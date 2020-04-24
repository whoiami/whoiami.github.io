---
layout: post
title: Rocksdb Code Analysis PUT--LOCK USAGE
---

![](/public/images/2020-04-15/rocksdb_cover.png)
<br>

### Previously on Rocksdb Put

在之前的博客文章Rocksdb Put 当中我们大致了解了Rocksdb 是如何调度线程写WAL和memtable的。最主要的合并写WAL的想法其实是从Leveldb继承而来。同时我们也注意到Rocksdb在其之上做了一些有意思的优化，下面我们就来了解一下Rocksdb 在上述Put流程中使用的一些解决线程间通信的一个技巧。如果合并写WAL的想法是让一个线程去写这一组线程的所有WAL，那么这个线程是如何通知其他线程写完WAL这个事情的呢？最直观的实现方式就是用条件变量CV，leveldb也正是这样实现的。然而基于一些有意思的发现，Rocksdb进行了进一步的优化，不得不说这是一个令人眼前一亮的优化。下面我来详细介绍。



<br>
### Introduction

根据Rocksdb作者自己的测试环境*4.0 SELinux test server with support for syscall auditing enabled* 平均一次 FUTEX_WAKE 切换到 FUTEX_WAIT的时间是10 微秒。如果每一次写入都损失这10微秒的时间显然是不够理想的。所以Rocksdb做了以下优化。其核心想法是自己先循环1微秒，看看Leader能不能在1微秒内把WAL写完，如果可以写完的话这次请求就少了10微秒的延迟。


如果不行的话也不会马上调用CV.wait，Rocksdb想要通过判断当前的OS是不是很忙，来确定这个Follower线程是不是需要再占用CPU资源一会，以至能够第一时间检测到Leader写完了WAL，可以马上继续处理客户端请求。Rocksdb 判断CPU是不是很忙的方式是调用std::this_thread::yield() 函数。

>  sched_yield() doesn't actually result in any
>  context switch overhead if there are no other runnable processes
>  on the current core, in which case it usually takes less than
>  a microsecond.

如果CPU 很闲，yield 函数甚至不会切换线程，并且在1微秒内返回。如果CPU不是很闲，那么yield返回的时间可能比我们预期的时间要长。这时候就需要调用CV.wait，毕竟CPU这么忙，我们也不好一只占用CPU资源不放。 



<br>
其这部分的代码如下：

```c++
uint8_t WriteThread::AwaitState(Writer* w, uint8_t goal_mask,
                                AdaptationContext* ctx) {
  // Busy loop using "pause" for 1 micro sec
  for (uint32_t tries = 0; tries < 200; ++tries) {
    state = w->state.load(std::memory_order_acquire);
    if ((state & goal_mask) != 0) {
      return state;
    }
    port::AsmVolatilePause();
  }
  
  // Update the yield_credit based on sample runs or right after a
  // hard failure
  bool update_ctx = false;
  // Should we reinforce the yield credit
  bool would_spin_again = false;
  if (enable_max_yield_usec) {
    // The samling base for updating the yeild credit.
    // The sampling rate would be  1/sampling_base.
    // const int sampling_base = 256;
    update_ctx = Random::GetTLSInstance()->OneIn(sampling_base);
    if (update_ctx || yield_credit.load(std::memory_order_relaxed) >= 0) {
      size_t slow_yield_count = 0;

      auto iter_begin = spin_begin;
      while ((iter_begin - spin_begin) <=
             std::chrono::microseconds(max_yield_usec_)) {
      
        std::this_thread::yield();

        state = w->state.load(std::memory_order_acquire);
        if ((state & goal_mask) != 0) {
          // success
          would_spin_again = true;
          break;
        }
        auto now = std::chrono::steady_clock::now();
        if (now == iter_begin ||
          now - iter_begin >= std::chrono::microseconds(slow_yield_usec_)){
          ++slow_yield_count;
          if (slow_yield_count >= kMaxSlowYieldsWhileSpinning) {
            update_ctx = true;
            break;
          }
        }
        iter_begin = now;
      }
    }
  }
  
  if ((state & goal_mask) == 0) {
    state = BlockingAwaitState(w, goal_mask);
  }

  if (update_ctx) {
    // update ctx->value by a factor
    v = v - (v / 1024) + (would_spin_again ? 1 : -1) * 131072;
    yield_credit.store(v, std::memory_order_relaxed);
  }
  return state;
}
```

<br>
1，Rocksdb调用了一个循环不停的检测状态state的变化，也就是检测WAL写没写完。这里可以看出这是类似一个自旋锁的实现。如果仔细看AsmVolatilePause 这个函数的调用我们会发现，可能就是受了自旋锁的实现的启发，这里也调用了asm volatile("pause") 这个函数，这个函数也是比较有意思的一个点，他的作用就是优化"自旋锁”的性能，具体来说是为了告诉CPU这是一个类似自旋锁的实现，请不要进行命令的重排。

摘抄自*intel-x86-and-64-manual*的一段解释：

> When executing a “spin-wait loop,” processors will suffer a severe performance penalty when exiting the loop because it detects a possible memory order violation. The PAUSE instruction provides a hint to the processor that the code sequence is a spin-wait loop. The processor uses this hint to avoid the memory order violation in most situations, which greatly improves processor performance. For this reason, it is recommended that a PAUSE instruction be placed in all spin-wait loops.


<br>

```c++
  // Busy loop using "pause" for 1 micro sec
  for (uint32_t tries = 0; tries < 200; ++tries) {
    state = w->state.load(std::memory_order_acquire);
    if ((state & goal_mask) != 0) {
      return state;
    }
    port::AsmVolatilePause();
  }
```

<br>


```c++

static inline void AsmVolatilePause() {
#if defined(__i386__) || defined(__x86_64__)
  asm volatile("pause");
#endif
}
```

<br>
2，全局有一个yield_credit 反应了CPU的繁忙程度，如果CPU真的特别忙，那这里就直接调用BlockingAwaitState 进行CV.wait()

```c++
uint8_t WriteThread::BlockingAwaitState(Writer* w, uint8_t goal_mask) {
  // condition wait
  w->StateCV().wait(guard, [w] {
    return w->state.load(std::memory_order_relaxed) != STATE_LOCKED_WAITING;
  });
}
```

<br>
3, 如果yield_credit 反应CPU不是很忙，Rocksdb的处理方式是进行yield，如果等待的时间超过write_thread_slow_yield_usec (default 3 us) 次数超过 kMaxSlowYieldsWhileSpinning (3) 的时候我们就应该放弃消耗CPU资源了，毕竟CPU忙起来了。退出yield循环后如果state 还没有改变，就调用BlockingAwaitState。

```c++
      while ((iter_begin - spin_begin) <=
             std::chrono::microseconds(max_yield_usec_)) {
        std::this_thread::yield();
        state = w->state.load(std::memory_order_acquire);
        if ((state & goal_mask) != 0) {
          // success
          would_spin_again = true;
          break;
        }
        auto now = std::chrono::steady_clock::now();
        if (now == iter_begin ||
          now - iter_begin >= std::chrono::microseconds(slow_yield_usec_)){
          ++slow_yield_count;
          if (slow_yield_count >= kMaxSlowYieldsWhileSpinning) {
            update_ctx = true;
            break;
          }
        }
        iter_begin = now;
      }
```


<br>

4，这里还时不时对CPU的使用情况进行采样，采样频率默认是256次调用采样一次。如果在上述yield的流程里面state改变了*((state & goal_mask) != 0）*，说明我们yield是有效果的，那么就增大yield_credit的值(would_spin_again = true)，如果yield没能等到Leader忙完，那么就减小yield_credit的值(defalut bool would_spin_again = false)，相应的会更多调用BlockingAwaitState。

```c++
v = v - (v / 1024) + (would_spin_again ? 1 : -1) * 131072;
```


<br>

### 总结：

Rocksdb这把锁的调用感觉是下了很大功夫，在CV.wait 之前的尝试最多花费10us (1us + 3us * 3 = 10us)的时间，而一次CV唤醒的平均时间也正好是10us，也就是说如果线程在这套机制内返回了，大概率会比单纯CV.wait要快，代价就是耗费了CPU资源。然而，如果CPU资源本身就很紧张，这种机制就又退化成了CV.wait的方式。


<br>

P.S. : 不知道有没有人注意过自旋1us这个操作，为什么是1us呢？Leader 写WAL 1us应该是一定写不完的。市面上的SSD写一次4k数据花费300us是比较正常的数据。但是写16k的内存差不多就是1us。那为什么写WAL算的是写内存的时间呢？这个跟WAL的默认实现有关，Rockdb默认实现利用了mmap实现了WAL的结构，mmap将内核的内存映射到了用户空间，减少了一次用户空间到内核空间上的拷贝，Rocksdb写入WAL实际上是写入内核空间内存，刷盘的时机交给了操作系统（不强制每次刷盘），即使Rocksdb进程down，也能保证所写数据能刷盘的功效，当然断电可能造成数据部分写入或丢失。所以1us 的时间就是写入WAL的时间，也是写入内存的时间。



<br>
### Reference：

[rocksdb/db/write_thread.cc](https://github.com/facebook/rocksdb/blob/v5.9.2/db/write_thread.cc#L56)

[Benefitting Power and Performance Sleep Loops](https://software.intel.com/en-us/articles/benefitting-power-and-performance-sleep-loops)

[Stackoverflow abotu pause op](https://stackoverflow.com/questions/12894078/what-is-the-purpose-of-the-pause-instruction-in-x86)

---
layout: post
title: Rocksdb Source Code Analysis PUT
---
![](/public/images/2020-04-15/rocksdb_cover.png)
<br>
### Introduction

<br>
Rocksdb 是基于Leveldb 实现的商业化数据库引擎。其最基本的想法就是基于LSMTree的特点，将随机写转化为一次顺序写加一次内存写的方式，极大提升了其写入的效率。这个结论是基于一个前提的，即目前的物理存储设备的随机写性能远小于顺序写性能，所以LSMTree以及Rocksdb就应运而生了。即然LSMTree的结构对于写请求有着天然的高性能，下面我们就来看一下Rocksdb对于写请求的处理又做了怎样的优化。



这里出现的代码基于5.9版本的Rocksdb。

Rocksdb写入的总体流程是先写入WAL 再写入memtable 然后返回客户端。其接口被封装在

```c++
// Default implementations of convenience methods that subclasses of DB
// can call if they wish
Status DB::Put(const WriteOptions& opt, ColumnFamilyHandle* column_family,
               const Slice& key, const Slice& value) {
  // Pre-allocate size of write batch conservatively.
  // 8 bytes are taken by header, 4 bytes for count, 1 byte for type,
  // and we allocate 11 extra bytes for key length, as well as value length.
  WriteBatch batch(key.size() + value.size() + 24);
  batch.Put(column_family, key, value);
  return Write(opt, &batch);
}
```

为了提高写入的效率，rocksdb提供batch写入，一次请求可以batch多条命令。

```c++
Status DBImpl::Write(const WriteOptions& write_options,
    WriteBatch* my_batch) {
  return WriteImpl(write_options, my_batch, nullptr, nullptr);
}
```

Put 接口和Write接口都共同掉用了DBImpl::WriteImpl 函数。


<br>
<br>

Rocksdb为了减少WAL的写入次数提高写入效率，进行了合并写的操作。首先将每个执行WriteImpl的线程，把自己加入到一个"此次需要写入的小组”的链表中，如果是第一个加入到这个链表，这个线程就是这组提交的Leader，Leader负责把这条链表上的数据整合一次写，写入WAL当中。其他线程会Wait在原地，等待Leader线程写完WAL之后的指令。如果是配置串行写memtable，那么Leader会写memtable，通知所有的其他人完成。如果配置可以并行写memetable，leader会通知其他正在Wait的线程开始写memtable。写完memtable的线程会等待最后一个写完的线程，等最后一个线程写完，所有人完成此次写操作。下面来看代码实现。

```c++
Status DBImpl::WriteImpl(const WriteOptions& write_options,
                         WriteBatch* my_batch, WriteCallback* callback,
                         uint64_t* log_used, uint64_t log_ref,
                         bool disable_memtable, uint64_t* seq_used) {
  WriteThread::Writer w(write_options, my_batch, callback, log_ref,
      disable_memtable);
  write_thread_.JoinBatchGroup(&w);
  if (w.state == WriteThread::STATE_PARALLEL_MEMTABLE_WRITER) {
    if (w.ShouldWriteToMemtable()) {
      w.status = WriteBatchInternal::InsertInto();
    }
    if (write_thread_.CompleteParallelMemTableWriter(&w)) {
      write_thread_.ExitAsBatchGroupFollower(&w);
    }
    assert(w.state == WriteThread::STATE_COMPLETED);
  }
  if (w.state == WriteThread::STATE_COMPLETED) {
    if (log_used != nullptr) {
      *log_used = w.log_used;
    }
    if (seq_used != nullptr) {
      *seq_used = w.sequence;
    }
    // write is complete and leader has updated sequence
    return w.FinalStatus();
  }
  assert(w.state == WriteThread::STATE_GROUP_LEADER);
  
  status = ConcurrentWriteToWAL(write_group, log_used, &last_sequence,
      seq_inc);
  // parallel write memtable
  write_thread_.LaunchParallelMemTableWriters(&write_group);
  w.status = WriteBatchInternal::InsertInto();
  // CompleteParallelWorker returns true if this thread should
  // handle exit, false means somebody else did
  should_exit_batch_group = write_thread_.CompleteParallelMemTableWriter(&w);
  if (should_exit_batch_group) {
    write_thread_.ExitAsBatchGroupLeader(write_group, status);
  }
  return status;
}
```

1,  将此线程上的Batch 请求 封装成WriteThread::Writer，其本身就是一个链表的成员。

```c++
struct Writer {
    WriteBatch* batch;
    std::atomic<uint8_t> state;  // write under StateMutex() or pre-link
    WriteGroup* write_group;
    Status status;            // status of memtable inserter
    Writer* link_older;  // read/write only before linking, or as leader
    Writer* link_newer;  // lazy,read/write only before linking, or as leader
}
```

2，调用JoinBatchGroup加入此次需要写入的小组链表中。如果是第一个加入链表则成为此次写入的Leader，通过LinkOne函数返回可知，如果自己不是Leader那么需要调用AwaitState在此等待其状态改变。

```c++
void WriteThread::JoinBatchGroup(Writer* w) {
  bool linked_as_leader = LinkOne(w, &newest_writer_);
  if (linked_as_leader) {
    SetState(w, STATE_GROUP_LEADER);
  }
  if (!linked_as_leader) {
    /**
     * Wait util:
     * 1) An existing leader pick us as the new leader when it finishes
     * 2) An existing leader pick us as its follewer and
     * 2.1) finishes the memtable writes on our behalf
     * 2.2) Or tell us to finish the memtable writes in pralallel
     * 3) (pipelined write) An existing leader pick us as its follower and
     *    finish book-keeping and WAL write for us, enqueue us as pending
     *    memtable writer, and
     * 3.1) we become memtable writer group leader, or
     * 3.2) an existing memtable writer group leader tell us to finish
     *      memtable writes in parallel.
     */
    AwaitState(w, STATE_GROUP_LEADER | STATE_MEMTABLE_WRITER_LEADER |
                      STATE_PARALLEL_MEMTABLE_WRITER | STATE_COMPLETED,
               &jbg_ctx);
  }
}
```

其中返回该线程是否为主的函数LinkOne比较有意思，该函数是用的无锁链表实现了链表里面的Add语意，同时由于第一个加入链表的线程*newest_writer == nullptr*所以有 *return (writers == nullptr)*的返回值。

```c++
// just point newest_write to w, add lock-free only leader can remove
// newest_writer is initialized to null, first thread invoke linkone who
// writers == nullptr is the leader
bool WriteThread::LinkOne(Writer* w, std::atomic<Writer*>* newest_writer) {
  assert(newest_writer != nullptr);
  assert(w->state == STATE_INIT);
  Writer* writers = newest_writer->load(std::memory_order_relaxed);
  while (true) {
    w->link_older = writers;
    if (newest_writer->compare_exchange_weak(writers, w)) {
      return (writers == nullptr);
    }
  }
}

```

3，如果是Leader的话需要继续执行ConcurrentWriteToWAL 函数，其主要作用是将所有线程的请求合并成一次写入，并调用WriteToWAL写入WAL中。

```c++
Status ConcurrentWriteToWAL(const WriteThread::WriteGroup& write_group,
                            uint64_t* log_used,
                            SequenceNumber* last_sequence,
                            size_t seq_inc) {
  WriteBatch* merged_batch = MergeBatch(write_group, &tmp_batch_,
                                      &write_with_wal, &to_be_cached_state);
  status = WriteToWAL(*merged_batch, log_writer, log_used, &log_size);
}

Status DBImpl::WriteToWAL(const WriteBatch& merged_batch,
                          log::Writer* log_writer, uint64_t* log_used,
                          uint64_t* log_size) {
  assert(log_size != nullptr);
  Slice log_entry = WriteBatchInternal::Contents(&merged_batch);
  *log_size = log_entry.size();
  Status status = log_writer->AddRecord(log_entry);
  return status;
}
```

4，如果是并发写入Memtable，调用LaunchParallelMemTableWriters。LaunchParallelMemTableWriters函数中，会将此次写入中的所有线程的状态置成STATE_PARALLEL_MEMTABLE_WRITER。同时自己调用WriteBatchInternal::InsertInto 本线程也写memtable。

```c++
void WriteThread::LaunchParallelMemTableWriters(WriteGroup* write_group) {
  assert(write_group != nullptr);
  write_group->running.store(write_group->size);
  for (auto w : *write_group) {
    SetState(w, STATE_PARALLEL_MEMTABLE_WRITER);
  }
}
```

5，Leader线程在写完memtable之后调用CompleteParallelMemTableWriter，如果不是最后一个写完memtable的线程，在此等待。

```c++
// This method is called by both the leader and parallel followers
bool WriteThread::CompleteParallelMemTableWriter(Writer* w) {
  if (write_group->running-- > 1) {
    // we're not the last one
    AwaitState(w, STATE_COMPLETED, &cpmtw_ctx);
    return false;
  }
  // else we're the last parallel worker and should perform exit duties.
  w->status = write_group->status;
  return true;
}
```

6，如果Leader是最后一个写完memtable的线程执行ExitAsBatchGroupLeader， 如果当前的链表上还有后续写入的Batch，直接选出来下一个主，并且将此次写入的小组成员状态全都置成STATE_COMPLETED。结束本次写入。

```c++
void WriteThread::ExitAsBatchGroupLeader(WriteGroup& write_group,
                                         Status status) {
  Writer* leader = write_group.leader;
  Writer* last_writer = write_group.last_writer;
  // if this link is added more writer the next writer is leader  
  SetState(last_writer->link_newer, STATE_GROUP_LEADER);
  while (last_writer != leader) {
     last_writer->status = status;
     auto next = last_writer->link_older;
     SetState(last_writer, STATE_COMPLETED);
     last_writer = next;
  }
}
```

7，回头看执行了步骤4后，其他线程由于自己的状态发生了改变（STATE_PARALLEL_MEMTABLE_WRITER），从JoinBatchGroup中醒来，跳回WriteImpl 函数执行。同样需要调用WriteBatchInternal::InsertInto写memtable,调用CompleteParallelMemTableWriter看看是否自己是最后一个完成的线程。如果不是，等待在CompleteParallelMemTableWriter函数里面。如果是，还需要调用WriteThread::ExitAsBatchGroupFollower。

```c++
void WriteThread::ExitAsBatchGroupFollower(Writer* w) {
  ExitAsBatchGroupLeader(*write_group, write_group->status);
  SetState(write_group->leader, STATE_COMPLETED);
}
```

<br>

### 总结

总体上Rocksdb的写入流程与Leveldb相似，都是用合并写的方式将多个线程上的写请求合并成一个Group，减少WAL的写入次数。不同与Leveldb的是，Rocksdb当中写入memtable是允许并发写的，在memtable的实现是skiplist的情况下，rocksdb 的选项*allow_concurrent_memtable_write* (default true)可以使线程并发的无锁插入skiplist当中，目前的memtable实现中也只有skiplist支持[Concurrent Insert](https://github.com/facebook/rocksdb/wiki/MemTable#comparison)操作。另外，AwaitState函数当中对于线程等待其他线程这件事rocksdb做了更详细的优化。可以期待一下之后的文章会对这两方面做更详细的介绍。


<br>
<br>

### Reference

[Rocksdb Source Code 5.9](https://github.com/facebook/rocksdb/tree/v5.9.2)

[https://github.com/facebook/rocksdb/wiki/](https://github.com/facebook/rocksdb/wiki/)

[https://zhuanlan.zhihu.com/p/33389807](https://zhuanlan.zhihu.com/p/33389807)

[https://cloud.tencent.com/developer/article/1143439](https://cloud.tencent.com/developer/article/1143439)


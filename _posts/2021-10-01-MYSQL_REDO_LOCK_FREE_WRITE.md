---
layout: post
title: MySQL Redo Lock Free Write
---


<img src="/public/images/2021-10-01/mysql_redo_lock_free_write_cover.png" width = "1200" height = "500" alt="cover" align=center />

<br>



### Basic Background


<br>



MySql8.0  对于mtr redo 的并发写入做了重构，通过一个atomic 原子变量对log_sys.buf空间进行reserve, 这样每个线程单独使用log_sys.buf  的一部分内存进行写入，因此也不需要对于log_sys.buf 进行加锁操作。log_sys.buf  是一个ring buffer，被预定的边界随着redo 的写盘不断推进，不断腾出新的位置给新写入的redo 进行预订。理论上只要redo 写盘足够快，每个写入redo 的线程只写入自己预定的一块内存，互不干预，从而实现高效的redo 写入。通常情况下并发写入mtr 的线程都是不需要等锁的。但是，如果线程mtr redo 写入log_sys.buf的速度过快，或者写盘异常或者写盘不够快，不能及时刷盘导致不能腾出更多的空间供新的mtr reserve，写mtr 的线程就会一直wait 在reserver 的阶段，等待释放更多的预定空间。

考虑到多线程写入log_sys.buf  内存，其中很可能会有空洞，这样的空洞会导致刷log_sys.buf 当中redo 的不连续。落盘的线程也就不知道哪一段redo 是连续的并且可以写盘。所以8.0使用了一个无锁内存变量recent_written，记录了log_sys.buf 当中已经写完的mtr 对应的起始跟结束的lsn 位置，通过收集各个mtr 写入状况可以得知，log_sys.buf当中哪一段redo 已经是连续的了，也就是这个位置之前的redo 都可以写盘了。

整体的设计思路就是用几块连续的内存换频繁的加锁开销，还是非常划算的。




<br>

### 数据结构


<br>



![](/public/images/2021-10-01/m_log_to_log_buf.png)


<br>
每个mtr结构中有一个m_log 的结构 ，负责存放这个mtr 对应的redo，在mtr commit 的时候需要先reserve，成功后将m_log 当中的redo 写到对应的log_sys.buf 当中。每个redo 文件当中放redo log 的最小单位是 log block。 在磁盘上的物理结构如下，一个Log BLock 是512个bytes，对于前12个bytes 用来存放一些控制信息，叫log block header，后4bytes 存放block的checksum叫做log block tailer。



![](https://raw.githubusercontent.com/jeremycole/innodb_diagrams/master/images/InnoDB_Log_Structures/Log%20Block.png)


<br>


recent_written是capacity 个lsn_t 组成的ring buffer。它可以给出目前已经连续写入log_sys.buf当中的redo 的最大值。

```c++
Link_buf<lsn_t> recent_written;
```



如下图所示，上半部分是log_sys.buf，下半部分是recent_written，对应的log_sys.buf 的一段内存的redo连续情况。当前recent_written 中的第一个slot 位置对应着一段redo 的起始位置lsn，记录的数值是当前mtr redo 的长度为5。那么将5写入第一个slot 当中。之后的log_sys.buf中有空洞。所以对应到recent_written当中是0表示没有写入，目前写入的redo 连续的位置应该是对应slot[0]+5 对应的redo lsn 的位置。

![](/public/images/2021-10-01/recent_write_before.png)

<br>
之后log_sys.buf 当中的空隙填充完毕，recent_written中slot[5] 的位置记录redo 长度为2。

![](/public/images/2021-10-01/recent_write_doing.png)

<br>
推进redo 的连续位置，到slot[12]对应的lsn。

![](/public/images/2021-10-01/recent_write_done.png)


<br>

### MTR commit



<br>
mtr_t  的mtr_buf_t m_log当中存放着mtr写入内存的redo。通过mtr_t::Command::execute() 的调用将mtr 内存的redo 按照Redo Log Block 的格式写入log_sys  的buf 当中，之后log_writer 线程会将log_sys 的buf 刷到盘上。execute 主要调用log_buffer_reserve， log_buffer_write，log_buffer_write_completed， log_wait_for_space_in_log_recent_closed， add_dirty_blocks_to_flush_list，log_buffer_close 这样几个函数。

```c++
void mtr_t::Command::execute() {
  auto handle = log_buffer_reserve(*log_sys, len);
  m_impl->m_log.for_each_block(write_log);
     |
    log_buffer_write(*log_sys, m_handle, block->begin(), block->used(), start_lsn);
    log_buffer_write_completed(*log_sys, m_handle, start_lsn, end_lsn);
  
  log_wait_for_space_in_log_recent_closed(*log_sys, handle.start_lsn);
  add_dirty_blocks_to_flush_list(handle.start_lsn, handle.end_lsn);
  log_buffer_close(*log_sys, handle);
}
```

<br>


log_buffer_reserve 通过log.sn的原子递增预留该线程会写入log_sys.buf 当中的位置。

```c++
Log_handle log_buffer_reserve(log_t &log, size_t len) {
  /* Reserve space in sequence of data bytes: */
  const sn_t start_sn = log_buffer_s_lock_enter_reserve(log, len);
  /* Translate sn to lsn (which includes also headers in redo blocks): */
  handle.start_lsn = log_translate_sn_to_lsn(start_sn);
  handle.end_lsn = log_translate_sn_to_lsn(end_sn);
  // 如果log buf 的空间写满了，会卡在这里
  if (unlikely(end_sn > log.buf_limit_sn.load())) {
    log_wait_for_space_after_reserving(log, handle);
  }
}

// log.sn 是一个atomic 变量。
static inline sn_t log_buffer_s_lock_enter_reserve(log_t &log, size_t len) {
  /* Reserve space in sequence of data bytes: */
  sn_t start_sn = log.sn.fetch_add(len);
}
```


<br>

log_buffer_write将m_log 的内容拷贝到log_sys.buf 当中。之后调用log_buffer_write_completed，如果recent_written 有空余的位置，将这次写入的lsn 插入到对应recent_written 当中。

```c++
lsn_t log_buffer_write(log_t &log, const Log_handle &handle, const byte *str,size_t str_len, lsn_t start_lsn) {
  // str 是所有的redo data不包括LOG_BLOCK_HDR_SIZE的头跟LOG_BLOCK_TRL_SIZE，copy到log_sys->buf里面之后需要带上LOG_BLOCK_HDR_SIZE的头跟LOG_BLOCK_TRL_SIZE信息。
  // 如果刚好写到log->buf 当中512对齐的最后一个blok，需要预创建LOG_BLOCK_HDR_SIZE的头跟LOG_BLOCK_TRL_SIZE。
}

void log_buffer_write_completed(log_t &log, const Log_handle &handle,
                                lsn_t start_lsn, lsn_t end_lsn) {
  while (!log.recent_written.has_space(start_lsn)) {
    os_event_set(log.writer_event);
    ++wait_loops;
    std::this_thread::sleep_for(std::chrono::microseconds(20));
  }
  log.recent_written.add_link_advance_tail(start_lsn, end_lsn);
}
```


<br>

将这次mtr redo 对应的位置写入到recent_closed 当中。

```c++
void log_wait_for_space_in_log_recent_closed(log_t &log, lsn_t lsn) {
  while (!log.recent_closed.has_space(lsn)) {
    ++wait_loops;
    std::this_thread::sleep_for(std::chrono::microseconds(20));
  }
}

void mtr_t::Command::add_dirty_blocks_to_flush_list 
  | 
  struct Add_dirty_blocks_to_flush_list 
  	|
    add_dirty_page_to_flush_list 
    	|	
    	buf_flush_note_modification
    		|	
    		buf_flush_insert_into_flush_list
 
void log_buffer_close(log_t &log, const Log_handle &handle) {
  log.recent_closed.add_link_advance_tail(start_lsn, end_lsn);   
}
```

<br>



### redo 写入相关线程：

<br>
 log_writer 线程主要控制redo 写到page cache中，log_flusher线程主要负责redo 的flush。

```c++
// 控制redo write
void log_writer(log_t *log_ptr)
  |
  /*log_sys—>buf里面有东西要写*/
  log_writer_write_buffer(log, ready_lsn);


// 控制redo flush
void log_flusher(log_t *log_ptr)
  |
  /* last_flush_lsn < log.write_lsn.load() */
  log_flush_low();/*update log.flushed_to_disk_lsn*/ 
  	|
  	fil_flush_file_redo(); 
```
<br>



### Reference

<br>
[https://github.com/mysql/mysql-server/tree/mysql-cluster-8.0.26](https://github.com/mysql/mysql-server/tree/mysql-cluster-8.0.26)

[http://mysql.taobao.org/monthly/2018/06/01/](http://mysql.taobao.org/monthly/2018/06/01/)

[MySQL 8.0: New Lock free, scalable WAL design](https://dev.mysql.com/blog-archive/mysql-8-0-new-lock-free-scalable-wal-design/)

[https://github.com/jeremycole/innodb_diagrams](https://github.com/jeremycole/innodb_diagrams/tree/master/images/InnoDB_Log_Structures)

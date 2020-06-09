---
layout: post
title: Rocksdb Code Analysis WAL Flush Strategy
---

![](/public/images/2020-04-15/rocksdb_cover.png)


<br>

Rocksdb 的写入流程分为写Write-Ahead Log（WAL）和写Memtable。不同的WAL刷盘策略对应了不同程度的容灾程度。提高容灾程度就意味着更频繁的刷盘，也就意味着会牺牲更多的写入性能。提高写入性能就意味着减少刷盘次数，也就意味着牺牲了容灾程度。这里介绍Rocksdb是如何平衡这个问题的。


<br>

### WAL Encapsulation 

```c++
// log::Writer(WritableFileWriter(WritableFile))
{
s = NewWritableFile(env_, log_fname, &lfile, opt_env_opt);
WritableFileWriter file_writer(
    new WritableFileWriter(lfile, log_fname, opt_env_opt, ...);
new_log = new log::Writer(file_writer, new_log_number, ...);
}

virtual Status NewWritableFile(const std::string& fname,
                               std::unique_ptr<WritableFile>* result,
                               const EnvOptions& options) override {
  // choose between PosixWritableFile and PosixMmapFile based on DBOptions
}

struct DBOptions {
  // Allow the OS to mmap file for writing.
  // Default: false
  bool allow_mmap_writes = false;
}
```

1，NewWritableFile 生成WritableFile ，主要负责Append，Flush 等最底层接口实现。Rocksdb的WritableFile 实现是由allow_mmap_writes 选项所决定的，默认实现是PosixWritableFile。另外一个可选的实现是PosixMmapFile。他们都继承于WritableFile 向上提供相同的基础接口。

2，将WritableFile作为参数，构建WritableFileWriter。WritableFileWriter主要作用是提供一个buffer，缓冲写入的数据，避免频繁的调用WritableFile的Append 操作和控制Sync 操作，是WritableFile之上的中间层。

3，将WritableFileWriter 作为参数，构建log::Writer。log::Writer 主要作用是将WAL 加上特定的header，并且封装成各个block。该概念从Leveldb即成而来，为保证在个别Block损坏的情况下，WAL文件可以跳过当损坏的Block继续解析出其他Block的数据。


<br>

### WAL Append Process

```c++
Status DBImpl::WriteToWAL(const WriteThread::WriteGroup& write_group,
                          log::Writer* log_writer, uint64_t* log_used,
                          bool need_log_sync, bool need_log_dir_sync,
                          SequenceNumber sequence) {
  Status status = log_writer->AddRecord(log_entry);
  if (status.ok() && need_log_sync) {
    for (auto& log : logs_) {
      status = log.writer->file()->Sync(immutable_db_options_.use_fsync);
      if (!status.ok()) {
        break;
      }
    }
  }
}
```



<br>
#### log::Writer

```c++
// log::Writer
Status Writer::AddRecord(const Slice& slice) {
  // Fragment the record if necessary and emit it.
  do {
    // choose RecordType: kFullType, kFirstType, kLastType, kMiddleType
    s = EmitPhysicalRecord(type, ptr, fragment_length);
  }
}

Status Writer::EmitPhysicalRecord(RecordType t, const char* ptr, size_t n) {
  // Write the header and the payload
  Status s = dest_->Append(Slice(buf, header_size));
  if (s.ok()) {
    s = dest_->Append(Slice(ptr, n));
    if (s.ok()) {
      s = dest_->Flush();
    }
  }
}
```

调用log::Writer的AddRecord 接口，内部调用WritableFileWriter->Append 和 WritableFileWriter->Flush 接口完成header 和payload 的写入。


<br>

#### WritableFileWriter

WritableFileWriter  内部控制buf， 缓存log::Writer Append的小数据，减少过多的writable_file->Append 调用。

```c++
Status WritableFileWriter::Append(const Slice& data) {
  const char* src = data.data();
  size_t left = data.size();
  if (buf_.Capacity() - buf_.CurrentSize() < left) {
    // See whether we need to enlarge the buffer to hold data in
  }
  // after enlarge still cant hold this data
  // means this data is huge, just prepare to bypass buffer
  // flush current buf data
  if (buf_.Capacity() - buf_.CurrentSize() < left) {
    Flush();
  }
  // small data write to current buffer
  if (buf_.Capacity() >= left) {
    buf_.Append(src, left);
  } else {
    // invoke writable_file_->Append
    s = WriteBuffered(src, left);
  }
}
```

1，尝试增大buf 容纳新写入的data。

2，如果buf可以容纳新写入的data，则写入到buf 中。

3，如果数据本身巨大，或者buf写满，则调用WriteBuffered->writable_file->Append。



<br>
#### PosixWritableFile

```c++
Status PosixWritableFile::Append(const Slice& data) {
  while(left!=0){
    write(fd_, src, left);
  }
}
```
PosixWritableFile 对于Append 的接口实现是直接调用系统调用write 写入数据。



<br>

### WAL Flush Strategy

<br>
#### 1，每次写入都刷盘。

DBImpl::WriteToWAL调用中在写入WAL之后，通过WriteOptions 配置sync == true 来实现每次写入WAL后都会对其进行刷盘，sync == true类似于每次系统调用write之后都进行fdatasync，这种场景下会极大拖慢写入效率，但是即使机器掉电也不会导致数据丢失。

```c++
Status DBImpl::WriteToWAL(const WriteThread::WriteGroup& write_group,
                          log::Writer* log_writer, uint64_t* log_used,
                          bool need_log_sync, bool need_log_dir_sync,
                          SequenceNumber sequence) {
  Status status = log_writer->AddRecord(log_entry);
  if (status.ok() && need_log_sync /* controled by WriteOptions.sync */) {
    for (auto& log : logs_) {
      status = log.writer->file()->Sync(immutable_db_options_.use_fsync);
      if (!status.ok()) {
        break;
      }
    }
  }
}

struct WriteOptions {
  // Default: false
  bool sync;
};
```

<br>
#### 2，完全交给操作系统刷盘。

配置sync==false，调用write 后就可以返回客户端写入完成，具体的刷盘时机由操作系统控制，这样在性能有所保证的基础上，机器掉电只会丢失一些最近写入的请求。对于更常见的进程crash 的场景，并不会造成数据丢失。这也是目前Rocksdb 默认的刷盘策略。



<br>
#### 3，用户可配刷盘策略。

如果刷盘完全交给操作系统，对于机器掉电数据的丢失程度是不可控的，所以Rocksdb 给出了一个用户可配置的刷盘参数。

```c++
struct DBOptions {
  // Applies to WAL files
  // Default: 0, turned off
  uint64_t wal_bytes_per_sync = 0;
}
```

WritableFileWriter 层面写入量超过WritableFileWriter 就会异步把page cache 刷入磁盘。具体实现如下。

```c++
Status WritableFileWriter::Flush() {
  // recent 1MB is not synced.
  const uint64_t kBytesNotSyncRange = 1024 * 1024;
  const uint64_t kBytesAlignWhenSync = 4 * 1024;    // Align 4KB.
  if (filesize_ > kBytesNotSyncRange) {
    uint64_t offset_sync_to = filesize_ - kBytesNotSyncRange;
    offset_sync_to -= offset_sync_to % kBytesAlignWhenSync;
    assert(offset_sync_to >= last_sync_size_);
    if (offset_sync_to > 0 &&
        offset_sync_to - last_sync_size_ >= bytes_per_sync_) {
      s = RangeSync(last_sync_size_, offset_sync_to - last_sync_size_);
      last_sync_size_ = offset_sync_to;
    }
  }
}

Status PosixWritableFile::Flush() { return Status::OK(); }
Status PosixWritableFile::RangeSync(uint64_t offset, uint64_t nbytes) {
  if (sync_file_range(fd_, static_cast<off_t>(offset),
      static_cast<off_t>(nbytes), SYNC_FILE_RANGE_WRITE) == 0) {
    return Status::OK();
  } else {
    return IOError;
  }
}

```

1，最近写入的1MB 不会Sync，原因稍后解释。

2，将filesize中超出1M的部分，取4K的整数段对齐。

3，调用系统调用sync_file_range 刷内存到磁盘上。这里使用了SYNC_FILE_RANGE_WRITE 参数，代表异步刷新，所以这个接口的调用结束并不能保证持久化完成。



至于为何保留1M 没有sync，是因为Xfs系统有neighbor page flushing 的机制，会刷新制定范围之外的page。试想没有这1M的Gap，page cache 分布如下：


> todo ragne sync part + cur_write_page(4k total 1k dirty)


如果neighbor page flushing机制也将cur_write_block 刷到磁盘，刷盘过程中由于有锁，后续写入该page的请求会因此而阻塞。


<br>

### Summary

Rocksdb 的默认刷盘策略是完全交给操作系统刷盘，这种策略可以应对绝大多数对于数据完整性不是非常苛刻的场景。对于机器掉电场景，数据损失大小完全取决于当时有多少数据没有刷入磁盘。对于其他的场景例如进程异常crash，在这种策略下是完全不会丢失数据的。另外，由于并不是每次写入都刷盘，这样写入的效率也可以得到保障。同时，由于操作系统行为的不可掌控，Rocksdb提供wal_bytes_per_sync 刷盘选项，用户可以此行选择刷盘的时机。总体来说，写入效率和数据完整性不可兼得。



<br>
### Reference

[Rocksdb Source Code 5.18.3](https://github.com/facebook/rocksdb/tree/v5.18.3)

[Write Ahead Log刷盘策略及实现](https://kernelmaker.github.io/Rocksdb_WAL)

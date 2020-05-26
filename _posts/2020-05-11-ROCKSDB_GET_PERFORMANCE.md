---
layout: post
title: Rocksdb Code Analysis Get Performance
---

![](/public/images/2020-04-15/rocksdb_cover.png)

<br>

### Previous On Rocksdb

之前的博客讨论了[DBImpl::Get接口实现](https://whoiami.github.io/ROCKSDB_GET)，Rocksdb Get 接口会依次在memtable， immutable memtable，和version 中查找当前key。如果每次都是内存查找命中例如在memtable 中命中，例如Put之后马上Get的场景，Get的性能当然非常好。但是考虑如果每次都查盘的场景，例如在rocksdb中读到很久之前的数据，这样的场景下rockdb的性能如何呢？下面让我们通过实际测试结合代码进行分析。



<br>
### Prepare

1，Physical Machine：40 core cpu，3T nvme ssd（4k **fio** 测试平均性能延迟100us左右）。

2，实验数据量10亿kv。kv大200bytes左右。由于有snappy压缩，最终落盘数据量220G左右。按照rocksdb数据分配的策略level1默认大小**max_bytes_for_level_base** 256MB，默认**max_bytes_for_level_multiplier** 10，这样level2最大容量2.56G，level3最大容量 25.6G，level4最大容量 256G。这样整个220G数据最终分布在level0-level4当中。

4， 关闭filter block和 index block cache。使其读盘获得filter block 和index block。

3，为了让Rocksdb读盘，我们需要在Get 实验之前将page cache清除。


<br>

### Result Analysis

期间我们发现多线程调用Get接口的性能大概在**1.4Wqps**左右，此时40个CPU使用率已经超过90%。这显然不符合我们对于Rocksdb这种高速kv引擎的期望。近一步我们通过pstack看看Rocksdb在做什么。

观察pstack 结果如下：

```c++
#2  GetLogicalBufferSize at env/io_posix.cc:70
#3  PosixRandomAccessFile::PosixRandomAccessFile at env/io_posix.cc:303
#4  PosixEnv::NewRandomAccessFile at env/env_posix.cc:252
#5  TableCache::GetTableReader at db/table_cache.cc:97
#6  TableCache::FindTable at db/table_cache.cc:150
#7  TableCache::Get at db/table_cache.cc:376
#8  Version::Get at db/version_set.cc:1002
#9  DBImpl::GetImpl at db/db_impl.cc:1024
#10 DBImpl::Get at db/db_impl.cc:941
#11 DB::Get at ./include/rocksdb/db.h:317

```

可以看出 **#5** 中TableCache::FindTable中由于待查找sst对应的TableReader没有被创建，需要调用TableCache::GetTableReader创建TableReader，并缓存到table_cache中。所以，怀疑是缓存TableReader对应的参数**max_open_files** 配置不够高，大多数请求需要重新创建TableReader。

再次测试时，配置**max_open_files** 为-1，缓存所有创建过的TableReader后，Get QPS 达到了**30W**左右，disk iops 也是30W左右。由于测试的是Rocksdb每次请求都读盘的场景，所以结果还比较符合预期。



<br>
### Code Analysis

然而，创建TableReader 过程到底发生了什么，导致这么消耗资源呢？由于有大量的CPU使用，所以怀疑是有大量的系统调用造成的。下面结合代码具体分析TableReader 的创建过程，重点关注其中涉及到的耗时操作。

创建TableReader主要系统调用代码如下：

```c++
Status TableCache::GetTableReader() {
  // invoke GetLogicalBufferSize
  Status s = ioptions_.env->NewRandomAccessFile(fname, &file, env_options);
  file->Hint(RandomAccessFile::RANDOM);
  std::unique_ptr<RandomAccessFileReader> file_reader(
        new RandomAccessFileReader(std::move(file), fname,...));
  s = ioptions_.table_factory->NewTableReader(file_reader);
}
```

1，创建NewRandomAccessFile。

2，调用Hint修改文件POSIX_FADV_RANDOM属性（"The specified data will be accessed in random order" from [posix_fadvise man page](https://linux.die.net/man/2/posix_fadvise)）。

3，用NewRandomAccessFile的结果创建RandomAccessFileReader。

4，用RandomAccessFileReader的结果创建TableReader。

```c++
virtual Status NewRandomAccessFile(const std::string& fname,
                                   unique_ptr<RandomAccessFile>* result,
                                   const EnvOptions& options) override {
  fd = open(fname.c_str(), flags, 0644);
  SetFD_CLOEXEC(fd, &options);
  //  stat(fname.c_str(), &sbuf)
  s = GetFileSize(fname, &size);
  if (options.use_mmap_reads) {
    void* base = mmap(nullptr, size, PROT_READ, MAP_SHARED, fd, 0);
    result->reset(new PosixMmapReadableFile(fd, fname, base,
                                            size, options));
  } else {
    result->reset(new PosixRandomAccessFile(fname, fd, options));
  }
}
```

1，调用open，SetFD_CLOEXEC，GetFileSize(即stat()) 等系统调用。

2，创建PosixRandomAccessFile。

3，调用BlockBasedTableFactory::NewTableReader创建TableReader。

```c++
PosixRandomAccessFile::PosixRandomAccessFile(const std::string& fname,
                                             int fd,
                                             const EnvOptions& options)
    : filename_(fname),
      fd_(fd),
      use_direct_io_(options.use_direct_reads),
      logical_sector_size_(GetLogicalBufferSize(fd_)) {
}

// get device page cache
size_t GetLogicalBufferSize(int __attribute__((__unused__)) fd) {
  int result = fstat(fd, &buf);
  // get device dir
  if (realpath(path, real_path) == nullptr) {
    return kDefaultPageSize;
  }
  std::string fname = device_dir + "/queue/logical_block_size";
  fp = fopen(fname.c_str(), "r");
  getline(&line, &len, fp);
  return size;
}
```

GetLogicalBufferSize 函数为了查找device使用的默认page cache大小，需要从一个设备文件目录读出来。期间调用realpath 利用lstat反复调用文件目录信息，获得文件完整路径（**非常耗时** 具体见strace结果）。之后调用fopen获得logical buffer cache。

```c++
BlockBasedTableFactory::NewTableReader {
  return BlockBasedTable::Open();
}

Status BlockBasedTable::Open() {
  s = prefetch_buffer->Prefetch(file.get(), prefetch_off, prefetch_len);
  // 53 bytes footer
  s = ReadFooterFromFile(file.get(), ..., file_size, &footer,
                         kBlockBasedTableMagicNumber);
}
```

最后，调用BlockBasedTableFactory::NewTableReader 进而调用 BlockBasedTable::Open，其中也会有一些系统调用，其中最主要的是ReadFooterFromFile会调用open读取 53bytes的footer。 



以上就是创建TableReader的主要流程以及其需要的系统调用。



<br>
### Strace SystemCall

为验证其正确性，打印了进程的strace 消息。其中打印了每个系统调用的调用时间。例如

```c++
4850  17:28:10.168420 futex(0x12cc9b0, FUTEX_WAKE_PRIVATE, 1 <unfinished ...>
4850  17:28:10.168520 <... futex resumed> ) = 1 <0.000096>
```

为线程4850 调用futex 所用时间0.000096s（96us）


具体的strace数据在[strace raw result](public/test_data/get_performance.txt)。

<br>

### Conclusion

结合上述strace结果分析，创建TableReader 花费1.2ms时间，没有filter block和 index block cache的情况下，Get 实际读取meta block 读取 data block 花费 0.3～0.5ms。从这里可以看出，TableReader创建是一个非常费CPU的事情，如果**max_open_files**参数设置过小，会导致在LRU cache中不停的刷出旧的TableReader，并且创建新的TableReader。另外，上述的strace结果能看出的有意思的东西非常多，有兴趣的同学可以自行研究，这里就不一一解释了。


<br>

### Reference

[Rocksdb Source Code 5.9](https://github.com/facebook/rocksdb/tree/v5.9.2)

[ROCKSDB_GET](https://whoiami.github.io/ROCKSDB_GET)

[posix_fadvise man page](http://man7.org/linux/man-pages/man2/fadvise64.2.html)

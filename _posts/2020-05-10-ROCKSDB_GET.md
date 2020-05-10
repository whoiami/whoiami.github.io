---
layout: post
title: Rocksdb Code Analysis Get
---

![](/public/images/2020-04-15/rocksdb_cover.png)

<br>
### Previous on Rocksdb

之前的博客中分享了[Rocksdb Put流程](https://whoiami.github.io/ROCKSDB_PUT)，实际上对于LSM-tree这种模型，其优点就是将随机写转换成为顺序写加内存写，极大的提升了写入的效率。与此同时，也带来了读效率的一系列问题。其中读放大 一直是Rocksdb最大的敌人。通过这里的介绍可以大致了解Rocksdb读流程，以便于后续更深入的研究。



<br>
### Introduction

```c++

Status DBImpl::Get(const ReadOptions& read_options,
                   ColumnFamilyHandle* column_family, const Slice& key,
                   PinnableSlice* value) {
  return GetImpl(read_options, column_family, key, value);
}

```

Rocksdb 的Get接口实现于DBImpl::Get其实现主要靠DBImpl::GetImpl 函数调用。

```c++
Status DBImpl::GetImpl(const ReadOptions& read_options,
                       ColumnFamilyHandle* column_family, const Slice& key,
                       PinnableSlice* pinnable_val, bool* value_found,
                       ...) {
  auto cfh = reinterpret_cast<ColumnFamilyHandleImpl*>(column_family);
  auto cfd = cfh->cfd();
  // Acquire SuperVersion
  SuperVersion* sv = GetAndRefSuperVersion(cfd);
  if (read_options.snapshot != nullptr) {
    snapshot = reinterpret_cast<const SnapshotImpl*>(
        read_options.snapshot)->number_;
  } else {
    // Since we get and reference the super version before getting
    // the snapshot number, without a mutex protection, it is possible
    // that a memtable switch happened in the middle and not all the
    // data for this snapshot is available. But it will contain all
    // the data available in the super version we have, which is also
    // a valid snapshot to read from.
    // We shouldn't get snapshot before finding and referencing the
    // super versipon because a flush happening in between may compact
    // away data for the snapshot, but the snapshot is earlier than the
    // data overwriting it, so users may see wrong results.
    snapshot = allocate_seq_only_for_data_ ? versions_->LastSequence()
                                  : versions_->LastAllocatedSequence();
  }
  if (sv->mem->Get(lkey, pinnable_val->GetSelf(), &s, &merge_context,
                   &range_del_agg, read_options, callback, is_blob_index)) {
      done = true;
  } else if (sv->imm->Get(lkey, pinnable_val->GetSelf(), &s, &merge_context,
                          &range_del_agg, read_options, callback,
                          is_blob_index)) {
      done = true;
  }
  if (!done) {
    sv->current->Get(read_options, lkey, pinnable_val, &s, &merge_context,
                     &range_del_agg, value_found, nullptr, nullptr, callback,
                     is_blob_index);
  }
  return s；
}
```

1，lock-free获取SuperVersion的消息

2，如果用户未指定snapshot，需要获取当前的snapshot

​	由于取SuperVersion是lock-free的，有可能当前的snapshot 已经不是全局最新的snapshot，但并不妨碍返回这个这个SuperVersion当中的最新数据。

```c++
struct SuperVersion {
  // Accessing members of this class is not thread-safe and requires external
  // synchronization (ie db mutex held or on write thread).
  MemTable* mem;
  MemTableListVersion* imm;
  Version* current;
}
```

3，SuperVersion 中按照数据的新旧程度排序MemTable >  MemTableListVersion > Version， 依次按序查找，如果在新的数据中找到符合snapshot规则的结果，就可以立即返回，完成本次查找。

下面分别介绍SuperVersion 各个数据结构中查找的流程.



<br>
### MemTable::Get

```c++
bool MemTable::Get(const LookupKey& key, std::string* value, Status* s,
                   ..., SequenceNumber* seq,
                   const ReadOptions& read_opts...) {
  prefix_bloom_->MayContain(prefix_extractor_->Transform(user_key);
  return false;
  Saver saver;
  saver.status = s;
  saver.key = &key;
  saver.value = value;
  saver.mem = this;
  ....
  table_->Get(key, &saver, SaveValue);
}
                            
class SkipListRep : public MemTableRep {
  virtual void Get(const LookupKey& k, void* callback_args,
                   bool (*callback_func)(void* arg,
                                         const char* entry)) override {
    SkipListRep::Iterator iter(&skip_list_);
    Slice dummy_slice;
    for (iter.Seek(dummy_slice, k.memtable_key().data());
         iter.Valid() && callback_func(callback_args, iter.key());
         iter.Next()) {
    }
  }
}
                            
static bool SaveValue(void* arg, const char* entry) {
  Saver* s = reinterpret_cast<Saver*>(arg);
  // Check that it belongs to same user key.  We do not check the
  // sequence number since the Seek() call above should have skipped
  // all entries with overly large sequence numbers.
  if (s->mem->GetInternalKeyComparator().user_comparator()->
      Equal(entry->key, s->key)) {
    // If the value is not less or equal thansnapshot, skip it,
    // continue for loop in Get
    if (IsCommitted){
      return true;
    }
    switch (entry->type) {
      // intentional fallthrough
      case kTypeValue: {
        *(s->status) = Status::OK();
        s->value->assign(v.data(), v.size());
        *(s->found_final_value) = true;
        // stop loop in Get
        return false;
      }
      case kTypeDeletion: {
        *(s->status) = Status::NotFound();
        *(s->found_final_value) = true;
        return false;
      }
    }
  }
}
```

1, 利用MemTableRep的Get 函数进行查找

2，以SkipListRep 实现为例，在skiplist中进行seek查找，从seek到的位置开始向后遍历，遍历entry是否符合SaveValue 定义的规则。skiplist 的seek实现可以参考[ROCKSDB_MEMTABLE](https://whoiami.github.io/ROCKSDB_MEMTABLE)。

​	SaveValue 查看当前skiplist entry是否还是当前查找的key如果不是则返回。

​	查看当前entry 的snapshot是否小于或等于需要查找的snapshot，不符合则继续调用SkipListRep 的loop。

​	如果entry 的snapshot符合上述条件，那么则跳出SkipListRep 的loop，返回查找结果。



<br>
### MemTableListVersion::Get

```c++
// Search all the memtables starting from the most recent one.
// Return the most recent value found, if any.
// Operands stores the list of merge operations to apply, so far.
bool MemTableListVersion::Get(const LookupKey& key, std::string* value,
                              Status* s, SequenceNumber* seq,
                              const ReadOptions& read_opts, ...) {
  return GetFromList(&memlist_, key, value, s, seq, read_opts, ...);
}

```

MemTableListVersion 用链表的形式保存了所有immutable memtable的结构，查找时，按时间序依次查找于每一个memtable，如果任何一个 memtable 查找到结果则立即返回，即返回最新的返回值。具体 memtable 查找见上述MemTable::Get接口。



<br>
### Version::Get

```c++
void Version::Get(const ReadOptions& read_options, const LookupKey& k,
                  PinnableSlice* value, Status* status,
                  ..., bool* value_found,
                  bool* key_exists, SequenceNumber* seq,...) {
  FdWithKeyRange* f = fp.GetNextFile();
  while (f != nullptr) {
    *status = table_cache_->Get(read_options, f->fd, ikey, ...);
    // if find or not found return
    f = fp.GetNextFile();
  }
}
```

1，Version::Get 从level0层开始，通过GetNextFile调用逐层查找可能包含LookupKey的sst文件。

2，调用TableCache::Get 遍历单个sst文件，如果查找到结果立即返回。

```c++
Status TableCache::Get(const ReadOptions& options,
                       const FileDescriptor& fd, const Slice& k, ...) {
  s = FindTable(env_options_, internal_comparator, fd, &handle, ...);
  if (s.ok()){
    s = t->Get(options, k, ...);
  }
}
```

1，获取table 的指针。

2，调用talbe内部的Get 接口，读取bloom fileter block和index block 最终定位到kv 所在位置。

```c++
Status TableCache::FindTable(
    const EnvOptions& env_options,
    const InternalKeyComparator& internal_comparator,
    const FileDescriptor& fd,
    Cache::Handle** handle, ...) {
  *handle = cache_->Lookup(key);
  if (*handle == nullptr) {
    // env->NewRandomAccessFile(fname, &file, env_options)
    // (if posix env : PosixEnv::NewRandomAccessFile)
    s = GetTableReader(env_options, internal_comparator, fd,
                       false, ...);
    if (s.ok()) {
      s = cache_->Insert(key, table_reader.get(), 1,
                         &DeleteEntry<TableReader>, handle);
    }
  }
}
```

TableCache::FindTable 函数中对于cache miss的table会新建sst的文件指针，将其缓存在cache中。


<br>

### 总结

至此，大致了解了Rocksdb 的Get 流程，对于Rocksdb使用者来说 Get流程是更值得关注的一部分，因为对于Put流程，Rocksdb 有着天然的优势，往往遇到瓶颈的正是其读请求。

Rocksdb 的Get 请求会不可避免的带来读放大（Read Amplification)。Rocksdb 的读操作需要从新到旧（从上到下）一层一层查找，直到找到想要的数据。这个过程可能需要不止一次 I/O。从level0 开始，由于level0上面的数据有可能会相互重叠，所以需要每个文件都进行查找，潜在的可能会造成I/O，对于level1及以下的层级，最差情况每一层都定位到一个sst 文件，进行一次潜在I/O。这样会造成大量的读放大。所幸，leveldb 和rocksb引进了bloom filter减少了可能的读盘次数。总而言之，读放大在一些极端场景，例如延迟性能要求很高的场景下，是非常致命的，会严重影响上层应用使用。之后的博客会做更详细的分析，并给出可能的优化方案。



<br>
### Reference

[Rocksdb Source Code 5.9](https://github.com/facebook/rocksdb/tree/v5.9.2)

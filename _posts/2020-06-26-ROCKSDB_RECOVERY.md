---
layout: post
title: Rocksdb Code Analysis Recovery
---

![](/public/images/2020-04-15/rocksdb_cover.png)

### Introduction

Rocksdb的写入流程是写入WAL和memtable，memtable 逐渐将内存的数据转换成sst。如果在转换成sst的过程中断电，对于已经返回客户端成功写入的数据，Rocksdb必须提供一定的恢复机制，能够保证在重启的时候将所有数据最终能够成功转换成sst文件，并且这些sst数据能够被后续的请求成功访问。为此，Rocksdb 在重启的时候提供了Recover的功能，其目的就是为了恢复上一次运行时候的状态，包括内存状态和磁盘数据状态。



<br>
### Version Control

通过CURRENT文件可以追溯到当前db正在使用的MANIFEST文件，Rocksdb利用MANIFEST文件记录某每一次对db的持久化操作，记录每一次操作的数据结构称之为version-edit，恢复过程所做的工作之一就是对version-edit的replay，最终在内存中生成version结构，version是对db的完整的描述。

根据[Rocksdb Wiki](https://github.com/facebook/rocksdb/wiki/MANIFEST)所示，CURRENT文件和MANIFEST文件的关系如下所示：

```
CURRENT = File pointer to the latest manifest log
MANIFEST-<seq no> = Contains snapshot of RocksDB state and subsequent
                    modifications
```

version，version-edit 和 manifest 的关系如下所示：

```
version-edit      = Any RocksDB state change
version           = { version-edit* }
manifest-log-file = { version, version-edit* }
                  = { version-edit* }
```


<br>

#### VersionSet

 ```c++
class VersionSet {
  Status Recover(const std::vector<ColumnFamilyDescriptor>& column_families,
                 bool read_only = false);
  Status VersionSet::LogAndApply(
    ColumnFamilyData* column_family_data,
    const autovector<VersionEdit*>& edit_list, ...);
  std::unique_ptr<ColumnFamilySet> column_family_set_;
  unique_ptr<log::Writer> descriptor_log_;
  std::deque<ManifestWriter*> manifest_writers_;
}

class ColumnFamilySet {
  std::unordered_map<uint32_t, ColumnFamilyData*> column_family_data_;
  ColumnFamilyData* dummy_cfd_;
}

class ColumnFamilyData {
  // pointers for a circular linked list.
  ColumnFamilyData* next_;
  ColumnFamilyData* prev_;
  std::unique_ptr<TableCache> table_cache_;  
  Version* current_;
  MemTable* mem_;
  MemTableList imm_;
  SuperVersion* super_version_;
}
 ```

Rocksdb 内部用VersionSet 管理所有的MANIFEST的更新和所有ColumnFamily的恢复。分别对应其内部的LogAndApply 和Recover 方法。

此外，VersionSet包含成员变量ColumnFamilySet，ColumnFamilySet包含了db的所有ColumnFamily 的信息。其内部包含了所有ColumnFamilyData一个集合，同时所有的ColumnFamilyData 是以链表的形式组织起来的，链表的表头是dummy_cfd。

ColumnFamilyData 记录了单个CF的所有数据，其中的next 和 prev指针是为了维护其链表的结构，table_cache缓存了打开了的文件句柄信息，current 指向当前CF的version，mem和imm是memtable数据。SuperVersion 是当前CF的全部数据信息，包括了memtable 和当前的version，是读写期间主要适用的数据结构。



<br>
#### Version

```c++
class Version {
  void Get(const ReadOptions&, const LookupKey& key,
           Status* status, MergeContext* merge_context, ...); 
  VersionStorageInfo storage_info_;
  Version* next_;               // Next version in linked list
  Version* prev_;               // Previous version in linked list
}

class VersionStorageInfo {
  int NumLevelFiles(int level) const {
    return static_cast<int>(files_[level].size());
  }
  const std::vector<FileMetaData*>& LevelFiles(int level) const {
    return files_[level];
  }
  // List of files per level, files in each level are arranged
  // in increasing order of keys
  std::vector<FileMetaData*>* files_;
}

struct FileMetaData {
  FileDescriptor fd;
  InternalKey smallest;            // Smallest internal key served by table
  InternalKey largest;             // Largest internal key served by table
  Cache::Handle* table_reader_handle;
  uint64_t num_entries;            // the number of entries.
  uint64_t num_deletions;          // the number of deletion entries.
  uint64_t raw_key_size;           // total uncompressed key size.
  uint64_t raw_value_size;         // total uncompressed value size.
  bool being_compacted;
  bool marked_for_compaction;      // True if client asked us nicely 
                                   // to compact this file.
}
```

Version存储了所有sst的信息，并且提供Get方法，Rocksdb的读写流程将会直接调用这个接口。其内部也是用链表组织的维护了next 和prev 两个指针。其内部最主要的数据结构是VersionStorageInfo。

VersionStorageInfo 维护一个二维数组files，保存了每一level上的所有sst文件信息，描述文件信息的数据结构称之为FileMetaData。

FileMetaData包括文件描述符，文件存储的最大最小key等等信息。



<br>
#### VersionBuilder

```c++
class VersionEdit {
  uint64_t log_number_;
  uint64_t prev_log_number_;
  uint64_t next_file_number_;
  DeletedFileSet deleted_files_;
  std::vector<std::pair<int, FileMetaData>> new_files_;
  uint32_t column_family_;
  bool is_column_family_drop_;
  bool is_column_family_add_;
  std::string column_family_name_;
  ...;
}

// A helper class so we can efficiently apply a whole sequence
// of edits to a particular state without creating intermediate
// Versions that contain full copies of the intermediate state.
class VersionBuilder {
  // apply VersionEdit into builder
  void Apply(VersionEdit* edit);
  // save builder info to VersionStorageInfo
  void SaveTo(VersionStorageInfo* vstorage);
};
```

Version的创建主要使用VersionEdit 和VersionBuilder 两个结构。

VersionEdit 描述了db 的任何一次持久化变动，包括ColumnFamily 的添加删除，sst文件的添加删除等等。

VersionBuilder 是用VersionEdit 生成Version的工具类。主要包括Apply方法，应用VersionEdit的改动到VersionBuilder。以及SaveTo 方法把所有的db改动输出到version类的VersionStorageInfo结构中。



<br>


### Recover

```c++
Status DBImpl::Open(...) {
  ...;
  DBImpl* impl = new DBImpl(db_options, dbname, seq_per_batch);
  s = impl->Recover(column_families);
  ...;
}

Status DBImpl::Recover(...) {
  // sanity check
  ...;
  Status s = versions_->Recover(column_families, read_only);
  // Recover from all newer log files than the ones named in the
  // descriptor (new log files may have been added by the previous
  // incarnation without registering them in the descriptor).
  s = env_->GetChildren(immutable_db_options_.wal_dir, &filenames);
  std::vector<uint64_t> logs;
  for (size_t i = 0; i < filenames.size(); i++) {
    uint64_t number;
    FileType type;
    if (ParseFileName(filenames[i], &number, &type) && type == kLogFile) {
      logs.push_back(number);
    }
  }
  if (!logs.empty()) {
    // Recover in the order in which the logs were generated
    std::sort(logs.begin(), logs.end());
    s = RecoverLogFiles(logs, &next_sequence, read_only);
    if (!s.ok()) {
      // Clear memtables if recovery failed
      for (auto cfd : *versions_->GetColumnFamilySet()) {
        cfd->CreateNewMemtable(*cfd->GetLatestMutableCFOptions(),
                               kMaxSequenceNumber);
      }
    }
  }
}
```

Rocksdb 的Recover流程是在DBImpl::Open期间调用的，其主要流程包括两个部分。

1，调用versions->Recover，从MANIFEST文件中读取VersionEdit，恢复VersionSet。

2，调用RecoverLogFiles，恢复LOG文件中没有记录到MANIFEST的部分。


<br>

#### VersionSet::Recover

```c++
Status VersionSet::Recover(
  const std::vector<ColumnFamilyDescriptor>& column_families) {
  // loading CURRENT and MANIFEST
  ...;
  std::unordered_map<uint32_t, BaseReferencedVersionBuilder*> builders;
  builders.insert({0, new BaseReferencedVersionBuilder(default_cfd)});
  log::Reader reader(NULL, std::move(manifest_file_reader), ...);
  while (reader.ReadRecord(&record, &scratch) && s.ok()) {
    VersionEdit edit;
    s = edit.DecodeFrom(record);
    ColumnFamilyData* cfd = nullptr;
    if (edit.is_column_family_add_) {
      // add cf
    } else if (edit.is_column_family_drop_) {
      // drop cf
    } else if (!cf_in_not_found) {
      builder->second->version_builder()->Apply(&edit);
    }
  }
  for (auto cfd : *column_family_set_) {
    if (cfd->IsDropped()) {
      continue;
    }
    auto builders_iter = builders.find(cfd->GetID());
    auto* builder = builders_iter->second->version_builder();
    Version* v =
        new Version(cfd, this, env_options_, current_version_number_++);
    builder->SaveTo(v->storage_info());
    // Install recovered version
    v->PrepareApply(*cfd->GetLatestMutableCFOptions(),
        !(db_options_->skip_stats_update_on_db_open));
    AppendVersion(cfd, v);
  }
}
```

1，创建builders用于生成每一个CF的Version类。

2，从MANIFEST读取所有VersionEdit进行逐个处理。如果是对于CF的操作，则生成或者删除对应的CF，如果不是对于CF的操作，则调用VersionBuilder::Apply 方法应用到VersionBuilder内部。

3，查看当前所有的column_family_set，对应生成新的Version结构。

3.1，调用VersionBuilder::SaveTo生成Version的storage_info。

3.2，调用Version::PrepareApply对Version进程初始化。

3.2，调用AppendVersion，把这个Version挂载到对应的CF内部。

```c++
void VersionSet::AppendVersion(ColumnFamilyData* column_family_data,
                               Version* v) {
  // compute new compaction score
  ...;
  // Make "v" current
  Version* current = column_family_data->current();
  column_family_data->SetCurrent(v);
  v->Ref();
  // Append to linked list
  v->prev_ = column_family_data->dummy_versions()->prev_;
  v->next_ = column_family_data->dummy_versions();
  v->prev_->next_ = v;
  v->next_->prev_ = v;
}
```

AppendVersion 过程中，将Version标记为ColumnFamilyData的当前使用过的Version，并且挂载到ColumnFamilyData维护的version链表中。


<br>

#### DBImpl::RecoverLogFiles

生成VersionSet后，就可以算出有哪些LOG文件需要恢复。之后调用RecoverLogFiles进行LOG恢复。

```c++
Status DBImpl::RecoverLogFiles(
  const std::vector<uint64_t>& log_numbers, ...) {
  std::unordered_map<int, VersionEdit> version_edits;
  for (auto cfd : *versions_->GetColumnFamilySet()) {
    VersionEdit edit;
    edit.SetColumnFamily(cfd->GetID());
    version_edits.insert({cfd->GetID(), edit});
  }
  for (auto log_number : log_numbers) {
    while (reader.ReadRecord(&record, &scratch, ...)) {
      status = WriteBatchInternal::InsertInto(
        &batch, column_family_memtables_.get(),
        &flush_scheduler_, ...);
      while ((cfd = flush_scheduler_.TakeNextColumnFamily()) != nullptr) {
        auto iter = version_edits.find(cfd->GetID());
        VersionEdit* edit = &iter->second;
        status = WriteLevel0TableForRecovery(job_id, cfd, cfd->mem(), edit);
        cfd->CreateNewMemtable(...);
      }
    }
  }
  for (auto cfd : *versions_->GetColumnFamilySet()) {
    auto iter = version_edits.find(cfd->GetID());
    VersionEdit* edit = &iter->second;
    // flush the final memtable (if non-empty)
    if (cfd->mem()->GetFirstSequenceNumber() != 0) {
      // WriteLevel0TableForRecovery && CreateNewMemtable
    }
    WriteLevel0TableForRecovery
    status = versions_->LogAndApply(
        cfd, *cfd->GetLatestMutableCFOptions(), edit, &mutex_);
  }
  return status;
}
```

1，为每一个CF分配VersionEdit，恢复过程中任何的需要持久化操作会通过VersionEdit 的方式记录，最终记录到MANIFEST中。

2，遍历需要恢复的LOG文件，依次处理从LOG文件中读取的record。

2.1，通过InsertInto接口写入memtable。

2.2，如果memtable需要刷盘，调用WriteLevel0TableForRecovery进行刷盘，之后调用CreateNewMemtable，生成一个新的memtable。

3，如果memtalbe中还有内容，再进行刷盘操作。

4，调用VersionSet::LogAndApply 将VersionEdit记录的改动持久化到MANIFEST文件中。



```c++
Status DBImpl::WriteLevel0TableForRecovery(
    int job_id, ColumnFamilyData* cfd,
    MemTable* mem, VersionEdit* edit) {
  s = BuildTable(...);
  edit->AddFile(level0, meta.fd.GetNumber(),...);
}
```

WriteLevel0TableForRecovery 主要调用BuildTable 写入sst文件，并且记录此次更新至VersionEdit。


<br>

### Summary

Rocksdb的Recover流程主要包括两个部分，VersionSet的恢复，和LOG文件的恢复。MANIFEST文件在恢复过程中起到了重要的作用，值得注意的是MANIFEST文件是只能追加不能覆盖的，这样可以保证新的VersionEdit即使写入失败也不会擦除之前的任何一次VersionEdit 数据。恢复过程最坏的情形是最后一个VersionEdit 恢复失败。由于任何的持久化操作都是先落盘再写MANIFEST的，这种情况可以依靠上一个VersionEdit 状态和当前db 文件推断出需要恢复的文件，进而保证数据的安全。同理LOG文件也是只追加不能覆盖的，所以其最后一条record有可能是残缺的，由于Rocksdb的写入逻辑是先写WAL LOG文件再写memtable，由于最后一条record是残缺的，这时候memtable一定还没有写入，也一定没有返回客户端写入成功，所以恢复过程可以忽略这一条record。这种顺序追加写的方式，其实大大简化了处理逻辑，只需要处理最后一条record失败的情况就可以了。



Rocksdb 和 Leveldb 将文件追加顺序写的方式应用到了实现的各个部分，获得各种好处的同时也带来了各种问题，比如说MANIFEST顺序写，意味着需要定期合并VersionEdit，清理多余的MANIFEST文件，意味着恢复流程需要一条一条VersionEdit的replay，一定程度上拖慢了open的时间。比如sst的顺序写，意味着需要定期的compaction，意味着一定程度上的写放大等等。


<br>

### Reference

[Rocksdb Source Code 5.18.3](https://github.com/facebook/rocksdb/tree/v5.18.3)

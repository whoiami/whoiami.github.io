---
layout: post
title: Rocksdb Code Analysis Transaction 2PC
---


![](/public/images/2020-04-15/rocksdb_cover.png)

<br>
### Background Introduction
<br>
#### Two Phase Commit

Two Phase Commit(2PC)适用于保证多Transaction 之间逻辑上的原子性。要不所有Txn都提交成功，要不所有Txn都没有提交成功。2PC常用于分布式场景下实现分布式事务，对于分布式事务来说将一个客户端的事务拆分成各个节点的小事务，通过2PC协议可以保证客户端整个Txn 的原子性。然而，对于支持2PC的引擎来说，不需要考虑2PC具体实现细节，只需要提供Prepare 和 Commit两个接口提供调用就可以了。具体的2PC 协议见[Two Phase Commit Protocol](https://en.wikipedia.org/wiki/Two-phase_commit_protocol). 这里只讨论Rocksdb 为上层实现2PC 提供的Prepare 和Commit 接口。


<br>
#### WriteBatch

WriteBatch 是在Rocksdb写入时使用的一种数据结构，负责存储要写入到WAL 和Memtable的数据。数据的存储结构举例如下：

<br>
> ***Sequence(0);NumRecords(3);Put(a,1);Merge(a,1);Delete(a);***

<br>
Sequence 和NumRecords 字段是其固定字段，之后是操作的命令和操作的数据本身。



<br>
### PessimisticTransaction 2PC

Rocksdb 实现的2PC接口的流程大体上思路是在之前的Txn流程中添加Prepare 调用，通过Prepare 将Txn流程写入WAL但不写入Memtable。之后通过Commit调用，在WAL中记录Commit Txn，再将本次Txn的数据部分写入Memtable。例如：

```c++
TransactionDB* db;
TransactionDB::Open(Options(), TransactionDBOptions(), DBPath, &db);

TransactionOptions txn_options;
txn_options.two_phase_commit = true
Transaction* txn = db->BeginTransaction(write_options, txn_options);

txn->Put(...);
txn->Get(...);
txn->Prepare();
txn->Commit();
```

为实现2PC 需要将WriteBatch内的数据结构进行改进。初始化时候，在Sequence和NumRecords字段后添加Noop占位符字段。WriteBatch 初始化如下

<br>
> ***Sequence(0);NumRecords(0);Noop;***

<br>


接下来调用Put Get 接口后，WriteBatch如下

<br>
> ***Sequence(0);NumRecords(0);Noop;Put(a,1);Get(a);***

<br>


调用Prepare 后Noop 标记会被BeginPrepare 替代，同时写入EndPrepare(XID)，之后会将此WriteBatch 写入WAL，WriteBatch 结构如下

<br>
> ***Sequence(0);NumRecords();BeginPrepare();Put(a,1);Get(a);EndPrepare(XID);***

<br>


调用Commit 后重新生成一个WriteBatch 如下，然后将此WriteBatch 分别写入WAL和Memtable，写入WAL的部分只是Commit 命令本身和其Xid，因为之前已经将数据部分写入过WAL了没有必要再写一次。

<br>
> ***Sequence(0);NumRecords();Commit(xid);BeginPrepare();Put(a,1);Get(a);EndPrepare(XID);***

<br>


### Code Analysis

目前只有PessimisticTransaction实现了2PC 的相关接口。对于PessimisticTransaction的原理和基本接口实现可以参考[ROCKSDB_TRANSACTION](https://whoiami.github.io/ROCKSDB_TRANSACTION)。这里只介绍跟2PC 相关的Prepare 和Commit 接口。
<br>
<br>

#### PessimisticTransaction::Prepare

```c++
Status PessimisticTransaction::Prepare() {
  s = PrepareInternal();
}

Status WriteCommittedTxn::PrepareInternal() {
  WriteOptions write_options = write_options_;
  write_options.disableWAL = false;
  WriteBatchInternal::MarkEndPrepare(GetWriteBatch()->GetWriteBatch(),name_);
  Status s =
      db_impl_->WriteImpl(write_options, GetWriteBatch()->GetWriteBatch(),
                          /*callback*/ nullptr, &log_number_, /*log ref*/ 0,
                          /* disable_memtable*/ true);
  return s;
}

Status WriteBatchInternal::MarkEndPrepare(WriteBatch* b, const Slice& xid) {
  // a manually constructed batch can only contain one prepare section
  assert(b->rep_[12] == static_cast<char>(kTypeNoop));
  // rewrite noop as begin marker
  b->rep_[12] = static_cast<char>(kTypeBeginPrepareXID);
  b->rep_.push_back(static_cast<char>(kTypeEndPrepareXID));
  PutLengthPrefixedSlice(&b->rep_, xid);
  return Status::OK();
}
```

通过PessimisticTransaction::Prepare => WriteCommittedTxn::PrepareInternal 。

1，调用MarkEndPrepare 将kTypeBeginPrepareXID 写到原本kTypeNoop 的位置，然后放入kTypeEndPrepareXID 和本次Txn 的Xid。此时的WriteBatch中的数据可能如下所示：

> ***Sequence(0);NumRecords();BeginPrepare();Put(a,1);Get(a);EndPrepare(XID);***

2，调用WriteImpl 将WriteBatch的内容写入到 WAL 中。



<br>
#### PessimisticTransaction::Commit

```c++
Status PessimisticTransaction::Commit() {
  s = CommitInternal();
}

Status WriteCommittedTxn::CommitInternal() {
  // We take the commit-time batch and append the Commit marker.
  WriteBatch* working_batch = GetCommitTimeWriteBatch();
  // push back kTypeCommitXID marker and xid
  WriteBatchInternal::MarkCommit(working_batch, name_);

  // any operations appended to this working_batch will be ignored from WAL
  working_batch->MarkWalTerminationPoint();

  // append WriteBatch content into new working batch. WriteBatch content 
  // will not append to WAL since MarkWalTerminationPoint is invoked
  WriteBatchInternal::Append(working_batch,GetWriteBatch()->GetWriteBatch());
  // write 
  auto s = db_impl_->WriteImpl(write_options_, working_batch,
                               nullptr, nullptr, log_number_);
  return s;
}
```

1，新建一个WriteBatch，在新的WriteBatch 中写入Commit 标记和Xid。

2，将准备写入WAL的位置标记在Commit之前，这样只有Commit 标记跟Xid 写入WAL。

3，将之前保存数据的WriteBatch 追加到新建的WriteBatch上。

4，调用WriteImpl 将Commit 标记跟Xid 写入WAL，将数据写入Memtable。

新建的WriteBatch可能如下所示：

> ***Sequence(0);NumRecords();Commit(xid);BeginPrepare();Put(a,1);Get(a);EndPrepare(XID);***


<br>

#### Recovery From WAL

恢复阶段主要是将WAL中的已经Commit的日志加载到Memtable中，之后Memtable 的恢复主要调用WriteBatchInternal::InsertInto 接口来完成。

```c++
Status WriteBatchInternal::InsertInto(
    const WriteBatch* batch, ColumnFamilyMemTables* memtables,
    ...) {
  MemTableInserter inserter(Sequence(batch), memtables,...);
  Status s = batch->Iterate(&inserter);
  return s;
}

Status WriteBatch::Iterate(Handler* handler) const {
  while() {
    switch (tag) {
      case kTypeColumnFamilyValue:
        handler->PutCF(column_family, key, value);
        break;
      case kTypeBeginPrepareXID:
        handler->MarkBeginPrepare();
        break;
      case kTypeEndPrepareXID:
        handler->MarkEndPrepare(xid);
        break;
      case kTypeCommitXID:
        handler->MarkCommit(xid);
        break;
        .......
    }
  }
}
```

通过对WriteBatch::Iterate 的调用，对WriteBatch 进行命令的遍历处理，例如正常的写入命令会调用handler->PutCF接口，这里调用的是MemTableInserter 的实现。这里主要关注2PC 相关的数据恢复。

```c++
class MemTableInserter : public WriteBatch::Handler {
  Status PutCFImpl(uint32_t column_family_id, const Slice& key,
                   const Slice& value, ValueType value_type) {
    WriteBatchInternal::Put(rebuilding_trx_, column_family_id, key, value);  
  }
  
  Status MarkBeginPrepare() override {
    rebuilding_trx_ = new WriteBatch();
  }
  
  Status MarkEndPrepare(const Slice& name) override {
    db_->InsertRecoveredTransaction(recovering_log_number_, name.ToString(),
                                      rebuilding_trx_, rebuilding_trx_seq_);
  }
  
  Status MarkCommit(const Slice& name) override {
    RecoveredTransaction* trx = 
      db_->GetRecoveredTransaction(name.ToString());
    s = trx->batch_->Iterate(this);
  }
}
```

1，BeginPrepare 标记处理，将新建一个用于后续Commit标记处理的WriteBatch。

2，Put 标记处理，将Put 标记压到WriteBatch 中。

3，EndPrepare 标记处理，将Txn记录到全局db 中，由于EndPrepare 跟Commit标记中间可能会有其他标记，例如单独对db进行若干次的读写操作。所以将此次WriteBatch 写入Txn，遇到Commit 标记再从全局db 中查找出来进行Commit 标记处理。

4，Commit 标记处理，从db 中找到步骤 3 写入的Txn，调用WriteBach的Iterate，对WriteBatch 的每一条读写标记进行处理。

<br>

### Summary

对于Rocksdb来说只是提供2PC 所需要的接口，对于上层如何使用实现2PC的细节完全不是Rocksdb该关心的事情。这里Rocksdb 把写WAL 跟写Memtable 流程通过Prepare 跟Commit 两个命令分别实现，这样可以保证Commit写入WAL后，这个Txn一定可以持久化，即使在写入Memtable当中发生断电，也可以从WAL中进行恢复。但是，抛开Rocksdb的实现来看，2PC 协议本身有许多诟病，有兴趣的读者可以自行查找。


<br>
### Reference

[RocksdbWiki: Two Phase Commit Implementation](https://github.com/facebook/rocksdb/wiki/Two-Phase-Commit-Implementation)

[Rocksdb Source Code 5.9](https://github.com/facebook/rocksdb/tree/v5.9.2)

[https://kernelmaker.github.io/transactiondb-intro](https://kernelmaker.github.io/transactiondb-intro)

[ROCKSDB_TRANSACTION](https://whoiami.github.io/ROCKSDB_TRANSACTION)

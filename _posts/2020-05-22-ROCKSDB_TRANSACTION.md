---
layout: post
title: Rocksdb Code Analysis Transaction
---

![](/public/images/2020-04-15/rocksdb_cover.png)

### Introduction


#### PessmisticTransaction & OptimisticTransaction

Rocksdb 实现了事务的功能，大体上分为PessimisticTransaction 和OptimisticTransaction。 其默认实现是PessimisticTransaction。PessimisticTransaction 的具体实现是通过行锁的方式，对于Transaction每一个write操作都进行尝试加锁，如果加锁失败则表明发生了冲突，在此次调用处返回异常。而OptimisticTransaction 不同的是，其不会在Transaction执行期间尝试加锁，发现冲突的过程是在commit命令内部，对Transaction操作的key 进行合法性检查，通过之后才可以commit。



#### Handle Conflict

因为Rocksdb 为每一次操作提供一个全局的sequence number，同一key的更新或删除都可以看作是对于这个key的一个新版本的写入，这种LSM-tree的数据结构是天然的Multiversion，非常适合MVCC。对于高并发读的场景，每个Transaction可以读到一个固定版本的数据。所以[Write–Read conflict](https://en.wikipedia.org/wiki/Write–read_conflict) 对于Rocksdb来说不需要做额外的操作，每一个Transaction Read操作都是读取一个key的某一个版本，同时这个版本一定是Commit之后的数据，所以dirty read 不可能产生。那么剩下的就是[Read-Write conflict](https://en.wikipedia.org/wiki/Read–write_conflict)，和[Write-Write conflict](https://en.wikipedia.org/wiki/Write–write_conflict)。


Read-Write conflict & Write-Write conflict

对于Read-Write conflict的处理，Rocksdb内部直接调用GetForUpdate (instead of Get)， 将一次读数据变成了一次写操作，实际上就是如果有可能出现Read-Write conflict，需要对读操作也要调用TryLock的操作。

> GetForUpdate Verifies there have been no commits to this key in the db since this sequence number.

可以保证在从这个sequence number之后，对于这个key，没有其他Transaction进行Commit。

这样就将Read-Write conflict 转化为了 Write-Write conflict。下面介绍Write-Write conflict 的处理。

<br>

Write-Write conflict detection: pessimistic approach

PessimistcTransaction 通过TryLock 接口将需要加锁的key依次加锁，如果所有key都加锁成功则可以commit。

Write-Write conflict detection: optimistic approach

OptimisticTransaction 通过记录需要加锁的key，在commit的时候调用TransactionUtil::CheckKeysForConflicts 进行总体的合法性检查。

> 检查Transaction 需要加锁的key，在DB中找到该key的最近的一个操作的sequence number，如果小于Transaction Sequence number 则合法（此次commit之后该key的commit 顺序也是按照seq顺序递增的），否则不合法。
>
> 由于在sst文件查找一个key的最新版本非常耗时，Rocksdb 默认只在内存（memtablelist）中确认合法性，如果内存中没有该key，默认合法性检测通过。

<br>

### Few Examples

#### Read Committed example

```c++
// Read Committed example
Transaction* txn = txn_db->BeginTransaction(write_options);
			// Put a key outside of transaction
			s = txn_db->Put(write_options, "abc", "abc");
// Read a key in this transaction
s = txn->Get(read_options, "abc", &value);
			// Put a key outside of transaction again
			s = txn_db->Put(write_options, "abc", "def");
// Read a key in this transaction again
s = txn->Get(read_options, "abc", &value);
s = txn->Commit();
```

这里Transaction 的操作如下Get(abc) Get(abc)。

在第一次Get之前DB已经commit了 Put("abc", "abc") 所以第一次Transaction内部的Get读到"abc"。在第一次Get之后第二次get之前，DB中又Commit了Put("abc", "def")，所以第二次Get读到"def"。

这也符合Read Committed 的定义，即Transaction读到的都是Committed的数据，但并不能保证多次对于同一个key读取结果是一致的。

<br>

#### Repeatable Read failed example

为保证Repeatable Read 需要调用GetForUpdate 将Read-Write conflict 转换成Write-Write conflict。

```c++
// Repeatable Read failed example
Transaction* txn = txn_db->BeginTransaction(write_options);
read_options.snapshot = txn_db->GetSnapshot();
		// Put a key outside of transaction
    s = txn_db->Put(write_options, "abc", "xyz");
s = txn->GetForUpdate(read_options, "abc", &value);
// s is Resource busy:
s = txn->GetForUpdate(read_options, "abc", &value);
// s is Resource busy:
```

GetForUpdate 函数检测到在本Transaction之外DB已经对这个key进行了更新，当前Transaction持有的sequence number是一个老的sequence number，为了保证相同key的提交顺序按照seq 递增，老的seq 不能在新的seq之后更新，所以当前Transaction失败。

<br>

#### Repeatable Read success example

```c++
// Repeatable Read success example
Transaction* txn = txn_db->BeginTransaction(write_options);
read_options.snapshot = txn_db->GetSnapshot();
		// Put a key outside of transaction
    s = txn_db->Put(write_options, "abc", "xyz");
txn->SetSnapshot();
s = txn->GetForUpdate(read_options, "abc", &value);
// s is ok res "xyz"
s = txn->GetForUpdate(read_options, "abc", &value);
// s is ok same res "xyz"
```

第一次调用GetForUpdate 时进行合法性检查，由于提前调用了SetSnapshot， Transaction的sequence number更新后大于第一次Put的sequence number，对于Write-Write conflict来说可以commit，对于GetForUpdate来说加锁成功。并且之后的Transaction在本Transaction放锁之前都不会获得锁，所以可以实现Repeatable Read。

下面来结合代码具体分析Rocksdb Transaction实现。


<br>

### Code Analysis


#### TransactionBaseImpl::Get

对于Get接口与正常的DB读取类似，这里就不详细分析了。这的注意的是，Get接口在DB查找之前，先在WriteBatchWithIndex::GetFromBatchAndDB 这个接口中进行查找，该结构里用skiplist存了当前Transaction修改的key，如果能读到当前Transaction修改的key，那么直接用这个key的value作为结果返回。如果没有查找到，则调用DBImpl::GetImpl在DB中进行查找，查找过程详见 [ROCKSDB_GET](https://whoiami.github.io/ROCKSDB_GET)。



#### TransactionBaseImpl::Put

```c++
Status TransactionBaseImpl::Put(ColumnFamilyHandle* column_family,
                                const Slice& key, const Slice& value) {
  Status s =
      TryLock(column_family, key, false /* read_only */, true/*exclusive*/);
  if (s.ok()) {
    s = GetBatchForWrite()->Put(column_family, key, value);
    if (s.ok()) {
      num_puts_++;
    }
  }
  return s;
}

```

Put接口对于PessimisticTransaction 和OptimisticTransaction 来说都是先调用其各自实现的TryLock 接口，然后调用Put接口放入WriteBatchWithIndex 中。



#### Commit

Commit 阶段，PessimisticTransaction Commit 调用db_impl_->WriteImpl，PessimisticTransaction Commit 调用 db_impl->WriteWithCallback 写入数据。

<br>

整体上，PessimisticTransaction 和OptimisticTransaction对于Transaction的接口实现类似，唯一不同的是他们冲突检测的时机，PessimisticTransaction的冲突检测在TryLock 调用当中，而OptimisticTransaction 的冲突检测在Commit中。我们分别来看这两种模式冲突检测的方法。

<br>

#### PessimisticTransaction::TryLock

```c++
Status PessimisticTransaction::TryLock(ColumnFamilyHandle* column_family,
                                       const Slice& key, bool read_only,
                                       bool exclusive, bool untracked) {
  const auto& tracked_keys = GetTrackedKeys();
  // check previously_locked and lock_upgrade
  if (!previously_locked || lock_upgrade) {
    s = txn_db_impl_->TryLock(this, cfh_id, key_str, exclusive);
  }
  if (snapshot_ == null /*snapshot not set*/) {
    // Need to remember the earliest sequence number that we know that this
    // key has not been modified after.
  } else {
    // If a snapshot is set,we need to make sure the key hasn't been modified
    // since the snapshot.  This must be done after we locked the key.
    s = ValidateSnapshot(column_family, key, current_seqno, &new_seqno);
  }
}
```

1，如果之前这个key没有加过锁，调用TransactionLockMgr TryLock加行锁。

2，如果设置了snapshot，调用ValidateSnapshot 进行合法性检查。

<br>

```c++
Status PessimisticTransaction::ValidateSnapshot(
    ColumnFamilyHandle* column_family, const Slice& key,
    SequenceNumber* new_seqno) {
  return TransactionUtil::CheckKeyForConflicts(db_impl_, cfh, key.ToString(),
                                              snapshot_->GetSequenceNumber(),
                                               false /* cache_only */);
}
Status TransactionUtil::CheckKeyForConflicts(
    DBImpl* db_impl,ColumnFamilyHandle* column_family,const std::string& key,
    SequenceNumber snap_seq, bool cache_only) {
  SuperVersion* sv = db_impl->GetAndRefSuperVersion(cfd);
  SequenceNumber earliest_seq =
        db_impl->GetEarliestMemTableSequenceNumber(sv, true);
  return CheckKey(db_impl, sv, earliest_seq, snap_seq, key, cache_only);
}
Status TransactionUtil::CheckKey(DBImpl* db_impl, SuperVersion* sv,
                                 SequenceNumber earliest_seq,
                                 SequenceNumber snap_seq,
                                 const std::string& key,
                                 bool cache_only) {
  // Since it would be too slow to check the SST files, we will only use
  // the memtables to check whether there have been any recent writes
  // to this key after it was accessed in this transaction.  But if the
  // Memtables do not contain a long enough history, we must fail the
  // transaction.
  if (snap_seq < earliest_seq) {
    need_to_read_sst = true;
    if (cache_only) {
      result = Status::TryAgain(msg);
    }
  }
  // get from memtable and immutable memtable and sst if necessary
  Status s = db_impl->GetLatestSequenceForKey(sv, key, !need_to_read_sst,
                                              &seq, &found_record_for_key);
  if (!(s.ok() || s.IsNotFound() || s.IsMergeInProgress())) {
    result = s;
  } else if (found_record_for_key) {
    bool write_conflict = snap_seq < seq;
    if (write_conflict) {
      result = Status::Busy();
    }
  }
  return result;
}
```

PessimisticTransaction::ValidateSnapshot => TransactionUtil::CheckKeyForConflicts => TransactionUtil::CheckKey

这里主要关注TransactionUtil::CheckKey

1，如果开启cache_only 只在memtalbelist 中查找该key。

2，调用 GetLatestSequenceForKey 获得这个key的最新版本。

> 如果 snapshot seq >= key seq，说明此次commit之后该key的commit 顺序也是按照seq顺序递增的。则返回ok。
>
> 否则 write_conflict 置true，返回Busy



#### OptimisticTransaction::Commit

由于OptimisticTransaction是在Commit处理冲突，这里TryLock 相当于没有做任何重要事情。所以这里重点关注一下OptimisticTransaction在Commit阶段的冲突检测。

```c++
Status OptimisticTransaction::Commit() {
  // Set up callback which will call CheckTransactionForConflicts() to
  // check whether this transaction is safe to be committed.
  OptimisticTransactionCallback callback(this);
  Status s = db_impl->WriteWithCallback(
      write_options_, GetWriteBatch()->GetWriteBatch(), &callback);
  if (s.ok()) {
    Clear();
  }
  return s;
}

```

1，调用db_impl->WriteWithCallback。

2，WriteWithCallback函数中，对于OptimisticTransactionCallback的调用在在写入WAL 和memtable 之前。

```c++
class OptimisticTransactionCallback : public WriteCallback {
  Status Callback(DB* db) override {
    return txn_->CheckTransactionForConflicts(db);
  }
}
Status OptimisticTransaction::CheckTransactionForConflicts(DB* db) {
  // Since we are on the write thread and do not want to block other writers,
  // we will do a cache-only conflict check.  This can result in TryAgain
  // getting returned if there is not sufficient memtable history to check
  // for conflicts.
  return TransactionUtil::CheckKeysForConflicts(db_impl, GetTrackedKeys(),
                                                true /* cache_only */);
}
Status TransactionUtil::CheckKeysForConflicts(DBImpl* db_impl,
                                            const TransactionKeyMap& key_map,
                                            bool cache_only) {
  Status result;
  for (auto& key_map_iter : key_map) {
    SequenceNumber earliest_seq =
        db_impl->GetEarliestMemTableSequenceNumber(sv, true);
    // For each of the keys in this transaction, check to see if someone has
    // written to this key since the start of the transaction.
    for (const auto& key_iter : keys) {
      result = CheckKey(db_impl, sv, earliest_seq, key_seq, key, cache_only);
      if (!result.ok()) {
        break;
      }
    }
  }
  return result;
}
```

OptimisticTransactionCallback::Callback 函数最终调用了CheckTransactionForConflicts 调用CheckKeysForConflicts 调用CheckKey 检查Transaction的每一个TrackedKeys 是否合法。

值得注意的是：

1，OptimisticTransaction的CheckTransactionForConflicts只检查内存中（memtablelist）的冲突检测。如果该key不在于内存，默认返回没有冲突。这样设计的主要原因是CheckTransactionForConflicts 是当前线程独占，并且会阻塞其他写线程。如果去sst中做冲突检测，会阻塞其他写线程很久，造成过大的性能损耗。

2，由于Rocksdb的写入模式是只有Leader线程在调用CheckTransactionForConflicts所以不可能出现当前在做key检测冲突的时候，有其他Transaction提交，[lost update](https://en.wikipedia.org/wiki/Write–write_conflict)也不存在。具体Rocksdb写入过程见[ROCKSDB_PUT](https://whoiami.github.io/ROCKSDB_PUT)。

<br>

### Summary

总体来说，Rocksdb通过其multiversion 的特性，可以实现Read Committed隔离性等级，利用GetForUpdate接口，将读请求转化为Write-Write Conflict实现Repeatable Read隔离性等级。对于Write-Write Conflict 冲突本身，Rocksdb利用加锁的方式来解决，这里Rocksdb提供了锁超时的方式来防止死锁，同时也提供了死锁检测机制。对于MVCC或者Snapshot Isolation来说，如果不使用加锁的机制，也可以通过timestamp实现高等级的隔离性等级，同时没有加锁带来的额外开销。其他的并发策略具体见[CONCURRENT_CONTROL](https://whoiami.github.io/CONCURRENT_CONTROL)。

<br>

### Reference

[Rocksdb Source Code 5.9](https://github.com/facebook/rocksdb/tree/v5.9.2)

[https://github.com/facebook/rocksdb/wiki/Transactions](https://github.com/facebook/rocksdb/wiki/Transactions)

[TRANSACTION CONCURRENT_CONTROL](https://whoiami.github.io/CONCURRENT_CONTROL)

[ROCKSDB_Code Analysis PUT](https://whoiami.github.io/ROCKSDB_PUT)

[ROCKSDB_Code Analysis GET](https://whoiami.github.io/ROCKSDB_GET)

[Database System Concepts](https://book.douban.com/subject/4740662/)

[Wikipedia](https://www.wikipedia.org)

---
layout: post
title: Rocksdb Code Analysis MergingIterator
---

![](/public/images/2020-04-15/rocksdb_cover.png)

<br>

### Basic Rocksdb Background


Rocksdb是一种LSM-tree的工业实现，其提供了一种可以遍历整个数据库的接口**Iterator**。利用如下的调用就可以很方便的遍历整个数据库。

 ```
rocksdb::Iterator* it = db->NewIterator(readOptions);
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  // Do something with it->key() and it->value().
}
 ```

利用Rocksdb 提供的Iterator，我们可以把Rocksdb 当作是全局有序数据来进行操作。然而真正的数据是不可能按照全局排序来组织的，Rocksdb利用了类似std::Iterator的概念，封装了底层的数据存储方式，使用户在遍历时可以屏蔽真正底层的数据存储方式。

<br>

### Basic Rocksdb Iterator


Rocksdb Iterator 的概念继承自Leveldb，对于每一种数据结构提供统一的查找和遍历接口。其基类函数如下。

```c++
class InternalIterator {
  bool Valid();
  void SeekToFirst();
  void SeekToLast();
  void Seek(const Slice& target);
  void Next();
  void Prev();
  // current key() and value()
  Slice key();
  Slice value();
}
```

Iterator的概念在Rocksdb中被大量使用，对于memtable, immutable memtable, sst block 都有相应的Iterator 实现。对于每一层数据，也有操作一层所有数据的Iterator，近而对于整个数据库，也有操作所有数据的rocksdb::Iterator。rocksdb::Iterator 本质是管理memtable 和所有的sst文件，这项管理工作Rocksdb内部抽象成了一种特殊的Iterator，就是今天的主角MergingIterator。

<br>

### MergingIterator


利用MergingIterator 可以为rocksdb::Iterator 提供全局顺序的遍历。MergingIterator继承自InternalIterator，向上提供统一的查找和遍历接口。

```c++
class MergingIterator::InternalIterator {
  // InternalIterator virtual func implementation
  ...
  autovector<IteratorWrapper, kNumIterReserve> children_;
  // Cached pointer to child iterator with the current key, or nullptr if no
  // child iterators are valid.  This is the top of minHeap_ or maxHeap_
  // depending on the direction.
  IteratorWrapper* current_;
}
```

除了需要实现InternalIterator 的接口，为了管理其他的Iterator，MergingIterator维护一个当前管理Iterator 集合children，为了缓存当前MergingIterator遍历结果，维护current指向了当前的key 和value 的位置

<br>

### MergingIterator创建


  ```c++
Iterator* DBImpl::NewIterator(const ReadOptions& read_options,
															ColumnFamilyHandle* column_family) {
  return NewInternalIterator(read_options, cfd, sv, db_iter->GetArena(),
    db_iter->GetRangeDelAggregator());
}

  ```

Rocksdb对外提供的DBImpl::NewIterator 接口主要是调用DBImpl::NewInternalIterator。

```c++
InternalIterator* DBImpl::NewInternalIterator(
    const ReadOptions& read_options, ColumnFamilyData* cfd,
    SuperVersion* super_version, Arena* arena,
    RangeDelAggregator* range_del_agg) {
  // Collect iterator for mutable mem
  // class MemTableIterator : public InternalIterator
  merge_iter_builder.AddIterator(
      super_version->mem->NewIterator(read_options, arena));
  // Collect all needed child iterators for immutable memtables
  // class MemTableIterator : public InternalIterator
  super_version->imm->AddIterators(read_options, &merge_iter_builder);
  // Collect iterators for files in L0 - Ln
  // 
  super_version->current->AddIterators(read_options, env_options_,
                                       &merge_iter_builder, range_del_agg);
}
```

1，在MergingIterator中添加memtable 和 immutable iterator。

2，在MergingIterator中添加version的iterator。

```c++
void Version::AddIterators(const ReadOptions& read_options,
                           const EnvOptions& soptions,
                           MergeIteratorBuilder* merge_iter_builder,
                           RangeDelAggregator* range_del_agg) {
  for (int level = 0; level < storage_info_.num_non_empty_levels(); level++){
    AddIteratorsForLevel(read_options, soptions, merge_iter_builder, level,
                         range_del_agg);
  }
}

void Version::AddIteratorsForLevel(const ReadOptions& read_options,
                                   const EnvOptions& soptions,
                                   MergeIteratorBuilder* merge_iter_builder,
                                   int level,
                                   RangeDelAggregator* range_del_agg) {
  if (level == 0) {
    //Level0 class TwoLevelIterator : public InternalIterator
    merge_iter_builder->AddIterator(cfd_->table_cache()->NewIterator();
  } else {
    // For levels > 0, we can use a concatenating iterator that
    // sequentially walks through the non-overlapping files in
    // the level, opening them lazily.
    auto* first_level_iter = new (mem) LevelFileNumIterator();
    merge_iter_builder->AddIterator(
      NewTwoLevelIterator(state, first_level_iter, arena, false));
  }
}
```

1，为level0的每一个文件都提供一个单独的iterator （因为level0的数据可能会有重叠）。

2，为level > 0的其他层，每一层提供一个TwoLevelIterator，用作每一层的查询和遍历。

<br>

### MergingIterator 遍历概述


MergingIterator 的遍历的主要流程是将各个数据结构的iterator指向下一个查找的位置，然后通过比较每一种数据结构当前指向key的大小，最终反馈给上层一个全局的搜索或者遍历结果。

例如：

MergingIterator 目前管理四个child iterator分别是memtable iterator，level0的第1个iterator和level0的第2个iterator。其中，level1的iterator可以操作整个level1所有的数据。单个memtable iterator 只能操作一个memtable，level0-1和level0-2也类似只能操作一个sst文件。

SeekToFirst

Memtable	       <font color='red'>(1,1)</font>, (2,2), (3,10), (10,1), (100,1)

Level0-1 		     <font color='red'>(3,9)</font>, (3,8), (3,7), (11,1)

Level0-2         <font color='red'>(2,1)</font>, (3,6), (3,5), (123,1)

Level1           <font color='red'>(3,4)</font>, (3,3), (3,2), (3,1)

current_ = (1,1) from Memtable

> 红字表示当前结构child iterator所在的位置，也是下一次需要搜索的位置。
>
> (3,1) 表示key3 的第一个版本，(3,10) 表示key3 的第10个版本。(3,10) 数据比(3,1) 更新，所在位置比(3,1) 更上层，正向查找时也应该优先搜索到。
>
> current 表示当前搜索的结果。
调用SeekToFirst 后，所有child 的iteraor被指到了起始位置，通过比较每一个child iterator指向的key值，最终对上层的返回值应该是(1,1)，并存储在current内部。


Next()

Memtable	 (1,1), <font color='red'>(2,2)</font>, (3,10), (10,1), (100,1)

Level0-1 		<font color='red'>(3,9)</font>, (3,8), (3,7), (11,1)

Level0-2         <font color='red'>(2,1)</font>, (3,6), (3,5), (123,1)

Level1            <font color='red'>(3,4)</font>, (3,3), (3,2), (3,1)

current_ = (2,2) from Memtable

current位置后移，查找各个结构当前位置最小的key。

<br>

Next()

Memtable	 (1,1), (2,2), <font color='red'>(3,10)</font>, (10,1), (100,1)

Level0-1 		<font color='red'>(3,9)</font>, (3,8), (3,7), (11,1)

Level0-2         <font color='red'>(2,1)</font>, (3,6), (3,5), (123,1)

Level1            <font color='red'>(3,4)</font>, (3,3), (3,2), (3,1)

current_ = (2,1) from Level0-2

<br>

Next()

Memtable	 (1,1), (2,2), <font color='red'>(3,10)</font>, (10,1), (100,1)

Level0-1 		<font color='red'>(3,9)</font>, (3,8), (3,7), (11,1)

Level0-2         (2,1), <font color='red'>(3,6)</font>, (3,5), (123,1)

Level1            <font color='red'>(3,4)</font>, (3,3), (3,2), (3,1)

current_ = (3,10) from Memtable

<br>

Next()

Memtable	 (1,1), (2,2), (3,10), <font color='red'>(10,1)</font>, (100,1)

Level0-1 		<font color='red'>(3,9)</font>, (3,8), (3,7), (11,1)

Level0-2         (2,1), <font color='red'>(3,6)</font>, (3,5), (123,1)

Level1            <font color='red'>(3,4)</font>, (3,3), (3,2), (3,1)

current_ = (3,9) from Level0-1

<br>

Next()

Memtable	 (1,1), (2,2), (3,10), <font color='red'>(10,1)</font>, (100,1)

Level0-1 		(3,9), <font color='red'>(3,8)</font>, (3,7), (11,1)

Level0-2         (2,1), <font color='red'>(3,6)</font>, (3,5), (123,1)

Level1            <font color='red'>(3,4)</font>, (3,3), (3,2), (3,1)

current_ = (3,8) from Level0-1

<br>

目前遍历的结果如下

> Current Traversal:  (1,1),  (2,2),  (2,1),  (3,10),  (3,9), (3,8) 

下面反向遍历

Prev()

direction_ = kReverse

Memtable	 (1,1), (2,2), <font color='red'>(3,10)</font>, (10,1), (100,1)

Level0-1 		<font color='red'>(3,9)</font>, (3,8), (3,7), (11,1)

Level0-2         <font color='red'>(2,1)</font>, (3,6), (3,5), (123,1)

Level1       <font color='red'>""</font>（3,4), (3,3), (3,2), (3,1)

current_ = (3,9) from Level0-1

反向遍历每一个结构应该指向上一次current的prev位置，即(3, 8)在各个结构的seek位置再prev，Memtable为例seek的位置应该是(10,1) ，再进行prev 位置为(3,10)。Level0-1，Level0-2，Level1做同样的seek & prev操作。之后对各个结构指向的key做比较，最小值为current(3,9) from level0-1。

<br>

Prev()

Memtable	  (1,1), (2,2), <font color='red'>(3,10)</font>, (10,1), (100,1)

Level0-1    <font color='red'>""</font>  (3,9), (3,8), (3,7), (11,1)

Level0-2         <font color='red'>(2,1)</font>, (3,6), (3,5), (123,1)

Level1       <font color='red'>""</font>（3,4）, (3,3), (3,2), (3,1)

current_ = (3,10) from Memtable

继续反向遍历，current位置向前移，寻找当前位置最大的key。

<br>

Next()

Memtable	  (1,1), (2,2), (3,10), <font color='red'>(10,1)</font>, (100,1)

Level0-1         <font color='red'>(3,9)</font>, (3,8), (3,7), (11,1)

Level0-2         (2,1), <font color='red'>(3,6)</font>, (3,5), (123,1)

Level1            <font color='red'>(3,4)</font>, (3,3), (3,2), (3,1)

current_ = (3,9) from level0-1         

正向遍历每一个结构应该指向上一次current的next位置，即(3, 10)在各个结构的seek位置（seek 本身的意思就是find key >= target）。Level0-1为例seek的位置应该是(3,9)。Level0-2，Level1做同样的seek 操作。memtable 因为本身指向了(3, 10)需要执行next指向current的next位置(10, 1)。之后对各个结构指向的key做比较，最小值为current(3,9) from level0-1。

<br>

### MergingIterator 优化

由于MergingIterator管理的child iterator集合随着rocksdb体积的膨胀会逐渐增加，对于上述所说的比较每一个child iterator指向的key值的这种比较方式显然存在性能问题。为解决这个问题，rocksdb在leveldb的基础上，引入了MergerMinIterHeap和MergerMaxIterHeap的结构。正向查找时，需要返回当前iterator集合的最小值，对应的current指向MergerMinIterHeap的top。反向查找时，需要返回当前iterator的最大值，对应的current指向MergerMaxIterHeap的top。由于用户正向查找的概率远大于反向查找，在初始化时只初始化MergerMinIterHeap，只有用户调用反向查找的接口的时候才创建MergerMaxIterHeap。

```c++
class MergingIterator : public InternalIterator {
	autovector<IteratorWrapper, kNumIterReserve> children_;
  // Cached pointer to child iterator with the current key, or nullptr if no
  // child iterators are valid.  This is the top of minHeap_ or maxHeap_
  // depending on the direction.
  IteratorWrapper* current_;
  MergerMinIterHeap minHeap_;
  // Max heap is used for reverse iteration, which is way less common than
  // forward.  Lazily initialize it to save memory.
  std::unique_ptr<MergerMaxIterHeap> maxHeap_;
}
```

<br>

### MergingIterator Next & Prev

接下来，具体看一下MergingIterator 的代码实现。这里主要介绍Next() 和 Prev() 两个接口。

```c++
  virtual void Next() override {
    // Ensure that all children are positioned after key().
    if (direction_ != kForward) {
      SwitchToForward();
    }
    // as the current points to the current record. move the iterator forward
    current_->Next();
    if (current_->Valid()) {
      minHeap_.replace_top(current_);
    } else {
      // current stopped being valid, remove it from the heap.
      minHeap_.pop();
    }
    current_ = CurrentForward();
  }
  
  void MergingIterator::SwitchToForward() {
    ClearHeaps();
    for (auto& child : children_) {
      if (&child != current_) {
        // seek to the next key after key() in this child
        child.Seek(key());
        if (child.Valid() && comparator_->Equal(key(), child.key())) {
          child.Next();
        }
      }
      if (child.Valid()) {
        minHeap_.push(&child);
      }
    }
    direction_ = kForward;
  }
 
  IteratorWrapper* CurrentForward() const {
    assert(direction_ == kForward);
    return !minHeap_.empty() ? minHeap_.top() : nullptr;
  }


```

1，调用current的Next向后移动iterator。

2，将新的current替换minHeap的顶端元素（应为之前未向后移动的current），然后minHeap 会找到当前的最小节点，交换到minHeap最顶端。如果current iterator已经遍历完，那么就从minHeap中移除当前iterator，之后minHeap 同样会将heap中的最小节点交换到最顶端。

3，如果之前用户操作的是Prev 这次调用了Next，这时候需要调用SwitchToForward，将child iterator 重置并重新构建minHeap。

<br>

```c++
  virtual void Prev() override {
    // Ensure that all children are positioned before key().
    if (direction_ != kReverse) {
      ClearHeaps();
      InitMaxHeap();
      for (auto& child : children_) {
        if (&child != current_) {
          // seek at posion entry >= key()
          child.Seek(key());
          // seek to the first key less then key() in this child
          if (child.Valid()) {
            child.Prev();
          } else {
            // Child has no entries >= key().  Position at last entry.
            child.SeekToLast();
          } 
        }
        if (child.Valid()) {
          maxHeap_->push(&child);
        }
      }
      direction_ = kReverse;
    }
    
    current_->Prev();
    if (current_->Valid()) {
      maxHeap_->replace_top(current_);
    } else {
      maxHeap_->pop();
    }
    current_ = CurrentReverse();
  }

  IteratorWrapper* CurrentReverse() const {
    assert(direction_ == kReverse);
    assert(maxHeap_);
    return !maxHeap_->empty() ? maxHeap_->top() : nullptr;
  }
```

Prev 跟Next 的做法基本相同。

1，调用current的Prev向前移动iterator。

2，将新的current替换maxHeap的顶端元素（应为之前未向前移动的current），然后maxHeap 会找到当前的最大节点，交换到maxHeap最顶端。如果current iterator已经遍历完，那么就从maxHeap中去除当前iterator，之后maxHeap 同样会将heap中的最大节点交换到最顶端。

3，如果之前用户操作的是Next 这次又调用Prev，这时候需要做一些额外工作，将child iterator 重置并重新构建maxHeap。

<br>

### Conclusion

MergingIterator 像其名字一样，把其管理的iterator 通过merge操作，向上返回一个有序的遍历结果。这样，rocksdb的使用者可以通过申请一个简单的DBImpl::NewIterator，有序遍历整个数据库的全部内容。

1， 深入到MergingIterator代码层面，我们发现不管是Next 还是Prev ，都需要对一系列的Iterator进行同时操作，甚至有可能造成多次的读盘，所以遍历本身还是一个比较昂贵的操作。

2，如果在遍历时候改变遍历方向，这种改变需要将所有的iterator 跟current 相比较，然后通过seek 等操作将child 重置下一个用于比较的位置。由于需要多次调用seek，在查找耗时同时也有可能会有读盘的操作，所以改变遍历方向对于MergingIterator来说是一个更加昂贵的操作。

<br>

###  Reference

[Rocksdb source code 5.9](https://github.com/facebook/rocksdb/tree/v5.9.2)

[https://github.com/facebook/rocksdb/wiki/](https://github.com/facebook/rocksdb/wiki/)

[Leveldb source code](https://github.com/google/leveldb)

[Leveldb源码分析--Iterator遍历数据库](https://www.cnblogs.com/KevinT/p/3826555.html)

[Rocksdb实现分析及优化 Iterator](http://kernelmaker.github.io/Rocksdb_Iterator)


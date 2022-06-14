---
layout: post
title: Innodb Undo Record Insert Update
---


<img src="/public/images/2022-05-19/crown_trick.jpeg"  alt="图片名称" align=center />


<br>



### 0, Basic

本文主要介绍Undo Segment 的申请流程，以及后续的Insert Undo Record 和Update Undo Record 的写入流程。Innnodb 内部对于Undo Record 分为 Insert 类型和Update 类型， 对于Delete 操作是包含在Update 类型当中的。Undo Record 的格式由于历史原因修改的版本比较多，看起来比较繁琐，这里基于8.0.27统一整理Undo Insert Update Record 格式，方便更好理解Undo的内部逻辑。

<br>
### 1, Undo Segment 的内存结构

每个事务内部会有trx_rsegs_t 这样一个结构来存放分配的rollback segment 和分配的undo segments。

trx_rsegs_t 内部给了两个结构，trx_undo_ptr_t m_redo是为非临时表提供的指针。trx_undo_ptr_t m_noredo 是临时表提供的指针。

trx_undo_ptr_t 内部是rollback segment 和undo segments 指针。如果一个事物既有insert 操作，又有update 操作，那么其内部会被分配两个undo segment, 分别存放insert undo record 和update undo record。这样操作主要是为了后续清理undo record方便。

```c++
struct trx_t {
  ...
  trx_rsegs_t rsegs;    /* rollback segments for undo logging */
}

/** Rollback segments assigned to a transaction for undo logging. */
struct trx_rsegs_t {
  /** undo log ptr holding reference to a rollback segment that resides in
  system/undo tablespace used for undo logging of tables that needs
  to be recovered on crash. */
  trx_undo_ptr_t m_redo;
  /** undo log ptr holding reference to a rollback segment that resides in
  temp tablespace used for undo logging of tables that doesn't need
  to be recovered on crash. （临时表分配）*/
  trx_undo_ptr_t m_noredo;
};

/**Represents an instance of rollback segment along with its state variables.*/
struct trx_undo_ptr_t {
  trx_rseg_t *rseg;        /*!< rollback segment assigned to the
                           transaction, or NULL if not assigned
                           yet */
  trx_undo_t *insert_undo; /*!< pointer to the insert undo log, or
                           NULL if no inserts performed yet */
  trx_undo_t *update_undo; /*!< pointer to the update undo log, or
                           NULL if no update performed yet */
};


```

<br>


### 2, Undo Segment 的申请

Undo segment 的申请分为两部分， Rollback segment 的申请在事务开启的时候，Undo Segment 的申请在写undo record 的时候。

#### Rollback Segment 分配逻辑：

按照轮询的方式分配，具体方式如下，如果只有一个space 的话，就是按照reg_id 的顺序轮询。

```c++
// 分配 rseg
trx_start_low() {
  /* read only trx will not  assign rseg */
  |-> trx_assign_rseg_durable(trx); // 分配到一个rseg到 m_redo 里面，这时候
                                   //  insert_undo 和update_undo 还没有分配
}
trx_assign_rseg_durable
  |->get_next_redo_rseg_from_undo_spaces
  /* Traverse the rsegs like this: (space, rseg_id)  (0,0), (1,0), ... (n,0),
     (0,1), (1,1), ... (n,1), ... */
```


<br>

#### Undo Segment 分配逻辑：

trx_rseg_t 结构体当中有nsert_undo_cached 和update_undo_cached 链表。在commit 阶段如果使用的Undo Segment的page 只有一个，并且小于page 大小的3/4，那么这个Undo Segment 就会被放入insert_undo_cached 或者 update_undo_cached  链表里面。

```c++
// 一个读写事务如果有insert 和update 操作就会，分配一个insert undo seg 一个update undo
// seg
// commit 阶段如果使用的page 写满大于了3/4 或者这个事物使用了2个page，这个undo seg 就不
// 会被重用。
// 如果是不能cache，update undo seg 会被标记TRX_UNDO_TO_PURGE，
// 紧接着调用 trx_undo_update_cleanup 进行清理，最终会把这个undo seg 释放，这个slot 就
// 空闲了，下次trx 又可以create 相同slot 的 undo seg
trx_undo_set_state_at_finish() {
  if (undo->size == 1 && mach_read_from_2(page_hdr + TRX_UNDO_PAGE_FREE) <
      TRX_UNDO_PAGE_REUSE_LIMIT) {
    state = TRX_UNDO_CACHED;
  } else if (undo->type == TRX_UNDO_INSERT) {
    state = TRX_UNDO_TO_FREE;
  } else {
    state = TRX_UNDO_TO_PURGE;
  }
}
```



事务需要申请Undo Segment 的时候，先去rseg->insert_undo_cached，rseg->update_undo_cached缓存的队列里面取一个Undo Segment，如果没有缓存的 Undo Segment，那么就新建一个Undo Segment。
新建流程见[https://whoiami.github.io/INNODB_UNDO_PHYSICAL_FORMAT](https://whoiami.github.io/INNODB_UNDO_PHYSICAL_FORMA)



```c++
// 分配undo seg
btr_cur_optimistic_insert /* 先写入undo 再写入数据 */
  |-> btr_cur_ins_lock_and_undo
  |-> page_cur_tuple_insert

btr_cur_optimistic_update /* 先写入undo 再写入数据 */
  |->btr_cur_upd_lock_and_undo
  |->btr_cur_insert_if_possible

btr_cur_ins_lock_and_undo
  |->trx_undo_report_row_operation

trx_undo_report_row_operation
  |-> trx_undo_assign_undo /* 找到一个undo segment */
  |-> trx_undo_page_report_insert or trx_undo_page_report_modify
  
trx_undo_assign_undo
  |-> trx_undo_reuse_cached
  if (undo == nullptr/* 没有从cache 当中取到undo seg */) {
    trx_undo_create
  }
  if (type == TRX_UNDO_INSERT) {
    UT_LIST_ADD_FIRST(rseg->insert_undo_list, undo); 
  } else {
    UT_LIST_ADD_FIRST(rseg->update_undo_list, undo);
  }

trx_undo_reuse_cached {
   从 UT_LIST_GET_FIRST(rseg->insert_undo_cached); 或
      UT_LIST_GET_FIRST(rseg->update_undo_cached);
}
```


<br>

### 3, Undo Record Insert Update

写Undo Record 的入口函数是trx_undo_report_row_operation。insert 类型调用trx_undo_page_report_insert ，update 类型调用trx_undo_page_report_modify。Undo Record 格式如下：

![](/public/images/2022-05-19/undo_insert_record.png)

字段说明：

```
Unique field <len, data> 可能有很多组，primary key 可以由多个field 组成。
```



```c++
ulint trx_undo_page_report_insert() {
  ulint first_free = mach_read_from_2(undo_page + TRX_UNDO_PAGE_HDR +
                                      TRX_UNDO_PAGE_FREE);
  ptr = undo_page + first_free;
  // 填写Undo Insert Record 各个字段
  ...
  // 写对应的redo
  trx_undof_page_add_undo_rec_log(); 
}
```



![](/public/images/2022-05-19/undo_update_record.png)

字段说明：

```
type_cmpl (1)
  低4位是update 类型
    （TRX_UNDO_DEL_MARK_REC，TRX_UNDO_UPD_DEL_REC（update a deleted rec,例如
     删除后马上插入相同rec）, TRX_UNDO_UPD_EXIST_REC)
  高4位是compiler info 和 TRX_UNDO_MODIFY_BLOB
  （abcd）
   b 位置代表时候否支持外部存储的格式 TRX_UNDO_MODIFY_BLOB
   c 位置UPD_NODE_NO_SIZE_CHANGE update 没有让record field size 改变
   d 位置UPD_NODE_NO_ORD_CHANGE1 update没有让任何索引索引顺序改变 
  	
 
 info bits(1) REC_INFO_MIN_REC_FLAG REC_INFO_DELETED_FLAG REC_INFO_INSTANT_FLAG
 
 Unique field <len, data> 可能有很多组，primary key 可以由多个field 组成。
 
 Old Cols in Update <field no, len, value> 这里对于外部存储会做特殊处理
 见 https://whoiami.github.io/INNODB_BLOB
```



```c++
ulint trx_undo_page_report_modify() {
  ulint first_free = mach_read_from_2(undo_page + TRX_UNDO_PAGE_HDR +
                                      TRX_UNDO_PAGE_FREE);
  ptr = undo_page + first_free;
  // 填写Undo Insert Record 各个字段
  ...
  // 写对应的redo
  trx_undof_page_add_undo_rec_log(); 
}
```



<br>




### 4, Reference

[https://github.com/mysql/mysql-server/tree/mysql-cluster-8.0.27](https://github.com/mysql/mysql-server/tree/mysql-cluster-8.0.27)

[http://mysql.taobao.org/monthly/2015/04/01/](http://mysql.taobao.org/monthly/2015/04/01/)

[http://mysql.taobao.org/monthly/2017/12/01/](http://mysql.taobao.org/monthly/2017/12/01/)

[https://zhuanlan.zhihu.com/p/165457904](https://zhuanlan.zhihu.com/p/165457904)

[https://zhuanlan.zhihu.com/p/263038786](https://zhuanlan.zhihu.com/p/263038786)

[http://catkang.github.io/2021/10/30/mysql-undo.html](http://catkang.github.io/2021/10/30/mysql-undo.html)

---
layout: post
title: Innodb Undo Physical Format
---


<img src="/public/images/2022-04-21/neon_abyss.jpeg"  alt="图片名称" align=center />



<br>

### 0, Basic

Innodb 引擎中，Undo 日志是存放本次修改之前的原值，方便MVCC 或者事物回滚的时候能够找到修改之前的状态。

这里主要介绍Undo 的物理格式。


<br>

Innodb 内部看到的主要数据结构是Undo Tablespaces，内部包括多个Undo tablespace，也就是支持把undo log存放到多个space 内部。这里主要介绍Undo space的数据物理格式。一个Undo sapce 内部包括128个Rollback Segemnts，每一个Rollback segment 又预留了1024个undo segment slot，事务开启的时候会随机申请Rollback sengment 以及 Rollback segment  当中的slot 槽位。找到指定的undo segment，新建一个undo log，写入这个事务的一些信息到undo log header 当中，之后这个事务产生的所有的undo record 都记录在这个undo log 当中。



<br>
### 1，主要Page 物理格式

<br>
![](/public/images/2022-04-21/rollback_segment_array.png)

RSEG_ARRAY_FSEG_HEADER_OFFSET(10)  /* 申请的file segment 的元信息 */



![](/public/images/2022-04-21/rollback_segment_page.png)



TRX_RSEG_FSEG_HEADER (10)  /* 申请的file segment 的元信息 */



![](/public/images/2022-04-21/undo_segment.png)

Undo Segment 的第一个Page 当中当中包括Undo Page Header 负责存放这个undo page 的信息，包括Undo Segment Header负责存放这个Undo Segment的一些信息，以及每一个undo log 都有的Undo Log Header，负责存放这个事务产生的所有undo record。


<br>

### 2，Undo Tablespace page 的物理组织形式

<br>
Undo tablespace 当中包括128个Rollback Sengment，这些Rollback Sengment Header 所在的page 位置存放在page 3当中。通过读取相对应的偏移的数据就可以获得对应Rollback Segment Header Page 的id，读取这个Rollback Segment Header page ，它里面包含了可以使用的undo segment slot，每个slot 当中记录了Undo Segment 的第一个Page id。



![](/public/images/2022-04-21/rollback_array_to_rseg.png)

通过读取Undo Segment 的第一个page 的Undo Segment Header可以获取这个Undo Segment 的元信息，通过读取这个page 当中的Undo Page Header，可以获取这个undo page 的元信息。从而找到某一个undo log，在找到这个undo log 之后，需要先读去这个undo log 的header，可以获取这个undo log 包含undo record 的元信息。最后进行相应的undo record 操作。

![](/public/images/2022-04-21/rseg_to_undo_seg.png)

<br>
### 3，Undo Tablespace 初始化代码路径

<br>
#### 初始化Rollback Segment Array Page

```c++
srv_undo_tablespaces_init 
|-> srv_undo_tablespaces_create /* 创建一个tablespace 还没有物理文件格式 */
|-> srv_undo_tablespaces_construct /* 创建undo tablespace 的page 3 */


srv_undo_tablespaces_construct
|-> trx_rseg_array_create

/* 初始化 rseg array的page*/
trx_rseg_array_create
|-> fseg_create 返回这个space 的page 3用来存放rseg array
Rollback segment array 各个字段初始化。初始化从 RSEG_ARRAY_SIZE_OFFSET 到
page_end - 200 的位置的字段，初始化成0xff 对应读到page no 应该是FIL_NULL。
```



<br>
#### 初始化Rollback Segment 

```c++
trx_sys_init_at_db_start
|-> trx_rsegs_parallel_init
|		| -> trx_rsegs_init_start 
|		|		|-> purge_sys->rsegs_queue
|		|-> trx_rseg_init_thread (used multiple threads)
|				|-> trx_rseg_physical_initialize
|-> trx_rsegs_init (created mem rsegs in undo_space) // 没有真正初始化，真正初始化在
trx_rseg_adjust_rollback_segments 里面做
  
trx_rseg_adjust_rollback_segments(); 
|-> trx_rseg_add_rollback_segments();
		|-> trx_rseg_create
		|-> trx_rseg_mem_create


/* 初始化一个rseg header page， 内存存放于undo_space->rsegs() */ 
trx_rseg_create
|->trx_rseg_header_create
/* 初始化rollback segment header page 的各个字段 */
/* 将这个rseg的page_no加入到 rollback segment array page header里面 */
      
      
/* 初始化rseg 的内存结构 trx_rseg_t */
trx_rseg_mem_create
/*申请内存创建trx_rseg_t ，初始化rseg 的相关参数。大多是从盘上undo page 读上来的参数。
其中比较关键的是这几个queue，存放内存的undo log segments。
  UT_LIST_INIT(rseg->update_undo_list);
  UT_LIST_INIT(rseg->update_undo_cached);
  UT_LIST_INIT(rseg->insert_undo_list);
  UT_LIST_INIT(rseg->insert_undo_cached);
  */
|->trx_undo_lists_init

trx_undo_lists_init() /* 初始化undo log list*/
|-> trx_undo_mem_init
```





<br>
#### 初始化Undo Segment 

Undo Segment  没有与预创建，在事务开启的时候，如果从Rollback Segment 当中cache 的undo 里面获取不到undo segment，说明当前没有可用的undo segment 就会新建Undo Segment。

```c++
static void trx_start_low(trx_t *trx, bool read_write);
|-> trx_assign_rseg_durable(trx)  /* Assign a durable rollback segment to a
                                     transaction */
	|-> get_next_redo_rseg_from_undo_spaces()  /* get trx_rseg_t */


trx_undo_reuse_cached 
/* get trx_undo_t *undo from  rseg->insert_undo_cached or
   rseg->update_undo_cached */
```

创建Undo Segement 流程

```c++
/*  Assigns an undo log for a transaction. A new undo log is created or a
    cached undo log reused. */
trx_undo_assign_undo
|-> trx_undo_create
		|->trx_undo_seg_create /*创建 undo segment header 和
                             undo page header */
		|->trx_undo_header_create /*创建 undo log header */
		|->trx_undo_mem_create /* 创建undo segment 内存结构 */

```





<br>
### Reference

[https://github.com/mysql/mysql-server/tree/mysql-cluster-8.0.27](https://github.com/mysql/mysql-server/tree/mysql-cluster-8.0.27)

[http://mysql.taobao.org/monthly/2015/04/01/](http://mysql.taobao.org/monthly/2015/04/01/)

[https://zhuanlan.zhihu.com/p/165457904](https://zhuanlan.zhihu.com/p/165457904)

[http://catkang.github.io/2021/10/30/mysql-undo.html](http://catkang.github.io/2021/10/30/mysql-undo.html)

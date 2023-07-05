---
layout: post
title: Blob Partial Update Crash
---

<img src="/public/images/2022-05-19/crown_trick.jpeg"  alt="图片名称" align=center />


<br>


### Long Story Short

<br>
Blob 在做partial update 并且符合small change 记录到undo record ，之后如果读请求走mvcc 读到这个undo record 的时候有概率造成crash。


<br>
### 场景复现

<br>
环境8.0.25

```sql
client1
create table t (a int primary key, b json);
insert into t values (1, '[ "abc", "def" ]');
update t set b=JSON_SET(b, '$[0]', REPEAT('w', 10000));

client2
set SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
begin;
select * from t where a =1;

client1
UPDATE t SET b = JSON_REMOVE(b, '$[1]') where a = 1;

client2
select * from t where a =1;
```


<br>

Core stack

```
#3  mach_read_from_1 (b=0x0) at /storage/innobase/include/mach0data.ic:66
#4  mach_read_next_compressed () at /storage/innobase/include/mach0data.ic:288
#5  trx_undo_read_blob_update () at /storage/innobase/trx/trx0rec.cc:985
#6  trx_undo_update_rec_get_update () at /storage/innobase/trx/trx0rec.cc:1882
#7  trx_undo_prev_version_build () at /storage/innobase/trx/trx0rec.cc:2550
#8  row_vers_build_for_consistent_read () at /storage/innobase/row/row0vers.cc:1305
#9  row_sel_build_prev_vers_for_mysql () at /storage/innobase/row/row0sel.cc:3144
#10 row_search_mvcc () at /storage/innobase/row/row0sel.cc:5373
#11 ha_innobase::index_read () at /storage/innobase/handler/ha_innodb.cc:9956
#12 handler::index_read_map () at /sql/handler.h:5071
#13 handler::ha_index_read_map () at /sql/handler.cc:3249
#14 read_const () at /sql/sql_executor.cc:3503
#15 join_read_const_table () at /sql/sql_executor.cc:3373
#16 JOIN::extract_func_dependent_tables () at /sql/sql_optimizer.cc:5515
#17 JOIN::make_join_plan () at /sql/sql_optimizer.cc:5035
#18 JOIN::optimize () at /sql/sql_optimizer.cc:535
#19 Query_block::optimize () at /sql/sql_select.cc:1819
#20 Query_expression::optimize () at /sql/sql_union.cc:678
#21 Sql_cmd_dml::execute_inner () at /sql/sql_select.cc:775
#22 Sql_cmd_dml::execute () at /sql/sql_select.cc:575
#23 mysql_execute_command () at /sql/sql_parse.cc:4412
#24 dispatch_sql_command () at /sql/sql_parse.cc:5000
#25 dispatch_command () at /sql/sql_parse.cc:1841
#26 do_command () at /sql/sql_parse.cc:1320
```


<br>

### Root Cause

<br>
对于Blob 字段进行partical update，由于是small change ，binary diff 记录到了undo record 里面。select 的时候走row_search_mvcc => trx_undo_read_blob_update 路径，如果读到diff 长度，需要申请而外的内存，lob_undo_data.copy_old_data 将binary diff 从undo record 拷贝到独立内存，但是如果长度为0（lob_diff.m_length 长度是0），申请内存返回了nullptr (注意这里是个坑，后面再说)，之后会将undo_ptr 直接置空，后续流程继续用undo_ptr 解析undo record 的时候crash。

```c++
static const byte * trx_undo_read_blob_update(...) {
  undo_ptr = lob_undo_data.copy_old_data(undo_ptr, lob_diff.m_length);
  ...
  ulint n_entry = mach_read_next_compressed(&undo_ptr);
}
```

<br>


很明显这里的处理流程有问题。修复的方法也比较直接，只要不更新lob_undo_data 的m_old_data 字段就可以了。但是这里需要明确的是为什么会记录undo 长度为0 的binary diff 在undo record 里面？


<br>

### Keep Digging
<br>
这个undo 的binary diff 的长度是从

Value::remove_in_shadow => TABLE::add_binary_diff 添加到binary diff

对应的数据结构是：m_partial_update_info->m_binary_diff_vectors

report undo record 的时候，函数 trx_undo_report_blob_update 调用

update->get_binary_diff_by_field_no()->get_binary_diffs 里面

返回 m_partial_update_info->m_binary_diff_vectors



所以这里需要看一下Value::remove_in_shadow 的逻辑。这个函数主要计算两部分binary diff，一个是Json 的number of elements部分的binary diff，另一个是json meta 部分的binary diff。具体的Json 格式格式解析可以参考https://developer.aliyun.com/article/598070，这里举例说明remove_in_shadow 是如何计算出长度为0 的binary diff 的。



<br>
create table t (a int primary key, b json);

insert into t values (1, '[ "abc", "def" ]');

执行后内存结构如下：

```
        0x02 - type: small JSON array
        0x02 - number of elements (low byte)
        0x00 - number of elements (high byte)
        0x12 - number of bytes (low byte)
        0x00 - number of bytes (high byte)
        0x0C - type of element 0 (string)
        0x0A - offset of element 0 (low byte)
        0x00 - offset of element 0 (high byte)
        0x0C - type of element 1 (string)
        0x0E - offset of element 1 (low byte)
        0x00 - offset of element 1 (high byte)
        0x03 - length of element 0
        'a'
        'b'  - content of element 0
        'c'
        0x03 - length of element 1
        'd'
        'e'  - content of element 1
        'f'
```

<br>
UPDATE t SET b = JSON_REMOVE(b, '$[1]') where a = 1;

执行后内存结构如下：

```
         0x02 - type: small JSON array
 CHANGED 0x01 - number of elements (low byte)
 CHANGED 0x00 - number of elements (high byte) // 高位虽然没有变但是也算是binary diff
         0x12 - number of bytes (low byte)
         0x00 - number of bytes (high byte)
         0x0C - type of element 0 (string)
         0x0A - offset of element 0 (low byte)
         0x00 - offset of element 0 (high byte)
 [Free]  0x0C - type of element 1 (string)
 [Free]  0x0E - offset of element 1 (low byte)
 [Free]  0x00 - offset of element 1 (high byte)
         0x03 - length of element 0
         'a'
         'b'  - content of element 0
         'c'
         0x03 - length of element 1
         'd'
         'e'  - content of element 1
         'f'
```

<br>
原本的逻辑是JSON_REMOVE需要生成两个binary diff，一个是修改number of elements的信息产生的binary diff，一个是meta 拷贝产生的binary diff。这里，第一个binary diff 修改了number of elements， 对应的是上面标注CHNAGED 的部分。第二个binary diff是meta 拷贝产生的diff，正常逻辑需要保证meta 信息的连续，所以需要在meta 信息被删除，有空洞的时候把下面的meta 信息拷贝上来，但是，由于这里是删除最后一个element，没有meta空洞，不需要meta数据拷贝，所以这里Value::remove_in_shadow 里面计算出来的数据binary diff 是0。也就是对应的标注FREE 的地方，这里的free meta内存可以给之后的新element使用，所以是不需要移动的。


<br>
### Follow Up

<br>
尝试查看最新官方代码8.0.33。实测还是可能binary diff 计算出等于0的情况的。但是其他逻辑并没有做任何特殊处理，但是在后续mvcc 读的时候没有出现crash。

<br>
查看对应的undo_data_t::copy_old_data 代码发现：

```c++
const byte *undo_data_t::copy_old_data(const byte *undo_ptr, ulint len) {
  m_old_data =
    ut::new_arr_withkey<byte>(UT_NEW_THIS_FILE_PSI_KEY, ut::Count{m_length});
  if (m_old_data == nullptr) {
    return nullptr;
  }
  ...
}
```

<br>
8.0.25  当中这里len = 0 的时候，申请内存的操作返回是nullptr，后续处理流程出错导致crash。

8.0.33 这里返回不是nullptr，而是一个正常的指针。

进一步探究发现，8.0.33官方申请array 内存的逻辑有改动。正常逻辑如果申请array 类型的内存会事先申请一个array meta 内存，当中存放array 的长度。8.0.33的逻辑是如果申请长度为0，会返回array meta 的地址（非空）。

由于8.0.33跟8.0.25 在处理申请array 内存的逻辑上，对于返回值是否是nullptr 的处理上有区别，8.0.33直接用不返回nullptr 的方式避过了文章开头的crash。所以8.0.33暂时没有这个问题。但是对于8.0.33来说这是一个隐患，因为之后如果官方修改array 类型申请内存的方式，对于申请0长度的array 如果返回nullptr ，这里依然会crash。


<br>

这里给官方提交了[bug report](https://bugs.mysql.com/bug.php?id=111647) 自认为事情描述清楚了，但是官方认为“can‘t repeat crash” 认为没有问题。


<br>

8.0.33代码返回非空代码路径：

```c++

trx_undo_read_blob_update
=>undo_data_t::copy_old_data
  =>ut::new_arr_withkey<byte>(UT_NEW_THIS_FILE_PSI_KEY, ut::Count{m_length}); 
    =>Alloc_pfs::alloc(std::size_t size,pfs_metadata::pfs_memory_key_t key);
  
static inline void *alloc(std::size_t size,pfs_metadata::pfs_memory_key_t key) {
  const auto total_len = size + Alloc_pfs::metadata_len;
  auto mem = Alloc_fn::alloc<Zero_initialized>(total_len);
  return static_cast<uint8_t *>(mem) + Alloc_pfs::metadata_len;
}
```

8.0.33 代码这里计算申请字节长度的时候，size 是上层传入的申请的大小（这里是0)，函数内部加上了Alloc_pfs::metadata_len长度，这个是array 的meta 部分长度，导致了总申请长度不是0，最终返回的指针也就不会是nullptr。

<br>
这里附上8.0.25 返回nullptr 的代码路径：

```c++
trx_undo_read_blob_update
=>undo_data_t::copy_old_data
  =>UT_NEW_ARRAY_NOKEY(byte, m_length)
    =>ut_allocator<type>(key).new_array(n_elements, UT_NEW_THIS_FILE_PSI_KEY)
      =>new_array(size_type n_elements, PSI_memory_key key)
        => allocate(...)

pointer allocate(size_type n_elements, ...) {
  if (n_elements == 0) {
    return (nullptr);
  }
}
```

<br>
8.0.25 可以复现(size 是0 就返回nullptr)。8.0.26 没问题(做了类似8.0.33 的处理返回array meta)。 8.0.33（目前最新版本） 接口做了重做，返回array meta。



<br>

### Reported bug

[https://bugs.mysql.com/bug.php?id=111647](https://bugs.mysql.com/bug.php?id=111647)

### Reference

[https://dev.mysql.com/blog-archive/mysql-8-0-optimizing-small-partial-update-of-lob-in-innodb/](https://dev.mysql.com/blog-archive/mysql-8-0-optimizing-small-partial-update-of-lob-in-innodb/)

[https://whoiami.github.io/INNODB_BLOB](https://whoiami.github.io/INNODB_BLOB)

[https://developer.aliyun.com/article/598070](https://developer.aliyun.com/article/598070)

[https://github.com/mysql/mysql-server/tree/mysql-8.0.33](https://github.com/mysql/mysql-server/tree/mysql-8.0.33)

[https://github.com/mysql/mysql-server/tree/mysql-8.0.25](https://github.com/mysql/mysql-server/tree/mysql-8.0.25)

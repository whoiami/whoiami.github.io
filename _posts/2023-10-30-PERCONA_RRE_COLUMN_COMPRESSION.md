---
layout: post
title: Percona Per-column Compression Code Analysis
---


<img src="/public/images/2023-07-05/diablo.jpg"  alt="图片名称" align=center />

<br>

### 简介

列压缩是针对某一列非常长的一种压缩策略，通常可压缩列类型为varchar，BLOB等类型。其最早由[印风](https://developer.aliyun.com/article/64891) 在AliSQL 实现并且提供给了Percona社区。Percona 在其基础上进行了代码重构，结合Zlib 提供的字典压缩特性，让列压缩提供以用户自建字典的方式进行压缩。Percona官方叫做Per-column Compression。

Percona引入了两个view，是位于`INFORMATION_SCHEM`下面的 `COMPRESSION_DICTIONARY` 跟`COMPRESSION_DICTIONARY_TABLES`。

`COMPRESSION_DICTIONARY`用于存储自建字典的具体信息，包括name, version 和具体dictionary string。

<br>
[INFORMATION_SCHEMA.COMPRESSION_DICTIONARY_TABLES](https://docs.percona.com/percona-server/8.0/compressed-columns.html#information_schemacompression_dictionary_tables)

| Column Name                        | Description                     |
| :--------------------------------- | :------------------------------ |
| ‘BIGINT(21)_UNSIGNED dict_version’ | ‘dictionary version’            |
| ‘VARCHAR(64) dict_name’            | ‘dictionary name’               |
| ‘BLOB dict_data’                   | ‘compression dictionary string’ |



`COMPRESSION_DICTIONARY_TABLES`用于存储哪些列关联到这个字典的信息。

[INFORMATION_SCHEMA.COMPRESSION_DICTIONARY_TABLES](https://docs.percona.com/percona-server/8.0/compressed-columns.html#information_schemacompression_dictionary_tables)

| Column Name                        | Description                                                  |
| :--------------------------------- | :----------------------------------------------------------- |
| ‘BIGINT(21)_UNSIGNED table_schema’ | ‘table schema’                                               |
| ‘BIGINT(21)_UNSIGNED table_name’   | ‘table ID from INFORMATION_SCHEMA.INNODB_SYS_TABLES’         |
| ‘BIGINT(21)_UNSIGNED column_name’  | ‘column position (starts from 0 as in INFORMATION_SCHEMA.INNODB_SYS_COLUMNS)’ |
| ‘BIGINT(21)_UNSIGNED dict_name’    | ‘dictionary ID’                                              |



<br>
通过如下命令创建字典。

```
mysql> SET @dictionary_data = 'one' 'two' 'three' 'four';
mysql> CREATE COMPRESSION_DICTIONARY numbers (@dictionary_data);
```

通过如下命令关联字典跟对应的table 列。

```
mysql> CREATE TABLE t1(
        id INT,
        a BLOB COLUMN_FORMAT COMPRESSED,
        b BLOB COLUMN_FORMAT COMPRESSED WITH COMPRESSION_DICTIONARY numbers
      ) ENGINE=InnoDB;
```

这样修改相关列的时候，就会读出相关字典的内容传入zlib 提供的字典压缩接口，进行相关的压缩/解压缩。



<br>
### Code Analysis

列压缩在Percona上经过5.6-8.0的迭代主要的commit 如下，这里只以8.0版本做代码分析。

[Implemented "Column Compression with optional Predefined Dictionary"](https://github.com/percona/percona-server/commit/35d5d3faf00db7e32f48dcb39f776e43b83f1cb2)

[Compressed columns functionality merged from 5.6 to 5.7](https://github.com/percona/percona-server/commit/d142eb0ffc7301f638060181748a9ff1b9910761)

[Re-implement compression dictionaries in 8.0](https://github.com/percona/percona-server/commit/14e2a96c6954433970ce15b613dbbe85fb7239f0)


<br>

#### Physical Format

Column最终的存储格式是

```
ZIP_COLUMN_HEADER_LENGTH + Len + Compressed_data
```

ZIP_COLUMN_HEADER_LENGTH: 压缩之后 Column Header 部分占用（2）bytes

Len:  压缩之前的长度，其占用的字节数是个变长的数值1-4bytes，具体这个column占用多长记录在ZIP_COLUMN_HEADER_LENGTH的zip_column_data_length里面。主要用户后续解压缩之后长度的校验。

Compressed_data: 压缩之后的数据


<br>

ZIP_COLUMN_HEADER_LENGTH 如下：

```c++
/* 'reserved', bit 0 */ /* 0000 0000 0000 0001 */
static constexpr uint zip_column_reserved = 0;
/* 'wrap', bit 1 */ /* 0000 0000 0000 0010 */
static constexpr uint zip_column_wrap = 1; // zlib used parameter
/* 'algorithm', bit 2,3,4,5,6 */ /* 0000 0000 0111 1100 */
static constexpr uint zip_column_algorithm = 2;
/* 'len-len', bit 7,8,9 */ /* 0000 0011 1000 0000 */
static constexpr uint zip_column_data_length = 7;
/* 'compressed', bit 10 */ /* 0000 0100 0000 0000 */
static constexpr uint zip_column_compressed = 10;
```



<br>
#### Row_compress_column && row_decompress_column

压缩跟解压缩主要逻辑位于row_compress_column跟 row_decompress_column两个函数当中。

row_compress_column逻辑相对简单，压缩接口调用zlib 的 deflateSetDictionary，如果成功就写入相应的column header，如果不成功也要占用2bytes 写入相应的column header，原因是解压的时候需要判断这个column 是否被压缩，需要读取column header 的zip_column_compressed 字段来做判断。

row_decompress_column  逻辑先解析column header，判断是否压缩。如果没有压缩，返回不带column header 的数据就可以。如果是压缩数据，就要调用inflateSetDictionary 解压并返回解压之后的数据。这里用column header 里的未压缩之前的数据长度跟这里解压之后的数据长度进行了一个校验。

```c++
byte *row_compress_column(
  const byte *data, ulint *len, ulint lenlen,
  const byte *dict_data, ulint, dict_data_len,
  mem_heap_t **compress_heap) {
  deflateSetDictionary(dict_data);
compress_success:
  column_set_compress_header();
do_not_compress:
  column_set_compress_header();
}

const byte *row_decompress_column(
  const byte *data, ulint *len,
  const byte *dict_data, ulint dict_data_len,
  mem_heap_t **compress_heap) {
  column_get_compress_header(data);
  if（!is_compresse）{
    return;
  }
  get_uncompressed_len();
  inflateSetDictionary(&d_stream, dict_data, dict_data_len);
}
```


<br>

#### 字典信息读取

字典信息是如何传入row_compress_column 函数里的呢？

```c++
mysql -h127.0.0.1 -uroot -P7788 -A （使用-A 参数use sbtest 的时候不会open table）
use sbtest;
INSERT INTO t1 VALUES (1, REPEAT('a', 200));

#0  fill_column_from_dd () at sql/dd_table_share.cc
#1  fill_columns_from_dd () sql/dd_table_share.cc
#2  open_table_def () sql/dd_table_share.cc
#3  get_table_share () at sql/sql_base.cc
#4  get_table_share_with_discover () sql/sql_base.cc
#5  open_table () at sql/sql_base.cc
#6  open_and_process_table () at sql/sql_base.cc
#7  open_tables () sql/sql_base.cc
#8  open_tables_for_query () at sql/sql_base.cc
#9  Sql_cmd_dml::prepare () sql/sql_select.cc
#10 Sql_cmd_dml::execute () sql/sql_select.cc
#11 mysql_execute_command () at sql/sql_parse.cc
#12 mysql_parse () at sql/sql_parse.cc
#13 dispatch_command () at sql/sql_parse.cc
```

<br>
第一次 open talbe的时候，fill_column_from_dd (sql/dd_table_share.cc:)调用 compression_dict::get_name_for_id(zip_dict_id)
（zip_dict_id 是create 时候column_options带的id）读mysql.compression_dictionary这个表，获取到了zip_dict_name和zip_dict_data 存到了TABLE_SHARE当中。
之后通过TABLE_SHARE存到了m_prebuilt->mysql_template
build_template_field(storage/innobase/handler/ha_innodb.cc) 
​	=> m_prebuilt->mysql_template = share->field->zip_dict_data 
之后在row_mysql_convert_row_to_innobase(storage/innobase/row/row0mysql.cc)  中row_prebuilt_t为参数传到了row_compress_column



<br>
#### New Added Table (Bootstrap and Invoke)

INFORMATION_SCHEMA.COMPRESSION_DICTIONARY_TABLES 和INFORMATION_SCHEMA.COMPRESSION_DICTIONARY 两张表是两个view，数据实际来源自mysql.compression_dictionary 和mysql.compression_dictionary_cols 两张表中。

<br>
sql/sql_zip_dict.cc // mysql.compression_dictionary和mysql.compression_dictionary_cols 调用接口以及两张表的初始化(bootstrap)

sql/dd/impl/system_views/compression_dictionary.cc // 定义INFORMATION_SCHEMA.COMPRESSION_DICTIONARY

sql/dd/impl/system_views/compression_dictionary_tables.cc // 定义 INFORMATION_SCHEMA.COMPRESSION_DICTIONARY_TABLES

sql/dd/info_schema/metadata.cc // create_system_views 创建information_schema 的view



<br>
#### Create Dict

```
SET @dictionary_data = 'one' 'two' 'three' 'four';
CREATE COMPRESSION_DICTIONARY numbers (@dictionary_data);

sql/sql_parse.cc // SQLCOM_CREATE_COMPRESSION_DICTIONARY 
create_zip_dict 
 => open_dictionary_table_write
    (open mysql.compression_dictionary and write table columns)
```



<br>
#### Combine Dict and Column

```
create table user_data (a int(10), b int(10), c varchar(100) COLUMN_FORMAT COMPRESSED WITH COMPRESSION_DICTIONARY numbers);

sql/sql_table.cc 
rea_create_base_table
 =>compression_dict::cols_table_insert
  =>open_dictionary_cols_table_write(open mysql.compression_dictionary_cols and write table columns)
```


<br>

#### Insert & Select From Compressed Column

```
CREATE TABLE t1 (id INT PRIMARY KEY, b1 varchar(200) COLUMN_FORMAT COMPRESSED);
INSERT INTO t1 VALUES (1, REPEAT('a', 200));

#0  row_compress_column() at storage/innobase/row/row0mysql.cc:392
#1  row_mysql_store_col_in_innobase_format () at storage/innobase/row/row0mysql.cc:911
#2  row_mysql_convert_row_to_innobase () at storage/innobase/row/row0mysql.cc:1088
#3  row_insert_for_mysql_using_ins_graph() at storage/innobase/row/row0mysql.cc:2040
#4  ha_innobase::write_row(unsigned char*) () at storage/innobase/handler/ha_innodb.cc:9633
#5  handler::ha_write_row () at sql/handler.cc:8305
#6  write_record() at sql/sql_insert.cc:2168
#7  Sql_cmd_insert_values::execute_inner(THD*) at sql/sql_insert.cc:631
#8  Sql_cmd_dml::execute(THD*) at sql/sql_select.cc:578
#9  mysql_execute_command(THD*, bool) at sql/sql_parse.cc:4974
#10 dispatch_sql_command(THD*, Parser_state*, bool) at sql/sql_parse.cc:5631
#11 dispatch_command(THD*, COM_DATA const*, enum_server_command) at sql/sql_parse.cc:2147
#12 do_command() at sql/sql_parse.cc:1501

select * from t1;

#0  row_decompress_column() at storage/innobase/row/row0mysql.cc:500
#1  row_sel_field_store_in_mysql_format_func () at storage/innobase/row/row0sel.cc:2557
#2  row_sel_field_store_in_mysql_format () at storage/innobase/include/row0sel.h:451
#3  row_sel_store_mysql_field () at storage/innobase/row/row0sel.cc:2881
#4  row_sel_store_mysql_rec() at storage/innobase/row/row0sel.cc:3021
#5  row_search_mvcc() at storage/innobase/row/row0sel.cc:5713
#6  ha_innobase::index_read () at storage/innobase/handler/ha_innodb.cc:10886
#7  ha_innobase::index_first () at storage/innobase/handler/ha_innodb.cc:11249
#8  rnd_next () at storage/innobase/handler/ha_innodb.cc:11438
#9  ha_innobase::rnd_next () at storage/innobase/handler/ha_innodb.cc:11426
#10 handler::ha_rnd_next () at sql/handler.cc:3151
#11 TableScanIterator::Read () at sql/iterators/basic_row_iterators.cc:223
#12 Query_expression::ExecuteIteratorQuery(THD*) () at sql/sql_union.cc:1771
#13 Query_expression::execute(THD*) () at sql/sql_union.cc:1824
#14 Sql_cmd_dml::execute(THD*) () at sql/sql_select.cc:578
#15 mysql_execute_command(THD*, bool) () at sql/sql_parse.cc:4974
#16 dispatch_sql_command(THD*, Parser_state*, bool) () at sql/sql_parse.cc:5631
#17 dispatch_command(THD*, COM_DATA const*, enum_server_command) () at sql/sql_parse.cc:2147
#18 do_command () at sql/sql_parse.cc:1501
```


<br>

#### Alter Column To Compressed Column (Copy DDL)

```
CREATE TABLE t1 (id INT PRIMARY KEY, b1 varchar(200) COLUMN_FORMAT COMPRESSED, b2 varchar(200));
INSERT INTO t1 VALUES (1, REPEAT('a', 200), REPEAT('a', 200));
ALTER TABLE t1 MODIFY COLUMN b2 varchar(200) COLUMN_FORMAT COMPRESSED;

这里走Copy ddl 先从原table 里面读出原有的record，再把需要修改的列压缩，之后写入一个新的table当中。这里原table有一部分列已经是压缩的了，所以读原table record的逻辑也会走到row_decompress_column。
完整的copy 逻辑在 copy_data_between_tables 函数内部。

读
#0  row_decompress_column()  at storage/innobase/row/row0mysql.cc:500
#1  row_sel_field_store_in_mysql_format_func() at storage/innobase/row/row0sel.cc:2557
#2  row_sel_field_store_in_mysql_format() at storage/innobase/include/row0sel.h:451
#3  row_sel_store_mysql_field() at storage/innobase/row/row0sel.cc:2881
#4  row_sel_store_mysql_rec() at storage/innobase/row/row0sel.cc:3021
#5  row_search_mvcc() at storage/innobase/row/row0sel.cc:5800
#6  ha_innobase::index_read () at storage/innobase/handler/ha_innodb.cc:10886
#7  ha_innobase::index_first() at storage/innobase/handler/ha_innodb.cc:11249
#8  rnd_next() at storage/innobase/handler/ha_innodb.cc:11438
#9  ha_innobase::rnd_next () at storage/innobase/handler/ha_innodb.cc:11426
#10 handler::ha_rnd_next () at sql/handler.cc:3151
#11 TableScanIterator::Read () at sql/iterators/basic_row_iterators.cc:223
#12 copy_data_between_tables () at sql/sql_table.cc:19137
#13 mysql_alter_table() at sql/sql_table.cc:18226
#14 Sql_cmd_alter_table::execute(THD*) at sql/sql_alter.cc:369
#15 mysql_execute_command(THD*, bool) at sql/sql_parse.cc:4974

写
#0  row_compress_column() at storage/innobase/row/row0mysql.cc:392
#1  row_mysql_store_col_in_innobase_format () at storage/innobase/row/row0mysql.cc:911
#2  row_mysql_convert_row_to_innobase () at storage/innobase/row/row0mysql.cc:1088
#3  row_insert_for_mysql_using_ins_graph() at storage/innobase/row/row0mysql.cc:2040
#4  ha_innobase::write_row(unsigned char*) () at storage/innobase/handler/ha_innodb.cc:9633
#5  handler::ha_write_row () at sql/handler.cc:8305
#6  copy_data_between_tables () at sql/sql_table.cc:19204
#7  mysql_alter_table() at sql/sql_table.cc:18226
#8  Sql_cmd_alter_table::execute(THD*) () at sql/sql_alter.cc:369
#9  mysql_execute_command(THD*, bool) () at sql/sql_parse.cc:4974
```


<br>

### 社区Bug

测试过程中发现一个导致OOM的bug，已经verify。具体见[https://jira.percona.com/browse/PS-8879](https://jira.percona.com/browse/PS-8879)



<br>
### Reference

[https://developer.aliyun.com/article/64891](https://developer.aliyun.com/article/64891)

[https://docs.percona.com/percona-server/8.0/compressed-columns.html#](https://docs.percona.com/percona-server/8.0/compressed-columns.html#)

[https://github.com/percona/percona-server](https://github.com/percona/percona-server)


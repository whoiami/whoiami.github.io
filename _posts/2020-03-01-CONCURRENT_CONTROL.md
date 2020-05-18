---
layout: post
title: Transaction Concurrent Control
---

<div style="text-align: center">
<img width = '800' height ='800' src ="/public/images/2020-03-01/transaction cover.png">
</div>

<br>
多数场景下，数据库（DB）是一个共享资源，用户从DB中有序获得数据资源。早期的DB，一次只能服务一个用户请求，不同客户的请求按照一定的顺序在服务端进行一系列的Transaction(Txn)事务处理，这样每个用户端的用户在其被服务的时间内独享整个DB资源。由于单个用户占用DB的行为和时间不定，会造成其他用户必须等待的情况。那么如何能让更多的用户在同一时间按照一定的顺序并发访问DB呢？这是Transaction Contrrent Control的研究范围，接下来就会讨论，如何实现高并发访问DB的同时，让每一个用户有独享DB的感觉。一切要从ACID说起。

本文是基于[Database System Concepts](https://book.douban.com/subject/4740662/) 加上自己的理解汇总而成。具体细节请感兴趣的同学自行阅读。



<br>
### ACID

Atomicity：所有Txn对数据库造成的变化只有两种状态，完全应用和完全没有应用。

Consistency：数据库整体的一致性。

Isolation： 每个Txn执行期间是不会相互影响的。每一个Txn仿佛独享数据库一样。

Durability：commit的数据就应该持久化。 

 其中，isolation是并发的头号敌人，因为如果多Txn并发并且在Txn中操作了相同的key，这几乎是不可能不相互影响的，同时最终多个Txn最后的结果还有可能影响数据库整体的Consistency。

具体到isolation levels，SQL 有如下标准：

> **Serializable** usually ensures serializable execution.
>
> **Repeatable read**  allows only committed data to be read and further requires that, between two reads of a data item by a transaction, no other transaction is allowed to update it.
> 
> **Read committed** allows only committed data to be read, but does not require repeatable reads. For instance, between two reads of a data item by the transaction, another transaction may have updated the data item and committed.
>
> **Read uncommitted** allows uncommitted data to be read. It is the lowest isolation level allowed by SQL.

这里所有的isolation levels 都不允许**dirty writes**，这基本上是isolation levels 的底线。

下面我们来总结一下，要实现上诉的isolation levels，目前可以采用的方法。



<br>
### Introduction to Current Control Method

要实现并发控制，最直观的方式就是使用锁，Txn执行过程中锁住整个DB就可以直接实现serializable了，最严格的isolation level。但是这样会极大的浪费资源，由于用户释放锁的时间不定，相当于庞大的DB资源只为一个用户使用。更一步的改进是，Txn中对要获得的资源上锁。Two-phase locking 就是这样的lock protocol。

另一种是基于timestamp的协议，read timestamp代表读过该数据的最大的时间戳，write timestamp是当前写入该数据的时间戳。如果出现冲突，通过时间戳控制每一条数据是否可以由当前 Txn 读写。

另外，目前比较流行的是Multiversion concurrency-control(**MVCC**)，单条数据同时拥有多个修改版本。读取操作只读去某一个版本数据，不会与其他 Txn造成冲突，可以有效解决读 Txn 较多的并发场景。

Predicate locking谓词锁是另外一种比较老的并发锁，与Two-phase locking 不同的是，其锁的是SQL里面的命令动作。


<br>

### **Lock-Based Protocols**


主要使用**shared-mode lock** (S)  和**exclusive-mode lock** (X)两种锁。需要对data进行读请求需要使用S，对data进行写请求需要使用X。S和X锁的兼容性关系如下：

|      |   S   |   X   |
| :--: | :---: | :---: |
|  S   | true  | false |
|  X   | false | true  |

只有shared-mode lock 和shared-mode lock 是兼容的，也就代表同一个数据的Read-Read并发是没有冲突的。

为了避免某些Txn有可能饿死（starved），我们需要使用一定的封锁协议。



#### **Two-Phase Locking Protocol**

Two-Phase Locking Protocol是一种可以实现serializability的封锁协议。执行过程如下：

>1. **Growing phase**. A transaction may obtain locks, but may not release any lock. 
>2. **Shrinking phase**. A transaction may release locks, but may not obtain any new locks. 

为了避免 Cascading rollbacks 我们可以使用 **strict two-phase locking protocol**.

为了实现 serializability 我们可以使用**rigorous two-phase locking protocol**，直到txn结束才会释放所有的锁。

Example：


<div style="text-align: center">
<img width = '800' height ='500' src ="/public/images/2020-03-01/two-phase locking.png">
</div>


#### **Graph-Based Protocols**

如果我们希望不使用two phase，我们就需要知道额外Txn的附加信息。最简单的模型是我们需要提前知道txn进入db的顺序。提供这种附加的信息，就有可能构建图基于形结构的协议，仍实现serializability。

其中 tree-locking protocol 就是这样一种协议，它是无锁的，也不需要回滚。tree-locking protocol 相比于two-phase locking 的另外一个好处是由于知道并发的顺序，所以可以提前放锁。然这种协议的缺点是在某些场景下会对自己不需要访问的数据枷锁。

具体实现是，将Txn访问的顺序，按照如下规则生成一个图。

> If *di* → *dj*, then any transaction accessing both *di* and *dj* must access *di* before accessing *dj*. 

<div style="text-align: center">
<img width = '800' height ='500' src ="/public/images/2020-03-01/Trees-space database graph.png">
</div>


> 1. The **first lock** by *Ti* may be on any data item. 
>
> 2. Subsequently, a data item *Q* can be locked by *T**i* only if the parent of *Q* is currently locked by *Ti*. 
>
> 3. Data items may be unlocked at any time. 
>
> 4. A data item that has been locked and unlocked by *Ti* cannot subsequently be relocked by *Ti*. 

<div style="text-align: center">
<img src ="/public/images/2020-03-01/Serializable schedule under the tree protocol.png">
</div>



#### **Deadlock Handling**

对于使用锁的协议来说死锁是必须要处理的一个问题。一部分协议使用Deadlock Prevention的一些方法避免造成死锁，另一部分协议使用 Deaklock Detection 和 Recovery from Deaklock 的方式来解决死锁的问题。

<br>
**Deadlock Prevention**

避免产生死锁，总体的思路分为两种，第一种是提前知道Txn需要加的所有锁，一次性加锁，第二种是每个Txn按照一定的顺序加锁。这里讨论第二种，这种方法按照时间顺序加锁，配合回滚机制确保可以防止死锁出现。具体如何保证按时间顺序加锁，这里讨论了**wait–die** 和**wait–die** 两种方法。

> 1. The **wait–die** scheme is a nonpreemptive technique. When transaction *Ti* requests a data item currently held by *Tj*, *Ti* is allowed to wait only if it has a timestamp smaller than that of *Tj* (i.e., *Ti* is older than *Tj* ). Otherwise, *Ti* is rolled back (dies). 

> 2. The **wait–die** scheme is a preemptive technique. It is a counterpart to the wait– die scheme. When transaction *Ti* requests a data item currently held by *Tj*, *Ti* is allowed to wait only if it has a timestamp larger than that of *Tj* (i.e., *Ti* is younger than *Tj*). Otherwise, *Tj* is rolled back (*Tj* is *wounded* by *Ti*). 
>
>    Returning to our example, with transactions *T*14, *T*15, and *T*16, if *T*14 requests a data item held by *T*15, then the data item will be preempted from *T*15, and *T*15 will be rolled back. If *T*16 requests a data item held by *T*15, then *T*16 will wait. 

<br>
**Deadlock Detection & Recovery from Deadlock**

死锁检测主要使用 **Wait-for graph** 来实现。

<img width = '500' height ='500' src ="/public/images/2020-03-01/Wait-for graph.png">

死锁的恢复主要有一下两种方式：

1. **Selection of a victim** to rollback

2. **Lock timeouts**



<br>
### **Timestamp-Based Protocols**

基于时间戳的协议中，Txn的时间戳时间就是实现serializability 的执行顺序。具体实现如下：

Suppose that transaction *Ti* issues read(*Q*). 

> If TS(*Ti*) *<* W-timestamp(*Q*), then *Ti* needs to read a value of *Q* that was already overwritten. Hence, the read operation is rejected, and *Ti* is rolled back. 
>
> If TS(*Ti*) ≥ W-timestamp(*Q*), then the read operation is executed, and R-timestamp(*Q*) is set to the maximum of R-timestamp(*Q*) and TS(*Ti*). 

Suppose that transaction *Ti* issues write(*Q*)

> If TS(*Ti*) *<* R-timestamp(*Q*), then the value of *Q* that *Ti* is producing was needed previously, and the system assumed that that value would never be produced. Hence, the system rejects the write operation and rolls *Ti* back. 
>
> If TS(*Ti*) *<* W-timestamp(*Q*), then *Ti* is attempting to write an obsolete value of *Q*. Hence, the system rejects this write operation and rolls *Ti* back. 
>
> Otherwise, the system executes the write operation and sets W-time- stamp(*Q*) to TS(*Ti*). 



<br>
### **Validation-Based Protocols**

1. **Read phase**.  
2. **Validation phase**. 
3. **Write phase**. 



<br>
### **Multiversion Schemes**

**MultiVersion Concurrency-Control**(MVCC) 是目前使用的最广泛的一种并发策略。每一次对同样的数据 Q 进行更新操作会创造一个新版本的Q，Txn的读操作会选取相应版本的Q进行读取，不会与其他Txn产生冲突，所以在读Txn较多的场景可以大幅度提升性能。

#### **Multiversion Timestamp Ordering**

> 1. If transaction *Ti* issues a read(*Q*), then the value returned is the content of version *Qk*. 
>
> 2. If transaction *Ti* issues write(*Q*), and if TS(*Ti*) *<* R-timestamp(*Qk*), then the system rolls back transaction *Ti*. On the other hand, if TS(*Ti*) = W- timestamp(*Qk*), the system overwrites the contents of *Qk*; Otherwise (if TS(*Ti*) *>* R-timestamp(*Qk*)), it creates a new version of *Q*. 

#### **Multiversion Two-Phase Locking**

将Txn分为**read-only transactions** 和 **update transactions**，对于read-only transactions 直接返回相应版本的数据，对于update transactions 实行rigorous two-phase locking，也就是加锁直到Txn结束才会放锁。

#### **MVCC GC**

相同数据的不同版本需要及时做垃圾回收，垃圾回收的规则如下：

> Suppose there are two versions, *Qk* and *Qj*, of a data item, and that both versions have a timestamp less than or equal to the timestamp of the oldest read-only transaction in the system. Then, the older of the two versions *Qk* and *Qj* will not be used again and it can be deleted.



<br>
### **Snapshot Isolation**

snapshot isolation 是并发控制策略最经常使用的一种方式。在Txn执行之前提供db的快照，Txn需要的所有read，write的数据都是在这个快照上发生的，Txn最终会有validation阶段，如果没有冲突才可以提交。

跟Timestamp最大的不同点是，Timestamp是基于每一条数据做检测是否合法，而snapshot isolation 是快照级别的，validation阶段检测的也是整个Txn在快照内更新是否合法。

#### **Multiversioning in Snapshot Isolation**

> *Tj* is concurrent with *Ti* if either
>
> StartTS*(*Tj) ≤ *StartTS*(*Ti*) ≤ *CommitTS*(*Tj*), or StartTS*(*Ti) ≤ *StartTS*(*Tj*) ≤ *CommitTS*(*Ti*).
>
> In validation step, for Write-write conflict, **first committer wins**. 

 需要注意的是**snapshot isolation** 并实现不了 serializability。需要额外的其他方法来保证。

<br>

### Summary

为保证ACID的性质，这里介绍了各种各样的并发控制协议。主要分为locking protocol，timestamp protocol，multiversion schemes 和 snapshot isolation。这些协议都是为了不同的用户可以相互不受影响的并发访问DB。然而，并不是所有的场景下都需要实现严格的Serializable。往往牺牲一定的隔离行等级，可以换取更好的并发性能。这也是snapshot isolation 近两年特别流行的原因。



<br>
### Reference

[Database System Concepts](https://book.douban.com/subject/4740662/)

[Designing Data-Intensive Applications](https://book.douban.com/subject/26197294/)

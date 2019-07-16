---
layout: post
title: Pika Codis Solution
---


### 基本概念

1， 支持 slot的添加删除

pkcluster addslots 0-3,8-11

pkcluster addslots 0-3,8,9,10,11

pkcluster addslots 0,2,4,6,8,10,12,14

2， 支持 slot级别的同步命令

pkcluster slotslaveof no one  0-3,8-11

pkcluster slotslaveof no one all

pkcluster slotslaveof ip port 0-3,8,9,10,11

pkcluster slotslaveof ip port 0,2,4,6 force

pkcluster slotslaveof ip port all

3， 提供集群查询命令

pkcluster info slot db0:123

pkcluster info slot 123

pkcluster info slot

如果不指定table名字，默认db0

该命令会根据slave slot的状态做全量同步和增量同步

4，codis的每一个slot 对应pika 内部的一个slot

5，关于之后分布式 default_table 概念

目前pika提供一个default_table，目前不支持table的添加删除（分布式支持），default_table 名字叫 db0，所有的slots写入都默认写入此table

6，slave slot 不提供写入服务

<br/>

### pika 兼容codis 方案

#### 1, 初始化拓扑

现有pika实例 pika1, pika3, pika2, pika4

pika1 和pika2 属于group1，pika2 是pika1 的全量备份，也就是pika2上面的slots全是pika1的slave， 监控发现pika1异常时可以让pika2执行 pkcluster slotslaveof no one all，然后切到pika2 上

同理 pika3 和pika4 属于 group2。

打算分配0-511slots 给 group1， 分配512-1023 slots 给 group2

以group1为例，group2同理

1.1，在pika1，pika2上执行pkcluster addslots 0-511

1.2，pika2 上执行pkcluster slotslave pika1_ip pika1_port 0-511

1.3，在codis分配相对应的slots给group1

<br/>

#### 2， 迁移

如果有新实例加入， 假设pika5 pika6 作为group3 加入集群。 打算迁移slot1 到pika5。

 pika5 需要预先按照1.1中所述建立slot 1

未做分配前，原始拓扑 如下图


![before](https://whoiami.github.io/public/images/images/before.png)

2.1 在pika5上通过pkcluster slotslaveof  pika1_ip  pika1_port  1 把slot1 同步到pika5上

2.2 在pika6上通过pkcluster slotslaveof pika5_ip pika5_port 1 把slot1 同步到pika6上 （保证slot1 有副本）保证2.1，2.2 的顺序，先建立pika5 和pika1 的同步关系，再建立pika6 和pika5的同步关系。因为pika5有可能会全同步，先建立pika5和pika6的关系会造成同步异常。

![middle1](https://whoiami.github.io/public/images/images/middle1.png)

2.3 在pika1 上用pkcluster info slot 1 观察pika5的lag很小的时候，修改codis meta信息，准备将slot1指向 group3(pika5)。


![middle2](https://whoiami.github.io/public/images/images/middle2.png)

2.4 观察pika1 上lag为0，pika5 做pkcluster slotslaveof no one  1。2.3 迁移过程中codis会cache来自客户端的请求，此时客户端请求不会流向pika1，之后观察pika1上对pika5的同步lag为0，在pika5上之后执行pkcluster slotsslaveof no one 1，pika1会向codis告知slot1迁移完成，slot1的meta信息才会指向group3，codis会把这段时间缓存的数据重新发送到slot1新的归属group上也就是group3。 

![after](https://whoiami.github.io/public/images/images/after.png)

2.5 用pkcluster slotslaveof no one 1, pkcluster removeslots 1清理pika1 pika2 上的slot1 数据

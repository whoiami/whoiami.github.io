---
layout: post
title: Pika Codis Solution
---


基本概念

Next Release:

1， 支持 partition级别的单独同步命令。

命令类似（还没定）partitionslaveof table partition/ partitionremoveslaveof table partition

该命令会根据slave partition的状态做全量同步和增量同步



2， 支持创建table

命令类似（还没定）createtable TableName partition_ids



3， codis的每一个slot 对应pika 内部的一个partition



pika 兼容codis 方案

1, 初始化拓扑 

现有pika实例 pika1, pika3, pika2, pika4

pika1 和pika2 属于group1，pika2 是pika1 的全备份，也就是pika2上面的partition全是pika1的slave， codis 发现pika1异常时可以让pika2 slaveof onone 切主到pika2 上

同理 pika2 和pika4 属于 group2。

打算分配0-511slots 给 pika1， 分配512-1024 slots 给 pika2

1.1，建表

在pika1 上建立table “default” 并建立partition 0-511 每个partition对应一个slot 详见基本概念3，在pika2 上建立table "default" 并建立partition 512-1024

1.2，在codis分配相对应的slot给pika1 pika2



2， 迁移

如果有新实例加入 假设pika5 pika6 作为group3 加入 打算分配slot1 给pika5

pika5 需要建表 default 建立partition1, 原始拓扑 如下图

![](https://whoiami.github.io/public/images/images/before.png)

2.1 在pika5上通过partitionslaveof default 1    pika1_ip    pika_1_port 把slot1 同步到pika5上

![](https://whoiami.github.io/public/images/images/middle1.png)

2.2 在pika6上通过partitionslaveof default 1    pika5_ip    pika_5_port 把slot1 同步到pika6上 （保证slot1 有副本）

2.3 观察pika1 上对pika5的lag很小的时候，修改codis meta信息将slot1指向 pika5

![](https://whoiami.github.io/public/images/images/middle2.png)

2.4 观察pika1 上lag为0，pika5 做partitionremoveslaveof default 1  pika1_ip    pika_1_port

![](https://whoiami.github.io/public/images/images/after.png)

2.5 清理pika1 pika2 上的partition1 数据










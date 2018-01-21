CRUSH

### PG分布平衡性测试

###### 4 * 10 * 10 拓扑

rule 0 (replicated_ruleset) num_rep 3 result size == 3:	1024/1024
  device 0:		 stored : 7	 expected : 7.68
  device 1:		 stored : 8	 expected : 7.68
  device 2:		 stored : 5	 expected : 7.68
  device 3:		 stored : 11	 expected : 7.68
  device 4:		 stored : 5	 expected : 7.68
  device 5:		 stored : 11	 expected : 7.68
  device 6:		 stored : 8	 expected : 7.68
  device 7:		 stored : 6	 expected : 7.68
  device 8:		 stored : 6	 expected : 7.68

….

 device 375:		 stored : 5	 expected : 7.68
  device 376:		 stored : 10	 expected : 7.68
  device 377:		 stored : 8	 expected : 7.68
  device 378:		 stored : 9	 expected : 7.68
  device 379:		 stored : 13	 expected : 7.68
  device 380:		 stored : 7	 expected : 7.68
  device 381:		 stored : 9	 expected : 7.68
  device 382:		 stored : 5	 expected : 7.68
​

2 * 10 * 2 拓扑
crushtool -o crushmap --build --num_osds 40  node straw 2  rack straw 10  root straw 0

  device 8:		 stored : 78	 expected : 76.8
  device 9:		 stored : 78	 expected : 76.8
  device 10:		 stored : 68	 expected : 76.8
  device 11:		 stored : 76	 expected : 76.8
  device 12:		 stored : 72	 expected : 76.8
  device 13:		 stored : 88	 expected : 76.8
  device 14:		 stored : 80	 expected : 76.8
  device 15:		 stored : 76	 expected : 76.8
  device 16:		 stored : 68	 expected : 76.8
  device 17:		 stored : 72	 expected : 76.8
  device 18:		 stored : 69	 expected : 76.8



### DPRD

### Partition初始化

由于实现的原因，本文中的Factor进行了放大，所有的Factor都乘了整个拓扑的总重量Sum_weight。

###### 测试中，所有的host和bucket都是负数ID，区别于host和bucket，node的ID是正数ID，root的ID为0。

Rule: 

Take root

Rack choose 3

Host choose 1

Node choose 1

​

Topology 4(rack) * 10(host) * 10(node)

​

root

name rack-12 factor 36 partition 9
name rack-34 factor 40 partition 10
name rack-1 factor 40 partition 10
name rack-23 factor 40 partition 10
Choose -12 type 1
Choose -34 type 1
Choose -1 type 1

rack-12

name host-17 factor 0 partition 0
name host-22 factor 40 partition 1
name host-14 factor 40 partition 1
name host-20 factor 40 partition 1
name host-19 factor 40 partition 1
name host-18 factor 40 partition 1
name host-16 factor 40 partition 1
name host-13 factor 40 partition 1
name host-15 factor 40 partition 1
name host-21 factor 40 partition 1
Choose -17 type 2

rack-34

name host-37 factor 40 partition 1
name host-36 factor 40 partition 1
name host-39 factor 40 partition 1
name host-41 factor 40 partition 1
name host-42 factor 40 partition 1
name host-40 factor 40 partition 1
name host-43 factor 40 partition 1
name host-38 factor 40 partition 1
name host-35 factor 40 partition 1
name host-44 factor 40 partition 1
Choose -37 type 2

rack-1

name host-7 factor 40 partition 1
name host-9 factor 40 partition 1
name host-3 factor 40 partition 1
name host-11 factor 40 partition 1
name host-6 factor 40 partition 1
name host-10 factor 40 partition 1
name host-8 factor 40 partition 1
name host-5 factor 40 partition 1
name host-4 factor 40 partition 1
name host-2 factor 40 partition 1
Choose -7 type 2

host-17

name node147 factor 0 partition 0
name node141 factor 0 partition 0
name node142 factor 0 partition 0
name node149 factor 0 partition 0
name node143 factor 0 partition 0
name node148 factor 0 partition 0
name node146 factor 0 partition 0
name node144 factor 0 partition 0
name node145 factor 0 partition 0
name node150 factor 0 partition 0
Choose 147 type 3

host-37

name node324 factor 0 partition 0
name node321 factor 0 partition 0
name node327 factor 0 partition 0
name node329 factor 0 partition 0
name node330 factor 0 partition 0
name node322 factor 0 partition 0
name node326 factor 0 partition 0
name node328 factor 0 partition 0
name node323 factor 0 partition 0
name node325 factor 400 partition 1
Choose 324 type 3

host-7

name node57 factor 0 partition 0
name node52 factor 0 partition 0
name node60 factor 0 partition 0
name node59 factor 0 partition 0
name node56 factor 0 partition 0
name node58 factor 0 partition 0
name node51 factor 0 partition 0
name node55 factor 0 partition 0
name node54 factor 0 partition 0
name node53 factor 400 partition 1
Choose 57 type 3



Choose 147, 324, 57





### 扩容测试

Test Reults

###### Partition 1000

4 * 12 * 10

BalancedTreeTest Basic Partition Initial Distribution

Changed partitions: 0
Moving rate: 0
Theoretical Moving rate: 0

BalancedTreeTest Add one Node
Changed partitions: 6
Moving rate: 0.002
Theoretical Moving rate: 0.002079

BalancedTreeTest Add one Host
Changed partitions: 61
Moving rate: 0.0203333
Theoretical Moving rate: 0.0204082

BalancedTreeTest Add one Rack
Changed partitions: 517
Moving rate: 0.172333
Theoretical Moving rate: 0.172414

​

5 * (18  12  10  15  16) * 10

UnbalancedTreeTest Basic Partition Initial Distribution

Changed partitions: 0
Moving rate: 0
Theoretical Moving rate: 0

UnbalancedTreeTest Add one Node
Changed partitions: 4
Moving rate: 0.00133333
Theoretical Moving rate: 0.00140647

UnbalancedTreeTest Add one Host
Changed partitions: 41
Moving rate: 0.0136667
Theoretical Moving rate: 0.0138889

UnbalancedTreeTest Add one Rack
Changed partitions: 370
Moving rate: 0.123333
Theoretical Moving rate: 0.123457



### 缩容测试

Test Result：

Partition 1000

4 * 12 * 10

BalancedTreeTest Basic Partition Initial Distribution

BalancedTreeTest Remove one Node
Changed partitions: 6
Moving rate: 0.002
Theoretical Moving rate: -0.00208333

BalancedTreeTest Remove one Host
Changed partitions: 62
Moving rate: 0.0206667
Theoretical Moving rate: -0.0208333

BalancedTreeTest Remove one Rack
Changed partitions: 750
Moving rate: 0.25
Theoretical Moving rate: -0.25

​

5 * (18  12  10  15  16) * 10


UnbalancedTreeTest Remove one Node
Changed partitions: 5
Moving rate: 0.00166667
Theoretical Moving rate: -0.00140845

UnbalancedTreeTest Remove one Host
Changed partitions: 43
Moving rate: 0.0143333
Theoretical Moving rate: -0.0140845

UnbalancedTreeTest Remove one Rack
Changed partitions: 760
Moving rate: 0.253333
Theoretical Moving rate: -0.253521

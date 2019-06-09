##Rang or Hash  this is a question



Key distribution design may usually include two general method, Range the key space or Hash the key space.

 

###1, RANGE

example ` mangodb`

Data sapce (from -infinity to +infinity) is devided into 5 chunks.

Chunk1 may store keys from [-infinity, "abc"],  Chunk2 ("abc","abcd" ] ect.

New key will be hashed. By checking meta,  it will locate one chunk this hashed key belongs to. 



Take key=12 for example. 

1, By hashing this key, a hashed key("abcabac") will locate at Chunk2. 

2, By checking meta, chunk2 belongs to shard1. 

                  x:12      x:16         x:20
           
                   |          |           |
                   |          |           |
                  
                           hash func
                 |        |        |       |          |
                 |        |        |       |          |
    -infinity                                                +infinity
              chunk1   chunk2   chunk3   chunk4     chunk5
meta: 

​	shard1 : {chunk1, chunk2}
​	shard2 : {chunk3, chunk4}
​	shard3 : {chunk5}



####Consider Split and Migrate

If new shard4 is added into this map, chunks need to be split and migrated into new shard. 

Split and migrate

1, chunk3 split into chunk3_1 and chunk3_2

  If keys are stored sequentially, split is relatively an easy work.

2, move chunk3_2 into shared4

3, change meta

4, del chunk3_2 in shared2



                      x:12      x:16         x:20
         
                       |          |           |
                       |          |           |
                  
                             hash func
                   
                  |       |         |        |           |        |
                  |       |         |        |           |        |
               chunk1   chunk2   chunk3_1  chunk3_2    chunk4   chunk5 
meta: 

  shard1 : {chunk1, chunk2}
  shard2 : {chunk3_1, chunk4}
  shard3 : {chunk5}
  shard4 : {chunk3_2}



###2, HASH
example `Pika`,  `Zeppelin`

#### Func : hash(key) % partition_num

IDEAL partition split

old partition number:      4
split partition number:    8


         0         1        2        3 
         |         |        |        |
        |  |      |  |     |  |     |  |
        0, 4      1, 5     2, 6     3, 7

Partition 0 split to partition 0 and partition 4. Under new partition number,
some keys stored in 0, need to be split from partition0 and be migrated to partition4. 

Partition split  may lead almost every  key need to moved to an new partition.  Double the partition size, mentioned above may ease this situation. 

Take partition0 for example,

Instead of rehashing and migrating every key to every other partition(usually other 7 partition), solution above just filter keys (in old partition 0) need to migrated to new partition4.

Still it is time consuming and really complexed. Usually, we dont support partition split when we design key distribution system using hash. 



###3, RESULT:

SO, Range may lead to maintaining more meta about key shard mapping. Every time you need to check this map. However, this may benefit shard split. On the other hand, Hash is EASY to implement, one hash function will replace checking key-shard map. However, this fixed hash func may really make partition limited.



Reference

Pika github

Zeppelin github

Mangodb Websit




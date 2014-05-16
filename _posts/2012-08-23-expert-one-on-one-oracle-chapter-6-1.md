---
layout: post
title: 读书笔记 《Expert One-on-One Oracle》 Chapter 6
description : ""
category :
tags: []
---

# 读书笔记 《Expert One-on-One Oracle》 Chapter 6

Chapter 6这一章讲的是table,也就是表.书中提到了多种不同的表,包括Heap Organized Table, Index Organized Table, Index Clustered Table, Hash Clustered Table, Nested Table, Temporary Table, Object Table.

  * **Terminology**

在讲表之前,书中提到了一些术语.

**High Water Mark**

首先要说明一个东西,虽然我们数据库里经常说到表,表是由一行行记录组成,但是表存在哪里?或者说表的每一行记录被存在哪里?存在block里!每个block可以存储一条至多条记录.如果讲存储记录的block"摊平",将block从左到右一字排开,那么High Water Mark指向的就是最右边存储了数据的block.往block里插入数据会导致这个Mark往右移动,但是删除表中的数据,也就是删除block中的数据不会导致这个Mark向左移动.所以,如果你要delete表中的所有数据,考虑使用TRUNCATE.这个Mark有什么用?书中只提到了扫描全表的时候用到.因为这个Mark会标记数据到哪里结束.

**FREELIST**

每个object至少有一个Freelist,这个freelist记录的high water mark下有空间的block.当一个object有可能同时被多个session修改时,可能会有多个freelist.这里的object可能指一张表.

**PCTFREE & PCTUSED**

这两种东西一般用百分比表示.PCTFREE表示一个block至少有百分之多少的剩余空间才能放在freelist里.PCTUSED表示一个当block剩余空间少于百分之多少时,该block就会被移除freelist中.

PCTUSED不能设置太高,这样block里稍微放一点数据,就会被移出freelist外,这样系统会"颠簸".

**Row Migration**

一个row要扩张,多数因为update,导致某个属性值占更多的空间.(突然想到,那个rollback segment和table放的位置有没有关系?不知道.一个row会扩张,update的时候,肯定要记录下之前的值,这些值就应该是放在rollback segment里,那和新值放在一起吗?不知道.)回到migration上.如果一个row扩张了,导致这个block放不下,那么就会发生migration.扩张的row会被移到新的block上.但是会在原来的block上留一个pointer,指向新的block里row的位置.为什么?因为如果这个table有index,就不需要重新建立index了.但这有利有也有弊,因为这样的pointer,会降低读的效率.

上面讲到FREELIST, PCTUSED.这里补充一点,一个block移出freelist,不代表它被打入冷宫,不再被用.因为row migration可能导致block的剩余空间增多,满足PCTFREE,就有可能被移回FREELIST中.

  * **Heap Organized Table**

这是最常见的一种表.一般的CREATE TABLE都是默认使用这种表.往这种结构的表中插入数据,数据会放在最先合适的地方,也就是放在最近的block空白处.所以,数据在block中存储的顺序很可能与插入的顺序不同.因为DELETE等操作会导致block中有空白空间出现,那么新插入的数据就可能放在这些空白空间上.

书中还提到,虽然CREATE TABLE是一个看似简单的操作,其实关于表的option是有很多的.但是比较重要的一般是FREELIST, PCTFREE, PCTUSED, INITRANS.

INITRANS指的是初始分配给每个block的事务槽,也就是transaction slot.如果这个值设得太低,就会影响并发.如果这个值设得太高,又会浪费空间.这又是一个trade-off了.

  * **Index Organized Table**

这种表指的就是加了索引的表.任何一个表都有一系列的属性,index organized table就是根据使用者的需要,使用一个或者多个属性建立索引,将相关的数据放在一起,方便访问.比起Heap Organized Table,数据要放在什么地方就会有一定的规定,而不是哪里有空位就放在哪里.因为index organized table中有索引作为约束.

关于Index Organized Table,书中提到两个option,一个是NOCompress/Compress,另一个是Overflow.

为了好理解,先说Compress,再说NOCompress. Compress会有一个参数N,表示压缩的属性个数.这里说的压缩是什么意思呢?举一个例子,若一个表的PK有(A,B,C)三个,Compress N=2,那么如果对于某些row,A的值都相同,B的值也都相同.那么在block中,A和B的值只记录一次,剩余的row只要记录C的值即可.这样的存储方式更节省空间,让每个block能存储更多的数据,但是如果要获得一条完整的row,就需要更多的cpu开销,因为需要找到那些被压缩的属性值"拼凑"出一条完整的记录.

而NOCompress就是不压缩,存储所有的属性值.

Overflow有两种模式,一种是PCTTHRESHOLD,另一种是INCLUDING.为什么要有Overflow?我们都希望一个block能装尽可能多的数据,而且最好是使用率高的数据.因为I/O的单位是block.block在内存停留的时间越长,I/O的次数就有可能越少.Overflow就是为了尽可能将经常被使用到的数据放在block里.PCTTHRESHOLD模式下规定,如果一个row的数据量超过了一定的大小,那么超过的部分就会被放在overflow中.而INCLUDING模式下规定,including之前,包括including的所有属性值,都存储在index block中,其余的属性值存储在overflow中.例如,某个表的属性有(A,B,C,D,E,F),INCLUDING设定为D,那么index block中只存储每个row的(A,B,C,D)的值,而(E,F)的值则存储在overflow中.所以INCLUDING模式更适合集中在前几个固定属性的访问.

  * **Index Clustered Tables**

Cluster,翻译为聚簇.书中对cluster的定义是"A cluster is a way to store a group of tables that share some common columns in the same database blocks and to store related data together on the same block".我认为重要的关键词,a group of tables, common columns, same block. 聚簇就是根据某个或者某些属性值,将表中具有相同属性值的row放在一起.

对聚簇还有一个解释我认为也很贴切,那就是"pre-join". join就是两张或以上的表根据一个或者多个属性值,进行结合.这里可能会觉得奇怪,难道一张表不能做聚簇吗?当然可以.那跟join不是要求多张表矛盾了?其实当一张表做聚簇的时候,其实无形中也"虚拟"了另一张表来做"pre-join".假如现在有一张表记录的是全校学生的信息,其中一个属性是专业.如果要求根据专业来对表中的数据进行聚类,其实不是无形中"虚拟"了一张关于专业的表来和这种总表做join吗?

cluster的出现是为了在index的基础上完成更好的查询.将相关的row放在一起,查询的时候就能够用更少的代价返回这些row所在的block.如果在heap table中,相关的row散落在不同的block中,那么就很有可能需要通过全表扫描来返回所有需要的block,代价不少的.

index cluster table更符合人类的生活习惯,因为人大部分都是群居动物,"物以类聚"也正是表示cluster.但是,人类世界里,组织家庭,这也是一种聚簇,是需要付出很多代价的,所以单身汉反而更自由自在,毕竟"一人吃饱,全家不饿"嘛.虽然cluster有的时候能让数据更易被查询,但有些时候,index clustered table也不见得好使.如果一个表是经常被修改的,尤其是导致row经常从一个簇移到另一个簇的,这样维护起来代价就会比较大.在index clustered table中,扫描全表需要花费更多的时间.因为表可能按照cluster key value被"拆散"分散在不同的block中,要拼凑出一张完整的表还是需要一点时间的.还有,drop,truncate这种操作也是不允许的.因为drop掉一张表,可能导致聚簇无法完成,或者失去了cluster key.

  * **Hash Cluster Table**
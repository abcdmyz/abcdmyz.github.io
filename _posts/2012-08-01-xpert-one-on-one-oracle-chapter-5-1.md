---
layout: post
title: 读书笔记 《Expert One-on-One Oracle》 Chapter 5
description : ""
category :
tags: []
---

# 读书笔记 《Expert One-on-One Oracle》 Chapter 5

这个chapter略长，读书笔记写了好几回,而且有一种越写越糟糕的感觉。不过，还是要坚持住写读书笔记。 

  * **Redo Log File**
为什么需要这个玩意？如果你的机器不会发生任何意外，你的操作永远不会“哎呀，不小心打错了”，那redo log file真的没用。它的出现就是为了弥补各种意外，突然断电、硬盘坏了、不小心的操作错误等之类的。 经大神提醒,log不仅仅是以上的作用,log在MVCC中会用到,根据事务开始的时间,选择合适的数据版本,这个需要用到log.还有事务执行的过程中,根据用户需要,会发生回滚,这个也需要用到log. redo log files有两种类型，一种是online，另一种是archived。online的至少有两个，假设是A和B。当online A装满以后，就会切换到online B。同时online A的数据会被转移到archived中。当online B的满了以后，同理切换。 
  * **Commit**
书中不断强调一个问题，将一个大的事务拆成几个小的事务提交不会比直接提交一个大事务要快，甚至速度会更慢。为什么？那首先看看一个事务在commit前已经做了什么。 
  1. rollback segment已经生成
  2. 数据已经修改。
  3. 根据1和2的内容，redo已生成。
  4. 1,2,3中部分的数据已经写到磁盘中（什么时候做flush，有几种可能，每隔一个时间段、累积到一定的数据、commit的时候，都会做flush）
  5. 锁已经都获得。
当commit的时候，要做的只是以下的操作了。 
  1. 生成SCN。
  2. 将剩余的redo log写到磁盘中。（根据WAL，先写log，再写数据）
  3. 释放锁。
  4. 数据块中事务留下的访问痕迹被清理掉。
因为commit之前要做的都基本完成,所以commit的时候要做的只是收拾收拾而已,所以无论事务多大或者多小,要做的活都差不多,不会因为事务越大,commit要花的时间越多.反而不断地commit会话费更多的时间. 书中给出了例子，数据的大小对inesrt等操作的时间、产生的rollback数据大小是有影响的，数据量越大，执行时间久越长，产生的rollback数据也越多。但是对commit的时间几乎没有影响。 

  * **Rollback**
上面说的是commit前后要做的，现在说说rollback做的。rollback之前，事务都在正常地执行中。当需要rollback时，要完成以下两个事情。 
  1. 从rollback segment中读取“历史数据”来还原数据。
  2. 释放锁。
  * **How much Redo Generate?**
这里讨论的是在事务执行的过程中，会有多少Redo log产生。要考虑几个因素对redo log大小的影响，分别是操作类型（包括INSERT, UPDATE, DELETE）、操作的数据大小（1 row，10 rows，100 rows...）、是否使用循环（用一个语句完成操作，称为in bulk，还是使用循环来操作每一个row，称为one row at a time）。 根据以上因素，有以下的结论： 
  1. 以上提到的三种操作都会产生redo，要操作的row越多，产生的redo也越多。
  2. 对于一条row的操作，INSERT和DELETE产生的REDO大小差不多，而UPDATE产生两倍的REDO Log。因为INSERT前和DELETE后都是没有数据的，只需要记录一份数据。而UPDATE要记录操作前和操作后的数据，数据量是INSERT和UPDATE的两倍。
  3. 关于in bulk操作和one row at a time的操作。前者会比后者产生更少的redo log。因为每次插入一条数据时，都要组织block内的数据。（表示不是很懂）
于是,可使用以下的方法来估计redo的量.

  1. 估计事务的"大小",也就是要写的数据量.
  2. 对于INSERT和DELETE操作,redo的数据量就是row大小的110%-120%.
  3. 对于UPDATE操作,redo的数据量是步骤2中数据量的两倍.

  * **Turn Off Redo Log Generation?**
NO! 这里指的"turn off",并不是完全的turn off.在书上有这么一句话"All data dictionary operations will be logged regardless of the logging mode".什么是data dictionary? 这个链接里有详细的解释. <http://docs.oracle.com/cd/B10500_01/server.920/a96524/c05dicti.htm> 用简单的话说,"数据字典"就是用来记录数据库的信息.例如,关于表的定义,表的索引,属性的类型,属性的默认值,数据库的使用者等等. 也就是说,虽然执行某个语句的时候,执行的模式是NOLOGGING, 但还是有数据会被log下来的. 能不能关闭Redo Log.其实这个问题的答案很显而易见.如果你不在乎信息的丢失,如果你敢打包票你的机器一定不会"当"掉,那这个redo log确实没什么必要了. 关于NOLOGGING以后以下两点是需要知道的: 

  1. 如果一个seciton A执行CREATE TABLE语句是使用NOLOGGING的,另一个section B对这个table进行其他操作,例如INSERT, DELETE, UPDATE,并且使用LOG模式. 对于section A而言,因为这个语句被标识为NOLOGGING,所以CREATE这个操作不会被log下来.而section B中的所有操作都会被log下来.如果这个时候机器down了,要根据硬盘和log恢复到当前的状态,此时问题出现了.因为A 建立的table没有log,所以就变成了不可恢复的了.于是B的各种操作也是白搭了.
  2. 在LOG模式下在执行完NONLOGGING的命令后,尽快将数据写回到硬盘上,以防突然机器down了,无法回复的窘况.
NOLOGGING的两个用法:

  1. 直接将"NOLOGGING"作为关键字放在SQL语句中.
  2. 令操作隐式地在NOLOGGING模式中执行.这个指将table, index之类的东西设置为NOLOGGING.这样对这些table和index的操作就是NOLOGGING的了.
那既然redo这么有用,为什么需要NOLOGGING的模式呢.因为在LOG模式下,NOLOGGING可以加快事务执行的速度.因为关于log这方面要做的活少了,自然就省时间了.例如要将一个table从一个tablespace移到另一个tablespace,可以先将这个table设置为NOLOGGING,然后移动,建立索引,再将table设置回LOG模式.

  * **Commit Clean Out**
在commit的时候提到,一个事务提交的时候,要做的其中一个工作是清除(clean out)数据块中事务留下的访问"痕迹".(其实这一部分,始终半懂半不懂) 当一个事务更新的数据块不超过缓冲区中数据块的10%,那么commit的时候,这些数据块中被访问的"痕迹"就会被清除.如果超过了10%,那就先不管了.为什么不管?书上好像没讲,个人的理解是,这样会增加了commit的工作量,还不如留着以后谁要访问了,谁再处理.不知道这样的理解对不对. 正因为超过10%不clean out这种情况,所以查询数据也会有产生redo的可能.例如,一个事务插入了大量的数据,超过了这个"10%",那么事务commit的时候数据块(data block)就没有被clean out.当这些插入的数据第一次被select的时候,因为数据块的block header没有被clean out,所以要先被clean out.clean out以后,数据块就变成"dirty"了,这样就会要产生 redo log了,还有可能导致block被再写回到disk中(如果他们之前已经写回到磁盘中). 当数据被第二次select的时候,就不会有redo log产生了. 
  * **What generate the most/least undo?**
从小到大的排列为: INSERT, UNPDATE, DELETE. 这个就很好理解了.INSERT前是没有数据的,所以产生的undo必然最少.DELETE把本来存在的数据弄走,所以产生的undo必然是最多的. 
  * **Snapshot Too Old**
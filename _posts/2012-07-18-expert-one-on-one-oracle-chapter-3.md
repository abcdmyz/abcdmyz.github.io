---
layout: post
title: 读书笔记 《Expert One-on-One Oracle》 Chapter 3
description : ""
category :
tags: []
---

# 读书笔记 《Expert One-on-One Oracle》 Chapter 3

今年5月份的时候说过,今年以内要看把这本书看完,结果,写完Chapter 1的读书报告,又停了下来.终于,又想起来该读这本书了.按照这种速度,这个计划很可能泡汤的.这本书的其中一个好处是,任何时候从任何一章读起都没问题.Chapter 3之前读过,但是没有写读书笔记,这两天重读了一下,现在要写读书笔记了. Chapter 3的题目是Locking and Concurrency.所以这个章节有两部分,一部分是Locking,一部分是Concurrency.这两个内容放在课本里讲的话,会很规范化,每个概念都会有明确的定义与例子.不过在这里,虽然没有了定义,但是结合实际应用来讲,让人觉得这些技术也没有这么复杂嘛.(不过书是2001年出版的,毕竟已经是10年前的技术了) 

  * **悲观锁与乐观锁**
锁一般会在更新数据的时候使用,因为现在的读操作都基本不会被阻塞.一般的写操作都是先query后update.如果使用悲观锁,query完了要update之前就要先申请锁.如果使用乐观锁,那就不用申请锁,直接update,后果再负. 在悲观锁种,update会有3种结果: 1\. 有其他事务已经申请了该数据的锁,这种情况下,要等待.如果使用了 FOR UPDATE NO WAIT, 那就是直接返回error,不等了. 2\. 在query和update之间,有其他事务已经更改了这个数据,所以当前的update是无效的,那么就会返回zero rows.表示没有更新成功. 3\. 如果没有以上两种情况,就会更新成功. 在乐观锁中,因为不会先申请锁,再去update数据,而是直接update,而这个后果就交给commit去负责.在commit的时候,如果只有一个事务更新了该数据,那就可以commit.否则就有以下两种处理方法: 1\. 重新执行事务 2\. 根据多个事务的update,整理出一个最终结果.(不过这个要有不少的代码量) 所以,作者更推荐的是悲观锁.其实两者的区别就在于update的时候要不要锁.要锁就要多点花销,不过commit就轻松一点,不要锁的话,就把责任全丢给commit. 

  * **Block**
想干的事干不了的时候就会被block,这些事一般指写数据,例如INSERT, UPDATE, DELETE等.书上提到了一些能避免block的方法. 对于INSERT,什么情况下会被block?当两个事务事务同时插入的目录有相同的Primary Key值时.所以,如果让系统来分配Primary Key就可能减少这种block.不过如果使用了这种方法,对于不能重复的属性加上CONSTRAIN.但是,插入的记录如果违反了约束,还能不能插入成功?还是依然会block? 对于UPDATE和DELETE,书中大力提倡使用SELECT FOR UPDATE NOWAIT,来确定数据是否能被写,如果不能,再根据实际情况修改方案. 
  * **DeadLock 死锁**
死锁的定义就不用多说了.这里提到一个实际系统中会导致死锁的重要原因unindexed foreign keys. 乍一看还没懂,书中是这么解释的.因为外键没有索引,于是当更新parent table的时候,会把整个child table都锁住.为什么要锁住整个表?因为如果要找到child table中对应的记录,就要扫描整个表,这样太耗时了.锁住整个child table会导致事务拥有太多的数据,加大了死锁的风险. 
  * **Types of Lock**
书中介绍了几种锁,主要有DML Lock和DDL Lock. DML Lock指的是Data Manipulation Language Lock,也就是执行INSERT, UPDATE, DELETE这一类操作时加的锁.它包括TX Lock和TM Lock.TX Lock是Lock数据,防止其他事务也写这些数据.TM Lock是Lock table,但不是不允许写这个表里的其他数据,而是不允许更改这个表的结构,也就是不能只行DROP, ALTER TABLE等操作. DDL Lock指的是Data Definition Lock,也就是当某个事务更改表的结构时加的锁,防治其他事务也更改这个表的结构.DDL Lock包括Exclusive DDL Lock, Share DDL Lock和Breakable Parse Lock.Exclusive DDL Lock是保证表的结构与数据都不能被更改.Share DDL Lock是不允许表的结构被更改,但允许表中的数据被更新. 
  * **并发控制**
并发控制有4个隔离等级,分别是读未提交, 读提交, 可重复读和可串行化.由于MVCC的使用,读未提交和读提交都不会出现,而且这两个隔离等级中,异常较多,大部分数据库都不会使用这两个等级. 至于可重复读是使用lock来实现.虽然能避免大部分的异常,但是由于读写互斥,会导致并发延时,还有可能导致死锁.书中提到,如果使用MVCC,能另读写不互斥,减少并发的延迟.同时使用SELECT FOR UPDATE NOWAIT,能知道要update的数据是否正在或者已经被其他事务更新,能减少block. 最后提到的等级是可串行化.书上举了一个很有趣的例子. 首先创建两张表. CREATE TABLE a ( x INT ); CREATE TABLE b ( x INT ); 有两个事务T1和T2 

T1
T2

INSERT INTO a SELECT COUNT(*) FROM b

INSERT INTO a SELECT COUNT(*) FROM b

COMMIT

 COMMIT
以上的操作如果在"可重复读"的隔离级别下,结果是a,b两张表都有新插入的行.但是按照可串行化的定义,a,b其中一张表必须是空的. 因为可串行化的定义是,并发执行的事务执行的结果等同于按照某个串行顺序执行的结果. 不过书中提到的Oracle的可串行化也不是真正的可串行化.书中是这么形容的"This is a gamble".因为最终实现的方法是,如果要update的数据被更新了,返回error.所以这个可串行化使用的条件是: 同时更新数据的可能性比较低, 要求读一致, 事务是短事务. 

  * **Read-Only Transaction**
这章的最后提到的一种事务.如果一个事务是只读事务,那么它不会被写操作阻塞.系统会根据事务的开始时间,选择将当前数据回滚到合适的时间点以供事务读取.不过这里也有可能会有异常."snapshot too old".如果在只读事务读的过程中有其他事务不断地修改数据,新的数据会把rollback segment填满,导致rollback segment会挤掉就的数据.这样只读事务就无法rollback.所以rollback segment的大小设置是一个需要注意的问题

## Comments

**[oahmoa](#14 "2012-07-18 14:54:32"):** 学习了！


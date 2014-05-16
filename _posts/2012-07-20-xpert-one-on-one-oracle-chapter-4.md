---
layout: post
title: 读书笔记 《Expert One-on-One Oracle》 Chapter 4
description : ""
category :
tags: []
---

# 读书笔记 《Expert One-on-One Oracle》 Chapter 4

Chapter 4很短,讲的是Transaction.放在课本里讲Transaction,讲的内容大概是ACID,一些异常现象,例如Dirty Read, Lost Update, Unrepeatable Read, Phantom之类的,一些隔离等级,然后是可串行化的问题.不过这里就完全不是讲这些,反而讲了一些与事务比较有关的实际例子. 一开始给了一些statement和procedure的例子,主要是说明了事务的原子性.大概的意思是,要干的活,要么都不干,要么都干完. 

  * **Constrain约束**
表里的属性是可能有约束的,例如要求值是唯一的,或者要在某个范围之内等.不过是否有想过,这个约束是在statement执行过程中检测呢?还是执行完成以后再检测呢?先看下面一个例子. CREATE TABLE t (x INT UNIQUE) INSERT INTO t VALUES(1) INSERT INTO t VALUES(2) UPDATE t SET x=x+1 如果是在执行过程中就判断约束,那么这个update就有50/50的机会会报错.因为先更新值为1的数据,就会和2冲突了,违反了唯一性约束.如果是在执行完update以后再检查约束,就不会有这样的问题出现了. 所以在实际系统中,Check Constrain可以有所设定.INITIALLY IMMEDIATE表示立即检查约束,也就是在执行过程中就检查约束.DEFERRABLE表示这个检查可以延迟,延迟到statement执行结束以后再检查约束. 
  * **Bad Transaction**
书中举了几个bad transaction的例子.其中一个是,很多人认为提交多个小事务比提交一个大事务要快得多.作者用一个实际例子说明,其实并不是这样.作者不断地重复,在Oracle里,锁和事务都不是什么scare source,并且鼓励开发者使用锁和事务.他认为,事务是在需要提交的时候才提交,而不是必须马上尽快地提交.(因为读写不互斥是他们一大优势?) 
  * **Redo and Rollback**
在书中提到,内存里会存有当前的表与索引,表的更新会记录在rollback segment中.table, index, rollback segment会构成redo log.内存的redo log每3秒就会写到磁盘中.当系统当掉时,必然有table和index的信息没有写到磁盘种,此时首先根据redo log将系统恢复到系统当掉的瞬间.由于事务还没有commit,所以一些然后再根据rollback segment将一
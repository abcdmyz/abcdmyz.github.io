---
layout: post
title: 读书笔记 《Expert One-on-One Oracle》 Chapter 1
description : ""
category :
tags: []
---

# 读书笔记 《Expert One-on-One Oracle》 Chapter 1

被推荐看<Expert One-on-One Oracle>，大牛的意思是这书是数据库菜鸟必看书籍。 糊里糊涂地看完前四章，想想，也该写点什么了，顺便给这个博客开封，否则会一直被唠叨进度太慢了。 个人认为，读书笔记，一是为了让自己对读过的内容有所梳理，二是记录下觉得重要的知识，三是提出自己的问题和观点。鉴于鄙人水平不够，先执行前两点吧。 刚拿到这本书的时候，没了解太多背景，只知道这本书很经典，作者很牛B，然后就开始看了。看完前四章的感觉是，作者眼中的Oracle数据库是世界上最好的数据库，很多技术都会让读者感觉很“惊喜”（Surprising）。多版本就是其中一个。于是我就纳闷，多版本不是很多数据库都有的吗？至少MySQL有。于是看看书籍的出版时间，2001年，哦，原来如此。 第一章的题目是“Developing Successful Oracle Applications", 所以作者在第一章中反复强调，想要用好数据库，了解数据库背后的原理非常重要，并且用大量的例子来说明这个观点。整个第一章有两个内容印象比较深刻，一个是多版本 (Multi-Version)，另一个是绑定变量 (Bind Variable)。对于多版本，作者的原话是”If you are not familiar with multi‐versioning, what you see below might be surprising.“ 2001年的时候还没有什么数据库有多版本这个技术？我不知道。说了这么多废话，以下简单地介绍一下这个surprising的多版本吧。  以下是一张表，记录银行中的账号和该账号对应的账户中的存款。 

Row
Account Number
Account Balance

1
 123
 500

2
 234
 250

3
 345
 400

4
 456
 100

有两个并发事务，T1，T2。并发的意思是T1和T2在时间上有交集，但怎么交，各种可能都有。 T1要计算所有账户的存款总数。 T2是将账户123的500块转给账户456。 如果这里没有多版本，以下的情况是有可能出现的。

  1. 使用两阶段锁 (Two Phase Lock)。那么T1事务要锁住整个表来计算总和，并且等事务完成了才释放锁，所以T2事务必须等待。也可能是T2获得锁，T1等待。这个要看RP。这种情况下，能保证正确性，但不能并发了。
  2. 使用两阶段锁，会有可能出现死锁。如果T1使用的是”读到哪锁到哪“的方法，它先锁了Row4，此时并发的T2锁了Row1，并且请求锁Row4，但是Row4已经被T1占了，所以T2只能等。占了Row4的T1读到了Row1想锁Row1，但是已经被T2占了，也只能等。这时候就出现了死锁。
而在Oracle中，如果使用了多版本，既可以使T1和T2并发，又能保证结果的正确性。 什么是多版本？简单一点的说法是，数据库会对数据维护多个版本，这里可以理解成对每个row都维护多个版本，数据根据被更新的时间划分不同的版本。当一个事务开始时，它开始的时间点就决定了它读的数据版本，一般会选择最新的数据版本。这些数据版本一定是已经提交的事务所形成的数据版本，具有一致性和持久性的 。所以其他与之并发的事务无论怎么读怎么写这些数据，对这个已经开始的事务的读操作都是没有影响的。 在Oracle里是怎么实现这个多版本的？当任何一个事务要更新数据时，redo log和rollback segment都会被添加新的内容。redo log是记录下当前的更新操作，例如记录下插入的一个数据项，或者需要删除的数据项所在的具体位置等。而rollback segment是记录旧版本的数据。当事务执行失败时，可以回滚到旧版本的数据。当事务要进行读操作时，也可以将当前的数据”回滚“到事务开始前的一个最新版本。这样任何事务的读操作都不需要经过加锁来完成，提高了事务并发的效率。 回到那个例子上。将事务T1和T2执行的过程做一个分析。 

time
T1 Query
T2 Account Transfer

Time0
Read Row 1, sum=500

Time1
Get exclusive lock on row 1, update row 1, row 1 has 100.

Time2
Read row 2, sum=750

Time3
Read row 3, sum=1150

Time4
Get exclusive lock on row 4, update row 4, row 4 has 500.

Time6
Read row 4, discover row 4 has been modified. Rollback the data to make it appear as it did at time=Time1. Read the value of row 4, it is 100.

Time7
Commit

Time8
 Sum = 1250

由于使用多版本，所以在Time6时间，事务T1就不会读T2修改后的row4，而是读回滚到Time1这个时间row 4的值。这样既保证了并发性又保证了正确性。

书中说到Oracle多版本有两大优点：Read-consistent query和No-blocking queries。前者是指查询得到的结果是有一致性的，后者是指任何查询操作都不会被阻塞。回到上面的例子，虽然row4被T2修改了，但是T1可以回滚到合适的版本，读”旧版本”的数据，保证了一致性。由于T1要读的数据版本是已经提交的事务所形成的，必然和其他事务的正在进行的写操作没关系，所以T1的读也不会被阻塞。

根据以上的例子，也就是说

最后，说一说书上提到的一个挺有趣的例子。

for x in ( select * from t ) loop insert into t values (x.username, x.user_id, x.created); end loop;

如果在多版本下，这个select操作是不会看到insert操作的结果的。 如果在非多版本下，或许这个循环是停不下来的。

Q： 要对数据维护多版本，那么每个数据岂不是要有大量的不同版本？ 读到后面的章节讲到，需要设定rollback segment的大小，如果rollback segment满了，那么太老的数据版本就会被丢掉，以腾出位置放新的版本。

好像写得很水，别喷了。

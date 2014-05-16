---
layout: post
title: bdb1.85源码学习笔记5_btree3_toolfunction
description : ""
category :
tags: []
---

# bdb1.85源码学习笔记5_btree3_toolfunction

bt_utils.c 这篇讲的是bt_utils.c里的函数,utils有使用的意思,所以这里提供的函数算是辅助函数吧,主要有两个__bt_ret和__bt_cmp.前者是根据给定的页和index获得数据,后者是比较数据. 要特别感谢以下这个博客,因为接下来bdb代码的阅读,参考了这个博客的很多内容. <http://blog.csdn.net/missyou03/article/category/713017> 在btree1这篇里说了btree的page,在btree2这篇里说了overflow page,那么这里的__bt_ret就很好理解了. 

  * **__bt_ret**
参数: 
  1. EPG *e,EPG结构体的定义就是指定页指针和index.
  2. key,data,它的注释是"user's key/data structure".而我的理解,对于非big data,如果不进行copy,那么key和data指的就是bytes这块空间,如果进行copy,那么指向的就是新的空间.对于big data,key和data指向的就是新空间.
  3. rkey,rdata,分别返回key的空间和buffer的空间,这里的空间是新创建的专门哪来放key和data的空间,而不是key和data本身所在的空间.
  4. copy,标志位,表示是否需要赋值数据,其实也就是rkey和rdata是否需要创建新的空间存放数据.
过程:

  1. 首先说明,使用_bt_ret的必须是叶子节点.所以注意bytes里放的内容,前ksize部分放的是key,后dsize部分放的是data.
  2. 使用GETBLEAF获得指定页面指定index的数据块.也就是找到linp[index]指向的数据块,获得其指针bl.
  3. 如果key是null,那么只要获取数据.也就是跳到7
  4. 这里规定,如果对于大数据,必须copy,才能保证其连续性.(因为overflow的链表中的每个page不一定是连续的).因此如果这个数据块的key是bigkey,那么就调用__ovfl_get函数来获得数据,正好返回的存放数据的空间的指针就是rkey->data.rkey->size和key->size都是存放key的大小.最后将key->data指针指向rkey->data指向的位置即可.
  5. 对于非big key,如果需要copy,那么就要根据需要分配空间.p指向新分配的空间.然后将rkey->data指向p指向的空间,rkey->size可以从bl->ksize中获得.如果rkey空间足够,那就省略分配空间这个步骤.然后直接使用memmove将数据从bl->bytes拷贝到rkey->data中,然后再给key赋值,也就是让key->data指针指向rkey->data指的地方,key->size也等于rkey->size.注意memmove的使用,src是bl->bytes,大小是bl->ksize.
  6. 如果非big key不需要复制,那就非常简单了,直接给key的size和data复制,也就是bl中ksize和bytes的值.注意,这里的值指的是指针,也就是地址值,而不是key值.
  7. 接下来是data的处理,其实与key基本一致,也是先处理大数据的拷贝,然后再处理非大数据的拷贝,最后处理非大数据不拷贝的情况.这里注意的是memmove的使用,src是bl->bytes+bl->ksize.前面已经提到了bytes的存储,前面存的是key,后面存的是data.所以这里的src要移动ksize的长度.
  8. 关于+1,在给rdata分配空间时,为什么bl->dsize要加1.我的理解,因为如果这里是没有data的,那么data的长度是0,但是不能申请长度为0的空间,所以这里就+1了.

  * **__bt_cmp**

用来对key进行比较的函数. 参数: 

  1. k1,要比较的key.
  2. EPG* e,另外一个要比较的key,我们成为k2,只是指定了页指针和index,所以要先把key提取出来
过程: 
  1. 获得e指定的页指针,h.
  2. 如果要比较的k2如果是在页的第一个位置,那么它一定是比后面的key都小,所以k1一定大于k2,因此返回1.为什么会这样?讲到后面的search的时候就会说明,这里暂且放着.
  3. 接下来就是获取h中指定index的key.这里分了两种情况讨论,如果是中间页,也就是INTERNAL的情况,另一种是叶子页,也就是LEAF的情况.根据不同的情况调用相应的宏定义(GETLEAF/GETINTERNAL)获得数据块.如果key不是big key,那么就好办了,可以直接给k2的data和size赋值.
  4. 如果key是bigkey,那么就先获得bl->bytes的内容,因为这个内容包括了overflow page链表的起始页号和页大小.然后调用__ovfl_get函数获得key.最后让k2的data指向返回的data.
  5. 调用btree本身的比较函数比较k1和k2.

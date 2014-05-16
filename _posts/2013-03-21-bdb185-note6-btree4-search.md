---
layout: post
title: bdb1.85源码学习笔记6_btree4_search
description : ""
category :
tags: []
---

# bdb1.85源码学习笔记6_btree4_search

终于开始写btree的最重要的部分了,search,put,delete,3大操作.先从search开始写,因为search算是最简单,并且put和delete中也被经常用到. 要特别感谢以下这个博客,因为接下来bdb代码的阅读,参考了这个博客的很多内容. <http://blog.csdn.net/missyou03/article/category/713017> 在说search之前,先再说说BLEAF和BINTERNAL [c] typedef struct _binternal { u_int32_t ksize; /* key size */ pgno_t pgno; /* page number stored on */ #define P_BIGDATA 0x01 /* overflow data */ #define P_BIGKEY 0x02 /* overflow key */ u_char flags; char bytes[1]; /* data */ } BINTERNAL; [/c] 看BINTERNAL里有一个很重要的变量,pgno.这个是什么作用?记录BINTERNAL所在的页编号?一个页内会有多个BINTERNAL,这些BINTERNAL都要记录下页编号?重点是,记下来好像没什么作用?!那这个pgno是干什么用的? ![](/wp-content/uploads/2013/03/search_key_1-300x146.png) 先看看上面的图,这是通常情况下,btree的其中一棵子树,假设key0,key1,key2都一个页面内,page0-page3是leaf page,里面存储就是key&data.page0存的data的key小于key0,page1存的data的key大于等于key0,小于key1,一次类推.那怎么将这棵子树存储起来?如果每个key0对应的就是一个BINTERNAL,但是,BINTERNAL怎么能指向两个page呢? 如果把图换成这样. ![](http://abcdmyz.me/wp-content/uploads/2013/03/search_key_2-300x128.png) 这样每个KEY都存放在一个INTERNAL里,并且每个INTERNAL可以指向一个LEAF,这里用什么指向?就是PGNO了!但是,问题是小于KEY0的数据放在哪里?在BDB中,这里有所改变.在INTERNAL中,每层最左边的页的最左边的KEY都认为是最小的KEY,任何一个KEY都会大于这个KEY.这样每个KEY指向的PAGE上的KEY都有满足以下条件:PAGE上的KEY都大于等于指向它的INTERNAL中的KEY,但又小于旁边INTERNAL的KEY.(有点绕口) 这样就能解释BINTERNAL中,pago的作用了.再看看BLEAF,它是不需要pgno的.因为它没有儿子节点. [c] typedef struct _bleaf { u_int32_t ksize; /* size of key */ u_int32_t dsize; /* size of data */ u_char flags; /* P_BIGDATA, P_BIGKEY */ char bytes[1]; /* data */ } BLEAF; [/c] 

  * **__bt_search**
查找函数,传入要查找的key,返回key所在的页面. 其实search的想起来也不难,就是从root开始,找到合适的key,然后继续从这个key指向的页面开始往下查找,一直找到叶子节点为止.这里的实现也差不多.只是在一个页面内查找的时候用的是二分(这就说明,页内的KEY是有序的),并且还会查找叶子节点的前一个页和后一个页. 参数:key,要查找的key; exactp,1表示查成功,0表示查找失败. 返回值: EPG*,记录查找到的data所在的页面指针,以及index. 过程: 
  1. 首先清空btree的stack. btree的stack用作记录查找的路径.stack在put和delete中发挥非常重要的作用.
  2. 从root开始查找,首先通过mpool_get获得指定页号的页面.
  3. 然后在此页面中进行二分查找,如果这个页面是BINTERNAL,那么就要找比key小的最大值,如果这个页面是BLEAF,那么就要找比key大的最小值.(结合上面BINTERNAL的使用,自己写点数据就很容易理解了).但是这个二分查找找到的都是比key大的最小值.
  4. 这里说说二分查找.我经常写的二分都是mid=(head+tail)/2的那种,这里有所不同.base是起点,lim表示当前有多少key,因此lim每循环完一次都会除以2.开始查找:index=base+(lim>>1),其实index就是取中间点.调用__bt_cmp比较参数中的key和index指向的key,如果相等,且当前的页是LEAF,那就说明找到了,可以返回.如果当前的页是BINTERNAL,那就将该页放入栈中和还给mpool,然后获取这个key指向的页继续查找.
  5. 若cmp的结果是不等的,那么分两种情况,如果key比中间值大,那么base=index+1,也就是改变查找区间的其实位置,查找后半段区间,并且lim-1.如果key比中间值小,那么就继续循环即可.因为其实位置不用变,lim会自动除以2.这里我纳闷很久的事,为什么大于的时候lim要减1?写了2组数据,分别有10个数,和9个数的,然后模拟了二分查找的过程,就能懂了.一个要点是,(偶数/2)!=((偶数-1)/2),(奇数/2)==((奇数-1)/2).
  6. 二分查找查找完毕后,如果一直查找到lim=0,说明没有找到相等的key.这里分两种情况讨论.如果当前的页面是BLEAF,那么跳转至7,如果是BINTERNAL,跳转是8.
  7. 如果它允许有重复建,且查找到最后是查找到了边界(最左或者最右),那么就要看看当前页左边和右边的页是否有合适的key.如果不允许重复建,那么就可以返回找不到了.
  8. 如果当前页是BINTERNAL的,上面说了,二分查找找到的都是比key大的最小值,而对于BINTERNAL来说,需要的是比key小的最大值,因此这里要减1.然后获得key指向的页面,将次页面放入栈还中,然后继续循环查找.
  * **__bt_snext**
比较指定page的下一个页面的第一个key与传入的key是否相等. 
  * **__bt_sprev**
比较指定page的上一个页面的最后一个key与传入的key是否相等.
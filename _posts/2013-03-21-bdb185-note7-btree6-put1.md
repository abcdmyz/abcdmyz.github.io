---
layout: post
title: bdb1.85源码学习笔记6_btree5_put
description : ""
category :
tags: []
---

# bdb1.85源码学习笔记6_btree5_put

put这个操作感觉是btree中最重要的,也是最麻烦,略难懂的.如果说简述put操作,可以说,找到data要放的页面,把data放入页面中,如果页面不够放,分裂页面,挤一个key到父节点中,然后一直往上到不用分裂或者到了root为止.这么说着好像挺简单的,但是,一看代码,好多好多细节呀. 要特别感谢以下这个博客,因为接下来bdb代码的阅读,参考了这个博客的很多内容. <http://blog.csdn.net/missyou03/article/category/713017> 先给两个图说说分别前后的子树有什么变化.假定该btree每个页面能5个key. ![](/wp-content/uploads/2013/03/btree1-300x150.png) 这是当前的子树,绿色部分代表INTERNAL,蓝色部分代表LEAF,现插入data=20 ![](http://abcdmyz.me/wp-content/uploads/2013/03/btree2-300x149.png) 确定要插入的,插入后,发现页面超出大小,于是分裂出另一个页面,即图中黄色的页面. ![](http://abcdmyz.me/wp-content/uploads/2013/03/btree3-300x139.png) 然后将黄色页面对应的key,20插入到其父节点中. 整个过程看起来也不是很复杂,但是实现起来很多细节都可能会被忽略或者想得不够清楚,那么很可能会出错了.在讲代码之前,先给出一个put操作的流程图.里面包括了put操作调用了什么函数,还有实现的一些大概的步骤. **bt_put.c**

  * **bt_put**
参数: key,data:要插入的key和data; flags:标志位. 过程: 
  1. cursor部分不懂,暂时被忽略.
  2. 处理key和data是bigkey和bigdata的情况,因为它们不能直接放入页中,要把它们变成链表,页中仅存储链表的首页编号.调用__ovfl_put将处理大数据,并返回首页编号pg.令tkey指向一个指定大小的空间kb,然后将页编号以及数据大小都复制到这个空间中(使用memmove),并且设置标志位BIGKEY.最后令key等于tkey的地址.data的处理方式相同.有意思的地方,如果这样处理一次后,key和data仍然是big key和big data,继续用同样的方法处理.
  3. 又有一个cursor处理,暂时忽略.
  4. 使用bt_fast和bt_search查找key/data应该放在哪个页的哪个index上,返回结果赋值给h和index.
  5. 做一些标志位的判断.如果设定的是不覆盖,但找到了,只能返回,但找不到,说明可以插入.如果设定的是覆盖,但没有找到又或者设定的key可重复,那么也是可以插入的.但找到,且设定key不能重复,那么就删原来那个,再插入.
  6. 获得要插入的key和data的大小nbytes,如果判断叶子页放不下,调用__bt_split分类页面,再在返回的页面h上插入数据.(同时还会返回index,指定插入位置)
  7. 将数据插入到页中,已经知道要插入到页中的什么位置(index),在前面提到,一个页中的数据是有顺序的,所以插入一个数据以后仍然要保证顺序.页中linp[]是保持有序的,但是其指向的数据不一定是有序的.(详见btree1_page)因此确定了index,就要移动linp[],腾出一个空位给新插入的key了.因此这里判断index是否是最后一个,如果不是,那么就移动linp[]吧.使用memmove,把下标为index后面的linp[]都往后移动一个位置.然后linp[index]就可以指向新插入的数据的起始位置了.dest也是指向新数据的起始位置,使用WR_BLEAF写入数据.
  8. 剩下的cursor又不懂了.

最后给个图说明7的情况,就更一目了然了. 以下是未插入数据前,页的情况. ![](/wp-content/uploads/2013/03/btree_insert1-300x198.png) 插入key=50,data=100的过程.(不需要分裂) 

![](/wp-content/uploads/2013/03/btree_insert2-300x192.png)![](http://abcdmyz.me/wp-content/uploads/2013/03/btree_insert3-300x197.png)

**bt_split.c**

  * **bt_page**
在写__bt_splict之前先写bt_page和bt_root.它们都是在为要分裂的页分配left和right page空间.在bdb中,如果一个页要分裂,不适把这个页一半的数据拷贝到新的页中,而是申请2个新的页(left和right),分别把原页中的数据分别拷到新的页中,然后再把原来指向原页的指针指向left页.为什么这么做?如果用方法一,那么原页就会因为数据被拷走,产生很多空隙,然后还要合并这些空隙,更麻烦.于是就采用了方法二.而这个函数的作用就是分裂非ROOT的页面,其实就是分配left和right页面空间然后设置一下该设置的. 参数: 
  1. h,原页,也就是要被分配的页.
  2. lp,rp,就是新申请的left和right页.
  3. skip,数据插入的index
  4. ilen,数据大小
过程:

  1. 申请右页r.设置r的各种变量.
  2. 接着不是马上申请左页,这里有一个trick.判断如果原页没有next page,并且要插入的index又是原页的最后一个,那么直接把数据插入到刚申请的右页就可以了.这样更省功夫.其实到了后面,原页还是要分裂的,只是这里不分裂,偷了一下懒.于是就把原页h的nextpg设为r的页号,然后指定插入的位置在*skip=0,给*lp和*rp分别赋值,然后返回r.表示在r的第0个位置插入数据.
  3. 如果没有这个特殊情况,就继续申请left page.
  4. 然后调用bt_psplit来真正分裂页面.注意调用时传入的参数是(l,r).lp和rp其实只是作为返回值用的.
  5. bt_psplit执行完后,也就是原页的数据copy到了left和right page中了,其返回值是个PAGE*,表示新数据应该插入到哪个页面(left or right),以及skip,表示插入到什么位置.
  6. tp作为bt_page的返回值,意义也是表示应该插入到哪个页面.执行完bt_psplit后,使用memmove将l的数据移回到h中,然后free掉l.注意最后*lp和*rp的赋值,left page其实是原页,而right page就是新申请的页.
附3个图说明h,l,r page的prevpg和nextpg是怎么变化的. 执行bt_page前 

![](/wp-content/uploads/2013/03/page_np_1-300x154.png)

执行bt_page中,bt_psplit前 ![](/wp-content/uploads/2013/03/page_np_2-300x234.png) 执行完bt_psplit后

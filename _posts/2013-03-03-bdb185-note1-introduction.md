---
layout: post
title: bdb1.85源码学习笔记1_概述
description : ""
category :
tags: []
---

# bdb1.85源码学习笔记1_概述

开始看berkeleydb,从1.85版下手,应该是能下载到的最旧的版本了,1992年的.比较了1.85版和2.77版的,代码大小差别1倍.1.85的压缩包是512K,而2.77版的是1.1M,对于我这种菜鸟,还是从最简单的开始吧.事实证明,这也是正确的.想当初,企图说想看mysql的代码,试着看了postgreSQL关于MVCC的小段代码,都说明了一个问题,我根本还没有能力"驾驭"这么多的代码,还是从最简单的开始吧. 解压1.85版的代码,文件目录其实挺清晰的. /PORT里面是各个不同平台的makefile. /include包括了一些头文件,db.h, mpool.h /db里有db.c,有部分db.h内函数的实现. /mpool里有mpool.c,主要是关于内存管理的函数实现. /hash, /btree, /recno 这三个文件夹内分别是3种存储方式的实现. /test里提供了使用bdb的样例. 

  * **编译**
在/PORT里挑一个合适平台makefile来执行.这里挑了/linux,通用平台.90年代的代码,现在编译起来,有问题是很正常的.主要问题有以下几个: 
  1.  hash.h,结构体HTAB内,有一个变量int errno,与errno.h里的宏定义重名了,执行的时候变量errno就会被这个宏定义替换.所以把int errno重命名,hash.c调用其的地方也要相应修改.
  2. 修改/PORT/Linux下的makefile,loader已经过时,所以可以把loader和后面的sort都去掉.
修改完以上的内容,就成功编译了. 要从哪里入手开始看?从/test开始吧.

  * **test**
/test内包括了3个测试,包括hash,btree的测试,还有一个是dbtest.dbtest可以看作是一个bdb使用的简单教程.使用dbtest前,也是要编译的,makefile是已经有的,只要指定PORTDIR的路径即可,这里的路径是 ../PORT/Linux.执行make,就可生成dbtest的可执行文件.输入合适的命令,就可以完成读写数据的操作.至于输入什么命令,稍微读一读dbtest.c里的case switch就知道输入的命令格式. 下面稍微分析一下dbtest就大概知道这份源码的基本架构. [c] DB *dbp = dbopen(fname, oflags, S_IRUSR | S_IWUSR, type, infop); [/c] 这里定义了一个只想DB类型的指针,DB就是一个数据库类型,通过调用dbopen函数,返回一个数据库的句柄.dbopen函数中,fname是数据库存放的文件;oflags是一些标志位,例如DB_LOCK等; S_IRUSR等是规定数据库的访问权限; type是指定数据库存储的方式,btree,hash还是recno; infop是存储方式的参数设置,例如使用btree,要设置页大小,页数量等. [c] dbp->get(dbp, kp, &data, flags); dbp->put(dbp, kp, dp, flags); dbp->del(dbp, kp, flags); [/c] 接着就可以通过调用各种函数来进行数据的读写. 
  * **db.h**
上面看到了DB这个类型,接下来就看看这个类型的具体定义,我个人认为它是bdb的开端. [c] typedef enum { DB_BTREE, DB_HASH, DB_RECNO } DBTYPE; typedef struct __db { DBTYPE type; /* Underlying db type. */ int (*close) __P((struct __db *)); int (*del) __P((const struct __db *, const DBT *, u_int)); int (*get) __P((const struct __db *, const DBT *, DBT *, u_int)); int (*put) __P((const struct __db *, DBT *, const DBT *, u_int)); int (*seq) __P((const struct __db *, DBT *, DBT *, u_int)); int (*sync) __P((const struct __db *, u_int)); void *internal; /* Access method private. */ int (*fd) __P((const struct __db *)); } DB; [/c] type,是个枚举类型,定义了这个数据库的存储方式.其实到现在为止都没有提到"表"这个东西,在bdb的源码里也没出现过这个"table"这个字样.在我的理解,对于bdb,每一个文件就是一个数据库,并且一个数据库在建立的时候指定了存储方式后,这个存储方式就不能再改变了.如果使用dbopen打开一个已建立的数据库,并且指定不同的type,就会返回出错. close, del, get, put, seq, sync是一系列的函数指针,用户可以通过调用这些函数来实现对指定数据库的读写操作. internal,暂时没弄懂具体做什么. fd, 全称是file descriptor,其实就是描述存储方式的一些相关设置的. dh.h里还声明了dbopen函数和3种存储方式的相关open函数. [c] DB *dbopen __P((const char *, int, int, DBTYPE, const void *)); DB *__bt_open __P((const char *, int, int, const BTREEINFO *, int)); DB *__hash_open __P((const char *, int, int, const HASHINFO *, int)); DB *__rec_open __P((const char *, int, int, const RECNOINFO *, int)); [/c] 接下来说说这些open是怎样的联系起来的. 
  * **db.c**
[c] dbopen(fname, flags, mode, type, openinfo) const char *fname; int flags, mode; DBTYPE type; const void *openinfo; { /* ... */ switch (type) { case DB_BTREE: return (__bt_open(fname, flags & USE_OPEN_FLAGS, mode, openinfo, flags & DB_FLAGS)); case DB_HASH: return (__hash_open(fname, flags & USE_OPEN_FLAGS, mode, openinfo, flags & DB_FLAGS)); case DB_RECNO: return (__rec_open(fname, flags & USE_OPEN_FLAGS, mode, openinfo, flags & DB_FLAGS)); } /* ... */ } [/c] db.c里有dbopen的定义,只列出最重要的部分.这里可以看出通过对type的判断,调用相应的open函数,并返回相应的句柄. 下一个问题是,对于用户来说,无论他们用什么存储方式打开数据库,他们都能用get, put等函数来进行读写.(恕我太水,看到以下的实现,才知道原来有如此好玩神奇的东西) 
  * **bt_open.c**
这里不是要讲btree的实现,只是把bt_open.c的一个片段拿出来,就能回答上面的问题了. [c] DB * __bt_open(fname, flags, mode, openinfo, dflags) const char *fname; int flags, mode, dflags; const BTREEINFO *openinfo; { /* ... */ dbp->type = DB_BTREE; dbp->internal = t; dbp->close = __bt_close; dbp->del = __bt_delete; dbp->fd = __bt_fd; dbp->get = __bt_get; dbp->put = __bt_put; dbp->seq = __bt_seq; dbp->sync = __bt_sync; /* ... */ return (dbp); } [/c] 上面提到结构体DB里定义了各种函数指针,那哪里给这些函数指针赋值?就是这里了!各个存储方式的open函数里,会将各自存储方式对这些函数的实现赋值给DB里的函数指针,这样对于用户,无论他用什么存储方式,调用的函数都是一样的. 看到这里,我想到的就是,这不就是多态么?恕我太没见识了,才略略体会到面向对象是一种思想呀,哪怕c不是一种面向对象语言,也能体现面向对象.
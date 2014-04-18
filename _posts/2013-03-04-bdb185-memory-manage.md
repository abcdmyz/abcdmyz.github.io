---
layout: post
title: "bdb1.85源码学习笔记2_内存管理"
description: ""
category:
tags: []
---

# bdb1.85源码学习笔记2_内存管理

记得以前做个一个课程设计,外排的实现,对tpc-h lineitem这个表1个scale的数据进行排序,大概600万条记录700多M的大小.当时的要求只能只用8个页,每个页大小为8K.那时候还没有"页管理"这个概念,就只知道能用的数组大小不能超过8*8K.看完bdb这部分的代码,算是对内存管理有一个更清晰的概念. 要特别感谢以下这个博客,因为接下来bdb代码的阅读,参考了这个博客的很多内容. <http://blog.csdn.net/missyou03/article/category/713017>

  * **概述**
bdb的内存管理,使用的是内存池.每一个数据库文件都会分配一个相应的内存池.内存池内有一定数量的页,这些页使用两个循环队列来存储,一个循环队列称为hash队列,一个是lru队列.(lru就是least recently used,最近最少使用).内存池提供了new, get, put, write, sync等操作. 
  * **mpool.h**
[c] /* The BKT structures are the elements of the queues. */ typedef struct _bkt { CIRCLEQ_ENTRY(_bkt) hq; /* hash queue */ CIRCLEQ_ENTRY(_bkt) q; /* lru queue */ void *page; /* page */ pgno_t pgno; /* page number */ #define MPOOL_DIRTY 0x01 /* page needs to be written */ #define MPOOL_PINNED 0x02 /* page is pinned into memory */ u_int8_t flags; /* flags */ } BKT; [/c] [c] #define CIRCLEQ_ENTRY(type) struct { struct type *cqe_next; /* next element */ struct type *cqe_prev; /* previous element */ } [/c]  结构体BKT,定义的是队列中的一个元素.也就是一个页. hq和q定义的是该页在这两个队列中的位置,查看CIRCLEQ_ENTRY的宏定义可知道,记录的是该页在队列中双向指针.这样就可以理解成,对于每个页,都会有4个指针,2个是在hash队列中,2个是在lru队列中. page,指向具体页面的指针.所以这里的BKT,记录的只是页的信息,而没有页的内容. pgno,页号. flags,记录页是否在内存池内,也就是pinned了,还有页是否被写,也就是dirty了. [c] #define HASHSIZE 128 #define HASHKEY(pgno) ((pgno - 1) % HASHSIZE) typedef struct MPOOL { CIRCLEQ_HEAD(_lqh, _bkt) lqh; /* lru queue head */ /* hash queue array */ CIRCLEQ_HEAD(_hqh, _bkt) hqh[HASHSIZE]; pgno_t curcache; /* current number of cached pages */ pgno_t maxcache; /* max number of cached pages */ pgno_t npages; /* number of pages in the file */ u_long pagesize; /* file page size */ int fd; /* file descriptor */ /* page in conversion routine */ void (*pgin) __P((void *, pgno_t, void *)); /* page out conversion routine */ void (*pgout) __P((void *, pgno_t, void *)); void *pgcookie; /* cookie for page in/out routines */ #ifdef STATISTICS u_long cachehit; u_long cachemiss; u_long pagealloc; u_long pageflush; u_long pageget; u_long pagenew; u_long pageput; u_long pageread; u_long pagewrite; #endif } MPOOL; [/c] [c] #define CIRCLEQ_HEAD(name, type) struct name { struct type *cqh_first; /* first element */ struct type *cqh_last; /* last element */ } [/c] 结构体MPOOL,定义了内存池内的两个队列,hash和lru. lqh(lru queue head), hqh(hash queue head),定义的是两个队列的首元素.查看CIRCLEQ_HEAD的定义可知.hqh是一个数组,可以理解成hash的桶,通过hash函数,确定页号对应的哈希值,也就是数组的下标,然后将该页放入对应的队列中. curcache, maxcache, npages, pagesize, fd,这些变量看注释都可以知道其意思了. pgin, pgout, pgcookie,这些我只知道是和文件读写有关的,大概的意思是,内存中的内容写入文件,从文件读入数据到内存,由于大小端的原因,会导致格式有差别,所以要有所调整.但是,这种问题一般发生在不同机器的通信当中,因为不同的机器可能大小端不同,所以导致格式不同.不知道这样的理解对不对. STATISTICS里就是各种数据的统计,如果想要统计,可以在makefile里加上这个宏定义. mpool.h最后剩下的就是各种内存池相关函数的声明.接下来就讲讲mpool.c里的这些函数吧.  
  * **mpool_new**
[c] void * mpool_new(mp, pgnoaddr) MPOOL *mp; pgno_t *pgnoaddr; { struct _hqh *head; BKT *bp; if (mp->npages == MAX_PAGE_NUMBER) { (void)fprintf(stderr, "mpool_new: page allocation overflow.\n"); abort(); } #ifdef STATISTICS ++mp->pagenew; #endif /* * Get a BKT from the cache. Assign a new page number, attach * it to the head of the hash chain, the tail of the lru chain, * and return. */ if ((bp = mpool_bkt(mp)) == NULL) return (NULL); *pgnoaddr = bp->pgno = mp->npages++; bp->flags = MPOOL_PINNED; head = &mp->hqh[HASHKEY(bp->pgno)]; CIRCLEQ_INSERT_HEAD(head, bp, hq); CIRCLEQ_INSERT_TAIL(&mp->lqh, bp, q); return (bp->page); } [/c] mpool_new是获取一个新的页面. 参数: mp,内存池指针. pgnoaddr,用作返回新页的页号. 返回值: 指向页内容的指针. 实现过程: 
  1. 判断目前内存池的页数是否到达总的页数.内存池的页数包括写入文件中的和cache里的,也就是总共分配的页数.如果到达总的页数,那就不能再申请了,就返回错误信息.
  2. 通过调用mpool_bkt获取新页面,页面信息指针返回给bp.
  3. 页面总数加1,同时将页号赋值给bp->pgno和*pgnoaddr.
  4. 将bp标志为pinned,既该页在cache中.
  5. 使用HASHKEY(bp->pgno)计算页号的哈希值,然后获得哈希值对应下标的队列的头指针.
  6. 将该页放入hash队列的首部和lru队列的尾部.放入hash队列的首部是因为,最近在使用的队列再次被使用的机会会更大,所以放在首部方便查找.而放在lru的尾部,是因为它是最新的页.
  7. 最后,返回页内容的指针.

  * **mpool_btk**
[c] static BKT * mpool_bkt(mp) MPOOL *mp; { struct _hqh *head; BKT *bp; /* If under the max cached, always create a new page. */ if (mp->curcache < mp->maxcache) goto new; /* * If the cache is max'd out, walk the lru list for a buffer we * can flush. If we find one, write it (if necessary) and take it * off any lists. If we don't find anything we grow the cache anyway. * The cache never shrinks. */ for (bp = mp->lqh.cqh_first; bp != (void *)&mp->lqh; bp = bp->q.cqe_next) if (!(bp->flags & MPOOL_PINNED)) { /* Flush if dirty. */ if (bp->flags & MPOOL_DIRTY && mpool_write(mp, bp) == RET_ERROR) return (NULL); #ifdef STATISTICS ++mp->pageflush; #endif /* Remove from the hash and lru queues. */ head = &mp->hqh[HASHKEY(bp->pgno)]; CIRCLEQ_REMOVE(head, bp, hq); CIRCLEQ_REMOVE(&mp->lqh, bp, q); #ifdef DEBUG { void *spage; spage = bp->page; memset(bp, 0xff, sizeof(BKT) + mp->pagesize); bp->page = spage; } #endif return (bp); } new: if ((bp = (BKT *)malloc(sizeof(BKT) + mp->pagesize)) == NULL) return (NULL); #ifdef STATISTICS ++mp->pagealloc; #endif #if defined(DEBUG) || defined(PURIFY) memset(bp, 0xff, sizeof(BKT) + mp->pagesize); #endif bp->page = (char *)bp + sizeof(BKT); ++mp->curcache; return (bp); } /* [/c] mpool_new创建一个新的页面,如果没有到达cache内页面的最大值,直接new页面,如果到达了,就将一个页面的内容写回文件中,然后返回该页面. 参数: mp,内存池指针. 返回值: 新分配页面的信息的指针,也就是BKT的结构体. 执行过程: 
  1. 判断cache里的页面是否到达最大值,如果没有到达最大值,可以直接new新页面.
  2. 如果已经超出最大值,那么就从cache里挑选一个页面写回文件,然后返回该页面.

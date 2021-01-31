---
title: Redis垂死笔记
date: 2020-02-29 21:42:42
tags: "C"
cover: /img/IMG_20200307_000726_822.jpg
---
没用过redis，但是去年闲着无聊翻了一遍源码，感觉就是一大坨数据结构合起来，还是非常naive的数据结构，就没怎么往脑子里去，没想到面试都在问。慌张.jpg。 决定先记个笔记把这些记下来。

**本文正在重写中**

## 从`server.h`开始

### 依赖

`server.h`中包含了这些头文件

```c
#include "ae.h"      /* Event driven programming library 应该是async event?*/
#include "sds.h"     /* Dynamic safe strings 为什么不是 safe dynamic string */
#include "dict.h"    /* Hash tables */
#include "adlist.h"  /* Linked lists */
#include "zmalloc.h" /* total memory usage aware version of malloc/free */
#include "anet.h"    /* Networking the easy way */
#include "ziplist.h" /* Compact list data structure */
#include "intset.h"  /* Compact integer set structure */
#include "version.h" /* Version macro */
#include "util.h"    /* Misc functions useful in many places */
#include "latency.h" /* Latency monitor API */
#include "sparkline.h" /* ASCII graphs API */
#include "quicklist.h"  /* Lists are encoded as linked lists of
                           N-elements flat arrays */
#include "rax.h"     /* Radix tree */
#include "connection.h" /* Connection abstraction */

#define REDISMODULE_CORE 1
#include "redismodule.h"    /* Redis modules API defines. */

/* Following includes allow test functions to be called from Redis main() */
#include "zipmap.h"
#include "sha1.h"
#include "endianconv.h"
#include "crc64.h"
```

接下来会有大量的宏定义，略过

module相关的因为不太熟悉，暂且不表

### 结构体

相对重要的`redisObject`定义

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4; 
    unsigned lru:LRU_BITS; /* 24
    						* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

字段不多，`type`指明对象表示上的类别，而`encoding`表记如`ziplist`之类的编码方式，`lru`见注释，使用了引用计数，最后用指针指向本体。

~~这里喷一下C系的神奇语法，下一行有这个东西`char *getObjectTypeName(robj*);`我这边还迟疑了一下，没能瞬间分清是定义并初始化类型为`char *`的变量`getObjectTypeName`还是声明一个函数~~

接着创建了客户端返回块~~为啥不上string~~

```c
typedef struct clientReplyBlock {
    size_t size, used;
    char buf[];
} clientReplyBlock;
```

接着是重要的`database`表示

```c
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
    unsigned long expires_cursor; /* Cursor of the active expire cycle. */
    list *defrag_later;         /* List of key names to attempt to defrag one by one, gradually. */
} redisDb;
```

包含着好几个键值对，到时候再展开看。

接下来还有许多相对不那么重要的结构，诸如记录redis的`pid`之类信息的`redisServer`等等，不细说。

最后就是大量的函数声明，密密麻麻，还挺吓人的。

---

接下来就按照`server.h`引用的头文件大概去分析

## 事件驱动

redis的事件模型相对于`rust`各种异步框架还是比较简单的

### 核心

```c
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    time_t lastTime;     /* Used to detect system clock skew */
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
    int flags;
} aeEventLoop;
```

最为重要的事件循环

主要有两类

```c
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE|BARRIER) */
    aeFileProc *rfileProc;
    aeFileProc *wfileProc;
    void *clientData;
} aeFileEvent;  // IO事件
typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
    aeTimeProc *timeProc;
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *prev;
    struct aeTimeEvent *next;
} aeTimeEvent;  //定时事件
```

### 实现

对于IO事件，ae复用了操作系统提供的api，依赖epoll之类

对于定时事件，ae使用了O(N)的算法遍历列表，获得最近的一个事件

这里ae甚至有系统时间回溯时的解决措施，算是大开眼界

## yet another dynamic string

没有多少特别的地方，上一次commit也是五个月之前，在此略过

## dict:redis的kv实现

接下来的就是redis的核心，dict，也就是键值对的实现，代码不多，但是性能几乎全是在这里压榨出来的

### 声明

```c
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

这里的type是个vtable

```c
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```

并且维护了两个hashtable用以满足rehash的需要

### 实现

这里说说最重要的rehash

什么是rehash？redis选择哈希表来保存键值对就不可避免的遭遇哈希冲突，冲突必然会导致性能退化，最坏情况下甚至会退化到O(N)，当然可以通过其他方式来减少冲突带来的损失，比如`tair`采用的冲突环。但是最简单也最好理解的方式就是：冲突多了要么是哈希函数不行，要么就是表小了。无论是前者还是后者，扩大哈希表都可以有不错的效果，一来通过不同的模数避免撞哈希，二来减小了冲突链的长度。

下面就是rehash的实现。

```c
int dictRehash(dict *d, int n) {
    int empty_visits = n*10; /* Max number of empty buckets to visit. */
    if (!dictIsRehashing(d)) return 0;

    while(n-- && d->ht[0].used != 0) {
        dictEntry *de, *nextde;

        /* Note that rehashidx can't overflow as we are sure there are more
         * elements because ht[0].used != 0 */
        assert(d->ht[0].size > (unsigned long)d->rehashidx);
        while(d->ht[0].table[d->rehashidx] == NULL) {
            d->rehashidx++;
            if (--empty_visits == 0) return 1;
        }
        de = d->ht[0].table[d->rehashidx];
        /* Move all the keys in this bucket from the old to the new hash HT */
        while(de) {
            uint64_t h;

            nextde = de->next;
            /* Get the index in the new hash table */
            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
            de->next = d->ht[1].table[h];
            d->ht[1].table[h] = de;
            d->ht[0].used--;
            d->ht[1].used++;
            de = nextde;
        }
        d->ht[0].table[d->rehashidx] = NULL;
        d->rehashidx++;
    }

    /* Check if we already rehashed the whole table... */
    if (d->ht[0].used == 0) {
        zfree(d->ht[0].table);
        d->ht[0] = d->ht[1];
        _dictReset(&d->ht[1]);
        d->rehashidx = -1;
        return 0;
    }

    /* More to rehash... */
    return 1;
}
```

几个地方要注意的

1. redis（过去）作为一个单线程软件，一旦一个操作花的时间太久，就会导致其它请求的阻塞，即使现在引入了多线程，更多也是用于IO方面，处理命令依然要保证阻塞不能太久，当然也有避免发生条件竞争的需求。每次rehash都不会全部做完，而是规定了处理数量上限`n`。
2. rehash的过程中会逐步把表0的结点转移到表1，所以数据可能会在任意一张表里，redis会两张表都查一遍。
3. redis的rehash是个漫长的过程，在rehash过程中，不仅更新操作会转移到表1进行，所有操作也都会进行一小步rehash
4. 触发rehash的时机，在`size > DICT_HT_INITIAL_SIZE && (used*100/size < HASHTABLE_MIN_FILL)`时发生

### 迭代器

本来想放到迭代器部分，但是想想还是专开一节

迭代器相关的函数只有四个

```c
dictIterator *dictGetIterator(dict *d);
dictIterator *dictGetSafeIterator(dict *d);
dictEntry *dictNext(dictIterator *iter);
void dictReleaseIterator(dictIterator *iter);
```

其中区分了safe的和unsafe的。

这两者的区别有些反直觉，safe可以修改dict内容，而unsafe只读。实际上，safe迭代器在迭代过程中会禁用rehash，以保证不破坏结构，而unsafe没有这样的限制，当然，为了防止unsafe迭代器破坏这样的规矩，redis引入了fingerprint记录unsafe迭代器开始迭代时的状态，并在最后进行比较，以确认没有违规操作。

```c
if (iter->entry == NULL) {
	dictht *ht = &iter->d->ht[iter->table];
    if (iter->index == -1 && iter->table == 0) { 
    	if (iter->safe)
        	iter->d->iterators++;
    .  .   . // 仅在开始迭代时自增
    }
    
static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1); // 在存在safe迭代器时不做rehash操作
}
```

用处的话，大概是删除超时key之类？

## 小学生也能看懂的adlist

naive的链表

## 你也配叫malloc？

`zmalloc`，非常敷衍的把几个主流allocator封装了一下，我甚至没有读的欲望

## anet

> Basic TCP socket stuff made a bit less boring.

并且读这些封装也挺boring的。

## zip-list

list的一种编码

只能保存int类型和string类型。 

`<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>` 

结构大概是这样的。

- zlbytes占4字节，标记整个zip-list长度
- zltail会标记最后一个entry所在偏移，用以逆序遍历。
- zllen，entry个数
- entry，每个entry至少两个字节
  - 对于int，第一个字节标识前一个entry的长度
  - 对于string，第二个字节用于标记编码方式，接下来记录raw_string
  - 对于entry长度超过一个byte表示范围的，使用更多的位数  

## intset

不得不说，这是个实现非常牛逼的整数集合

### struct

```C
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

最引人注目的大概是这个encoding了，intset会探测当前集合所需要的最小编码长度，然后以该长度组织数组。

### functions

先看search

```C
while(max >= min) {
        mid = ((unsigned int)min + (unsigned int)max) >> 1;
        cur = _intsetGet(is,mid);
        if (value > cur) {
            min = mid+1;
        } else if (value < cur) {
            max = mid-1;
        } else {
            break;
        }
    }

    if (value == cur) {
        if (pos) *pos = mid;
        return 1;
    } else {
        if (pos) *pos = min;
        return 0;
    }
```

看到这个二分了吗，显然这个set是有序的

上面说到redis的自动编码探测，这样的魔法显然不会是免费的，我们从add中就可窥得一二

```c
intset *intsetAdd(intset *is, int64_t value, uint8_t *success) {
    uint8_t valenc = _intsetValueEncoding(value);
    uint32_t pos;
    if (success) *success = 1;
    if (valenc > intrev32ifbe(is->encoding)) {
        return intsetUpgradeAndAdd(is,value);
    } else {
        if (intsetSearch(is,value,&pos)) {
            if (success) *success = 0;
            return is;
        }

        is = intsetResize(is,intrev32ifbe(is->length)+1);
        if (pos < intrev32ifbe(is->length)) intsetMoveTail(is,pos,pos+1);
    }

    _intsetSet(is,pos,value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}
static intset *intsetUpgradeAndAdd(intset *is, int64_t value) {
    uint8_t curenc = intrev32ifbe(is->encoding);
    uint8_t newenc = _intsetValueEncoding(value);
    int length = intrev32ifbe(is->length);
    int prepend = value < 0 ? 1 : 0;

    is->encoding = intrev32ifbe(newenc);
    is = intsetResize(is,intrev32ifbe(is->length)+1);

    while(length--)
        _intsetSet(is,length+prepend,_intsetGetEncoded(is,length,curenc));
    if (prepend)
        _intsetSet(is,0,value);
    else
        _intsetSet(is,intrev32ifbe(is->length),value);
    is->length = intrev32ifbe(intrev32ifbe(is->length)+1);
    return is;
}

```

在编码上升后，intset就会进入upgrade环节，这个环境不仅会发生一次realloc，还会发生一次逐个移动这种开销极大的东西。当然，出于编码本身的性质，插入新值会直接在最前或最后，而不需要额外的二分。

该死，这样做的性能开销真的值吗？

## version

```c
#define REDIS_VERSION "999.999.999"
```

卧槽！牛逼！

## util(又名：剑指offer之redis重制版)

redis教你写string2int

## latency

[官网的介绍](https://redis.io/topics/latency-monitor)，老实讲没太懂。但是report里的冷笑话显然是讲的不错的。

总之，这是一个延迟监控器，事实上，它的工作也非常依赖server.c，或者说，这就是一个server端的插件，用以记录命令时延。

另外这个file上个commit还是五个月前修的一个typo

## quick-list

结合了zip-list的特殊结构体，用来加快查询速度

### struct

```cpp
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;

typedef struct quicklistBookmark {
    quicklistNode *node;
    char *name;
} quicklistBookmark;

typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned long len;          /* number of quicklistNodes */
    int fill : QL_FILL_BITS;              /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;
```

这里的数据结构开始不naive了。

quicklist的结构体构造看起来极似一个普通的双链表，但是多了几个其它的东西，fill与compress两者解释了quickList快在哪里。

- ziplist将列表压缩得到了极高的内存利用率，同时也是cache友好的，但是这导致了它的修改操作开销较大
- 双链表插入删除开销极小，但是内存利用率很难过

为了结合两者，redis采用了quicklist结合二者。quicklist的每一个结点都是一个ziplist。

但是这里还有些问题，每一个node保存多长的ziplist？你可以当个神经病对长度不做限制，但是得到的就是完全退化成ziplist的一坨。你也可以非常封建作风，坚持保持社交距离，两个value之间务必分开，那和普通链表也没区别了。

于是redis提供了可配置项，list-max-ziplist-size与list-compress-depth，前者记录每个node的value数量，后者则是对于不常访问的那些node进一步压缩。压缩后得到的就是quicklistLZF，LZF也是其所使用的压缩算法。fill与compress两个字段正是记录了对应的配置。

### 实现

不妨以pushhead为例

```c
int quicklistPushHead(quicklist *quicklist, void *value, size_t sz) {
    quicklistNode *orig_head = quicklist->head;
    if (likely(
            _quicklistNodeAllowInsert(quicklist->head, quicklist->fill, sz))) {
        quicklist->head->zl =
            ziplistPush(quicklist->head->zl, value, sz, ZIPLIST_HEAD);
        quicklistNodeUpdateSz(quicklist->head);
    } else {
        quicklistNode *node = quicklistCreateNode();
        node->zl = ziplistPush(ziplistNew(), value, sz, ZIPLIST_HEAD);

        quicklistNodeUpdateSz(node);
        _quicklistInsertNodeBefore(quicklist, quicklist->head, node);
    }
    quicklist->count++;
    quicklist->head->count++;
    return (orig_head != quicklist->head);
}
```

可以观察到，首先尝试在ziplist上插入，如果已满，则创建新的node

## rax

```
 *              (f) ""
 *                \
 *                (o) "f"
 *                  \
 *                  (o) "fo"
 *                    \
 *                  [t   b] "foo"
 *                  /     \
 *         "foot" (e)     (a) "foob"
 *                /         \
 *      "foote" (r)         (r) "fooba"
 *              /             \
 *    "footer" []             [] "foobar"
```

稍有见地的人都能看出，这是一个字典树。

## 以`server.c`结束

先是用**900**行创建了所有命令的结构体数组`redisCommandTable`，以至于明明在它前面的`redisServer`实例都变得存在感低下了。













---

**！！！以下内容为原版，极不推荐阅读！！！**

## zip-list
只能保存int类型和string类型。  
`<zlbytes> <zltail> <zllen> <entry> <entry> ... <entry> <zlend>`  
结构大概是这样的。

- zlbytes占4字节，标记整个zip-list长度
- zltail会标记最后一个entry所在偏移，用以逆序遍历。
- zllen，entry个数
- entry，每个entry至少两个字节
    - 对于int，第一个字节标识前一个entry的长度
    - 对于string，第二个字节用于标记编码方式，接下来记录raw_string
    - 对于entry长度超过一个byte表示范围的，使用更多的位数  
## zip-map
保存键值对。  
`<zmlen><len><key><len><free><key> ... <len><value><len><free><value> ...`  
key和value都是string  
- zmlen，一字节，记录大小，塞不下就塞不下
- len，1字节或5字节，表示后面跟着的key或value长度
- free，1字节，留作扩展  
## adlist
``` cpp
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct listIter {
    listNode *next;
    int direction;
} listIter;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```
非常naive的双链表，特殊一点就在于这个head里保存的几个函数指针吧，有多态内味了。遍历的函数也没啥要说的。
## sds
redis这个神奇的命名……
``` cpp
struct sdshdr {
    unsigned int len;
    unsigned int free;
    char buf[];
};
```
同样不展开说，天下的字符串类基本上大同小异，也同样写了一大坨函数，比string.h料多得多了。自己实现的格式化字符串，说不准有攻击的空间，不过这个函数似乎对外不太好访问到，但是这些不在本文讨论范围之内。
## quick-list
结合了zip-list的特殊结构体，用来加快查询速度
```cpp
typedef struct quicklistNode {
    struct quicklistNode *prev;
    struct quicklistNode *next;
    unsigned char *zl;
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
typedef struct quicklistLZF {
    unsigned int sz; /* LZF size in bytes*/
    char compressed[];
} quicklistLZF;

typedef struct quicklistBookmark {
    quicklistNode *node;
    char *name;
} quicklistBookmark;

typedef struct quicklist {
    quicklistNode *head;
    quicklistNode *tail;
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned long len;          /* number of quicklistNodes */
    int fill : QL_FILL_BITS;              /* fill factor for individual nodes */
    unsigned int compress : QL_COMP_BITS; /* depth of end nodes not to compress;0=off */
    unsigned int bookmark_count: QL_BM_BITS;
    quicklistBookmark bookmarks[];
} quicklist;

```
这是一个比较有趣的东西，先放着，日后再说
## dict
重头戏，redis的核心，redis高速缓存的核心竞争力
```cpp
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```
一个结点，保存了一个键值对
```c
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;
```
vtable一样的东西，用以实现多态。
```c
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```
非常有趣的在单线程下解决扩展哈希表耗时过长的方式。默认产生了两张表，相当于天然的读写锁，一张表用来读，另一张表用来写，数据同步也令人称道。
```c
/* Rehash for an amount of time between ms milliseconds and ms+1 milliseconds */
int dictRehashMilliseconds(dict *d, int ms) {
    long long start = timeInMilliseconds();
    int rehashes = 0;
 
    while(dictRehash(d,100)) {
        rehashes += 100;
        if (timeInMilliseconds()-start > ms) break;
    }
    return rehashes;
```
隔一小段时间重定位一次，每次刷新一点点，超时就停止，将长时间的操作转化为无数短时间，提供了高可用。
除此之外，保证同步时的数据可靠也有相应的处理方法，不多展开。

---
写到这里才发现一个严肃的问题，面试时问的redis类型和上述似乎不是一个东西，面试所问及的是源码中以`t_`开头的几种结构体
## t_string
直接就是sds
## t_list
一般情况是quick-list，在少数时刻会是zip-list，此时会迅速转换成quick-list
## t-hash
时而是zip-list,时而是dict
## t-set
时而是dict,时而是int-set
## t-zset
不太妙，这是个独立的东西，目前知道是一个有序集合，具体实现以后再看。
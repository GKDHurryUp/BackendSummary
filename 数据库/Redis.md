#   Redis #
## redis简介 ##
- redis是一款高性能的NOSQL系列的非关系型数据库，**redis数据存在内存中**，读写速度非常快，广泛应用于**缓存方向**。
- redis也经常用来做**分布式锁**
- redis支持事务、持久化、LUA脚本、LRU驱动事件、多种集群方案

# redis基本数据

redis存储的是：key,value格式的数据，其中key都是字符串，value是一个redisObject结构体
1. 字符串类型 string

2. 哈希类型 hash ： map格式  

3. 列表类型 list ： linkedlist格式。支持重复元素

4. 集合类型 set  ： 不允许重复元素

   应用：共同好友，推荐好友SUNION、SDIFF

5. 有序集合类型 sortedset：不允许重复元素，且元素有顺序

   应用：排行榜，有序事件，评论（时间，点赞数）+分页（动态）

6. Stream类型, redis 5.0 加入，完善消息队列

查看编码方式

```
object encoding key
```

## string

字符串是最基础的类型，因为所有的键都是字符串类型，字符串长度不能超过512MB。

Redis使用自己的简单动态字符串(simple dynamic string, SDS)的抽象类型作为默认字符串表示

```C
struct sdshdr {    
  // 用于记录buf数组中使用的字节的数目
  // 和SDS存储的字符串的长度相等  
	int len;    
  // 用于记录buf数组中没有使用的字节的数目   
	int free;    
  // 字节数组，用于储存字符串
	char buf[];   //buf的大小等于len+free+1，其中多余的1个字节是用来存储’\0’的。
};
```

对比：

| C字符串                                    | SDS                                        |
| ------------------------------------------ | ------------------------------------------ |
| 获取字符串长度的复杂度为O(N)               | 获取字符串长度的复杂度为O(1)               |
| API是不安全的，可能会造成缓冲区溢出        | API是安全的，不会造成缓冲区溢出            |
| 修改字符串长度N次必然需要执行N次内存重分配 | 修改字符串长度N次最多需要执行N次内存重分配 |
| 只能保存文本数据                           | 可以保存文本或者二进制数据                 |
| 可以使用所有库中的函数                     | 可以使用一部分库的函数                     |

编码方式：

- **int**，8个字节的长整型。字符串值是整型时，使用long整型表示
- **embstr**，这种形式使得 RedisObject和SDS 内存地址是连续的
- **raw**，这种形式使得内存不连续

## list

使用：

```
LPUSH key value [value …]
```

将一个或多个值 `value` 插入到列表 `key` 的表头，返回列表的长度

```
LPUSHX key value
```

将值 `value` 插入到列表 `key` 的表头，当且仅当 `key` 存在并且是一个列表，当 `key` 不存在时，LPUSHX命令什么也不做

```
LPOP key
```

移除并返回列表 key 的头元素

原理：

==在版本3.2之前==

- 压缩列表ziplist
- 双向链表linkedlist 

转换条件：

- 试图往列表新添加一个字符串值，且这个字符串的长度超过 server.list_max_ziplist_value （默认值为 64 ）
- ziplist 包含的节点超过 server.list_max_ziplist_entries （默认值为 512 ）

**==在版本3.2之后==**

- 全部由quicklist实现

list数据结构：

```C
typedef struct list{
     //表头节点
     listNode *head;
     //表尾节点
     listNode *tail;
     //链表所包含的节点数量
     unsigned long len;
     //节点值复制函数
     void (*free) (void *ptr);
     //节点值释放函数
     void (*free) (void *ptr);
     //节点值对比函数
     int (*match) (void *ptr,void *key);
}list;
```

listNode 节点：

```C
typedef  struct listNode{
       //前置节点
       struct listNode *prev;
       //后置节点
       struct listNode *next;
       //节点的值
       void *value;  
}listNode
```

特点：

1. 可以直接获得头、尾节点。
2. 常数时间复杂度得到链表长度。
3. 是双向链表。

### **双向链表(linkedlist)**

linkedlist是标准的双向链表，Node节点包含prev和next指针，可以进行双向遍历；
还保存了 head 和 tail 两个指针，因此，对链表的表头和表尾进行插入的复杂度都为 O(1) —— 这是高效实现 LPUSH 、 RPOP、 RPOPLPUSH 等命令的关键

### 压缩列表(ziplist)

ziplist是redis为了节约内存而开发的顺序型数据结构。它被用在列表键和哈希键中。一般用于小数据存储。

1. ziplist是一种特殊编码的双向列表，**特殊编码**是为了节省存储空间
2. ziplist允许同时存放字符串和整型类型，**并且整型数被编码成真实的整型数而不是字符串序列(节省空间)**
3. ziplist列表支持在头部和尾部进行push和pop操作的时间复杂度都在常量范围O(1)，但是每次操作都涉及内存重新分配，**尤其在头部操作时，会涉及大段的内存移动操作**，增加了操作的复杂性

==没有维护双向指针prev，next；而是维护一个指向当前的指针，存储上一个节点的长度和当前节点的长度，通过长度推算下一个元素在什么地方。==
牺牲读取的性能，获得高效的存储空间，因为(简短字符串的情况)存储指针比存储entry长度 更费内存。这是典型的“时间换空间”，目的是节省内存，这种结构并不擅长做修改操作。一旦数据发生改动，就会引发内存realloc，可能导致内存拷贝

每一个存储节点（entry）都是一个zlentry (zip list entry)

``` c
typedef struct zlentry {    // 压缩列表节点
    unsigned int prevrawlensize, prevrawlen;    // prevrawlen是前一个节点的长度，prevrawlensize是指prevrawlen的大小，有1字节和5字节两种
    unsigned int lensize, len;  // len为当前节点长度 lensize为编码len所需的字节大小
    unsigned int headersize;    // 当前节点的header大小
    unsigned char encoding; // 节点的编码方式
    unsigned char *p;   // 指向节点的指针
} zlentry;
```

**prevrawlen是变长编码，有两种表示方法**

- 如果前一节点的长度小于 254 字节，则使用1字节(uint8_t)来存储prevrawlen；
- 如果前一节点的长度大于等于 254 字节，那么将第 1 个字节的值设为 254 ，然后用接下来的 4 个字节保存实际长度。

**ziplist连锁更新问题**

因为在ziplist中，每个zlentry都存储着前一个节点所占的字节数，而这个数值又是**变长编码**的，假设在链表中插入一个元素，它的长度超过了254字节，那么它的后一个的prevrawlen需要扩充为5字节，这个zlentry的整体长度就会发生变化，一直传递下去，最坏情况下，需要进行N次空间再分配，而每次空间再分配的最坏时间复杂度为O(N)，因此连锁更新的总体时间复杂度是O(N^2)

### 快速列表(quicklist)

quicklist，是ziplist和linkedlist二者的结合，quicklist通过quicklistNode来形成一个双向链表，每一个quicklistNode 包含 一个ziplist，quicklist是对ziplist进行一次封装，使用小块的ziplist来既保证了少使用内存，也保证了性能

```C
typedef struct quicklist {
    quicklistNode *head; //头结点
    quicklistNode *tail; //尾节点
    unsigned long count;        /* total count of all entries in all ziplists */
    unsigned int len;           /* number of quicklistNodes */
    int fill : 16;              /* fill factor for individual nodes *///负数代表级别，正数代表个数
    unsigned int compress : 16; /* depth of end nodes not to compress;0=off *///压缩级别
} quicklist;
```

```c
typedef struct quicklistNode {
    struct quicklistNode *prev; //上一个node节点
    struct quicklistNode *next; //下一个node
    unsigned char *zl;            //保存的数据 压缩前ziplist 压缩后压缩的数据
    unsigned int sz;             /* ziplist size in bytes */
    unsigned int count : 16;     /* count of items in ziplist */
    unsigned int encoding : 2;   /* RAW==1 or LZF==2 */
    unsigned int container : 2;  /* NONE==1 or ZIPLIST==2 */
    unsigned int recompress : 1; /* was this node previous compressed? */
    unsigned int attempted_compress : 1; /* node can't compress; too small */
    unsigned int extra : 10; /* more bits to steal for future usage */
} quicklistNode;
```

### list是如何实现阻塞队列的？

阻塞队列，就像ArrayBlockingQueue那样，消费端取数据时，如果列表为空，就阻塞。Redis是如何实现的呢？
`blpop 、 brpop 和 brpoplpush`这个几个阻塞命令的实现原理：

- 将客户端的状态设为“正在阻塞”，并记录阻塞这个客户端的各个键，以及阻塞的最长时限（timeout）等数据
- 将客户端的信息记录到 server.db[i]->blocking_keys 中（其中 i 为客户端所使用的数据库号码），blocking_keys 是一个字典， 字典的键是那些造成客户端阻塞的键， 而字典的值是一个链表， 链表里保存了所有因为这个键而被阻塞的客户端
- 继续维持客户端和服务器之间的网络连接，但不再向客户端传送任何信息，造成客户端阻塞。

## hash

Hash表节点：

```C
typedef struct dictEntry {
    // 键
    void *key;
    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;  // 单链表结构
} dictEntry;
```

1. Reids的Hash采用**链地址法**来处理冲突，然后它没有使用红黑树优化。
2. 哈希表节点采用单链表结构。
3. rehash优化。

编码方式：

- **ziplist**
- **hashtable**

## set

底层使用了intset和hashtable两种数据结构存储

intset我们可以理解为数组

hashtable就是普通的哈希表（key为set的值，value为null）

转换条件：

- 集合对象保存的所有元素都是整数值
- 集合对象保存的元素数量不超过512个

编码方式：

- **intset**
- **hashtable**

intset结构：

```cpp
typedef struct intset {
    
    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];

} intset;
```

intset内部其实是一个数组，==而且存储数据的时候是有序的，因为在查找数据的时候是通过二分查找来实现的==

## zset

底层分别使用**ziplist（压缩链表）**和**skiplist（跳表）**实现。

- 什么时候使用ziplist什么时候使用skiplist？

1. 保存的元素少于128个
2. 保存的所有元素大小都小于64字节

使用ziplist时，头部就是ziplist的，之后跟着zset element，包括value和score



字典的键保存元素的值，字典的值则保存元素的分值；

跳跃表节点的 object 属性保存元素的成员，跳跃表节点的 score 属性保存元素的分值。

### skiplist

跳表(skip List)是一种随机化的数据结构，基于并联的链表，实现简单，插入、删除、查找的复杂度均为O(logN)
```cpp
typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```
在每一层的上面继续添加一层，随机一个层数信息（level），而且插入操作只需要修改插入节点前后的指针，不会影响其它节点的层数。这样查找过程就非常类似于一个二分查找，使得查找的时间复杂度可以降低到O(log n)

如我们单独使用字典，虽然能以 O(1) 的时间复杂度查找成员的分值，但是因为字典是以无序的方式来保存集合元素，所以**每次进行范围操作的时候都要进行排序**；假如我们单独使用跳跃表来实现，虽然能执行范围操作，**但是查找操作由 O(1)的复杂度变为了O(logN)**。因此**Redis使用了两种数据结构来共同实现有序集合。**

```C
#define ZSKIPLIST_MAXLEVEL 32 /* Should be enough for 2^32 elements */
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */

/* Returns a random level for the new skiplist node we are going to create.
 * The return value of this function is between 1 and ZSKIPLIST_MAXLEVEL (32)
 * (both inclusive), with a powerlaw-alike distribution（也就是说符合80-20法则） where higher
 * levels are less likely to be returned. */
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;  
}
```

(random()&0xFFFF，int是4字节32位，0xFFFF是2字节，它的作用就是将高位置0，得到小于等于0xFFFF的随机数，这个随机数比(ZSKIPLIST_P * 0xFFFF)小的概率为ZSKIPLIST_P，0.25，每一次wihle为true的概率都为 ZSKIPLIST_P，那么level n的概率 为 ZSKIPLIST_P ^ (n - 1)

### 为什么redis跳表上升概率设置为1/4而不是1/2？

时间换空间，较少的向上随机造层，避免过的创建节点

### redis内部编码格式

redis是key-value存储数据的，它实际是以一个redisObject的结构体来保存的

```C
typedef struct redisObject {
    // 类型
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 对象最后一次被访问的时间
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
    // 引用计数
    int refcount;
    // 指向实际值的指针
    void *ptr;
} robj;
```

其中type就指定了是五大基本数据中的哪一个，encoding代表其编码格式

```C
#define OBJ_ENCODING_RAW 0        /* Raw representation */
#define OBJ_ENCODING_INT 1        /* Encoded as integer */
#define OBJ_ENCODING_HT 2         /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3     /* Encoded as zipmap */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5    /* Encoded as ziplist */
#define OBJ_ENCODING_INTSET 6     /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7   /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8     /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9  /* Encoded as linked list of ziplists */
```

string：

1. 如果一个字符串对象保存的是整数值，并且这个整数值可以用 long 类型标识，对应为INT编码

2. 如果字符串对象保存的是一个字符串值，并且这个字符串的长度大于 32 字节，那么字符串对象将使用一个简单动态字符串（SDS）来保存这个字符串值，并将对象的编码设置为 RAW。
3. 如果字符串对象保存的是一个字符串值，并且这个字符串的长度小于等于 32 字节，那么字符串对象将使用 EMBSTR编码的方式来保存这个字符串。

## 为什么要使用redis/为什么要使用缓存？ ##
1. 高性能
	如果用户第一次访问数据库的数据，要进行I/O操作，过程比较慢。将该用户访问的数据存在缓存中，下一次访问可以直接从缓存中换取（操作缓存就是直接操作内存）
2. 高并发
	直接操作缓存能够承受的请求是远远大于直接访问数据库的，可以把数据库的部分数据转移到缓存中去，提高并发量

## 为什么要使用redis而不用map/guava做缓存？ ##
缓存分为本地缓存和分布式缓存。
Java自带的map或者guava实现的是本地缓存，主要的特点是轻量，快速，在多实例的情况下，每个实例都需要各自保存一份缓存，不具有一致性
使用redis或memcached实现的是分布式缓存，在多实例的情况下，各实例公用一份缓存数据，缓存具有一致性

## redis和memcached的区别 ##
1. redis支持更丰富的数据类型list, set, zset, hash等数据结构，memcached支持简单的数据类型String
2. redis支持数据的持久化，memcached把数据全部放在内存中
3. redis支持集群模式，memcached没有原生的集群模式
4. redis使用单线程的多路IO复用，memcached是多线程，非阻塞IO复用的网络模型

# Redis淘汰删除策略 #

redis数据库在数据库服务器中使用了`redisDb`数据结构，结构如下：

```c
typedef struct redisDb {
     dict *dict;     /* 键空间 key space */
     dict *expires;    /* 过期字典 */
     dict *blocking_keys;  /* Keys with clients waiting for data (BLPOP) */
     dict *ready_keys;   /* Blocked keys that received a PUSH */
     dict *watched_keys;   /* WATCHED keys for MULTI/EXEC CAS */
     struct evictionPoolEntry *eviction_pool; /* Eviction pool of keys */
     int id;      /* Database ID */
     long long avg_ttl;   /* Average TTL, just for stats */
} redisDb;
```

## 为什要使用Redis淘汰删除策略？ ##
Redis性能很高，一般作为缓存来使用，很可能会碰到内存空间瓶颈，如果超过了内存的限制，它和磁盘会发生交换，使得Redis性能下降

## Redis内存淘汰机制 ##
首先Redis客户端发起一个内存申请，然后Redis服务器端检测是否查过了一个最大的内存限制值，如果超出了的话就是用淘汰策略

redis提供6种数据淘汰策略：
1. volatile-lru
	从已设置过期时间的数据集(server.db[i].expires)中挑选最近最少使用的数据淘汰
	
2. volatile-ttl
	从已设置过期时间的数据集(server.db[i].expires)中挑选将要过期的数据淘汰
	
3. volatile-random
	从已设置过期时间的数据集(server.db[i].expires)中任意选择数据淘汰
	
4. allkeys-lru（常用）
	从数据库(server.db[i].dict)中挑选最近最少使用的数据淘汰
	
	实现：在redisObject结构体中定义了一个字段，用来存储对象最后一次被命令程序访问的时间。并不需要一个完全准确的LRU算法，就算移除了一个最近访问过的Key，也没有关系。最开始随机挑选几个key，把最大的那个删除，之后采用了缓冲池，加入了历史信息的比较
	
5. allkeys-random
    从数据集(server.db[i].dict)中任意选择数据淘汰

6. no-eviction
    禁止驱逐数据，内存满时，再写入数据就会报错

## Redis的过期键删除策略 ##
redis数据库键的过期时间都保存在过期字典中，根据系统时间和存活时间判断是否过期。

1. 定时删除：实现方式，创建定时器

   在设置键的过期时间的同时，创建一个定时器，让定时器在键的过期时间来立即执行对键的删除操作

2. 惰性删除：每次获取键时，检查是否过期

3. 定期删除：每隔一段时间，对数据库进行一次检查，删除过期键，由算法决定删除多少过期键和检查多少数据库

优缺点
1. 定时删除，对内存友好，但是对cpu很不友好
2. 惰性删除，对cpu友好，对内存很不友好
3. 定期删除，是两种折中，但是，如果删除太频繁，将退化为定时删除，如果删除次数太少，将退化为惰性删除。



Redis服务器实际使用的是惰性删除和定期删除两种策略

通过配合使用这两种策略，服务器可以很好地在合理使用CPU时间和避免浪费内存空间之间取得平衡

- 惰性删除在所有读写数据库的Redis命令在执行之前都会调用expireIfNeeded函数对输入键进行检查

- Redis的服务器周期性操作redis.c/serverCron函数（默认100ms执行一次），作为时间事件来运行。在其中调用databasesCron函数，然后activeExpireCycle函数就会被调用，它在规定的时间内，分多次遍历服务器中的各个数据库，从数据库的expires字典中随机检查一部分键的过期时间，并删除其中的过期键

### activeExpireCycle函数的工作模式：

1. 函数每次运行时，都会遍历CRON_DBS_PER_CALL（默认16）个db，针对每个db，每次循环随机选择20个（`ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP`）中取出一定数量的随机键进行检查，并删除其中的过期键
2. 有一个全局变量current_db会记录当前activeExpireCycle函数检查的进度，并在下一次activeExpireCycle函数调用时，接着上一次处理的数据库进行操作

### AOF、RDB和复制功能对过期键的处理

- 生成RDB，程序会数据库中的键进行检查，已过期的键不会保存到新创建的RDB文件中
- 载入RDB
  - 主服务载入RDB文件，会对文件中保存的键进行检查会忽略过期键加载未过期键
  - 从服务器载入RDB文件，会加载文件所保存的所有键（过期和未过期的），但从主服务器同步数据同时会清空从服务器的数据库
- 写入AOF文件：当过期键被删除后，会在AOF文件增加一条DEL命令，来显式地记录该键已被删除
- AOF重写：已过期的键不会保存到重写的AOF文件中
- 复制，当服务器运行在复制模式下时，从服务器的过期键删除动作由主服务器控制的，这样的好处主要为了保持主从服务器数据一致性
  - 主服务器在删除一个过期键之后，会显式地向所有的从服务器发送一个DEL命令，告知从服务器删除这个过期键
  - 从服务器在执行客户端发送的读取命令时，即使碰到过期键也不会将过期键删除，不作任何处理
  - 只有接收到主服务器 DEL命令后，从服务器进行删除处理

# redis持久化机制 #

在redis启动之后，有一个定时任务进行持久化，形成持久化文件。在关机之后，内存中的数据会丢失，再次启动redis之后会读取持久化文件，把数据从磁盘读回到内存

## RDB持久化 ##
快照（snapshotting），redis会单独创建(fork)一个与当前进程一模一样的子进程来进行持久化，这个子进程的所有数据（变量、环境变量、程序计数器）都和原进程一模一样，会先将数据写入到一个**临时文件**中，等待持久化结束，使用临时文件替换上次持久化好的文件，整个过程中，**主进程不进行任何io操作，确保了极高的性能**

## 再另一个目录启动redis会丢失数据？ ##
在配置文件中，工作空间默认为./当前目录，会把生成的文件生成在配置的路径下

## 什么时候触发rdb持久化机制？ ##

手动触发：

1. 执行save或者bgsave命令
   - save使用主进程进行持久化（不使用）
   - redis会在后台异步进行快照操作，同时可以相应客户端请求

自动触发：

1. 配置文件中默认的快照配置
2. 执行SHUTDOWN时（意外宕机不触发），没有开启aof
3. 从节点全量复制时，主节点发送rdb给从节点完成复制操作，主节点会触发bgsave
4. flushall命令
   如果执行了flushall命令，数据库意外宕机，命令会白执行

## 为什么要fork子进程？ ##
redis5.0之前是单IO线程（redis6.0之后多IO线程，五一期间刚刚发布），进行相关io操作只有一个线程。在主进程进行持久化会进行io操作，导致效率低下，如果此时有客户端发来命令会阻塞，而fork子进程，交给子进程进行持久化，保证性能

## 为什会出现AOF持久化？ ##
数据库意外宕机时，RDB方式不会进行持久化，因此造成数据丢失时间长的问题

## AOF持久化  ##
只追加文件（append-only file，AOF），采用日志记录的方式，以追加的方式写入文件，读操作是不记录的。这种方式通过开辟了内存与硬盘之间的一个缓冲区，保证操作效率，而不会fork子进程

配置：
appendonly yes开启AOF
appendfsync 有三个参数

1. no 表示等操作系统内核进行数据缓存同步到磁盘（效率快，持久化没保证）
2. always 同步持久化，每次发生数据变更时,立即记录到磁盘（效率慢，安全）
3. everysec 每秒种同步一次（默认值，上两种方案的折中，可能会丢失一秒左右的数据）

手动触发：bgrewriteaof

自动触发： 就是根据配置规则来触发，当然自动触发的整体时间还跟Redis的定时任务频率有关

## AOF重写机制 ##

当AOF文件增长到一定大小时候Redis能够调用bgrewriteaof对日志文件进行重写，替换旧的AOP文件会fork子进程---为了给AOF文件瘦身，剔除冗余数据

重写时机：
1. AOF文件达到一定大小（一般往大设置）
	auto-aof-rewrite-min-size 5GB	
2. AOF文件增长率到达一定值（超过原来大小的百分之多少）
	auto-aof-rewrite-percentage 100

## redis4.0之后增加了混合持久化机制 ##

当开启混合持久化时，fork出的子进程会先将共享内存副本全量以RDB方式写入AOF文件，然后在将重写缓冲区的增量命令以AOF方式写入文件，写入完成后通知主进程更新统计信息。

新的AOF前半段是RDB格式的全量数据，后半段是AOF格式的增量数据

配置
aof-use-rdb-preamble yes

优点：rdb数据紧凑，占用空间小, 由于绝大部分都是RDB格式，加载速度快，同时结合AOF，增量的数据以AOF方式保存了，数据更少的丢失。

缺点：兼容性差，一旦开启了混合持久化，在4.0之前版本都不识别该aof文件，同时由于前部分是RDB格式，阅读性较差

## RDB和AOF优缺点 ##
RDB
优点：

1. RDB 是一个非常紧凑（compact）的文件，体积小，因此在传输速度上比较快，因此适合灾难恢复。 

2. RDB 可以最大化 Redis 的性能：父进程在保存 RDB 文件时唯一要做的就是 fork 出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无须执行任何磁盘 I/O 操作。

3. RDB 在恢复大数据集时的速度比 AOF 的恢复速度要快。

缺点：

1. RDB是一个快照过程，无法完整的保存所以数据，尤其在数据量比较大时候，一旦出现故障丢失的数据将更多。

2. 当redis中数据集比较大时候，RDB由于RDB方式需要对数据进行完成拷贝并生成快照文件，fork的子进程会耗CPU，并且数据越大，RDB快照生成会越耗时。

3. RDB文件是特定的格式，阅读性差，由于格式固定，可能存在不兼容情况。

# Redis事务 #
Redis事务就是一次性、顺序性的执行一个队列中的一系列命令，单条命令是原子性执行的，但是事务不保证原子性，且没有自动回滚。事务中任意命令执行失败，其余的命令仍会被执行

基本命令：

1. multi：标记事务块开始
2. exec：执行所有事务块
3. discard：取消事务
4. watch 监视一或多个key,如果在事务执行之前，被监视的key被其他命令改动，则事务被打断 （ 类似乐观锁 ）
5. unwatch : 取消watch对所有key的监控



# 主从复制 #

互联网“三高”架构
- 高并发
- 高性能
- 高可用

可用性=(一年的时间-服务器宕机的时间)/一年的时间
业界追求5个9，即99.999%，服务器年宕机时长低于315秒

## 单redis的风险 ##

1. 单点故障

   机器硬盘故障，数据丢失

2. 容量瓶颈，无法无限制升级内存

3. CPU运算压力

解决方案，AKF拆分原则：

- X轴，全量，镜像，存放一致的数据

- Y轴，业务，功能，存放不一致的数据
- Z轴，优先级，逻辑再拆分

拆分之后的问题：

- 数据同步，强一致性要求所有节点阻塞到数据全部一致，为什么要一变多？解决可用性

主依旧为单点模式，如何解决？

使用其他节点监控主节点，但是如何保证监控节点不出现问题？

使用过半机制n/2+1，但为什么要用奇数？

拿3台和4台来距离，他们的容错机器数量都为1，但是4台却多使用了一台机器的价钱，而且4台机器比4台机器更容易出现故障

## 主从复制形式 ##

Redis使用默认的异步复制，其特点是低延迟和高性能

一个master对多个slave，一个slave只对应一个master
master：

- 写数据
- 执行写操作时，将变换的数据自动同步到slave
- 读数据（可忽略）

slave：
- 读数据
- 写数据（禁止）

核心工作：将master的数据复制到slave中

## 主从复制作用 ##
1. 读写分离：master写，slave读
2. 负载均衡：根据需求变化，改变slave的数量
3. 故障恢复：master宕机，由slave提供服务
4. 数据冗余：实现数据热备份
5. 高可用基石：基于主从复制，构建哨兵模式与集群

## 工作流程 ##

1. 从库配置主库的IP、端口号

2. 当一个从库启动时，会向主库发送sync命令

3. 主库收到sync命令后，会在后台保存快照（rdb），并将保存期间接收到的命令缓存到缓存区中

4. 快照完成之后，redis会将快照文件和所有缓存区中的命令发送给从数据库

5. master有任何写操作都会生成日志记录，发送改变时，就会通知到给下面所有的从服务器

6. 如果slave挂掉后，再次上线，不会进行rdb的全量回复，而是进行增量 

      如果开启AOF，会全量同步

      

7. 建立连接
     建立slave到master的连接，使master能够识别slave，保存slave的端口号
       建立连接：slaveof 主机ip 端口

  8. 客户端发送命令

  9. 服务器启动参数

  10. 服务器配置文件
       断开连接：slaveof no one 

11. 数据同步

  - 全量复制：
  	1. slave发送psync2 ？ -1

    				psync2 <runid> <offset>
  	2. master执行bgsave，记录当前的复制偏移量offset
  	3. 第一个slave连接时，创建命令缓冲区
  	4. 生成RDB文件，通过socket发送给slave
  	5. slave接收RDB，清空数据，执行RDB文件恢复

  - 部分复制：
  	6. slave发送命令告知RDB恢复已经完成
  	7. master发送复制缓冲器信息
  	8. slave接收信息，执行bgrewriteaof，恢复数据 

8. 命令传播

  心跳机制

  

部分复制三个核心
	- 服务器的运行id(run id)，40位16进制字符，用于身份识别
	- 主服务器的复制积压缓冲区
	- 主从服务器的复制

## 复制缓冲区 ##
又称复制积压缓冲区，是一个FIFO队列，用于存储服务器执行过的命令，每次传播命令都会把传播的命令记录下来，并存储在复制缓冲区

1. 偏移量offset
2. 字节值

通过offset区分不同的slave数据传播的差异
master记录已发送的信息对应的offset
slave记录已接收的信息对应的offset

## 数据同步过程应注意什么？ ##

master端：
1. master数据量巨大，数据同步阶段应该避开流量高峰期
2. 复制缓存区大小设定不合理会导致数据溢出，此时需要全量复制，容易导致死循环状态
	repl-backlog-size
3. 多个slave同时对master请求数据同步，master发送的RDB文件增多，会对带宽造成巨大冲击

slave端：
1. 为避免slave进行全量复制、部分复制时服务器响应阻塞或数据不同步，应该关闭对外服务
	slave-serve-stale-data yes|no
2. 数据同步阶段，master发给slave信息可以理解master是slave的一个客户端，主动向slave发送命令
3. slave过多，需要使用树状结构，从主从结构变为树状结构

##  频繁的全量复制 ##
1. master重启，runid将发送变化，导致所有的slave进行全量复制操作
	内部创建master_replid变量，使用runid相同的生成策略，长度41位，在关闭进行rdb持久化,会将runid与offset保存在rdb文件中
2. 复制缓冲区过小，断网后slave的offset越界，触发全量复制	
	设置建议：2 × 重连平均时长 × 每秒写命令数据总量

## 频繁的网络中断 ##
1. master的cpu占用过高或者slave频繁断开连接
	设置合理的超时时间，确认是否释放slave   repl-timeout

# 哨兵模式 #
普通的主从模式，当主数据库崩溃时，需要手动切换从数据库成为主数据库:

在主从模式下，如果master宕机下线，会找一个slave作为master，通知所有的slave连接新的master，可能会存在（全量复制×N+部分复制×N）。

哨兵(sentinel)是一个分布式系统，用于对主从结构的每台服务器进行**监控**，当出现故障时通过**投票机制选择**新的master并将所有的slave连接到新的master。
当使用sentinel模式的时候，客户端就不要直接连接Redis，而是连接sentinel的ip和port，由sentinel来提供具体的可提供服务的Redis实现，这样当master节点挂掉以后，sentinel就会感知并将新的master节点提供给使用者。

作用：
1. 监控
2. 通知
3. 自动故障转移

注：
哨兵也是一台redis服务器，不提供数据服务
通常哨兵的配置数量为单数

## 哨兵工作原理 ##
监控：同步各个节点的状态信息
1. 获取各个sentinel的状态
2. 获取master的状态
	master属性
	- runid
	- role：master
3. 获取所有slave的状态（根据master中的slave信息）
	slave属性
	- runid
	- role：slave
	- master_host、master_port
	- offset

# 集群 #

## 现状问题 ##
1. redis提供的服务OPS达到10万/秒，当前业务OPS已经达到20万/秒
2. 内存单机容量达到256G，当前业务需求内存容量1T

集群就是使用网络将若干计算机联通起来,提供统一的管理方式,对外呈现单机的效果

sentinel模式基本可以满足一般生产的需求，具备高可用性。但是当数据量过大到一台服务器存放不下的情况时，主从模式或sentinel模式就不能满足需求了，这个时候需要对存储的数据进行分片，将数据存储到多个Redis实例中。**cluster模式的出现就是为了解决单机Redis容量有限的问题**，将Redis的数据根据一定的规则分配到多台机器。

1. 数据分配

    Redis 集群使用哈希槽（hash slot），因为一致哈希性算法有数据倾斜的问题，如果在分片的集群中，节点太少，并且分布不均，一致性哈希算法就会出现部分节点数据太多，部分节点数据太少。也就是说无法控制节点存储数据的分配。

   Redis Cluster是自己做的crc16的简单hash算法一致性哈希的空间是一个圆环，节点分布是基于圆环的，无法很好的控制数据分布。而 redis cluster 的槽位空间是自定义分配的，类似于 windows 盘分区的概念。

2. 数据同步，节点通信

   多个 Redis 节点之间进行数据共享的分布式集群，在服务端，通过节点之间的特殊协议进行通讯，即Gossip协议

## Cluster配置 ##
1. cluster-enable yes|no

2. cluster-config-file <filename>
	cluster配置文件名，自动生成
	
3. cluster-node-timeout <milliseconds>
	节点服务响应超时时间
	
4. cluster-migration-barrier <count>
	master连接的slave最小数量
	
	

### 为什么RedisCluster会设计成16384个槽呢?

1. **如果槽位为65536，发送心跳信息的消息头达8k，发送的心跳包过于庞大，浪费带宽。**

   在消息头中，最占空间的是 myslots[CLUSTER_SLOTS/8]。当槽位为65536时，这块的大小是: 65536÷8=8kb因为每秒钟，redis节点需要发送一定数量的ping消息作为心跳包，如果槽位为65536，这个ping消息的消息头太大了，浪费带宽。 

2. **redis的集群主节点数量基本不可能超过1000个。**

   如果节点过1000个，也会导致网络拥堵。因此redis作者，不建议redis cluster节点数量超过1000个。那么，对于节点数在1000以内的redis cluster集群，16384个槽位够用了。没有必要拓展到65536个。 

3. **槽位越小，节点少的情况下，压缩率高。**

   Redis主节点的配置信息中，它所负责的哈希槽是通过一张bitmap的形式来保存的，在传输过程中，会对bitmap进行压缩，但是如果bitmap的填充率slots / N很高的话(N表示节点数)，bitmap的压缩率就很低。如果节点数很少，而哈希槽数量很多的话，bitmap的压缩率就很低。而16384÷8=2kb

注意：对于槽位的转移和分派，Redis集群是不会自动进行的，而是需要人工配置的。所以Redis集群的高可用是依赖于节点的主从复制与主从间的自动故障转移。

## Gossip协议

Gossip协议是一个通信协议，一种传播消息的方式

使用Gossip协议的有：Redis Cluster、Consul

**六度分隔理论**，你和任何一个陌生人之间所间隔的人不会超过六个，也就是说，最多通过六个人你就能够认识任何一个陌生人

Gossip协议基本思想就是：一个节点想要分享一些信息给网络中的其他的一些节点。那么，它**周期性**的**随机**选择一些节点，并把信息传递给这些节点。这些收到信息的节点接下来会做同样的事情，即把这些信息传递给其他一些随机选择的节点。


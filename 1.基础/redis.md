# Redis

## 1. SDS

```c
struct __attribute ((__packed__)) sdshdr8 {
	uint8_t len;    /* used */
	uint8_t alloc;  /* excluding the header and null terminator */
	unsigned char flags;
	char buf[];
};

struct __attribute ((__packed__)) sdshdr16 {...};
struct __attribute ((__packed__)) sdshdr32 {...};
struct __attribute ((__packed__)) sdshdr64 {...};
```

### 1.1. 说明SDS和C字符串的不同之处，解释为什么Redis要使用SDS而不是C字符串？

| C字符串                                    | SDS                                        |
| :----------------------------------------- | :----------------------------------------- |
| 获取字符串长度的复杂度为O(N)               | 获取字符串长度的复杂度为O(1)               |
| API是不安全的，可能会造成缓冲区溢出        | API是安全的，不会造成缓冲区溢出            |
| 修改字符串长度N次必然需要执行N次内存重分配 | 修改字符串长度N次最多需要执行N次内存重分配 |
| 只能保存文本数据                           | 可以保存文本或者二进制数据                 |
| 可以使用所有<string.h>库中的函数           | 可以使用部分<string.h>库中的函数           |

#### 1.1.1. O(1) 复杂度获取字符串长度

C字符串：需要遍历整个字符串

SDS：直接获取len属性

#### 1.1.2. 杜绝缓冲区溢出

strcat函数不会判断空间是否足够

sdscat函数会先检查sds空间是否满足需求，若不满足，会自动进行sds空间扩展，然后才进行修改操作。

#### 1.1.3. 内存重分配优化

内存重分配通常是一个耗时的操作。通过未使用的空间，SDS实现了空间预分配和惰性空间释放两种优化策略。

##### 1.1.3.1. 空间预分配
```c
#define SDS_MAX_PREALLOC (1024*1024)

if (newlen < SDS_MAX_PREALLOC)
	newlen *= 2;
else
	newlen += SDS_MAX_PREALLOC;
```
##### 1.1.3.2. 惰性空间释放

SDS 避免了缩短字符串时的内存重分配操作，并为未来可能的增长操作提供优化。

SDS 提供了API，需要时，可真正的释放SDS未使用的空间。

#### 1.1.4. 二进制安全

C字符串中不能包含空字符，使得C字符串只能保存文本数据，而不能保存图片/音视频等二进制数据。

#### 1.1.5. 二进制安全

SDS总会多分配一个字节来容纳末尾的空字符，这样可以让保存文本数据的SDS重用一部分<string.h>库函数。

### 1.2. 应用

- 保存数据库中的字符串
- 缓冲区
	- AOF 缓冲区
	- 客户端状态的输入缓冲区

## 2. 链表

```c
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedef struct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsigned long len;
} list;
```

### 2.1. 特性

- 双端
- 无环
- 获取表头、表尾的复杂度为O(1)
- 获取节点数量的复杂度为O(1)
- 多态：void * 指针来保存节点值，结合dup、free、match，可以保存各种不同类型的值。

### 2.2. 应用

- 列表键
	- 比如：LRANGE integers 0 10
- 发布和订阅
- 慢查询
- 监视器
- Redis服务器：
	- 使用链表保存多个服务器的信息
	- 使用链表来构建客户端输出缓冲区

## 3. 字典

```c
struct dict {
    dictType *type;

    dictEntry **ht_table[2];
    unsigned long ht_used[2];

    long rehashidx; /* rehashing not in progress if rehashidx == -1 */

    /* Keep small vars at end for optimal (minimal) struct padding */
    int16_t pauserehash; /* If >0 rehashing is paused (<0 indicates coding error) */
    signed char ht_size_exp[2]; /* exponent of size. (size = 1<<exp) */
};

typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;     /* Next entry in the same hash bucket. */
    void *metadata[];           /* An arbitrary number of bytes (starting at a
                                 * pointer-aligned address) of size as returned
                                 * by dictType's dictEntryMetadataBytes(). */
} dictEntry;
```

### 3.1. 特性

- 底层使用哈希表
- 使用链地址法解决键冲突
- 为了让哈希表的负载因子（load factor）维持在一个合理的范围内，程序通过rehash操作来扩展和收缩哈希表
- 每个字典带有两个哈希表，一个平时使用，另一个仅在rehash时使用
- rehash不是一次性完成的，而是分多次、渐进式地完成的。将计算工作均摊到对字典的添加、删除、查找、更新操作上。

### 3.2. 应用

- 数据库
- 哈希键

## 4. 跳跃表

```c
typedef struct zskiplistNode {
    sds ele;
    double score;                        // 分值
    struct zskiplistNode *backward;      // 后退指针
    struct zskiplistLevel {
        struct zskiplistNode *forward;   // 前进指针
        unsigned long span;              // 跨度
    } level[];  // 层
} zskiplistNode;

typedef struct zskiplist {
    struct zskiplistNode *header, *tail;  // 跳跃表头尾节点
    unsigned long length;                 // 节点数量（不算表头）
    int level;          // 层数最大的那个节点的层数（不算表头）
} zskiplist;
```

### 4.1. 特性

- zskiplist用于保存跳跃表信息（头尾节点、长度）、zskiplistNode用于表示跳跃表节点
- 每个跳跃表节点的层高都是 1 至 32 之间的随机数
- 同一个跳跃表中，多个节点可以包含相同的score，但ele必须是唯一的
- 节点按score大小进行排序（升序），分值相同时，按ele排序

#### 4.1.1. 随机生成层数

```c
#define ZSKIPLIST_P 0.25      /* Skiplist P = 1/4 */

int zslRandomLevel(void) {
    static const int threshold = ZSKIPLIST_P*RAND_MAX;
    int level = 1;
    while (random() < threshold)
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```

随机：采用类似幂律分布的方式，较高的level不太可能返回

- 首先，random 随机结果 < threshold 的概率为 P
- level 为 1 的概率为 1-P
- level 为 2 的概率为 P(1-P)
- level 为 3 的概率为 P^2(1-P)
- level 为 4 的概率为 P^3(1-P)
- ......

一个节点的平均层数为：1(1-p) + 2P(1-P) +  3P^2(1-P) + 4P^3(1-P) + ... ≈ 1/(1-P)

所以，当 P = 1/4 时，每个节点所包含的平均指针数目为1.33 。这也是Redis里的skiplist实现在空间上的开销。

#### 4.1.2. 插入

简要步骤：

1. 查找插入位置
2. 为新节点随机生成level，并调整表最大层数
3. 创建并插入新节点，更新指向新节点的每一层forward指针和span
4. 层数大于新节点层数的前置节点，span加1
5. 调整新节点的backward指针
6. 表长度加1

```c
zskiplistNode *zslInsert(zskiplist *zsl, double score, sds ele) {
	// update 记录每一层需要指向新节点的节点，x 为新节点
    zskiplistNode *update[ZSKIPLIST_MAXLEVEL], *x;
    // rank 记录每一层需要指向新节点的节点排名，用于更新span值
    unsigned long rank[ZSKIPLIST_MAXLEVEL];
    int i, level;

    serverAssert(!isnan(score));
    
    // 查找插入位置
    x = zsl->header;
    for (i = zsl->level-1; i >= 0; i--) {
        /* store rank that is crossed to reach the insert position */
        rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
        while (x->level[i].forward &&
                (x->level[i].forward->score < score ||
                    (x->level[i].forward->score == score &&
                    sdscmp(x->level[i].forward->ele,ele) < 0)))
        {
            rank[i] += x->level[i].span;
            x = x->level[i].forward;
        }
        update[i] = x;
    }
    
    // 随机生成新节点的level
    level = zslRandomLevel();
    
    // 如果新节点的level超过了最大层数
    if (level > zsl->level) {
    	// 增加的这些层，需要更新header节点指向新节点，先记录该层需更新的节点（设置为header）
        for (i = zsl->level; i < level; i++) {
            rank[i] = 0;
            update[i] = zsl->header;
            update[i]->level[i].span = zsl->length;
        }
        // 修改表最大层数
        zsl->level = level;
    }
    // 创建新节点
    x = zslCreateNode(level,score,ele);
    for (i = 0; i < level; i++) {
    	// 插入新节点，针对新节点的每一层，修改链表指针
        x->level[i].forward = update[i]->level[i].forward;
        update[i]->level[i].forward = x;

        // 修改新节点每一层的span值
        x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
        // 修改指向新节点的节点每一层的span值
        update[i]->level[i].span = (rank[0] - rank[i]) + 1;
    }

    // 对于节点层数大于新节点的前置节点，给它们的span+1
    for (i = level; i < zsl->level; i++) {
        update[i]->level[i].span++;
    }

	// 设置新节点的后退指针
    x->backward = (update[0] == zsl->header) ? NULL : update[0];
    if (x->level[0].forward)
    	// 设置后退指针指向新节点
        x->level[0].forward->backward = x;
    else
        zsl->tail = x;
    // 表长度加1
    zsl->length++;
    return x;
}
```

### 4.2. 应用

- 有序集合键
- 在集群节点中用作内部数据结构

### 4.3. skiplist 和 平衡树、哈希表 的比较（Redis为什么用skiplist而不用平衡树？）

- skiplist 和 平衡树 的元素是有序排列的，而哈希表不是有序的。因此，哈希表上只能做单个Key查找，不适用做 **范围查找**（指的是查找那些大小在指定的两个值之间的所有节点）。
- 在做范围查找的时候，平衡树比skiplist操作要复杂。在平衡树上，我们找到指定范围的小值之后，还需要以中序遍历的顺序继续寻找其它不超过大值的节点。这里的中序遍历不易实现。**而在skiplist上进行范围查找就非常简单，只需要在找到小值之后，对第1层链表进行若干步的遍历就可以实现。**
- **平衡树的插入和删除操作可能引发子树的调整，逻辑复杂，而skiplist的插入和删除只需要修改相邻节点的指针，操作简单又快速。**
- 从内存占用上来说，skiplist比平衡树更灵活一些。平衡树每个节点包含2个指针，而skiplist每个节点包含的指针数目平均为1/(1-p)。
- 查找单个key，**skiplist和平衡树的时间复杂度都为O(log n)**。而哈希表在保持较低的哈希值冲突概率的前提下，查找时间复杂度接近O(1)，性能更高一些。
- 从算法实现难度上来比较，skiplist比平衡树要简单得多。

### 4.4. 参考资料

http://zhangtielei.com/posts/blog-redis-skiplist.html
https://leetcode.cn/problems/design-skiplist/
https://zhuanlan.zhihu.com/p/340929045

## 4. 整数集合

```c
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

### 4.1. 特性

- 底层实现为数组，以有序、无重复的方式保存集合元素
- 升级操作为整数集合带来了操作上的灵活性，并尽可能地节约内存
- 不支持降级

#### 4.1.1. 升级

1. 根据新元素类型，扩展整数集合底层数组的空间大小，并为新元素分配空间
2. 将底层数组所有元素转换为新类型，并放置到正确位置上
3. 将新元素添加到底层数组里面

### 4.2. 应用

- 集合键（当一个集合只包涵整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数作为集合键的底层实现）

## 5. 压缩列表

```c
-------------------------------------------------------------------------
|  zlbytes   |   zltail   |   zllen   | entry1 | ... | entryN |  zlend  |
-------------------------------------------------------------------------
|  uint32_t  |  uint32_t  |  uint16_t |          ...          | uint8_t |
```

- `zlbytes`：记录整个压缩列表占用的内存字节数
- `zltail`  ：记录起始地址到表尾节点的字节数
- `zllen`    ：zllen < UINT16_MAX 记录压缩列表包含的节点数量；zllen == UINT16_MAX，节点的真实数量需要遍历整个列表得出。
- `zlend`    ：特殊值 0xFF，用于标记压缩列表的末端


```c
-------------------------------------------------------------------------
|  previous_entry_length  |      encoding       |   content   |
-------------------------------------------------------------------------
|       1 or 5 bytes      |  1 or 2 or 5 bytes  |     ...     |
```

- `previous_entry_length`： 记录前一个节点的长度。如果前一个节点长度小于 254 字节，则previous_entry_length占用 1 字节。否则，占用5字节，第一个字节设置为 0xFE，后面四个字节保存前一节点的长度。
- `encoding`： 记录了节点的content属性所保存数据的类型和长度，前两位编码代表类型：
	- 00：1 字节长的数组编码，content长度 小于 1 << 6 
	- 01：2 字节长的数组编码，content长度 小于 1 << 14 
	- 10：5 字节长的数组编码，content长度 小于 1 << 32 （只使用后面4个字节） 
	- 11：整数编码（占用1字节）
- `content`：保存节点的值，可以是字节数组或整数

### 5.1. 连锁更新

添加新节点或删除节点，可能会引发连锁更新操作（平均O(n)，最坏O(n^2)），但这种操作出现的几率并不高。

为了让每个节点的previous_entry_length都符合压缩列表对节点的要求，程序需要不断地对压缩列表执行空间重分配操作，直到entryN为止。

最坏情况下，需执行N次空间重分配，每次空间重分配的最坏复杂度为O(N)，所以连锁更新的最坏复杂度为O(n^2)。

### 5.2. 应用

- 列表键（当一个列表键只包含少量列表项（小整数值、短字符串），Redis就会使用压缩列表来做列表键的底层实现）
- 哈希键（当一个哈希键只包含少量键值对（小整数值、短字符串），Redis就会使用压缩列表来做哈希键的底层实现）

## 6. 对象

```c
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits access time). */
    int refcount;
    void *ptr;
} robj;
```

Redis数据库保存的键值对，键总是一个字符串对象，值可以是字符串对象、列表对象、哈希对象、集合对象、有序集合对象。

`TYPE` 命令返回的结果是 数据库键对应 **值的对象类型**
`OBJECT ENCODING` 命令可以查看 数据库键对应 **值对象的编码**

对象类型和对象编码：
| 类型   | 编码                |
| ------ | ------------------- |
| string | int、embstr、raw    |
| list   | quicklist           |
| hash   | listpack、hashtable |
| set    | intset、hashtable   |
| zset   | listpack、skiplist  |

备注：redis后面用listpack替代ziplist。

看源码定义，zipmap、linkedlist、ziplist不再使用了。
```c
#define OBJ_ENCODING_RAW 0     /* Raw representation */
#define OBJ_ENCODING_INT 1     /* Encoded as integer */
#define OBJ_ENCODING_HT 2      /* Encoded as hash table */
#define OBJ_ENCODING_ZIPMAP 3  /* No longer used: old hash encoding. */
#define OBJ_ENCODING_LINKEDLIST 4 /* No longer used: old list encoding. */
#define OBJ_ENCODING_ZIPLIST 5 /* No longer used: old list/hash/zset encoding. */
#define OBJ_ENCODING_INTSET 6  /* Encoded as intset */
#define OBJ_ENCODING_SKIPLIST 7  /* Encoded as skiplist */
#define OBJ_ENCODING_EMBSTR 8  /* Embedded sds string encoding */
#define OBJ_ENCODING_QUICKLIST 9 /* Encoded as linked list of listpacks */
#define OBJ_ENCODING_STREAM 10 /* Encoded as a radix tree of listpacks */
#define OBJ_ENCODING_LISTPACK 11 /* Encoded as a listpack */
```

### 6.1. 为什么有序集合需要同时使用跳跃表和字典实现？
```c
typedef struct zset {
    dict *dict;
    zskiplist *zsl;
} zset;
```

首先，

- 跳跃表用于范围操作
- 字典用于ele到score的映射

1. 单独使用字典，字典是无序的，如果做范围操作（ZRANK、ZRANGE），需要排序至少O(NlogN)复杂度，额外O(N)内存空间。
2. 单独使用跳跃表，根据ele查找score复杂度将从O(1)变为O(logN)。

### 6.2. 内存回收

Redis在对象中使用 **引用计数** 实现内存回收机制

### 6.3. 对象共享

多个键共享同一个值对象：
1. 将数据库键的值指针指向一个现有的值对象
2. 将被共享的值对象的引用计数加 1

## 7. 数据库

```c
struct redisServer {
	...
    redisDb *db;
    int dbnum;                      /* Total number of configured DBs */
    ...
};
```
初始化服务器时，会创建dbnum个数据库。（默认 16 个）

```c
static struct config {
    int dbnum; /* db num currently selected */
} config;

int main(int argc, char **argv) {
    config.dbnum = 0;
}
```
redis-cli 默认选择0号数据库，可以使用 SELECT 命令切换数据库。

```c
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
};
```
使用dict保存了数据库中所有的键值对，称这个字典为键空间。
操作：`SET`、`GET`、`DEL`、`FLUSHDB`、`RANDOMKEY`、`EXISTS`、`KEYS`...
过期操作：`EXPIRE`、`PEXPIRE`、`EXPIREAT`、`PEXPIREAT`、’`TTL`、`PTTL`

### 7.1. Redis 过期键删除策略

1. 惰性删除（expireIfNeeded）
	所有读写数据库的命令在执行之前都会调用 expireIfNeeded 做检查：
	a. 键过期，则删除
	b. 键未过期，则不做动作
2. 定期删除（activeExpireCycle）
	在规定时间内，分多次遍历各个数据库，从db->expires字典中检查一部分键的过期时间，并删除其中的过期键。

- 生成RDB文件时，过期键不会被保存。
- 载入RDB文件时，
	- master模式：过期键被忽略。
	- slave模式：过期键被载入。（主从同步会清空从数据库，无影响）
- 过期键被删除后，AOF文件中会追加 DEL 命令。
- 过期键不会被保存到重写后的AOF文件中。
- 复制模式下，
	- 主服务器删除过期键，会向所有从服务器发送DEL命令
	- 从服务器执行读命令时，即使键过期，也不会被删除
	- 从服务器只有接收到主服务器的DEL命令，才会删除过期键

### 7.2. 数据库通知

redis命令对数据库进行修改后，服务器会根据配置向客户端发送数据库通知。

## 8. RDB 持久化

- SAVE 命令会阻塞Redis服务器进程
- BGSAVE 会派生子进程处理
- 如果开启了AOF，服务器会优先使用AOF文件来还原数据库状态。
- 服务器在载入 RDB 文件期间，会一直处于阻塞状态


## X. FK

https://baijiahao.baidu.com/s?id=1718316080906441023&wfr=spider&for=pc

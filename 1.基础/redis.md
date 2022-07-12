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

首先，random 随机结果 < threshold 的概率为 P
level 为 1 的概率为 1-P
level 为 2 的概率为 P(1-P)
level 为 3 的概率为 P^2(1-P)
level 为 4 的概率为 P^3(1-P)
......

一个节点的平均层数为：1(1-p) + 2P(1-P) +  3P^2(1-P) + 4P^3(1-P) + ... ≈ 1/(1-P)

所以，当 P = 1/4 时，每个节点所包含的平均指针数目为1.33 。这也是Redis里的skiplist实现在空间上的开销。

#### 4.1.2. 插入


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
https://blog.csdn.net/qq_42604176/article/details/110550937

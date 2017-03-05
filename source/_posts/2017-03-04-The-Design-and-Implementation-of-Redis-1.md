title: Redis设计与实现总结——数据结构与对象
date: 2017-03-04 11:44:38
tags:
---


### 基本数据结构
#### 简单的动态字符串 
Redis自己构建的一种名为简单动态字符串(simple dynamic string, SDS)的抽象类型，并将SDS用做Redis的默认字符串表示。
SDS的定义在`sds.h/sdshdr`结构中:
```c
struct sdshdr {
// 记录buf数组中已使用的字节的数量
// 等于SDS所保存字符串的长度
int len;

// 记录buf数组中未使用字节的数量
int free;

// 字节数组，用于保存字符串
char buf[];
}
```
![SDS结构](/assets/img/redis/redis_sds.png)
C字符串和SDS之间的区别:

|C字符串|SDS|
|-|-|
|获取字符串长度的复杂度为O(N)|获取字符串长度的复杂度为O(1)|
|API是不安全的，可能会造成缓冲区溢出|API是安全的，不会造成缓冲区溢出|
|修改字符串长度N次必然需要执行N次内存重分配|修改字符串长度N次最多需要执行N次内存重分配|
|只能保持文本数据|可以保持文本或者二进制数据|
|可以使用所有<string.h>库中的函数|可以使用一部分<string.h>库中的函数|


#### 链表
链表提供了高效的节点重排能力，以及顺序性的节点访问方式，并且可以通过增删节点来灵活地调整链表的长度。链表在redis链表键，发布与订阅，慢查询，监视器等功能都用到了。
链表结构分为链表和链表节点，每个链表由多个链表的节点组合而成。每个节点都有一个指向前置节点和后置节点的指针，所以Redis的链表实现是一个双端链表。表头节点和表尾节点都指向NULL, 是一个无环链表。保存链表值的类型是void, 可以保持不同类型的值。
```c
// 链表定义adlist/list
typedef struct list {
    // 表头节点
    listNode *head;
    
    // 表尾节点
    listNode *tail;
    
    // 链表所浩瀚的节点数量 
    unsigned long len;

    // 节点复制函数
    void *(*dup) (void *ptr);
    
    // 节点释放函数
    void (*free) (void *ptr);
    
    // 节点值对比函数
    int (*match) (void *ptr, void *key);
} list;

//链表节点定义adlist.h/listNode
typedef struct listNode {
    // 表头节点
    struct listNode *prev;
    
    // 表尾节点
    struct listNode *next;

    // 节点值
    void *value;
} listNode;
```

![list结构](/assets/img/redis/redis_list.png)

#### 字典
字典又称为符号表(symbol table), 关联数组(associative array)或映射(map),是一种用于保存键值对(key-value pair)的抽象数据结构。字典中的每一个键都是独一无二的。字典在Redis中应用相当广泛，比如Redis的数据库就是使用字典作为底层实现的，对数据库的CRUD也是建立在字典的操作上。字典还是哈希键的底层实现之一。
![dict结构](/assets/img/redis/redis_dict.png)
字典的结构如上图所示，字典是由多个结构连接而成，首先是字典结构`dict.h/dict`：
```c
typedef struct dict {
    
    // 类型特定函数, 保存了一簇用于操作特定类型键值对的函数,
    // Redis会为用途不同的字典设置不同的类型特定函数
    dictType *type;
    
    // 私有数据, 保存了需要传给那些类型特定的函数的可选参数
    void *privdata;

    // 哈希表, 注意这里hash表定义两个，其中一个是实际中使用的，
    // 另一个是在扩展或收缩的时候使用的，类似于GC复制算法的原理
    dictht ht[2];

    // rehash 索引, 记录rehash目前的进度
    // 当rehash不在进行时，值为-1
    int trehashidx;
} dict;
```
字典所使用的哈希表`dict.h/dictht`：
```c
typedef struct dictht {
    
    // 哈希表数组
    dictEntry **table;
    
    // 哈希表大小
    unsigned long size;

    // 哈希表大小掩码，用于计算索引值
    // 总是等于size-1
    unsigned long sizemark;    
    
    // 该哈希表已有节点的数量
    unsigned long used;
} dictht;
```
哈希表节点:
```c
typedef struct dictEntry {

    // 键
    void *key;
    
    // 值 
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;
    
    // 指向下个哈希表节点,形成链表
    // 使用next指针解决哈希冲突的问题
    // 哈希算法为MurmurHash2
    struct dictEntry *next;
} dictEntry;
```
随着操作的不断执行，哈希表保存的键值对会逐渐的增多或减少，为了让哈希负载因子(load factor)维持在一个合理的范围之内，当哈希表保持的键值对对数量太多或太少时，程序需要对哈希表的大小进行相应的扩展或收缩。
为了避免rehash对服务器性能造成影响，服务器不是一次性将`ht[0]`里面的所有键值对全部`rehash`到`ht[1]`，而是分多次，渐进式地将`ht[0]`里面的键值对慢慢地`rehash`到`ht[1]`。

#### 跳跃表
跳跃表(skiplist)是一种有序数据结构, 它通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。跳跃表支持平均O(longN)，最坏O(N)复杂度的节点查找，还可以通过顺序性操作来批量处理节点。更多介绍参考[wiki](https://zh.wikipedia.org/wiki/%E8%B7%B3%E8%B7%83%E5%88%97%E8%A1%A8)。
Redis使用跳跃表作为有序集合键的底层实现之一，如果一个有序集合包含的元素数量比较多，又或者有序集合中元素的成员是比较长的字符串时，Redis就会使用跳跃表来作为有序集合键的底层实现。跳跃表的另一个应用就是作为集群节点中的内部数据结构。除了这两个地方，其它地方没有用到。
跳跃表有`redis.h/zskiplistNode`和`redis.h/zskiplist`两个结构定义，其中`zskiplistNode`结构用于表示跳跃表节点，而`zskiplist`结构则用于保存跳跃表节点信息。
```c
/*
 * 跳跃表
 */
typedef struct zskiplist {

    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;

    // 表中节点的数量
    unsigned long length;

    // 表中层数最大的节点的层数
    int level;

} zskiplist;

/*
 * 跳跃表节点
 */
typedef struct zskiplistNode {

    // 成员对象
    robj *obj;

    // 分值
    double score;

    // 后退指针
    struct zskiplistNode *backward;

    // 层
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 跨度
        unsigned int span;

    } level[];

} zskiplistNode;
```
![skiplist结构](/assets/img/redis/redis_skiplist.png)
* 层: 每个层带有两个属性，前进指针和跨度。前进指针用于访问表尾方向的其节点，而跨度则记录了前进指针所指向节点和当前节点的距离。上图中连线数字上带有数字的箭头就代表前进指针，而那个数字就是跨度。当程序从表头向表尾进行遍历时，访问就会沿着层的前进指针进行。每次创建一个新跳跃表节点的时候，程序根据幂次定律(power law, 越大的数出现的概率越小)随机生成一个介于1和32之间的值作为level数组的大小，这个大小就是层的高度。
* 前进指针: 每个层都有一个指向表尾方向的前进指针(level[i].forward属性), 用于从表头向表尾方向的访问节点。
* 后退指针: 节点中用BW字样标记节点的后退指针，它指向位于当前节点的前一个节点。后退指针在程序从表尾向表头遍历时使用。
* 分值: 各个节点中的1.0，2.0和3.0是节点所保存的分值。在跳跃表中，节点各自所保存的分值从小到大排列。 跳跃表中的节点按照分值进行排序，当分值相同时，节点按照成员对象的大小进行排序。
* 成员对象: 各个节点中的o1, o2和o3是节点所保存的成员对象。

具体的操作过程参考[http://blog.csdn.net/ict2014/article/details/17394259](http://blog.csdn.net/ict2014/article/details/17394259)

#### 整数集合
整数集合(intset)是集合键的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合键的底层实现。
每个`intset.h/intset`结构表示一个整数集合:
```c
tyepdef struct intset {
    // 编码方式, 决定contents的类型
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;
    
    // 保持元素的数组
    int8_t contents[];
} intset;
```
![intset结构](/assets/img/redis/redis_intset.png)
`contents`数组是整数集合的底层实现: 整数集合的每个元素都是`contents`数组的一个数组项(item), 各个项在数组中按值的大小从小到大有序地排列，并且数组中不包含任何重复项。虽然`contetns`声明为`int8_t`类型的数组,但实际上`contents`并不保存任何`int8_t`类型的值，`contents`数组的真正类型取决于`encoding`属性的值。
每当我们要将一个新元素添加到整数集合里面，并且新元素的类型比整数集合现有所有元素的类型都要长时，整数集合需要先进行`升级(upgrade)`, 然后才能将新元素添加到整数集合里面。整数集合不支持`降级操作`, 一旦对数组进行了升级，编码就会一致保持升级后的状态。

#### 压缩列表
压缩列表(ziplist)是列表建和哈希键的底层实现之一。当一个列表建只包含少量列表项，并且每个列表项要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做列表建的底层实现。
压缩列表是Redis为了节约内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型(sequential)数据结构。一个压缩列表可以包含任意多个节点(entry),每个节点可以保持一个字节数组或一个整数值。
![ziplist结构](/assets/img/redis/redis_ziplist.png)
压缩列表各个组成部分的详细说明:

|属性|类型|长度|用途|
|--|--|--|--|
|zlbytes|uint32_t|4字节|记录整个压缩列表占用的内存字节数:在对压缩列表进行内存重分配,或者计算zlend的位置时使用|
|zltail|uint32_t|4字节|记录压缩列表表尾节点距离压缩列表的起始地址有多少字节:通过这个偏移量，程序无须遍历整个压缩列表就可以确定表尾节点的地址|
|zllen|uint16_t|2字节|记录了压缩列表包含的节点数量，当节点数小于UINT16_MAX时取这个值，大于时需要遍历列表才能得出|
|entryX|列表节点|不定|压缩列表包含的节点，节点的长度由节点保存的内容决定|
|zlend|uint8_t|1字节|特殊值`0xFF(十进制255)`，用于标记压缩列表的末端|

压缩列表的节点构成:
* `previous_entry_length`: 以字节为单位，记录了压缩列表中前一个节点的长度。只要我们拥有了一个指向某个节点的起始地址的指针，那么通过这个指针及这个节点的`previous_entry_length`属性，程序就可以一直向前一个节点回溯，最终达到压缩列表的表头节点。
* `encoding`: 记录了节点的`content`属性所保存数据的类型及长度。
* `content`: 负责保存节点的值，节点值可以使一个字节数组或整数，值的类型和长度由节点的`encoding`属性决定。

`连锁更新`问题是指当插入新节点或删除节点后，`previous_entry_length`属性所记录的长度不能够满足改变后的节点的记录，需要扩容以便记录，最差的情况是后面的每个节点都会改变位置。最差的复杂度为O(N^2)。但是这种情况很少见，一般复杂度为O(N)。

### 对象
前面介绍了Redis的主要数据结构，但是Redis并没有直接使用这些数据结构实现键值对数据库，而是基于这些数据结构创建了一个`对象系统`, 这个系统包含字符串对象，列表对象，哈希对象，结合对象和有序集合对象这五种类型的对象，每种对象都用到了至少一种我们面前所介绍的数据结构。我们可以针对不同的使用场景，为对象设置多种不同的数据结构实现，从而优化对象在不同场景下的使用效率，而这些对用户是透明的。
Redis的对象系统还实现了基于引用计数技术的内存回收机制(GC), 当程序不再是由某个对象的时候，这个对象所占用的内存就会被自动释放；另外Redis还通过引用计数法实现了对象共享机制，这一机制可以在适当的条件下，通过让多个数据库键共享同一个对象来节约内存。
每当我们在Redis数据库中新创建一个键值对时，我们至少会创建两个对象，一个对象用作键值对的键(键对象),另一个对象用作键值对的值(值对象)。
```c
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

使用`TYPE`命令可以看到对象的类型，对象的类型及type属性的值对应关系如下表:

|type类型常量|对象的名称|TYPE命令的输出|
|--|--|--|
|REDIS_STRING|字符串对象|"string"|
|REDIS_LIST|列表对象|"list"|
|REDIS_HASH|哈希对象|"hash"|
|REDIS_SET|集合对象|"set"|
|REDIS_ZSET|有序集合对象|"zset"|

每种TYPE对象的底层编码都是由上面说的数据结构组成的，使用`OBJECT ENCODING`命令可以查看一个数据库键的值对象的编码，具体的对应关系如下表:

|对象所使用的底层数据结构|编码常量|OBJECT ENCODING命令输出|
|--|--|
|整数|REDIS_ENCODING_INT|"int"|
|embstr编码的简单动态字符串(SDS)|REDIS_ENCODING_EMBSTR|"embstr"|
|简单动态字符串|REDIS_ENCODING_RAW|"raw"|
|字典|REDIS_ENCODING_HT|"hashtable"|
|双端链表|REDIS_ENCODING_LINKEDLIST|"linkedlist"|
|压缩列表|REDIS_ENCODING_ZIPLIST|"ziplist"|
|整数集合|REDIS_ENCODING_INTSET|"intset"|
|跳跃表和字典|REDIS_ENCODING_SKIPLIST|"skiplist"|

每种类型对象可以使用哪些数据结构，下面做了一个总结:

|对象类型|"int"|"embstr"|"raw"|
|--|--|--|--|
|"string"|如果字符串对象保存的是整数值，并且其可以用long类型来表示|如果字符串对象保持的是一个字符串值，并且其长度小于39字节|如果字符串对象保持的是一个字符串值，并且其长度大于39字节|

|对象类型|"linkedlist"|"ziplist"|
|--|--|--|
|"list"|不满足ziplist的条件的情况|同时满足:<br>1. 所有字符串元素的长度都小于64字节;<br>2. 元素数量小于512个|

|对象类型|"hashtable"|"ziplist"|
|--|--|--|
|"hash"|不满足ziplist的条件的情况|同时满足:<br>1. 哈希对象保存的所有键值对的键和值的字符串长度都小于64字节;<br>2. 哈希对象保存的键值对数量小于512个|

|对象类型|"intset"|"hashtable"|
|--|--|--|
|"set"|同时满足:<br>1. 集合对象保存的所有元素都是整数值；<br>2. 结合对象保存的元素数量不超过512个|不满足"intset"的条件的情况|

|对象类型|"ziplist"|"skiplist"|
|--|--|--|
|"zset"|同时满足:<br>1. 有序集合保持的元素数量小于128个；<br>2. 有序集合保持的所有元素成员的长度都小于64字节|不满足"ziplist"的条件的情况|

`redisObject`有一个`lru`属性,这个属性记录了对象最后一次被命令程序访问的时间,`OBJECT IDLETIME`命令可以打印出给定键的空转时长(当前时间-lru时间), 另外当开启`maxmemory`选项，并且服务器用于内存回收的算法为`volatile-lru`或者`allkeys-lru`，那么当服务器占用的内存数超过了`maxmemory`选项所设置的上限值时，空转时长较高的那部分键会优先被服务器释放，从而回收内存。

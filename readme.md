## Redis 

### 1、引言

Redis全称REmote DIctionary Server，是一个高性能key-value存储系统。

- Redis与其他key-value缓存产品（如memcache）有以下几个特点。 
- Redis支持数据的持久化，可以将内存中的数据保存在磁盘中，重启的时候可以再次加载进行使用。 
- Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储。 
- Redis支持数据的备份，即master-slave模式的数据备份。

Redis的性能极高且拥有丰富的数据类型，同时，Redis**所有操作都是原子性的**，也支持对几个操作合并后原子性的执行。

Redis更为优秀的地方在于，它的代码风格极其精简，整个源码只有23000行，很有利于阅读和赏析！

```bash
# 登录redis 并设置密码
root@ubuntu:/home/perry# redis-cli
127.0.0.1:6379> config get requirepass
1) "requirepass"
2) ""    # 默认情况下，redis 是没有登录密码的
127.0.0.1:6379> config set requirepass root
OK
127.0.0.1:6379> config get requirepass  # 设置成功后需要输入密码
(error) NOAUTH Authentication required.
127.0.0.1:6379> auth root
OK
127.0.0.1:6379> config get requirepass
1) "requirepass"
2) "root"
127.0.0.1:6379> quit
```



## Redis 基础

### 2、Redis 的简单动态字符串 SDS

redis 中的sds字符串相比于c语言中的字符串的区别：

​	（1）常数复杂度获取字符串长度

```go
/*
 * 类型别名，用于指向 sdshdr 的 buf 属性
 */
typedef char *sds; // 简单动态字符串（simple dynamic string）

/*
 * 保存字符串对象的结构
 */
struct sdshdr {
    
    // buf 中已占用空间的长度
    int len;

    // buf 中剩余可用空间的长度
    int free;

    // 数据空间
    char buf[];
};
```

​	通过SDS而不是C字符串，Redis将获取字符串长度所需的复杂度从O(N) 降低到了O(1)。这确保了获取字符串长度的工作不会成为Redis的性能瓶颈。

​	(2) 杜绝缓冲区溢出

​	C字符串不记录本身的长度很容易造成缓冲区溢出，如<string.h>/strcat 函数，可以将src字符串的内容拼接到dest字符串的末尾：

```cpp
char *strcat(char *dest, const char *src);
```

strcat 函数默认dest已经分配了足够的内存空间，可以容纳stc字符串中的所有内容，而当空间不够时，则会造成缓冲区溢出。

而SDS中因为有自己的空间分配策略，可以完全杜绝发送缓冲区溢出的可能性，SDS中也有一个执行拼接操作的sdscat函数，它可以将一个C字符串拼接到给定的SDS保存的字符串后面，在执行拼接操作前，sdscat会先检查给定SDS的空间是否足够，如果不够的话，sdscat会先扩展SDS的空间，然后再执行拼接操作。



​	(3) 减少修改字符串带来的内存重分配次数

​		空间预分配：当字符串增长时，需要对SDS的空间进行扩展，策略不仅会分配修改所必须的空间大小，还会为SDS分配额外的未使用空间。有两种情况：当对SDS修改后，SDS的长度小于1MB时，程序会为SDS再额外分配和len属性同样大小的未使用空间；

如果对SDS修改后，其长度大于1MB，那么程序会分配1MB的未使用空间。

​		惰性空间释放：当sds中的字符被删除之后，它并不会马上将空间释放掉，而是用free记录此时空闲的空间，当下次再需要扩展时就不需要再进行分配，而是直接使用。节省了空间释放申请的开销。

​		

​	(4)  二进制安全

​		因为有len的属性参数，可以保证使用sds时直到它的具体长度，所以buff中存放的就不再像c字符串一样只是文本数据了，也可以是图片、音视频等的二进制文件了。不再有c字符串中不能包含空字符会被误认为是字符串末尾的情况了。

​	(5) 兼容部分C字符串函数

​		可以直接使用c字符串中的一些函数，而不需要再单独去实现。因为sds最后也是和c字符串一样是空字符结尾的。

**总结**

![image-20210626135133925](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210626135133925.png)

##### 阅读源码中的疑问

```go
/*
 * 返回 sds 实际保存的字符串的长度
 * T = O(1)
 */
static inline size_t sdslen(const sds s) {
    struct sdshdr *sh = (void*)(s-(sizeof(struct sdshdr))); // 这一句看不懂！！！！
    return sh->len;
}

// 1、首先通过sds结构体的定义：
typedef char *sds; // 简单动态字符串（simple dynamic string）
struct sdshdr {
    int len;
    int free;
    char buf[];
};
// 其中buf[]数组是没有大小的，是柔性数组，是不占据内存的，所以sizeof(struct sdshdr) 的大小为8；
// 2、紧接着，我们看一下这个结构的初始化，在src/sds.c中：
	// 关键是最后 这个函数返回的是指向buf的char* 类型的sds。
/*
 * 根据给定的初始化字符串 init 和字符串长度 initlen
 * 创建一个新的 sds
 *
 * 参数
 *  init ：初始化字符串指针
 *  initlen ：初始化字符串的长度
 *
 * 返回值
 *  sds ：创建成功返回 sdshdr 相对应的 sds
 *        创建失败返回 NULL
 *
 * 复杂度
 *  T = O(N)
 */
sds sdsnewlen(const void *init, size_t initlen) {
    struct sdshdr *sh;
    // 根据是否有初始化内容，选择适当的内存分配方式
    // T = O(N)
    if (init) {
        // zmalloc 不初始化所分配的内存
        sh = zmalloc(sizeof(struct sdshdr)+initlen+1);
    } else {
        // zcalloc 将分配的内存全部初始化为 0
        sh = zcalloc(sizeof(struct sdshdr)+initlen+1);
    }

    // 内存分配失败，返回
    if (sh == NULL) return NULL;

    // 设置初始化长度
    sh->len = initlen;
    // 新 sds 不预留任何空间
    sh->free = 0;
    // 如果有指定初始化内容，将它们复制到 sdshdr 的 buf 中
    // T = O(N)
    if (initlen && init)
        memcpy(sh->buf, init, initlen);
    // 以 \0 结尾
    sh->buf[initlen] = '\0';

    // 返回 buf 部分，而不是整个 sdshdr
    return (char*)sh->buf;
}

```

**此时分析下结构体的内存情况：**

sds指向buf，之前的空间正好是(sizeof(struct sdshdr))，

当 (s-(sizeof(struct sdshdr))) 之后，正好就指到了结构体的同步。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190322141830306.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQzMDM2NDc=,size_16,color_FFFFFF,t_70)

> 柔性数组：
>
> 结构体中最后一个元素允许是未知大小的数组，这个数组就是柔性数组。但结构中的柔性数组前面必须至少一个其他成员,柔性数组成员允许结构中包含一个大小可变的数组，sizeof返回的这种结构大小不包括柔性数组的内存。包含柔数组成员的结构用malloc函数进行内存的动态分配,且分配的内存应该大于结构的大小以适应柔性数组的预期大小。



##### 比较有代表性的惰性空间释放源码

```go
/*
 * 在不释放 SDS 的字符串空间的情况下，
 * 重置 SDS 所保存的字符串为空字符串。
 * 直接修改free和len属性，再将空字符 '\0'直接放到buf[0]，很简洁
 *
 * 复杂度
 *  T = O(1)
 */
void sdsclear(sds s) {

    // 取出 sdshdr
    struct sdshdr *sh = (void*) (s-(sizeof(struct sdshdr)));

    // 重新计算属性
    sh->free += sh->len;
    sh->len = 0; 

    // 将结束符放到最前面（相当于惰性地删除 buf 中的内容）
    sh->buf[0] = '\0'; 
}
```

##### 空间预分配的源码

```go
/*
 * 对 sds 中 buf 的长度进行扩展，确保在函数执行之后，
 * buf 至少会有 addlen + 1 长度的空余空间
 * （额外的 1 字节是为 \0 准备的）
 *
 * 返回值
 *  sds ：扩展成功返回扩展后的 sds
 *        扩展失败返回 NULL
 * 复杂度
 *  T = O(N)
 */
sds sdsMakeRoomFor(sds s, size_t addlen) {
    struct sdshdr *sh, *newsh;
    // 获取 s 目前的空余空间长度
    size_t free = sdsavail(s);
    size_t len, newlen;
    // s 目前的空余空间已经足够，无须再进行扩展，直接返回
    if (free >= addlen) return s;
    
    // 获取 s 目前已占用空间的长度
    len = sdslen(s);
    sh = (void*) (s-(sizeof(struct sdshdr)));
    // s 最少需要的长度
    newlen = (len+addlen);

    // 根据新长度，为 s 分配新空间所需的大小
    // sds.h 文件中已经声明，#define SDS_MAX_PREALLOC (1024*1024)
    if (newlen < SDS_MAX_PREALLOC)
        // 如果新长度小于 SDS_MAX_PREALLOC 
        // 那么为它分配两倍于所需长度的空间
        newlen *= 2;
    else
        // 否则，分配长度为目前长度加上 SDS_MAX_PREALLOC
        newlen += SDS_MAX_PREALLOC;
    // T = O(N)
    newsh = zrealloc(sh, sizeof(struct sdshdr)+newlen+1);

    // 内存不足，分配失败，返回
    if (newsh == NULL) return NULL;

    // 更新 sds 的空余长度
    newsh->free = newlen - len;

    // 返回 sds
    return newsh->buf;
}
```

### 3、链表 linked list

链表节点的定义：

**每个节点都是一个双向链表，而这一串连着的双向链表又是通过一个list结构体来管理的.**

```go
/* Node, List, and Iterator are the only data structures used currently. */

/*
 * 双端链表节点
 */
typedef struct listNode {

    // 前置节点
    struct listNode *prev;

    // 后置节点
    struct listNode *next;

    // 节点的值
    void *value;

} listNode;

/*
 * 双端链表迭代器
 */
typedef struct listIter {

    // 当前迭代到的节点
    listNode *next;

    // 迭代的方向
    int direction;

} listIter;


```

**通过list 这个双端链表结构，可以更好的管理整理链表，其中记录了整个链表的头指针head，尾指针tail，以及链表长度计数器len，dup、free和match成员则是用于实现多态链表所需要的类型特定函数。**

```go
/*
 * 双端链表结构
 */
typedef struct list {

    // 表头节点
    listNode *head;

    // 表尾节点
    listNode *tail;

    // 节点值复制函数
    void *(*dup)(void *ptr);

    // 节点值释放函数
    void (*free)(void *ptr);

    // 节点值对比函数
    int (*match)(void *ptr, void *key);

    // 链表所包含的节点数量
    unsigned long len;

} list;
```

简单的示意图：

<img src="C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210626163806971.png" alt="image-20210626163806971" style="zoom: 50%;" />



小结：
![image-20210626163852846](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210626163852846.png)

### 4、字典（map、映射、关联数组、符号表）

Redis的字典使用哈希表作为底层实现，一个哈希表中可以有多个哈希表节点，而每个哈希表节点就保存了字典中的要给键值对。



**字典底层既然是哈希表，我们先看看哈希表是如何设计的：**

```go
/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
/*
 * 哈希表
 *
 * 每个字典都使用两个哈希表，从而实现渐进式 rehash 。
 */
typedef struct dictht {
    
    // 哈希表数组
    dictEntry **table; // 二维数组存放的是 指向哈希节点dictEntry 的指针

    // 哈希表大小，即table数组的大小
    unsigned long size;
    
    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;

} dictht;


/*
 * 哈希表节点
 */
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
    struct dictEntry *next; // 用来解决键冲突的问题

} dictEntry;

```

下图是个简单的例子，有助于理解dictht和dictEntry及其中元素的关系：

![image-20210627141518315](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210627141518315.png)

介绍完哈希表和哈希表节点后，紧接着来看看**字典的结构**：

```go
/*
 * 字典
 */
typedef struct dict {

    // 类型特定函数
    dictType *type; // 指向dictType 结构，每个dictType 结构保存了一簇用于操作特定类型键值对的函数
					// redis 会为用途不同的字典设置不同的类型特定函数
    // 私有数据
    void *privdata; // 保存了需要传给哪些类型特点函数的可选参数

    // 哈希表
    dictht ht[2]; // ht属性是一个包含两个项的数组，数组中的每个项都是一个dictht哈希表，
    			  // 一般情况下字典只是用ht[0]哈希表，ht[1]哈希表只会在对ht[0]哈希表进行rehash时使用
    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 ， 记录了rehash的进度*/

    // 目前正在运行的安全迭代器的数量
    int iterators; /* number of iterators currently running */

} dict;

// dictType 结构定义如下
/*
 * 字典类型特定函数
 ？？？？？？ 这些函数指针是如何实现的？？？？？
 */
typedef struct dictType {

    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key); // 函数指针，指针指向的是函数而非对象。

    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);

    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);

    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);

    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);
    
    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);

} dictType;

```

一个普通状态下的字典的结构图：

![image-20210627222218444](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210627222218444.png)

**哈希算法：**如何将一个新的键值对添加到字典里面

先根据键值对的键计算出哈希值和索引值，然后根据索引值将包含键值对的哈希节点放到哈希表数组的指定索引上。

redis 计算哈希值和索引如下：

```c++
// 使用字典设置里的哈希函数计算键 key 的哈希值
hash = dict->type->hashFunction(key);
// 使用哈希表的sizemask属性和哈希值，计算出索引值
// 根据情况不同，ht[x] 可以是ht[0]或者ht[1]
index = hash & dict->ht[x].sizemask;
```

一个添加了键值对之后的字典如下：

![image-20210628200325971](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210628200325971.png)

**解决键冲突**

Redis解决键冲突使用的是 **链地址法**，每个哈希表节点都有一个next指针，被分配到同一个索引上的多个节点可以使用这个单向链表连接起来，解决了冲突的问题。

![image-20210628200648531](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210628200648531.png)

**rehash**：就是对哈希表的大小进行相应的扩展或者收缩。

什么条件下，程序会需要进行rehash来对哈希表进行扩展呢

- 服务器目前没有执行 BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1；
- 服务器目前正在执行 BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5；

什么条件下，程序会需要进行rehash来对哈希表进行收缩呢

### 5、跳跃表 

跳跃表是一种 **有序数据结构**，通过在每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。

支持**平均O(logN)，最坏O(N)** 复杂度的节点查找，还可以通过顺序性操作来批量处理节点。

redis使用调表作为有序集合的底层实现之一，如果一个有序集合包含的元素数量比较多，又或者有序集合中元素的成员是比较长的字符串时，redis就会使用跳跃表来作为有序集合键的底层实现。

和链表、字典等数据结构被广泛的应用在redis内部不同，redis只在两个地方用到了跳跃表，一个是实现有序集合键，另一个是在集群节点中用作内部数据结构，除此之外，跳跃表在redis中没有其他的用途。

![image-20210630230838200](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210630230838200.png)

下面是源码中zskiplist结构体的具体实现：

```c++
/*
 * 跳跃表: 图中最左边的结构体
 */
typedef struct zskiplist {

    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;

    // 表中节点的数量，即跳跃表的长度（表头节点不计算在内）
    unsigned long length;

    // 表中层数最大的节点的层数
    int level;

} zskiplist;

```

zskiplist指向的即为zskiplistNode：

```c++
/* ZSETs use a specialized version of Skiplists */
/*
 * 跳跃表节点
 */
typedef struct zskiplistNode {

    // 成员对象，指向一个字符串对象，该字符串对象中保存一个SDS值。
    robj *obj; // 必须唯一

    // 分值，所有节点都是按照分值从小到大排序。
    double score; // 可以相同

    // 后退指针，每次只能后退至前一个节点
    struct zskiplistNode *backward;

    // 层，level数组可以包含多个元素，层数越多访问其他节点的速度越快
    struct zskiplistLevel {

        // 前进指针，从表头向表尾访问元素
        struct zskiplistNode *forward;

        // 跨度，用于记录两个节点之间的距离。主要是用来计算排位的（rank）：
        // 在查找某个节点的过程中，将沿途访问过的所有层的跨度累计起来，得到的结果就是目标节点在跳跃表中的排位
        unsigned int span;

    } level[];

} zskiplistNode;

// 成员对象中的robj 结构体的定义
/* A redis object, that is a type able to hold a string / list / set */
/* The actual Redis Object */
/*
 * Redis 对象
 */
#define REDIS_LRU_BITS 24
#define REDIS_LRU_CLOCK_MAX ((1<<REDIS_LRU_BITS)-1) /* Max value of obj->lru */
#define REDIS_LRU_CLOCK_RESOLUTION 1000 /* LRU clock resolution in ms */
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

需要指出的是表头节点，也是和其他的节点一样有后退指针、分值

成员对象等，但是这些属性不会用到，所以图中省略了。

**如下是由多个跳跃节点组成的跳跃表**

![image-20210701004726728](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210701004726728.png)

### 6、整数集合

intset整数集合是集合键的底层实现之一，当一个集合只包括整数值元素，并且这个集合的元素数量不多时，redis就会使用整数集合来作为集合键的底层实现。

如下是inset结构体的定义：

```go
typedef struct intset {
    
    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组，类型并不是int8_t的，真正存储的数据类型由 encoding 来决定
    int8_t contents[];

} intset;
// 常用的一些接口函数如下：
intset *intsetNew(void);
intset *intsetAdd(intset *is, int64_t value, uint8_t *success);
intset *intsetRemove(intset *is, int64_t value, int *success);
uint8_t intsetFind(intset *is, int64_t value);
int64_t intsetRandom(intset *is);
uint8_t intsetGet(intset *is, uint32_t pos, int64_t *value);
uint32_t intsetLen(intset *is);
size_t intsetBlobLen(intset *is);
```

contents数组是整数集合的底层实现，各个项在数组中是按值的大小从小到大有序排列的，并且数组中不包含任何的重复项。

**升级**：当需要将一个新的元素添加到整数集合中时，如果新元素的类型比整数集合现有所有元素的类型都要长时，就需要进行升级，只有升级之后才能将新元素添加到整数集合中去。

升级的分三步：

​	（1）根据新元素的类型，扩展整个整数集合底层数组的空间大小。（包括了新元素需要的空间）

​	（2）将底层数组中所有的元素转换成 与新元素相同的类型，并将类型转换后的元素放置到正确的位置（找位置用的是二分法，移动元素用的是memove函数）

​	（3）将新元素添加到底层数组中

**升级的好处：**

- 提升灵活性：C语言是静态语言，不会将不同类型的值放在同一个数据结构中，整数集合可以通过升级来适应新元素，很灵活。

- 节约内存：C中如果要让一个数组同时存放几种类型的数据，一般是将最大的类型设为底层类型，存放较小类型的元素也是一样，但是整数集合的升级操作，既可以让集合同时保存多种不同类型的值，还能够确保升级操作只会在需要的时候进行，可尽量节约内存。

**整数集合不支持降级操作**



### 7、压缩列表

压缩表是为了 **节约内存**开发的。

被用作 **列表键**和 **哈希键**的底层实现之一。

![image-20210702191210022](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210702191210022.png)

压缩列表可以包含多个节点，每个节点可以保存一个字节数组或者整数值。

添加新节点到压缩列表，或者从压缩列表中删除节点，可能会引发连锁更新操作，但这种操作出现的几率不高。



### 8、对象

redis 使用对象来表示数据库中的键和值，当在redis的数据库中创建一个键值对时，**至少会创建两个对象**，一个对象用作键值对的键（**键对象**），另一个用作键值对的值（**值对象**）。

redis 中的每个对象由一个 **redisObject** 结构表示，对于redis保存的键值对来说，键总是一个字符串对象，而值则可以是 字符串对象、列表对象、哈希对象、集合对象 或 有序集合对象 中的一种。

```c++
/* A redis object, that is a type able to hold a string / list / set */
/* The actual Redis Object */
/*
 * Redis 对象
 */
#define REDIS_LRU_BITS 24
#define REDIS_LRU_CLOCK_MAX ((1<<REDIS_LRU_BITS)-1) /* Max value of obj->lru */
#define REDIS_LRU_CLOCK_RESOLUTION 1000 /* LRU clock resolution in ms */
typedef struct redisObject {

    // 类型，说的都是数据库键值对中的 值对象。type属性可以是 8-1表中列出的一个
    unsigned type:4;

    // 编码
    unsigned encoding:4;

    // 对象最后一次被访问的时间
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */

    // 引用计数
    int refcount;

    // 指向实际值的指针，指向底层实现的数据结构. 由encoding属性来决定
    void *ptr;

} robj;
```

![image-20210706133156462](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706133156462.png)

**当我们使用 TYPE 命令时，返回的是 数据库键对象的值对象类型，而不是键对象类型：**

![image-20210706133409521](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706133409521.png)



不同类型值对象的**TYPE**命令输出：

![image-20210702193000571](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210702193000571.png)



```c++

/* Object types */
// 对象类型
#define REDIS_STRING 0
#define REDIS_LIST 1
#define REDIS_SET 2
#define REDIS_ZSET 3

/* Objects encoding. Some kind of objects like Strings and Hashes can be
 * internally represented in multiple ways. The 'encoding' field of the object
 * is set to one of this fields for this object. */
// 对象编码
#define REDIS_ENCODING_RAW 0     /* Raw representation 简单动态字符串 */
#define REDIS_ENCODING_INT 1     /* Encoded as integer. long类型的整数*/
#define REDIS_ENCODING_HT 2      /* Encoded as hash table */
#define REDIS_ENCODING_ZIPMAP 3  /* Encoded as zipmap */
#define REDIS_ENCODING_LINKEDLIST 4 /* Encoded as regular linked list 双端链表*/
#define REDIS_ENCODING_ZIPLIST 5 /* Encoded as ziplist  压缩列表*/
#define REDIS_ENCODING_INTSET 6  /* Encoded as intset 整数集合*/
#define REDIS_ENCODING_SKIPLIST 7  /* Encoded as skiplist 跳跃表和字典*/
#define REDIS_ENCODING_EMBSTR 8  /* Embedded sds string encoding. embstr编码的简单动态字符串*/
```

**每种类型的对象都至少使用了两种不同的编码：**（下表是每种类型的 对象可以使用的编码）

![image-20210702193959497](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210702193959497.png)

**不同编码对象对应的OBJECT ENCODING 命令输出：**

![image-20210702194341726](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210702194341726.png)

![image-20210702194358838](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210702194358838.png)



#### 字符串对象

字符串对象编码可以是 int 、 raw 、embstr。

- 如果一个字符串对象保存的是整数值，并且这个整数值可以用 long 类型来表示，字符串对象的编码会设为int，这个整数值会被保存在字符串对象结构的ptr属性里面（将void* 转换为long）。

- 如果字符串对象保存的是一个字符串值，并且长度大于32字节，那么字符串对象使用SDS（简单动态字符串）来保存这个字符串值，并将对象的编码设置为raw。

- 如果字符串对象保存的是一个字符串值，并且长度小于等于32字节，那么字符串对象使用embstr编码方式来保存这个字符串值。

  > （embstr编码是专门用于保存短字符串的一种优化编码方式，这种编码方式和raw编码一样，都使用redisObject结构和sdshdr结构来表示字符串对象。但不同的是，raw编码会调用两次内存分配函数来分别创建redisObject结构和sdshdr结构，而embstr编码则通过调用一次内存分配函数来分配一块连续的空间，空间中依次包含redisObject和sdshdr两个结构）
  >
  > embstr编码的有点：
  >
  > **创建字符串对象时从内存中分配空间相对raw编码只需要一次**
  >
  > **释放字符串对象同样也只需要调用一次内存释放函数**
  >
  > **因为字符串对像所有数据保存在连续的内存空间，相比raw编码更能利用缓存带来的优势。**

##### 编码转换

int和embstr编码的字符串对象在一定条件下会被 **转换为raw编码的字符串对象。**

- 对int编码对象执行一定的操作，使其保存的不再是整数值而是字符串值，此时字符串对象的编码将从int变为raw。如下：

![image-20210702203829793](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210702203829793.png)

- redis 没有为 embstr编码的字符串对象编写任何相应的修改程序（只有int和raw编码对象有），所以embstr实际上是只读的。当对embstr编码字符串对象进行修改时，程序会先将对象的编码从embstr转为raw，然后再执行修改命令。如下：

  ![image-20210702204232563](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210702204232563.png)



- [ ] redis 中用long double 类型表示的浮点数 也是作为 **字符串值保存的**。如果，保存一个浮点数到字符串对象里，程序会先将这个浮点数转换为字符串值，然后再保存转换所得的字符串值。

![image-20210706140320379](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706140320379.png)



##### 字符串命令的实现

![image-20210702204344378](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210702204344378.png)



#### 列表对象

列表对象的编码可以是 **ziplist 或 linkedlist**。

- ziplist 编码的 列表对象：

```bash
redis> RPUSH numbers 1 "three" 5
(integer) 3
```



![image-20210706141013774](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706141013774.png)

- linkedlist 编码的列表对象：



![image-20210706141147443](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706141147443.png)

##### **编码转换**

当列表对象同时满足以下两个条件时 ，列表对象使用ziplist 编码：

- [ ] ​	列表对象保存的所有字符串元素的长度都小于64字节

- [ ] ​    列表对象保存的元素数量小于512个

  不能满足这个两个对象的列表对象需要使用linkedlist 编码。（**两个条件的上限可以通过配置文件修改**）

当使用ziplist 编码时，两个条件的任意一个不能满足时，对象的编码转换就会被执行。保存在压缩列表中的所有列表都会被移动到双端链表里，对象的编码也会从ziplist转变为linkedlist。



##### **列表命令的实现**

![image-20210706141902151](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706141902151.png)



#### 哈希对象

哈希对象的编码可以是ziplist或者hashtable。

- ziplist 编码的哈希对象使用 压缩列表作为底层实现，新的键值对加入哈希对象时，程序会 依次 保存键的压缩列表节点 和值的压缩列表节点到 压缩列表**表尾**，

  ```bash
  redis> HSET profile name "Tom"
  (interger) 1
  redis> HSET profile age 25
  (interger) 1
  redis> HSET profile career "Programmer"
  (interger) 1
  ```

  ![image-20210706143616329](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706143616329.png)

- hashtable 编码的哈希对象使用字典作为底层实现，哈希对象中的每个键值对都是用一个字典键值对来保存：

  ![image-20210706143750084](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706143750084.png)

##### 编码转换

当哈希对象同时满足以下两个条件时 ，哈希对象使用ziplist 编码：

- [ ] ​	哈希对象保存的所有键值对的键和值的字符串长度都小于64字节；

- [ ] ​    哈希对象保存的键值对数量小于512个

  否则使用hashtable 编码来实现哈希对象。

当使用ziplist 编码时，两个条件的任意一个不能满足时，对象的编码转换就会被执行。保存在压缩列表中的所有键值对都会被移动到字典里，对象的编码也会从ziplist转变为hashtable。

```bash
# 键值都不超过64个字节
127.0.0.1:6379> Hset book name "Mastering C++ in 1 days"
(integer) 1
127.0.0.1:6379> OBJECT encoding book
"ziplist"
127.0.0.1:6379> Hset book description_gogoogogog_longlonglonglllllllllllllllllllllllllllllllllllll "C++"
(integer) 1
127.0.0.1:6379> object encoding book
"hashtable" # 编码已经改变
127.0.0.1:6379>
# 通过改变 value 值来进行类型转换
127.0.0.1:6379> HSET blah greeting "hello world"
(integer) 1
127.0.0.1:6379> OBJECT encoding blah
"ziplist"
127.0.0.1:6379> HSET blah story "my longlonglonglonglnglong strin string strng strjing string string strin ddddddddddddddddddddddddddddddddddd"
(integer) 1
127.0.0.1:6379> OBJECT encoding blah
"hashtable"
# 因键值对数量过多，引起的编码转换
# 创建一个包含512个键值对的哈希对象
127.0.0.1:6379> EVAL "for i=1, 512 do redis.call('HSET', KEYS[1], i, i) end" 1 "numbers"
(nil)
127.0.0.1:6379> HLEN numbers
(integer) 512
127.0.0.1:6379> object encoding numbers
"ziplist"
# 此时类型依旧是 ziplist，再向哈希对象添加一个新的键值对，使得键值对数量变成 513个，再来看编码方式
127.0.0.1:6379> HMSET numbers "key" "value"
OK
127.0.0.1:6379> HLEN numbers
(integer) 513
127.0.0.1:6379> OBJECT encoding numbers
"hashtable" # 编码改变了
127.0.0.1:6379>
```

##### 哈希命令的实现

![image-20210706152010186](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706152010186.png)



#### 集合对象

集合对象的编码可以是 intset 或 hashtable。

- intset 编码的集合对象是用整数集合作为底层实现，集合对象包含的元素都保存在整数集合里。

- hashtable 编码的集合对像使用字典作为底层实现，字典的每个键都是一个字符串对象，每个字符串对象包含一个集合元素，而字典的值则全部被设置为NULL。

```bash
127.0.0.1:6379> SADD numbers 1 3 5
(integer) 3
127.0.0.1:6379>
127.0.0.1:6379> SADD fruits "apple" "banana" "cherry"
(integer) 3
127.0.0.1:6379>
```

![image-20210706152745870](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706152745870.png)

##### 编码转换

当集合对象同时满足以下两个条件时 ，集合对象使用intset编码：

- [ ] ​	集合对象保存的所有元素都是整数值；

- [ ] ​    集合对象保存的元素数量不超过512个。

  否则使用hashtable 编码来实现集合对象。

```bash
127.0.0.1:6379> dbsize
(integer) 0
127.0.0.1:6379> SADD numbers 1 3 5
(integer) 3
127.0.0.1:6379> OBJECT encoding numbers
"intset"
127.0.0.1:6379> SADD numbers "seven" # 增加一个字符串元素到 只包含整数元素的集合对象numbers中
(integer) 1
127.0.0.1:6379> OBJECT encoding numbers
"hashtable"  # 编码转换为hashtable
127.0.0.1:6379>

# 创建一个包含 512 个整数元素的集合对象
127.0.0.1:6379> EVAL "for i=1, 512 do redis.call('SADD', KEYS[1], i) end" 1 intergers
(nil)
127.0.0.1:6379> SCARD intergers
(integer) 512
127.0.0.1:6379> OBJECT encoding intergers
"intset"
127.0.0.1:6379> SADD intergers 10086
(integer) 1
127.0.0.1:6379> SCARD intergers
(integer) 513
127.0.0.1:6379> OBJECT encoding intergers
"hashtable"
127.0.0.1:6379>
```

##### 集合命令实现

![image-20210706155451462](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706155451462.png)

![image-20210706155503219](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706155503219.png)



#### 有序集合对象

有序集合的编码可以是ziplist 或skiplist。

- ziplist 编码的有序集合对象使用压缩列表作为底层实现，每个集合元素使用两个紧挨在一起的压缩列表节点来保存，第一个节点保存元素的成员（member），而第二个元素则保存元素的分值（scores）。

  压缩列表内的集合元素按分值从小到大进行排序，分值较小的元素被放置在靠近表头的方向，而分值较大的元素放置在靠近表尾的方向。

  ```bash
  127.0.0.1:6379> ZADD price 8.5 apple 5.0 banana 6.0 cherry
  (integer) 3
  ```

  ![image-20210706160806955](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706160806955.png)

- skiplist 编码实现的有序集合使用zset结构作为底层实现，一个zset结构同时包含一个字典和一个跳跃表：

  ```c++
  typedef struct zset {
      zskiplist *zsl;
      dict *dict;
  }zset;
  ```

  - zset结构中的zsl跳跃表按分值从小到大保存了所有集合元素，每个跳跃表节点都保存了一个集合元素：跳跃表节点的object属性保存了元素的成员，score属性则保存了元素的分值。通过这个跳跃表，程序可以对有序集合进行范围型操作，如 ZRANK、ARANGE 等命令就是基于跳跃表的API实现的。
  - zset结构中的dict字典为有序集合创建了一个从成员到分值的映射，字典中的每个键值对都保存了一个集合元素：字典的键保存了元素的成员，值则保存了元素的分值。通过字典，可以O(1)复杂度查找给定成员的分值，ZSCORE 命令就是根据这一特性实现的，很多其他有序集合命令都在实现内部用到了这个特性。
  - 有序集合每个元素的成员都是一个字符串对象，而每个元素的分值都是一个double类型的浮点数。**虽然zset结构同时使用了跳跃表和字典来实现，但是两种数据结构都会通过指针来共享相同元素的成员和分值，所以这样的实现策略并没有产生额外的重复成员或分值，也没有浪费额外的内存。**

##### 为什么有序集合同时使用跳跃表和字典来实现

![image-20210706162658886](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706162658886.png)

**如下，为skiplist 编码的有序集合对象：**

![image-20210706162808001](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706162808001.png)

![image-20210706162824641](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706162824641.png)

##### 编码转换

当有序集合对象同时满足以下两个条件时 ，有序集合对象使用ziplist编码：

- [ ] ​	有序集合对象保存的元素数量小于128个；

- [ ] ​    有序集合对象保存的所有元素成员的长度都小于64个字节。

  否则使用skiplist 编码来实现有序集合对象。

```bash
# 因元素过多引起的编码转换
127.0.0.1:6379> EVAL "for i=1, 128 do redis.call('ZADD', KEYS[1], i, i) end" 1 numbers
(nil)
127.0.0.1:6379> ZCARD numbers
(integer) 128 # 128 个元素时为 ziplist 编码
127.0.0.1:6379> OBJECT encoding numbers
"ziplist"
127.0.0.1:6379> ZADD numbers 3.14 pi
(integer) 1
127.0.0.1:6379> ZCARD numbers
(integer) 129
127.0.0.1:6379> OBJECT encoding numbers
"skiplist" # 编码方式转换为 skiplist
127.0.0.1:6379>

# 因元素成员过长引起的编码转换
127.0.0.1:6379> ZADD blah 1.0 www
(integer) 1
127.0.0.1:6379> OBJECT encoding blah
"ziplist"
# 添加一个成员超过 64字节的元素
127.0.0.1:6379> ZADD blah 2.0 tttttttttttttttttttttttttttttttttttttttttttttttttttttttttttttttttttttttttttttttt
(integer) 1
127.0.0.1:6379> OBJECT encoding blah
"skiplist"
127.0.0.1:6379>
```

##### 有序集合命令的实现

![image-20210706163915308](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210706163915308.png)

#### 类型检查与命令多态

redis中操作键的命令基本 分为 两大类型。

- 一种是对任何类型键都能执行，如：DEL、EXPIRE、RENAME、TYPE、OBJECT命令等。

- 另一种则是，只能对特定的类型的键执行：

  ![image-20210707101914626](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210707101914626.png)

```bash
127.0.0.1:6379> SET msg "hello"
OK
127.0.0.1:6379> RPUSH numbers 1 2 3 4
(integer) 4
127.0.0.1:6379> SADD fruits apple banana cherry
(integer) 3
127.0.0.1:6379> dbsize
(integer) 4
127.0.0.1:6379> DEL msg
(integer) 1
127.0.0.1:6379> DEL numbers
(integer) 1
127.0.0.1:6379> DEL fruits
(integer) 1
127.0.0.1:6379> SET msg "hello redis"
OK
127.0.0.1:6379> get msg
"hello redis"
127.0.0.1:6379> append msg "2021"
(integer) 15
127.0.0.1:6379> LLEN msg  # LLEN 是只有列表键才能执行的命令（此处执行进行了 类型检查）
(error) WRONGTYPE Operation against a key holding the wrong kind of value
127.0.0.1:6379>

```

##### 类型检查的实现

redis 在执行命令前，会先检查输入键的类型是否正确，然后再决定是否执行给定的命令。

类型检查是通过 **redisObject 结构的 type 属性来实现的**：

![image-20210707102432074](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210707102432074.png)

##### 多态命令的实现

redis 除了会根据值对象的类型来判断键是否执行指定命令之外，还会根据对象的编码方式，先择正确的命令实现代码来执行命令。

如：列表对象的编码可以是 ziplist 或 linkedlist 两种，如果我们使用LLEN命令，服务器不仅需要确保执行命令的是列表键，还要根据键的值对象所用的编码方式来选择正确的LLEN命令实现。

![image-20210707145232030](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210707145232030.png)



DEL 、EXPIRE 、TYPE等命令 和LLEN命令的区别：

前者是基于类型的多态，即一个命令可以同时用于处理多种不同类型的键；

后者则是基于编码的多态，即一个命令可以同时用于处理多种不同的编码。



#### 内存回收

因为c语言没有自动回收内存的功能，redis在自己的对象系统中使用**引用计数（reference counting）** 技术实现了内存回收机制。程序通过跟踪对象的引用计数信息，在适当的时候自动释放对象并进行内存回收。

```c
/* The actual Redis Object */
/*
 * Redis 对象
 */
typedef struct redisObject {
    // 类型，说的都是数据库键值对中的 值对象。type属性可以是 8-1表中列出的一个
    unsigned type:4;
    // 编码
    unsigned encoding:4;
    // 对象最后一次被访问的时间
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
    // 引用计数: 随着对象的使用状态不断改变
    int refcount;
    // 指向实际值的指针，指向底层实现的数据结构. 由encoding属性来决定
    void *ptr;

} robj;
```

引用计数改变的几种情况：

- 在创建一个新对象时，引用计数的值会被初始化为1；
- 当对象被一个新程序使用时，引用计数加1；
- 当对象不再被一个程序使用时，引用计数减1；
- 当对象的引用计数变为0时，对象所占的内存会被释放。

**修改引用计数的API：**

![image-20210707150342141](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210707150342141.png)

对象的整个声明周期可以分为： **创建对象、 操作对象、释放对象 三个阶段**。

如下，是一个字符串对象从创建到释放的整个过程

```c
// 创建一个字符串对象s，对象的引用计数为1
robj *s = createStringObject(...)
// 对象s执行各种操作...
    
// 将对象s的引用计数减1，使得对象的引用计数变为0
// 导致对象s被释放
decreRefCount(s)
```



#### 对象共享

redis的对象引用计数属性除了可以实现**内存回收机制**外，还能实现**对象共享**的作用。

如下例，键A创建一个包含整数值100的字符串对象作为值对象，如果此时键B也要创建一个同样保存了整数值100的字符串对象作为值对象，redis会将数据库键的值指针指向一个现有的值对象，被共享的值对象的引用计数加1。

![image-20210707151613103](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210707151613103.png)

目前，redis在初始化服务器时，会创建一万个字符串对象，这些对象包含了从 0 到 9999 的所有整数值，当服务器需要用到值为 0 - 9999 的字符串对象时，服务器就会使用这些对象，而不是新创建对象。

```BASH
### 实际操作redis发现和书中有所不同了
127.0.0.1:6379> SET A 1000
OK
127.0.0.1:6379> OBJECT refcount A
(integer) 2147483647
127.0.0.1:6379> SET B 1000
OK
127.0.0.1:6379> OBJECT refcount A
(integer) 2147483647
127.0.0.1:6379> OBJECT refcount B
(integer) 2147483647
### 新版本吧，自己看的是 redis 3.0 版本，对于此处的涉及稍有不同
# 源码中将 REDIS_SHARED_INTEGERS 设为 10000，并定义不被销毁的全局对象的引用数量为 INT_MAX = 2^31 - 1 = 2147483647
### redis.h
#define REDIS_SHARED_INTEGERS 10000

```

```c
### object.c
/* Try to encode a string object in order to save space */
// 尝试对字符串对象进行编码，以节约内存。
robj *tryObjectEncoding(robj *o) {
    long value;
    sds s = o->ptr;
    size_t len;
    
    /* Make sure this is a string object, the only type we encode
     * in this function. Other types use encoded memory efficient
     * representations but are handled by the commands implementing
     * the type. */
    redisAssertWithInfo(NULL,o,o->type == REDIS_STRING);

    /* We try some specialized encoding only for objects that are
     * RAW or EMBSTR encoded, in other words objects that are still
     * in represented by an actually array of chars. */
    // 只在字符串的编码为 RAW 或者 EMBSTR 时尝试进行编码
    if (!sdsEncodedObject(o)) return o;

    /* It's not safe to encode shared objects: shared objects can be shared
     * everywhere in the "object space" of Redis and may end in places where
     * they are not handled. We handle them only as values in the keyspace. */
     // 不对共享对象进行编码
     if (o->refcount > 1) return o;

    /* Check if we can represent this string as a long integer.
     * Note that we are sure that a string larger than 21 chars is not
     * representable as a 32 nor 64 bit integer. */
    // 对字符串进行检查
    // 只对长度小于或等于 21 字节，并且可以被解释为整数的字符串进行编码
    len = sdslen(s);
    if (len <= 21 && string2l(s,len,&value)) {
        /* This object is encodable as a long. Try to use a shared object.
         * Note that we avoid using shared integers when maxmemory is used
         * because every object needs to have a private LRU field for the LRU
         * algorithm to work well. */
        if (server.maxmemory == 0 &&
            value >= 0 &&
            value < REDIS_SHARED_INTEGERS) 
            // redis 对象引用只对 value >= 0 && value < REDIS_SHARED_INTEGERS 的数值类对象生效，除此之外的其他redis对象，都不会互相引用
        {
            decrRefCount(o);
            incrRefCount(shared.integers[value]);
            return shared.integers[value];
        } else {
            if (o->encoding == REDIS_ENCODING_RAW) sdsfree(o->ptr);
            o->encoding = REDIS_ENCODING_INT;
            o->ptr = (void*) value;
            return o;
        }
    }
    .....
    return o;
}
```



#### 对象的空转时长

redisObject结构中还有一个属性为 lru属性，记录了对象最后一次被命令程序访问的时间：

```c
/*
 * Redis 对象
 */
typedef struct redisObject {
    ....
    // 对象最后一次被访问的时间
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
	....

} robj;
```

**OBJECT IDLETIME 命令实现特殊，命令访问键的值对象时，不会修改值对象lru 的属性。**

```bash
# 对于之前创建的 A
127.0.0.1:6379> OBJECT idletime A
(integer) 514709
# 使用之后
127.0.0.1:6379> get A
"1000"
# 键处于活跃状态，空转时长很短
127.0.0.1:6379> OBJECT idletime A
(integer) 4
```

#### 回顾

![image-20210707160552384](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210707160552384.png)



## 单机数据库的实现



### 9、数据库

redis服务器将所有的数据库都保存在服务器状态结构的db数组中，每个redisDb结构代表一个数据库：

```c
struct redisServer {

    /* General */
    // 配置文件的绝对路径
    char *configfile;           /* Absolute config file path, or NULL */

    // serverCron() 每秒调用的次数
    int hz;                     /* serverCron() calls frequency in hertz */

    // 数据库 数组，保存这服务器中所有的数据库
    redisDb *db;

    // 命令表（受到 rename 配置选项的作用）
    dict *commands;             /* Command table */
    // 命令表（无 rename 配置选项的作用）
    dict *orig_commands;        /* Command table before command renaming. */

    // 事件状态
    aeEventLoop *el;

    // 最近一次使用时钟
    unsigned lruclock:REDIS_LRU_BITS; /* Clock for LRU eviction */

	...
	int dbnum;      /* Total number of configured DBs 默认情况下会设置为 16个数据库 */
   	...
};

```

redis 服务器默认会创建 16 个数据库：

<img src="C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708113845374.png" alt="image-20210708113845374" style="zoom: 80%;" />



#### 数据库切换

每个客户端都由自己的目标数据库，**可以使用select number 来切换数据库**。

redisClient 结构的db属性记录了客户端当前的目标数据库，指向的是一个redisDb的结构体。

```c
typedef struct redisClient {

    // 套接字描述符
    int fd;

    // 当前正在使用的数据库
    redisDb *db;

    // 当前正在使用的数据库的 id （号码）
    int dictid;

    // 客户端的名字
    robj *name;             /* As set by CLIENT SETNAME */

    // 查询缓冲区
    sds querybuf;

    // 查询缓冲区长度峰值
    size_t querybuf_peak;   /* Recent (100ms or more) peak of querybuf size */

    // 参数数量
    int argc;

    // 参数对象数组
    robj **argv;

    // 记录被客户端执行的命令
    struct redisCommand *cmd, *lastcmd;

    // 请求的类型：内联命令还是多条命令
    int reqtype;

    // 剩余未读取的命令内容数量
    int multibulklen;       /* number of multi bulk arguments left to read */

    // 命令内容的长度
    long bulklen;           /* length of bulk argument in multi bulk request */

    // 回复链表
    list *reply;

    // 回复链表中对象的总大小
    unsigned long reply_bytes; /* Tot bytes of objects in reply list */

    // 已发送字节，处理 short write 用
    int sentlen;            /* Amount of bytes already sent in the current
                               buffer or object being sent. */

    // 创建客户端的时间
    time_t ctime;           /* Client creation time */

    // 客户端最后一次和服务器互动的时间
    time_t lastinteraction; /* time of the last interaction, used for timeout */

    ...
} redisClient;
```

![image-20210708114410780](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708114410780.png)



**为避免不知道在哪个数据库中出现误操作的情况，最好先执行一个select命令显示的切换到指定的数据库，然后再执行别的命令。**

#### 数据库键空间

redis是一个键值对数据库服务器，服务器中的每个数据库都是由一个redisDb结构的dict字典保存数据库中所有的键值对，将这个**字典称为 键空间**。

```c
/* Redis database representation. There are multiple databases identified
 * by integers from 0 (the default database) up to the max configured
 * database. The database number is the 'id' field in the structure. */
typedef struct redisDb {

    // 数据库键空间，保存着数据库中的所有键值对
    dict *dict;                 /* The keyspace for this DB */

    // 键的过期时间，字典的键为键，字典的值为过期事件 UNIX 时间戳
    dict *expires;              /* Timeout of keys with a timeout set */

    // 正处于阻塞状态的键
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP) */

    // 可以解除阻塞的键
    dict *ready_keys;           /* Blocked keys that received a PUSH */

    // 正在被 WATCH 命令监视的键
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */

    struct evictionPoolEntry *eviction_pool;    /* Eviction pool of keys */

    // 数据库号码
    int id;                     /* Database ID */

    // 数据库的键的平均 TTL ，统计信息
    long long avg_ttl;          /* Average TTL, just for stats */

} redisDb;
```

键空间和用户所见的数据库是直接对应的，**键空间的键也是数据库的键，每个键都是一个字符串对象； 键空间的值也就是数据库的值，每个值可以是字符串对象、列表对象、哈希表对象、集合对象、有序集合对象中的任意一种redis对象。**

```bash
127.0.0.1:6379> SET message "hello world"
OK
127.0.0.1:6379> RPUSH alphabet "a" "b" "c"
(integer) 3
127.0.0.1:6379> HSET book name "Redis in Action"
(integer) 1
127.0.0.1:6379> HSET book author "Joshiah L. Carlson"
(integer) 1
127.0.0.1:6379> HSET book publisher "Manning"
(integer) 1
```

![image-20210708124730668](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708124730668.png)

**添加新键**

```bash
127.0.0.1:6379> SET date "2013.12.1"
OK
127.0.0.1:6379>
```

![image-20210708124848334](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708124848334.png)

**删除键**

```bash
127.0.0.1:6379> DEL book
(integer) 1
127.0.0.1:6379>
```

![image-20210708125004399](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708125004399.png)



**更新键**

```bash
127.0.0.1:6379> SET message "blah blah"
OK
127.0.0.1:6379>
```

![image-20210708125200926](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708125200926.png)

```bash
127.0.0.1:6379> HSET book page 320
(integer) 1
127.0.0.1:6379>
```

![image-20210708125259717](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708125259717.png)

**对键取值**

对一个数据库的键进行取值，就是在键空间中取出键所对应的值对象，根据值对象的类型不同，具体的取值方法不同。

```bash
127.0.0.1:6379> GET message
"hello world"
127.0.0.1:6379>
```

![image-20210708131100061](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708131100061.png)



```bash
127.0.0.1:6379> LRANGE alphabet 0 -1
1) "a"
2) "b"
3) "c"
127.0.0.1:6379>
```

![image-20210708131152897](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708131152897.png)

#### 设置键的生存时间或过期时间

通过 **EXPIRE 、PEXPIRE 命令**，客户端可以以 秒 或者 毫秒精度为数据库中的某个键设置生存时间（Time To Live，TTL），经过指定的秒数 或 毫秒数后，服务器就会自动删除生存时间为0的键；

客户端还可以通过 **EXPIREAT 、PEXPIREAT** 命令，以秒或毫秒精度给数据库中的某个键设置过期时间，这个过期时间是个 UNIX 时间戳，当键的过期时间来临时，服务器就会自动从数据库中删除这个键：

```bash
#########################################
127.0.0.1:6379> SET key value
OK
127.0.0.1:6379> EXPIRE key 5
(integer) 1
127.0.0.1:6379> GET key
"value"
127.0.0.1:6379> GET key
(nil)


#########################################
127.0.0.1:6379>SET key value
OK
127.0.0.1:6379> time
1) "1625721731"
2) "533534"
127.0.0.1:6379> EXPIREAT key 1625721780
(integer) 1
127.0.0.1:6379> time
1) "1625721768"
2) "894450"
127.0.0.1:6379> GET key
"value"
127.0.0.1:6379> time
1) "1625721783"
2) "167470"
127.0.0.1:6379> GET key
(nil)
```

##### 计算并返回剩余生存时间

**TTL、PTTL 命令接收一个带有生存时间或者过期时间的键，返回这个键的剩余生存时间，即，返回距离这个键被服务器删除还有多长时间：**

```bash
127.0.0.1:6379> SET key value
OK
127.0.0.1:6379> EXPIRE key 100
(integer) 1
127.0.0.1:6379> TTL key
(integer) 96
127.0.0.1:6379> TTL key
(integer) 87
#################################################
127.0.0.1:6379> SET newkey newvalue
OK
127.0.0.1:6379> time
1) "1625722064"
2) "659195"
127.0.0.1:6379> EXPIREAT newkey 1625722666
(integer) 1
127.0.0.1:6379> ttl newkey
(integer) 558
127.0.0.1:6379>
```

**实际上，EXPIRE、PEXPIRE、EXPIREAT 三个命令都是使用 PEXPIREAT 命令实现的：无论客户端执行以上四个命令中的哪一个，经过转换后，最后的执行效果都和执行 PEXPIREAT 命令一样。**

![image-20210708133216062](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708133216062.png)

##### 保存过期时间

redisDb结构的expires 字典保存了数据库中所有键的过期时间，称这个字典为过期字典：

```c
typedef struct redisDb {

    // 数据库键空间，保存着数据库中的所有键值对
    dict *dict;                 /* The keyspace for this DB */

    // 键的过期时间，字典的键为键，字典的值为过期事件 UNIX 时间戳
    dict *expires;              /* Timeout of keys with a timeout set */

  	...

} redisDb;

/*
 * 字典
 */
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表
    dictht ht[2];

    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */

    // 目前正在运行的安全迭代器的数量
    int iterators; /* number of iterators currently running */

} dict;

```

![image-20210708133825637](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708133825637.png)

过期字典新增一个键值对：

![image-20210708133909223](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708133909223.png)



##### 移除过期时间

**使用 PERSIST 命令就可以在 过期字典中查找给定的键，并删除键和值（过期时间）在过期字典中的关联。**



#### Redis过期键的删除策略

- 定时删除：设置键的过期时间同时，创建一个定时器，让定时器在键的过期时间来临时，立即执行对键的删除操作。缺点： **对CPU时间不友好，对服务器的响应时间造成影响**
- 惰性删除：放任键不管，但每次从键空间中获取键时，都检查取得的键是否过期，如果过期的话就删除改键，如果没有过期，就返回该键。**对CPU时间最友好，但是对内存不友好，只要过期键不被删除就会占据内存，对于一些永远不会被访问的键，甚至可以看作是一种内存泄漏。**
- 定期删除：每个一段时间，程序就对数据库就行一次检查，删除里面的过期键。删除不超过 25% 的过期键。



##### 惰性删除策略实现

由**db.c/expireIfNeeded** 函数实现：

```c
/*
 * 检查 key 是否已经过期，如果是的话，将它从数据库中删除。
 *
 * 返回 0 表示键没有过期时间，或者键未过期。
 *
 * 返回 1 表示键已经因为过期而被删除了。
 */
int expireIfNeeded(redisDb *db, robj *key) {

    // 取出键的过期时间，mstime_t 是 millisecond time type 毫秒 long long 类型
    mstime_t when = getExpire(db,key);
    mstime_t now;

    // 没有过期时间
    if (when < 0) return 0; /* No expire for this key */

    /* Don't expire anything while loading. It will be done later. */
    // 如果服务器正在进行载入，那么不进行任何过期检查
    if (server.loading) return 0;

    /* If we are in the context of a Lua script, we claim that time is
     * blocked to when the Lua script started. This way a key can expire
     * only the first time it is accessed and not in the middle of the
     * script execution, making propagation to slaves / AOF consistent.
     * See issue #1525 on Github for more information. */
    // 如果此时正在执行 lua 脚本，数据库时间会阻塞在脚本开始的时间，所以now = server.lua_time_start 
    now = server.lua_caller ? server.lua_time_start : mstime();

    /* If we are running in the context of a slave, return ASAP:
     * the slave key expiration is controlled by the master that will
     * send us synthesized DEL operations for expired keys.
     *
     * Still we try to return the right information to the caller, 
     * that is, 0 if we think the key should be still valid, 1 if
     * we think the key is expired at this time. */
    // 当服务器运行在 replication(复制) 模式时
    // 附属节点并不主动删除 key
    // 它只返回一个逻辑上正确的返回值
    // 真正的删除操作要等待主节点发来删除命令时才执行
    // 从而保证数据的同步
    if (server.masterhost != NULL) return now > when;

    // 运行到这里，表示键带有过期时间，并且服务器为主节点

    /* Return when this key has not expired */
    // 如果未过期，返回 0
    if (now <= when) return 0;

    /* Delete the key */
    server.stat_expiredkeys++;

    // 向 AOF 文件和附属节点传播过期信息
    propagateExpire(db,key);

    // 发送事件通知
    notifyKeyspaceEvent(REDIS_NOTIFY_EXPIRED,
        "expired",key,db->id);

    // 将过期键从数据库中删除
    return dbDelete(db,key);
}


```

![image-20210708140020321](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708140020321.png)



##### 定期删除策略实现

过期键的定期删除策略由 **redis.c/activeExpireCycle** 函数实现，每次redis执行服务器周期性操作 **redis.c/serverCron** 函数（Redis 的时间中断器，每秒调用 server.hz 次）时，**activeExpireCycle 函数就会被调用**，该函数在规定的时间内，分多次遍历服务器中的各个数据库，从数据库的 expires 字典中随机检查一部分的过期键，并删除其中的过期键。



```c
void activeExpireCycle(int type) {
    ....
   	....

    // 遍历数据库
    for (j = 0; j < dbs_per_call; j++) {
        int expired;
        // 指向要处理的数据库
        redisDb *db = server.db+(current_db % server.dbnum);

        /* Increment the DB now so we are sure if we run out of time
         * in the current DB we'll restart from the next. This allows to
         * distribute the time evenly across DBs. */
        // 为 DB 计数器加一，如果进入 do 循环之后因为超时而跳出
        // 那么下次会直接从下个 DB 开始处理
        current_db++;

        /* Continue to expire if at the end of the cycle more than 25%
         * of the keys were expired. */
        do {
            unsigned long num, slots;
            long long now, ttl_sum;
            int ttl_samples;

            /* If there is nothing to expire try next DB ASAP. */
            // 获取数据库中带过期时间的键的数量
            // 如果该数量为 0 ，直接跳过这个数据库
            if ((num = dictSize(db->expires)) == 0) {
                db->avg_ttl = 0;
                break;
            }
            // 获取数据库中键值对的数量
            slots = dictSlots(db->expires);
            // 当前时间
            now = mstime();

            /* When there are less than 1% filled slots getting random
             * keys is expensive, so stop here waiting for better times...
             * The dictionary will be resized asap. */
            // 当 filled slots 小于 1% 时，随机获得 keys 的开销太大了，所以退出
            // 这个数据库的使用率低于 1% ，扫描起来太费力了（大部分都会 MISS）
            // 跳过，等待字典收缩程序运行
            if (num && slots > DICT_HT_INITIAL_SIZE &&
                (num*100/slots < 1)) break;

            /* The main collection cycle. Sample random keys among keys
             * with an expire set, checking for expired ones. 
             *
             * 样本计数器
             */
            // 已处理过期键计数器
            expired = 0;
            // 键的总 TTL 计数器
            ttl_sum = 0;
            // 总共处理的键计数器
            ttl_samples = 0;

            // 每次最多只能检查 LOOKUPS_PER_LOOP 个键
            if (num > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP)
                num = ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP;

            // 开始遍历数据库
            while (num--) {
                dictEntry *de;
                long long ttl;

                // 从 expires 中随机取出一个带过期时间的键
                if ((de = dictGetRandomKey(db->expires)) == NULL) break;
                // 计算 TTL
                ttl = dictGetSignedIntegerVal(de)-now;
                // 如果键已经过期，那么删除它，并将 expired 计数器增一
                if (activeExpireCycleTryExpire(db,de,now)) expired++;
                if (ttl < 0) ttl = 0;
                // 累积键的 TTL
                ttl_sum += ttl;
                // 累积处理键的个数
                ttl_samples++;
            }

            /* Update the average TTL stats for this database. */
            // 为这个数据库更新平均 TTL 统计数据
            if (ttl_samples) {
                // 计算当前平均值
                long long avg_ttl = ttl_sum/ttl_samples;
                
                // 如果这是第一次设置数据库平均 TTL ，那么进行初始化
                if (db->avg_ttl == 0) db->avg_ttl = avg_ttl;
                /* Smooth the value averaging with the previous one. */
                // 取数据库的上次平均 TTL 和今次平均 TTL 的平均值
                db->avg_ttl = (db->avg_ttl+avg_ttl)/2;
            }

            /* We can't block forever here even if there are many keys to
             * expire. So after a given amount of milliseconds return to the
             * caller waiting for the other active expire cycle. */
            // 我们不能用太长时间处理过期键，
            // 所以这个函数执行一定时间之后就要返回

            // 更新遍历次数
            iteration++;

            // 每遍历 16 次执行一次
            if ((iteration & 0xf) == 0 && /* check once every 16 iterations. */
                (ustime()-start) > timelimit)
            {
                // 如果遍历次数正好是 16 的倍数
                // 并且遍历的时间超过了 timelimit
                // 那么断开 timelimit_exit
                timelimit_exit = 1;
            }

            // 已经超时了，返回
            if (timelimit_exit) return;

            /* We don't repeat the cycle if there are less than 25% of keys
             * found expired in the current DB. */
            // 如果在当前数据库中发现过期的键少于将要检查的键的   25%，我们不会重复循环。
            // 如果已删除的过期键占当前总数据库带过期时间的键数量的 25 %
            // 那么不再遍历
        } while (expired > ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP/4);
    }
}
```



![image-20210708141345736](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708141345736.png)



#### AOF 、RDB 和复制功能对过期键的处理



##### 生成 和 载入RDB 文件

- 执行 SAVE 、BGSAVE 命令创建一个新的 RDB文件时，程序会对数据库中的键进行检查，已经过期的键不会被保存到新创建的RDB文件中。因此，**数据库中包含过期键不会对生成新的RDB文件造成影响。**

- 启动Redis服务器时，如果服务器开启了 RDB 功能，那么服务器将对 RDB 文件进行载入：
  - 服务器以主服务器模式运行，载入RDB时，程序会对文件中保存的键进行检查，未过期的键会被载入到数据库中，而过期键会被忽略。所以，**过期键对载入RDB文件的主服务器不会造成影响。**
  - 服务器已从服务器模式运行，载入RDB文件时，文件中保存的所有键，不论是否过期，都会被载入到数据库中。不过，主从服务器在进行数据同步的时候，从服务器的数据库就会被清空，所以，一般来说，过期键对载入RDB文件的从服务器也不会造成影响。

##### AOF文件写入

当服务器以AOF持久化模式运行时，如果数据库中的某个键已经过期，但还没有被惰性删除或者定期删除，那么AOF文件不会因为这个过期键而产生影响。

当过期键被惰性删除或者定期删除后，程序会像AOF文件**追加(append)**一条DEL命令，来显示的记录该键已被删除。

##### AOF 重写

和生成RDB文件类似，执行AOF重写过程中，程序会对数据库中的键键进行检查，已过期的键不会被保存到重写后的AOF文件中。因此，**数据库中包含过期键不会对AOF重写造成影响。**



#### 复制

当服务器运行在复制模式下时，从服务器的过期键删除动作由主服务器控制：

- 主服务器在删除一个过期键后，会显示的向所有从服务器发送一个 DEL 命令，告知从服务器删除这个过期键；（**由主服务器来控制从服务器统一的删除过期键，可以保证主从服务器数据的一致性**）
- 从服务器在执行客户端发送的读命令时，及时碰到过期键也不会将过期键删除，而是继续像处理未过期键一样来处理过期键。
- 从服务器只有在接到主服务器发来的 DEL 命令后，才会删除过期键。		



![image-20210708144358876](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708144358876.png)

![image-20210708144348305](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708144348305.png)

![image-20210708144410915](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708144410915.png)

![image-20210708144419042](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708144419042.png)

#### 数据库通知

TODO



#### 回顾

![image-20210708144528295](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210708144528295.png)



### 10、RDB持久化

redis的RDB持久化功能，可以将redis内存中的数据库状态保存到磁盘里，避免数据丢失。

RDB持久化既可以手动执行，也可以根据服务器的配置选项定期执行。

RDB文件是一个经过压缩的二进制文件，通过该文件可以还原生成RDB文件时的数据库状态。

#### RDB文件的创建与载入

**save**命令和**bgsave**命令都可以生成RDB文件。

- save命令会**阻塞Redis服务器进程，直到RDB文件创建完毕为止**，在Redis服务器阻塞期间，服务器不能处理任何命令请求。

- **bgsave**命令会**创建一个子进程**，**由子进程来负责创建RDB文件，父进程(即Redis主进程)则继续处理请求。**
- bgsave命令执行过程中，只有fork子进程时会阻塞服务器，而对于save命令，整个过程都会阻塞服务器，因此***save已基本被废弃***，线上环境要杜绝save的使用。此外，在自动触发RDB持久化时，Redis也会选择bgsave而不是save来进行持久化。



创建 RDB 文件的实际工作是由 rdb.c/rdbSave 函数完成的，SAVE命令和 BGSAVE 命令会以不同的方式调用这个函数：

```c
/* Save the DB on disk. Return REDIS_ERR on error, REDIS_OK on success 
 *
 * 将数据库保存到磁盘上。
 *
 * 保存成功返回 REDIS_OK ，出错/失败返回 REDIS_ERR 。
 */
int rdbSave(char *filename) {
    dictIterator *di = NULL;
    dictEntry *de;
    char tmpfile[256];
    char magic[10];
    int j;
    long long now = mstime();
    FILE *fp;
    rio rdb;
    uint64_t cksum;

    // 创建临时文件
    snprintf(tmpfile,256,"temp-%d.rdb", (int) getpid());
    fp = fopen(tmpfile,"w");
    if (!fp) {
        redisLog(REDIS_WARNING, "Failed opening .rdb for saving: %s",
            strerror(errno));
        return REDIS_ERR;
    }

    // 初始化 I/O
    rioInitWithFile(&rdb,fp);

    // 设置校验和函数
    if (server.rdb_checksum)
        rdb.update_cksum = rioGenericUpdateChecksum;

    // 写入 RDB 版本号
    snprintf(magic,sizeof(magic),"REDIS%04d",REDIS_RDB_VERSION);
    if (rdbWriteRaw(&rdb,magic,9) == -1) goto werr;

    // 遍历所有数据库
    for (j = 0; j < server.dbnum; j++) {

        // 指向数据库
        redisDb *db = server.db+j;

        // 指向数据库键空间
        dict *d = db->dict;

        // 跳过空数据库
        if (dictSize(d) == 0) continue;

        // 创建键空间迭代器
        di = dictGetSafeIterator(d);
        if (!di) {
            fclose(fp);
            return REDIS_ERR;
        }

        /* Write the SELECT DB opcode 
         *
         * 写入 DB 选择器
         */
        if (rdbSaveType(&rdb,REDIS_RDB_OPCODE_SELECTDB) == -1) goto werr;
        if (rdbSaveLen(&rdb,j) == -1) goto werr;

        /* Iterate this DB writing every entry 
         *
         * 遍历数据库，并写入每个键值对的数据
         */
        while((de = dictNext(di)) != NULL) {
            sds keystr = dictGetKey(de);
            robj key, *o = dictGetVal(de);
            long long expire;
            
            // 根据 keystr ，在栈中创建一个 key 对象
            initStaticStringObject(key,keystr);

            // 获取键的过期时间
            expire = getExpire(db,&key);

            // 保存键值对数据
            if (rdbSaveKeyValuePair(&rdb,&key,o,expire,now) == -1) goto werr;
        }
        dictReleaseIterator(di);
    }
    di = NULL; /* So that we don't release it again on error. */

    /* EOF opcode 
     *
     * 写入 EOF 代码
     */
    if (rdbSaveType(&rdb,REDIS_RDB_OPCODE_EOF) == -1) goto werr;

    /* CRC64 checksum. It will be zero if checksum computation is disabled, the
     * loading code skips the check in this case. 
     *
     * CRC64 校验和。
     *
     * 如果校验和功能已关闭，那么 rdb.cksum 将为 0 ，
     * 在这种情况下， RDB 载入时会跳过校验和检查。
     */
    cksum = rdb.cksum;
    memrev64ifbe(&cksum);
    rioWrite(&rdb,&cksum,8);

    /* Make sure data will not remain on the OS's output buffers */
    // 冲洗缓存，确保数据已写入磁盘
    if (fflush(fp) == EOF) goto werr;
    if (fsync(fileno(fp)) == -1) goto werr;
    if (fclose(fp) == EOF) goto werr;

    /* Use RENAME to make sure the DB file is changed atomically only
     * if the generate DB file is ok. 
     *
     * 使用 RENAME ，原子性地对临时文件进行改名，覆盖原来的 RDB 文件。
     */
    if (rename(tmpfile,filename) == -1) {
        redisLog(REDIS_WARNING,"Error moving temp DB file on the final destination: %s", strerror(errno));
        unlink(tmpfile);
        return REDIS_ERR;
    }

    // 写入完成，打印日志
    redisLog(REDIS_NOTICE,"DB saved on disk");

    // 清零数据库脏状态
    server.dirty = 0;

    // 记录最后一次完成 SAVE 的时间
    server.lastsave = time(NULL);

    // 记录最后一次执行 SAVE 的状态
    server.lastbgsave_status = REDIS_OK;

    return REDIS_OK;

werr:
    // 关闭文件
    fclose(fp);
    // 删除文件
    unlink(tmpfile);

    redisLog(REDIS_WARNING,"Write error saving DB on disk: %s", strerror(errno));

    if (di) dictReleaseIterator(di);

    return REDIS_ERR;
}
```

如 rdbSaveBackground 函数中就是调用的 rdvSave 函数：

```c
int rdbSaveBackground(char *filename) {
    pid_t childpid;
    long long start;

    // 如果 BGSAVE 已经在执行，那么出错
    if (server.rdb_child_pid != -1) return REDIS_ERR;

    // 记录 BGSAVE 执行前的数据库被修改次数
    server.dirty_before_bgsave = server.dirty;

    // 最近一次尝试执行 BGSAVE 的时间
    server.lastbgsave_try = time(NULL);

    // fork() 开始前的时间，记录 fork() 返回耗时用
    start = ustime();

    if ((childpid = fork()) == 0) {
        int retval;

        /* Child */

        // 关闭网络连接 fd
        closeListeningSockets(0);

        // 设置进程的标题，方便识别
        redisSetProcTitle("redis-rdb-bgsave");

        // 执行保存操作
        retval = rdbSave(filename);

        // 打印 copy-on-write 时使用的内存数
        if (retval == REDIS_OK) {
            size_t private_dirty = zmalloc_get_private_dirty();

            if (private_dirty) {
                redisLog(REDIS_NOTICE,
                    "RDB: %zu MB of memory used by copy-on-write",
                    private_dirty/(1024*1024));
            }
        }

        // 向父进程发送信号
        exitFromChild((retval == REDIS_OK) ? 0 : 1);

    } else {

        /* Parent */

        // 计算 fork() 执行的时间
        server.stat_fork_time = ustime()-start;

        // 如果 fork() 出错，那么报告错误
        if (childpid == -1) {
            server.lastbgsave_status = REDIS_ERR;
            redisLog(REDIS_WARNING,"Can't save in background: fork: %s",
                strerror(errno));
            return REDIS_ERR;
        }

        // 打印 BGSAVE 开始的日志
        redisLog(REDIS_NOTICE,"Background saving started by pid %d",childpid);

        // 记录数据库开始 BGSAVE 的时间
        server.rdb_save_time_start = time(NULL);

        // 记录负责执行 BGSAVE 的子进程 ID
        server.rdb_child_pid = childpid;

        // 关闭自动 rehash
        updateDictResizePolicy();

        return REDIS_OK;
    }

    return REDIS_OK; /* unreached */
}
```

通过伪代码简单看看两个命令之间的区别：

![image-20210709090743233](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709090743233.png)

![image-20210709090753036](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709090753036.png)

**RDB文件的载入工作是在服务器启动时自动执行的，所以redis没有专门用于RDB文件载入的命令。**



- [ ] AOF文件的更新频率通常比RDB文件更新频率高，如果服务器开启了AOF持久化功能，那么服务器会优先使用AOF文件来还原数据库状态；
- [ ] 只有在AOF持久化功能关闭时，服务器才会使用RDB文件来还原数据库状态。

载入RDB文件的实际工作是由 rdb.c/ rdbLoad 函数完成：

```c
/*
 * 将给定 rdb 中保存的数据载入到数据库中。
 */
int rdbLoad(char *filename) {
    uint32_t dbid;
    int type, rdbver;
    redisDb *db = server.db+0;
    char buf[1024];
    long long expiretime, now = mstime();
    FILE *fp;
    rio rdb;

    // 打开 rdb 文件
    if ((fp = fopen(filename,"r")) == NULL) return REDIS_ERR;

    // 初始化写入流
    rioInitWithFile(&rdb,fp);
    rdb.update_cksum = rdbLoadProgressCallback;
    rdb.max_processing_chunk = server.loading_process_events_interval_bytes;
    if (rioRead(&rdb,buf,9) == 0) goto eoferr;
    buf[9] = '\0';

    // 检查版本号
    if (memcmp(buf,"REDIS",5) != 0) {
        fclose(fp);
        redisLog(REDIS_WARNING,"Wrong signature trying to load DB from file");
        errno = EINVAL;
        return REDIS_ERR;
    }
    rdbver = atoi(buf+5);
    if (rdbver < 1 || rdbver > REDIS_RDB_VERSION) {
        fclose(fp);
        redisLog(REDIS_WARNING,"Can't handle RDB format version %d",rdbver);
        errno = EINVAL;
        return REDIS_ERR;
    }

    // 将服务器状态调整到开始载入状态
    startLoading(fp);
    while(1) {
        robj *key, *val;
        expiretime = -1;

        /* Read type. 
         *
         * 读入类型指示，决定该如何读入之后跟着的数据。
         *
         * 这个指示可以是 rdb.h 中定义的所有以
         * REDIS_RDB_TYPE_* 为前缀的常量的其中一个
         * 或者所有以 REDIS_RDB_OPCODE_* 为前缀的常量的其中一个
         */
        if ((type = rdbLoadType(&rdb)) == -1) goto eoferr;

        // 读入过期时间值
        if (type == REDIS_RDB_OPCODE_EXPIRETIME) {

            // 以秒计算的过期时间

            if ((expiretime = rdbLoadTime(&rdb)) == -1) goto eoferr;

            /* We read the time so we need to read the object type again. 
             *
             * 在过期时间之后会跟着一个键值对，我们要读入这个键值对的类型
             */
            if ((type = rdbLoadType(&rdb)) == -1) goto eoferr;

            /* the EXPIRETIME opcode specifies time in seconds, so convert
             * into milliseconds. 
             *
             * 将格式转换为毫秒*/
            expiretime *= 1000;
        } else if (type == REDIS_RDB_OPCODE_EXPIRETIME_MS) {

            // 以毫秒计算的过期时间

            /* Milliseconds precision expire times introduced with RDB
             * version 3. */
            if ((expiretime = rdbLoadMillisecondTime(&rdb)) == -1) goto eoferr;

            /* We read the time so we need to read the object type again.
             *
             * 在过期时间之后会跟着一个键值对，我们要读入这个键值对的类型
             */
            if ((type = rdbLoadType(&rdb)) == -1) goto eoferr;
        }
            
        // 读入数据 EOF （不是 rdb 文件的 EOF）
        if (type == REDIS_RDB_OPCODE_EOF)
            break;

        /* Handle SELECT DB opcode as a special case 
         *
         * 读入切换数据库指示
         */
        if (type == REDIS_RDB_OPCODE_SELECTDB) {

            // 读入数据库号码
            if ((dbid = rdbLoadLen(&rdb,NULL)) == REDIS_RDB_LENERR)
                goto eoferr;

            // 检查数据库号码的正确性
            if (dbid >= (unsigned)server.dbnum) {
                redisLog(REDIS_WARNING,"FATAL: Data file was created with a Redis server configured to handle more than %d databases. Exiting\n", server.dbnum);
                exit(1);
            }

            // 在程序内容切换数据库
            db = server.db+dbid;

            // 跳过
            continue;
        }

        /* Read key 
         *
         * 读入键
         */
        if ((key = rdbLoadStringObject(&rdb)) == NULL) goto eoferr;

        /* Read value 
         *
         * 读入值
         */
        if ((val = rdbLoadObject(type,&rdb)) == NULL) goto eoferr;

        /* Check if the key already expired. This function is used when loading
         * an RDB file from disk, either at startup, or when an RDB was
         * received from the master. In the latter case, the master is
         * responsible for key expiry. If we would expire keys here, the
         * snapshot taken by the master may not be reflected on the slave. 
         *
         * 如果服务器为主节点的话，
         * 那么在键已经过期的时候，不再将它们关联到数据库中去
         */
        if (server.masterhost == NULL && expiretime != -1 && expiretime < now) {
            decrRefCount(key);
            decrRefCount(val);
            // 跳过
            continue;
        }

        /* Add the new object in the hash table 
         *
         * 将键值对关联到数据库中
         */
        dbAdd(db,key,val);

        /* Set the expire time if needed 
         *
         * 设置过期时间
         */
        if (expiretime != -1) setExpire(db,key,expiretime);

        decrRefCount(key);
    }

    /* Verify the checksum if RDB version is >= 5 
     *
     * 如果 RDB 版本 >= 5 ，那么比对校验和
     */
    if (rdbver >= 5 && server.rdb_checksum) {
        uint64_t cksum, expected = rdb.cksum;

        // 读入文件的校验和
        if (rioRead(&rdb,&cksum,8) == 0) goto eoferr;
        memrev64ifbe(&cksum);

        // 比对校验和
        if (cksum == 0) {
            redisLog(REDIS_WARNING,"RDB file was saved with checksum disabled: no check performed.");
        } else if (cksum != expected) {
            redisLog(REDIS_WARNING,"Wrong RDB checksum. Aborting now.");
            exit(1);
        }
    }

    // 关闭 RDB 
    fclose(fp);

    // 服务器从载入状态中退出
    stopLoading();

    return REDIS_OK;

eoferr: /* unexpected end of file is handled here with a fatal exit */
    redisLog(REDIS_WARNING,"Short read or OOM loading DB. Unrecoverable error, aborting now.");
    exit(1);
    return REDIS_ERR; /* Just to avoid warning */
}
```

![image-20210709092114289](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709092114289.png)







#### **设置自动保存参数**

服务器状态结构体的参数：

```c
struct redisServer { 
    ...
	struct saveparam *saveparams;   /* Save points array for RDB */
    // 自从上次 SAVE 执行以来，数据库被修改的次数
    long long dirty;               /* Changes to DB from the last save */
    
    // BGSAVE 执行前的数据库被修改次数
    long long dirty_before_bgsave;  /* Used to restore dirty on failed BGSAVE */
    ...
        
    // 最后一次完成 SAVE 的时间
    time_t lastsave;                /* Unix time of last successful save */

    // 最后一次尝试执行 BGSAVE 的时间
    time_t lastbgsave_try;          /* Unix time of last attempted bgsave */

    // 最近一次 BGSAVE 执行耗费的时间
    time_t rdb_save_time_last;      /* Time used by last RDB save run. */

    // 数据库最近一次开始执行 BGSAVE 的时间
    time_t rdb_save_time_start;     /* Current RDB save start time. */

    // 最后一次执行 SAVE 的状态
    int lastbgsave_status;          /* REDIS_OK or REDIS_ERR */
    int stop_writes_on_bgsave_err;  /* Don't allow writes if can't BGSAVE */
};
// 服务器的保存条件（BGSAVE 自动执行的条件）
struct saveparam {

    // 多少秒之内
    time_t seconds;

    // 发生多少次修改
    int changes;

};
```

服务器状态中的saveparams数组：

```bash
save 900 1
save 300 10
save 60 10000
```



![image-20210709094213297](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709094213297.png)

##### 检查保存条件是否满足

serverCron 函数默认每隔 100 毫秒就会执行一次，该函数用于对正在运行服务器进行维护，其中的一项工作就是检查 save 选项所设置的保存条件是否满足，满足就执行 BGSAVE 命令。

```c
/* This is our timer interrupt, called server.hz times per second.
 *
 * 这是 Redis 的时间中断器，每秒调用 server.hz 次。
 *
 * Here is where we do a number of things that need to be done asynchronously.
 * For instance:
 *
 * 以下是需要异步执行的操作：
 *
 * - Active expired keys collection (it is also performed in a lazy way on
 *   lookup).
 *   主动清除过期键。
 *
 * - Software watchdog.
 *   更新软件 watchdog 的信息。
 *
 * - Update some statistic.
 *   更新统计信息。
 *
 * - Incremental rehashing of the DBs hash tables.
 *   对数据库进行渐增式 Rehash
 *
 * - Triggering BGSAVE / AOF rewrite, and handling of terminated children.
 *   触发 BGSAVE 或者 AOF 重写，并处理之后由 BGSAVE 和 AOF 重写引发的子进程停止。
 *
 * - Clients timeout of different kinds.
 *   处理客户端超时。
 *
 * - Replication reconnection.
 *   复制重连
 *
 * - Many more...
 *   等等。。。
 *
 * Everything directly called here will be called server.hz times per second,
 * so in order to throttle execution of things we want to do less frequently
 * a macro is used: run_with_period(milliseconds) { .... }
 *
 * 因为 serverCron 函数中的所有代码都会每秒调用 server.hz 次，
 * 为了对部分代码的调用次数进行限制，
 * 使用了一个宏 run_with_period(milliseconds) { ... } ，
 * 这个宏可以将被包含代码的执行次数降低为每 milliseconds 执行一次。
 */

int serverCron(struct aeEventLoop *eventLoop, long long id, void *clientData) {
	.... 
        
    /* Start a scheduled AOF rewrite if this was requested by the user while
     * a BGSAVE was in progress. */
    // 如果 BGSAVE 和 BGREWRITEAOF 都没有在执行
    // 并且有一个 BGREWRITEAOF 在等待，那么执行 BGREWRITEAOF
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1 &&
        server.aof_rewrite_scheduled)
    {
        rewriteAppendOnlyFileBackground();
    }

    /* Check if a background saving or AOF rewrite in progress terminated. */
    // 检查 BGSAVE 或者 BGREWRITEAOF 是否已经执行完毕
    if (server.rdb_child_pid != -1 || server.aof_child_pid != -1) {
        ....
        ....
    } else {

        /* If there is not a background saving/rewrite in progress check if
         * we have to save/rewrite now */
        // 既然没有 BGSAVE 或者 BGREWRITEAOF 在执行，那么检查是否需要执行它们

        // 遍历所有保存条件，看是否需要执行 BGSAVE 命令
         for (j = 0; j < server.saveparamslen; j++) {
            struct saveparam *sp = server.saveparams+j;

            /* Save if we reached the given amount of changes,
             * the given amount of seconds, and if the latest bgsave was
             * successful or if, in case of an error, at least
             * REDIS_BGSAVE_RETRY_DELAY seconds already elapsed. */
            // 检查是否有某个保存条件已经满足了
            if (server.dirty >= sp->changes &&
                server.unixtime-server.lastsave > sp->seconds &&
                (server.unixtime-server.lastbgsave_try >
                 REDIS_BGSAVE_RETRY_DELAY ||
                 server.lastbgsave_status == REDIS_OK))
            {
                redisLog(REDIS_NOTICE,"%d changes in %d seconds. Saving...",
                    sp->changes, (int)sp->seconds);
                // 执行 BGSAVE
                rdbSaveBackground(server.rdb_filename);
                break;
            }
         }

         /* Trigger an AOF rewrite if needed */
        // 出发 BGREWRITEAOF
         if (server.rdb_child_pid == -1 &&
             server.aof_child_pid == -1 &&
             server.aof_rewrite_perc &&
             // AOF 文件的当前大小大于执行 BGREWRITEAOF 所需的最小大小
             server.aof_current_size > server.aof_rewrite_min_size)
         {
            // 上一次完成 AOF 写入之后，AOF 文件的大小
            long long base = server.aof_rewrite_base_size ?
                            server.aof_rewrite_base_size : 1;

            // AOF 文件当前的体积相对于 base 的体积的百分比
            long long growth = (server.aof_current_size*100/base) - 100;

            // 如果增长体积的百分比超过了 growth ，那么执行 BGREWRITEAOF
            if (growth >= server.aof_rewrite_perc) {
                redisLog(REDIS_NOTICE,"Starting automatic rewriting of AOF on %lld%% growth",growth);
                // 执行 BGREWRITEAOF
                rewriteAppendOnlyFileBackground();
            }
         }
    }

    return 1000/server.hz;
}
```

简单的伪码逻辑：

![image-20210709101958307](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709101958307.png)

#### RDB文件结构

RDB文件的各个部分：

![image-20210709103656867](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709103656867.png)

一个RDB文件的database 部分可以保存任意多个非空数据库。

![image-20210709110311521](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709110311521.png)

![image-20210709110341488](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709110341488.png)

每个key_value_pairs 部分都保存了一个或以上数量的键值对。

![image-20210709112914040](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709112914040.png)

**每个value部分保存一个值对象，每个值对象的类型由对应的TYPE记录，根据类型的不同，value部分的结构、长度会不同。**



#### 分析RDB文件

```bash
# 使用 od 命令可以打印 RDB文件
# -c 参数可以以 ASCII编码的方式打印输入文件
# -x 以 十六进制格式打印
# 可以用 -cx 参数，同时以ASCII编码和十六进制格式打印RDB文件
od -c dump.rdb

```



### 11、AOF持久化

**AOF（Append Only File）**

与 RDB 持久化通过**保存数据库中的键值对**来记录数据库状态不同的是，AOF 是通过**保存redis服务器所执行的写命令**来记录数据库状态的。

#### AOF持久化的实现

**实现可以分为 命令追加（append）、文件写入、文件同步（sync）。**

- 命令追加：就是服务器执行完一个写命令之后，会以协议格式将被执行的写命令追加到服务器状态的 aof_buf 缓冲区的末尾；

  ```c
  struct redisServer{
      .....
  /* AOF persistence */
  
      // AOF 状态（开启/关闭/可写）
      int aof_state;                  /* REDIS_AOF_(ON|OFF|WAIT_REWRITE) */
  	....
      ....    
      // AOF 缓冲区
      sds aof_buf;      /* AOF buffer, written before entering the event loop */
  
      // AOF 文件的描述符
      int aof_fd;       /* File descriptor of currently selected AOF file */
  
      // AOF 的当前目标数据库
      int aof_selected_db; /* Currently selected DB in AOF */
      .....
  };
  
  ```

- 文件写入与同步：服务器处理文件事件可能会执行写命令，使得一些内容被追加到了 aof_buf 缓冲区里，所以在每次结束一个事件循环之前，都会调用 flushAppendOnlyFile 函数，看看是否需要将 aof_buf 缓冲区中的**内容写入和保存到** AOF 文件里面。

  （redis的服务器进程就是一个事件循环（loop），这个事件负责接收客户端的命令请求，及向客户端发送命令回复，而时间事件则负责像serverCron函数这样需要定时运行的函数。）

  伪代码：

  ![image-20210709151805586](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709151805586.png)

  flushAppendOnluFile 函数的行为由服务器配置的 appendfsync 选项的值来决定，各个值的不同行为如下：

  ![image-20210709151949519](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709151949519.png)

> 写入 和 同步是 两个步骤，因为考虑到写入磁盘的效率，用户调用的write函数会先将要写入的数据保存到内存缓冲区，等缓冲区满或者超过指定时限后，再将缓冲区的数据写入到磁盘里面。

![image-20210709152425936](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709152425936.png)

**appendfsync** 选项的选择优劣比较：

![image-20210709152606536](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709152606536.png)

![image-20210709152616438](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709152616438.png)

#### AOF文件的载入与数据还原

数据库读入并重新执行一遍 AOF 文件中保存的写命令，就可以还原服务器关闭之前的数据库状态。

<img src="C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709152928659.png" alt="image-20210709152928659" style="zoom: 67%;" />

#### AOF 重写

因为服务器运行的越久 AOF 文件内容就会越多，文件体积会越大，使用AOF 文件进行数据还原的时间就会越多。为了解决文件体积膨胀的问题，redis提供了 AOF 重写功能：

**服务器可以创建一个新的AOF文件来替代现有的AOF 文件，新旧两个 AOF 文件保存的数据库状态相同，但新 AOF 文件不会包含任何浪费空间的冗余命令，所以新的AOF文件通常比旧 AOF 文件体积小的多。**

##### AOF文件重写的实现

原理：**首先从数据库中读取键现在的值，然后用一条命令去记录键值对，代替之前记录这个键值对的多条命令。**

因为 aof_rewrite 函数生成的新 AOF 文件只包含还原当前数据库状态所必须的命令，所以新 AOF 文件不会浪费任何硬盘空间。

**（REWRITEAOF 似乎已经是一个废弃的命令）**

```c
/* 
 * 将足以还原当前数据集的命令写入到 filename 指定的文件中。
 *
 * 这个函数被 REWRITEAOF 和 BGREWRITEAOF 两个命令调用。
 * （REWRITEAOF 似乎已经是一个废弃的命令）
 * 为了最小化重建数据集所需执行的命令数量，
 * Redis 会尽可能地使用接受可变参数数量的命令，比如 RPUSH 、SADD 和 ZADD 等。
 *
 * 不过单个命令每次处理的元素数量不能超过 REDIS_AOF_REWRITE_ITEMS_PER_CMD = 64。
 */
int rewriteAppendOnlyFile(char *filename) {
    dictIterator *di = NULL;
    dictEntry *de;
    rio aof;
    FILE *fp;
    char tmpfile[256];
    int j;
    long long now = mstime();

    /* 创建临时文件
     * 注意这里创建的文件名和 rewriteAppendOnlyFileBackground() 创建的文件名稍有不同
     */
    snprintf(tmpfile,256,"temp-rewriteaof-%d.aof", (int) getpid());
    fp = fopen(tmpfile,"w");
    if (!fp) {
        redisLog(REDIS_WARNING, "Opening the temp file for AOF rewrite in rewriteAppendOnlyFile(): %s", strerror(errno));
        return REDIS_ERR;
    }

    // 初始化文件 io
    rioInitWithFile(&aof,fp);

    // 设置每写入 REDIS_AOF_AUTOSYNC_BYTES 字节，就执行一次 FSYNC 
    // 防止缓存中积累太多命令内容，造成 I/O 阻塞时间过长
    if (server.aof_rewrite_incremental_fsync)
        rioSetAutoSync(&aof,REDIS_AOF_AUTOSYNC_BYTES);

    // #### 遍历所有数据库
    for (j = 0; j < server.dbnum; j++) {

        char selectcmd[] = "*2\r\n$6\r\nSELECT\r\n";

        redisDb *db = server.db+j;

        // 指向键空间
        dict *d = db->dict;
        if (dictSize(d) == 0) continue; #### 忽略数据库

        // 创建键空间迭代器
        di = dictGetSafeIterator(d);
        if (!di) {
            fclose(fp);
            return REDIS_ERR;
        }

        /* SELECT the new DB 
         *
         * 首先写入 SELECT 命令，确保之后的数据会被插入到正确的数据库上
         */
        if (rioWrite(&aof,selectcmd,sizeof(selectcmd)-1) == 0) goto werr;
        if (rioWriteBulkLongLong(&aof,j) == 0) goto werr;

        /* Iterate this DB writing every entry 
         *
         * ##### 遍历数据库所有键，并通过命令将它们的当前状态（值）记录到新 AOF 文件中
         */
        while((de = dictNext(di)) != NULL) {
            sds keystr;
            robj key, *o;
            long long expiretime;

            // 取出键
            keystr = dictGetKey(de);

            // 取出值
            o = dictGetVal(de);
            initStaticStringObject(key,keystr);

            // 取出过期时间
            expiretime = getExpire(db,&key);

            /* If this key is already expired skip it 
             *
             * 如果键已经过期，那么跳过它，不保存
             */
            if (expiretime != -1 && expiretime < now) continue;

            /* Save the key and associated value 
             *
             * ##### 根据值的类型，选择适当的命令来保存值
             */
            if (o->type == REDIS_STRING) {
                /* Emit a SET command */
                char cmd[]="*3\r\n$3\r\nSET\r\n";
                if (rioWrite(&aof,cmd,sizeof(cmd)-1) == 0) goto werr;
                /* Key and value */
                if (rioWriteBulkObject(&aof,&key) == 0) goto werr;
                if (rioWriteBulkObject(&aof,o) == 0) goto werr;
            } else if (o->type == REDIS_LIST) {
                if (rewriteListObject(&aof,&key,o) == 0) goto werr;
            } else if (o->type == REDIS_SET) {
                if (rewriteSetObject(&aof,&key,o) == 0) goto werr;
            } else if (o->type == REDIS_ZSET) {
                if (rewriteSortedSetObject(&aof,&key,o) == 0) goto werr;
            } else if (o->type == REDIS_HASH) {
                if (rewriteHashObject(&aof,&key,o) == 0) goto werr;
            } else {
                redisPanic("Unknown object type");
            }

            /* Save the expire time 
             * 保存键的过期时间
             */
            if (expiretime != -1) {
                char cmd[]="*3\r\n$9\r\nPEXPIREAT\r\n";

                // 写入 PEXPIREAT expiretime 命令
                if (rioWrite(&aof,cmd,sizeof(cmd)-1) == 0) goto werr;
                if (rioWriteBulkObject(&aof,&key) == 0) goto werr;
                if (rioWriteBulkLongLong(&aof,expiretime) == 0) goto werr;
            }
        }

        // 释放迭代器
        dictReleaseIterator(di);
    }

    /* Make sure data will not remain on the OS's output buffers */
    // 冲洗并关闭新 AOF 文件
    if (fflush(fp) == EOF) goto werr;
    if (aof_fsync(fileno(fp)) == -1) goto werr;
    if (fclose(fp) == EOF) goto werr;

    /* Use RENAME to make sure the DB file is changed atomically only
     * if the generate DB file is ok. 
     *
     * 原子地改名，用重写后的新 AOF 文件覆盖旧 AOF 文件
     */
    if (rename(tmpfile,filename) == -1) {
        redisLog(REDIS_WARNING,"Error moving temp append only file on the final destination: %s", strerror(errno));
        unlink(tmpfile);
        return REDIS_ERR;
    }

    redisLog(REDIS_NOTICE,"SYNC append only file rewrite performed");

    return REDIS_OK;

werr:
    fclose(fp);
    unlink(tmpfile);
    redisLog(REDIS_WARNING,"Write error writing append only file on disk: %s", strerror(errno));
    if (di) dictReleaseIterator(di);
    return REDIS_ERR;
}
```

![image-20210709164620265](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709164620265.png)

![image-20210709164648887](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210709164648887.png)



#### AOF 后台重写

**（REWRITEAOF 似乎已经是一个废弃的命令）**

因为 REWRITEAOF 函数会进行大量的写入操作，所以调用这个函数的线程会被长时间阻塞，因为redis是单线程模式，这样的话在重写 AOF 期间，服务器将无法处理客户端发来的请求。 AOF 重写则是将 重写程序放到**子进程**里执行，这样做的两个目的：

​	子进程进行aof重写期间，服务器进程（父进程）可以继续处理命令请求；

​	子进程带有服务器进程的副本，使用子进程而不是子线程，可以避免在使用锁的情况下，保证数据的安全性。





### 12、事件

redis 服务器是一个事件驱动的程序，程序需要处理的事件分为两类：

- 文件事件：文件事件就是服务器对套接字操作的抽象。

  redis服务器通过套接字与客户端（或者其他redis服务器）进行连接。服务器与客户端（或者其它啊服务器）的通信会产生响应的文件事件，而服务器则通过监听并处理这些事件来完成一系列网络通信操作。

- 时间时间：redis服务器中的一些操作（如 serverCron 函数）需要在给定的时间点执行，而时间事件就是服务器对这类定时操作的抽像。



#### 文件事件



![image-20210712150413075](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210712150413075.png)

I/O多路复用程序负责监听多个套接字，并向文件事件分派器传送那些产生了事件的套接字，虽然多个文件事件可能会并发的出现，但是 I/O 多路复用程序会将所有产生事件的套接字都放到一个队列中，然后通过这个队列，以有序、同步、每次一个套接字的方式向文件事件分派器传送套接字。

##### I/O 多路复用程序的实现

redis是通过包装 select 、 epoll 、evport 和 kqueue 这些I/O 多路复用库来实现的。

每个多路复用函数库都实现了相同的 API，所以 I/O 多路复用程序的底层实现是可以互换的：

![image-20210712151043036](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210712151043036.png)

```c
// redis 中 多路复用的选择规则
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```

##### 事件的类型

I/O 多路复用程序可以监听多个套接字的 ae.h/AE_READABLE 事件和 ae.h/AE_WRITABLE 事件，两类事件和套接字操作间的关系：

- 当套接字变得**可读**时（**客户端对套接字执行write操作，或者执行close操作**），或者 **有新的可应答 套接字出现**时（**客户端对服务器的监听套接字执行 connect 操作**），套接字产生**AE_READABLE**事件；
- 当套接字变得可写时（**客户端对套接字执行read操作**），套接字产生AE_WRITABLE事件。

I/O 多路复用允许服务器同时监听套接字的两个事件（AE_READABLE 事件AE_WRITABLE 事件），如果一个套接字同时产生两种事件，那么文件事件分派器优先处理 AE_READABLE 事件，等到AE_READABLE 事件处理完之后，再处理AE_WRITABLE 事件。

##### 文件事件的接口API 介绍

![image-20210712153025427](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210712153025427.png)

![image-20210712153039648](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210712153039648.png)

##### 文件事件的处理器

###### 1、连接应答处理器

![image-20210712160028069](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210712160028069.png)

![image-20210712160041681](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210712160041681.png)

###### 2、命令请求处理器

![image-20210712160100105](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210712160100105.png)

![image-20210712160119340](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210712160119340.png)

###### 3、命令回复处理器

![image-20210712160140294](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210712160140294.png)



#### 时间事件

redis 时间事件分为两类：

- 定时事件
- 周期性shijian



##### 时间事件的实现

服务器将所有的事件都放在一个无序链表中，每当时间事件执行器运行时，就遍历整个链表，查到所有已到达的时间事件，并调用响应的事件处理器。

**无序链表不会影响事件处理器的性能：目前阅读的 3.0  版本中只使用 severCron 一个时间事件，在benchmark 模式下，也只使用两个时间事件。这种情况下，无序链表退化成一个指针使用。**



##### serverCron函数

![image-20210712161058958](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210712161058958.png)

##### 事件的调度和执行 

事件的调度和执行由 ae.c/aeProcessEvents 函数负责：

```c
/* Process every pending time event, then every pending file event
 * (that may be registered by time event callbacks just processed).
 *
 * 处理所有已到达的时间事件，以及所有已就绪的文件事件。
 *
 * Without special flags the function sleeps until some file event
 * fires, or when the next time event occurs (if any).
 *
 * 如果不传入特殊 flags 的话，那么函数睡眠直到文件事件就绪，
 * 或者下个时间事件到达（如果有的话）。
 *
 * If flags is 0, the function does nothing and returns.
 * 如果 flags 为 0 ，那么函数不作动作，直接返回。
 *
 * if flags has AE_ALL_EVENTS set, all the kind of events are processed.
 * 如果 flags 包含 AE_ALL_EVENTS ，所有类型的事件都会被处理。
 *
 * if flags has AE_FILE_EVENTS set, file events are processed.
 * 如果 flags 包含 AE_FILE_EVENTS ，那么处理文件事件。
 *
 * if flags has AE_TIME_EVENTS set, time events are processed.
 * 如果 flags 包含 AE_TIME_EVENTS ，那么处理时间事件。
 *
 * if flags has AE_DONT_WAIT set the function returns ASAP until all
 * the events that's possible to process without to wait are processed.
 * 如果 flags 包含 AE_DONT_WAIT ，
 * 那么函数在处理完所有不许阻塞的事件之后，即刻返回。
 *
 * The function returns the number of events processed. 
 * 函数的返回值为已处理事件的数量
 *
 * 此函数是： 文件事件分派器
 */
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    	....
        struct timeval tv, *tvp;
		....
        // ##### 1、获取最近的时间事件
        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);
        if (shortest) {
            // 如果时间事件存在的话
            // 那么根据最近可执行时间事件和现在时间的时间差来决定文件事件的阻塞时间
 			....
            // ##### 2、计算距今最近的时间事件还要多久才能达到
            // ##### 3、并将该时间距保存在 tv 结构中
            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;
            tvp->tv_sec = shortest->when_sec - now_sec;
            if (shortest->when_ms < now_ms) {
                tvp->tv_usec = ((shortest->when_ms+1000) - now_ms)*1000;
                tvp->tv_sec --;
            } else {
                tvp->tv_usec = (shortest->when_ms - now_ms)*1000;
            }

            // ##### 4、时间差小于 0 ，说明事件已经可以执行了，将秒和毫秒设为 0 （不阻塞）
            if (tvp->tv_sec < 0) tvp->tv_sec = 0;
            if (tvp->tv_usec < 0) tvp->tv_usec = 0;
        } else {
            // 执行到这一步，说明没有时间事件
            // 那么根据 AE_DONT_WAIT 是否设置来决定是否阻塞，以及阻塞的时间长度
            ....
        }

        // ##### 5、处理文件事件，阻塞时间由 tvp 决定
        numevents = aeApiPoll(eventLoop, tvp);
        for (j = 0; j < numevents; j++) {
            // 从已就绪数组中获取事件
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
			....
            // 读事件
            if (fe->mask & mask & AE_READABLE) {
                // rfired 确保读/写事件只能执行其中一个
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            // 写事件
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
            processed++;
        }
    }

    // ##### 6、执行时间事件
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```

**将aeProcessEvents 函数置于redis 服务器的主函数：**

```bash
def main():
	# 初始化服务器
	init_server()
	# 一直处理事件，直到服务器关闭为止
	while server_is_not_shutdown();
		aeProcessEvents()
	# 服务器关闭，执行清理操作
	clean_server()
```

![image-20210712162823566](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210712162823566.png)

(因为时间事件是在文件事件之后执行，并且事件之间不会出现抢占，所以时间事件的实际处理时间，通常会比时间事件设定的到达时间稍晚一些)

![image-20210712163332023](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210712163332023.png)

#### 回顾

![image-20210712163424372](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210712163424372.png)

Redis 服务采用 Reactor 的方式来实现文件事件处理器（每一个网络连接其实都对应一个文件描述符）

![redis-reactor-pattern](https://img.draveness.me/2016-11-26-redis-reactor-pattern.png-1000width)

Redis 对于 I/O 多路复用模块的设计非常简洁，通过宏保证了 I/O 多路复用模块在不同平台上都有着优异的性能，将不同的 I/O 多路复用函数封装成相同的 API 提供给上层使用。

整个模块使 Redis 能以**单进程**运行的**同时服务成千上万个文件描述符**，避免了由于多进程应用的引入导致代码实现复杂度的提升，减少了出错的可能性。



#### 从源码看看对 select 的封装

**首先是看看 select 函数使用的大致流程**

```c

int fd = /* file descriptor */
// 1. 初始化一个可读的 fd_set 集合，保存需要监控可读性的 FD；
fd_set rfds;
FD_ZERO(&rfds);
// 2. 使用 FD_SET 将 fd 加入 rfds
FD_SET(fd, &rfds)

for ( ; ; ) {
    // 3. 调用 select 方法监控 rfds 中的 FD 是否可读
    select(fd+1, &rfds, NULL, NULL, NULL);
    // 4. 当 select 返回时，检查 FD 的状态并完成对应的操作
    if (FD_ISSET(fd, &rfds)) {
        /* file descriptor `fd` becomes readable */
    }
}
```

**Redis中ae_select 文件中的select模块封装：**

1、首先在`aeApiCreate` 函数中初始化 rfds 和 wfds

```c
static int aeApiCreate(aeEventLoop *eventLoop) {
    aeApiState *state = zmalloc(sizeof(aeApiState));

    if (!state) return -1;
    FD_ZERO(&state->rfds);
    FD_ZERO(&state->wfds);
    eventLoop->apidata = state;
    return 0;
}
```

2、而 `aeApiAddEvent` 和 `aeApiDelEvent` 会通过 `FD_SET` 和 `FD_CLR` 修改 `fd_set` 中对应 FD 的标志位

```c
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;

    if (mask & AE_READABLE) FD_SET(fd,&state->rfds);
    if (mask & AE_WRITABLE) FD_SET(fd,&state->wfds);
    return 0;
}

static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;

    if (mask & AE_READABLE) FD_CLR(fd,&state->rfds);
    if (mask & AE_WRITABLE) FD_CLR(fd,&state->wfds);
}
```

3、整个 `ae_select` 子模块中最重要的函数就是 **`aeApiPoll`**，它是实际调用 `select` 函数的部分，

   其作用就是在 I/O 多路复用函数返回时，将对应的 FD 加入 `aeEventLoop` 的 `fired` 数组中，并返回事件的个数。

```c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, j, numevents = 0;

    memcpy(&state->_rfds,&state->rfds,sizeof(fd_set));
    memcpy(&state->_wfds,&state->wfds,sizeof(fd_set));

    retval = select(eventLoop->maxfd+1,
                &state->_rfds,&state->_wfds,NULL,tvp);
    if (retval > 0) {
        for (j = 0; j <= eventLoop->maxfd; j++) {
            int mask = 0;
            aeFileEvent *fe = &eventLoop->events[j];

            if (fe->mask == AE_NONE) continue;
            if (fe->mask & AE_READABLE && FD_ISSET(j,&state->_rfds))
                mask |= AE_READABLE;
            if (fe->mask & AE_WRITABLE && FD_ISSET(j,&state->_wfds))
                mask |= AE_WRITABLE;
            eventLoop->fired[numevents].fd = j;
            eventLoop->fired[numevents].mask = mask;
            numevents++;
        }
    }
    return numevents;
}
```



#### 封装 epoll 函数

Redis 对 `epoll` 的封装其实也是类似的，使用 `epoll_create` 创建 `epoll` 中使用的 `epfd`：

```c
static int aeApiCreate(aeEventLoop *eventLoop) {
    aeApiState *state = zmalloc(sizeof(aeApiState));

    if (!state) return -1;
    state->events = zmalloc(sizeof(struct epoll_event)*eventLoop->setsize);
    if (!state->events) {
        zfree(state);
        return -1;
    }
    state->epfd = epoll_create(1024); /* 1024 is just a hint for the kernel */
    if (state->epfd == -1) {
        zfree(state->events);
        zfree(state);
        return -1;
    }
    eventLoop->apidata = state;
    return 0;
}
```

在 `aeApiAddEvent` 中使用 `epoll_ctl` 向 `epfd` 中添加需要监控的 FD 以及监听的事件：

```c
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;
    struct epoll_event ee = {0}; /* avoid valgrind warning */
    /* If the fd was already monitored for some event, we need a MOD
     * operation. Otherwise we need an ADD operation. */
    int op = eventLoop->events[fd].mask == AE_NONE ?
            EPOLL_CTL_ADD : EPOLL_CTL_MOD;

    ee.events = 0;
    mask |= eventLoop->events[fd].mask; /* Merge old events */
    if (mask & AE_READABLE) ee.events |= EPOLLIN;
    if (mask & AE_WRITABLE) ee.events |= EPOLLOUT;
    ee.data.fd = fd;
    if (epoll_ctl(state->epfd,op,fd,&ee) == -1) return -1;
    return 0;
}
```

由于 `epoll` 相比 `select` 机制略有不同，在 `epoll_wait` 函数返回时并不需要遍历所有的 FD 查看读写情况；

在 `epoll_wait` 函数返回时会提供一个 `epoll_event` 数组：

```c
typedef union epoll_data {
    void    *ptr;
    int      fd; /* 文件描述符 */
    uint32_t u32;
    uint64_t u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events; /* Epoll 事件 */
    epoll_data_t data;
};
```

> 其中保存了发生的 `epoll` 事件（`EPOLLIN`、`EPOLLOUT`、`EPOLLERR` 和 `EPOLLHUP`）以及发生该事件的 FD。

`aeApiPoll` 函数只需要将 `epoll_event` 数组中存储的信息加入 `eventLoop` 的 `fired` 数组中，将信息传递给上层模块：

```c
static int aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp) {
    aeApiState *state = eventLoop->apidata;
    int retval, numevents = 0;

    retval = epoll_wait(state->epfd,state->events,eventLoop->setsize,
            tvp ? (tvp->tv_sec*1000 + tvp->tv_usec/1000) : -1);
    if (retval > 0) {
        int j;

        numevents = retval;
        for (j = 0; j < numevents; j++) {
            int mask = 0;
            struct epoll_event *e = state->events+j;

            if (e->events & EPOLLIN) mask |= AE_READABLE;
            if (e->events & EPOLLOUT) mask |= AE_WRITABLE;
            if (e->events & EPOLLERR) mask |= AE_WRITABLE;
            if (e->events & EPOLLHUP) mask |= AE_WRITABLE;
            eventLoop->fired[j].fd = e->data.fd;
            eventLoop->fired[j].mask = mask;
        }
    }
    return numevents;
}
```

##### 子模块的选择

```c
#ifdef HAVE_EVPORT
#include "ae_evport.c"
#else
    #ifdef HAVE_EPOLL
    #include "ae_epoll.c"
    #else
        #ifdef HAVE_KQUEUE
        #include "ae_kqueue.c"
        #else
        #include "ae_select.c"
        #endif
    #endif
#endif
```



### 13、客户端



服务器状态结构的 clients 属性是一个链表，保存了所有与服务器连接的客户端的状态。

![image-20210713112047713](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210713112047713.png)





```c
/* With multiplexing we need to take per-client state.
 * Clients are taken in a liked list.
 *
 * 因为 I/O 复用的缘故，需要为每个客户端维持一个状态。
 *
 * 多个客户端状态被服务器用链表连接起来。
 */
typedef struct redisClient {

    // 套接字描述符，等于-1为伪客户端，大于-1为普通客户端。
    // 伪客户端的命令请求来自 AOF 或者 lua 脚本，而不是网络，所以这种客户端不需要套接字连接。
    // 使用 client list 命令可以查看当前连接的所有 普通客户端
    int fd;

    // 当前正在使用的数据库
    redisDb *db;

    // 当前正在使用的数据库的 id （号码）
    int dictid;

    // 客户端的名字
    robj *name;             /* As set by CLIENT SETNAME */

    // 查询缓冲区，用于保存 客户端发送的命令请求
    sds querybuf;

    // 查询缓冲区长度峰值
    size_t querybuf_peak;   /* Recent (100ms or more) peak of querybuf size */

    // 参数数量
    int argc;

    // 参数对象数组
    robj **argv; // argv[0] 中是要执行的命令，之后则是传给命令的参数

    // 记录被客户端执行的命令
    struct redisCommand *cmd, *lastcmd; // redisCommand 具体见下一段源码

    // 请求的类型：内联命令还是多条命令
    int reqtype;

    // 剩余未读取的命令内容数量
    int multibulklen;       /* number of multi bulk arguments left to read */

    // 命令内容的长度
    long bulklen;           /* length of bulk argument in multi bulk request */

    // 回复链表，服务器可以为客户端保存一个非常长的命令回复，而不必受到固定大小16KB的限制。
    // 保存长度较大的回复，如 一个非常长的字符串值、一个由很多项组成的列表、一个包含了很多元素的集合等。
    list *reply; // 双端链表

    // 回复链表中对象的总大小
    unsigned long reply_bytes; /* Tot bytes of objects in reply list */

    // 已发送字节，处理 short write 用
    int sentlen;            /* Amount of bytes already sent in the current
                               buffer or object being sent. */

    // 创建客户端的时间，CLIENT list 命令中的 age 域记录了这个秒数
    time_t ctime;           /* Client creation time */

    // 客户端最后一次和服务器互动的时间，CLIENT list 命令中的 idle，记录了客户端空转的时间
    time_t lastinteraction; /* time of the last interaction, used for timeout */

    // 客户端的输出缓冲区超过软性限制的时间
    time_t obuf_soft_limit_reached_time;

    // 客户端状态标志
    int flags;              /* REDIS_SLAVE | REDIS_MONITOR | REDIS_MULTI ... */

    // 当 server.requirepass 不为 NULL 时
    // 代表认证的状态
    // 0 代表未认证， 1 代表已认证
    int authenticated;      /* when requirepass is non-NULL */

    // 复制状态
    int replstate;          /* replication state if this is a slave */
    // 用于保存主服务器传来的 RDB 文件的文件描述符
    int repldbfd;           /* replication DB file descriptor */

    // 读取主服务器传来的 RDB 文件的偏移量
    off_t repldboff;        /* replication DB file offset */
    // 主服务器传来的 RDB 文件的大小
    off_t repldbsize;       /* replication DB file size */
    
    sds replpreamble;       /* replication DB preamble. */

    // 主服务器的复制偏移量
    long long reploff;      /* replication offset if this is our master */
    // 从服务器最后一次发送 REPLCONF ACK 时的偏移量
    long long repl_ack_off; /* replication ack offset, if this is a slave */
    // 从服务器最后一次发送 REPLCONF ACK 的时间
    long long repl_ack_time;/* replication ack time, if this is a slave */
    // 主服务器的 master run ID
    // 保存在客户端，用于执行部分重同步
    char replrunid[REDIS_RUN_ID_SIZE+1]; /* master run id if this is a master */
    // 从服务器的监听端口号
    int slave_listening_port; /* As configured with: SLAVECONF listening-port */

    // 事务状态
    multiState mstate;      /* MULTI/EXEC state */

    // 阻塞类型
    int btype;              /* Type of blocking op if REDIS_BLOCKED. */
    // 阻塞状态
    blockingState bpop;     /* blocking state */

    // 最后被写入的全局复制偏移量
    long long woff;         /* Last write global replication offset. */

    // 被监视的键
    list *watched_keys;     /* Keys WATCHED for MULTI/EXEC CAS */

    // 这个字典记录了客户端所有订阅的频道
    // 键为频道名字，值为 NULL
    // 也即是，一个频道的集合
    dict *pubsub_channels;  /* channels a client is interested in (SUBSCRIBE) */

    // 链表，包含多个 pubsubPattern 结构
    // 记录了所有订阅频道的客户端的信息
    // 新 pubsubPattern 结构总是被添加到表尾
    list *pubsub_patterns;  /* patterns a client is interested in (SUBSCRIBE) */
    sds peerid;             /* Cached peer ID. */

    /* Response buffer 
       每个客户端有两个输出缓冲区，一个是大小固定的，另一个是大小可变的（前面定义的 list *reply 回复链表）。
       此处为固定大小的缓冲区：保存长度较小的回复，如 ok、简短的字符串值、整数值、错误回复等
    */
    // 回复偏移量
    int bufpos;
    // 回复缓冲区，
    char buf[REDIS_REPLY_CHUNK_BYTES]; 
    // #define REDIS_REPLY_CHUNK_BYTES (16*1024) /* 16k output buffer */

} redisClient;

```



```c
/*
 * Redis 命令
 */
struct redisCommand {

    // 命令名字
    char *name;

    // 实现函数
    redisCommandProc *proc;

    // 参数个数
    int arity;

    // 字符串表示的 FLAG
    char *sflags; /* Flags as string representation, one char per flag. */

    // 实际 FLAG
    int flags;    /* The actual flags, obtained from the 'sflags' field. */

    /* Use a function to determine keys arguments in a command line.
     * Used for Redis Cluster redirect. */
    // 从命令中判断命令的键参数。在 Redis 集群转向时使用。
    redisGetKeysProc *getkeys_proc;

    /* What keys should be loaded in background when calling this command? */
    // 指定哪些参数是 key
    int firstkey; /* The first argument that's a key (0 = no keys) */
    int lastkey;  /* The last argument that's a key */
    int keystep;  /* The step between first and last key */

    // 统计信息
    // microseconds 记录了命令执行耗费的总毫微秒数
    // calls 是命令被执行的总次数
    long long microseconds, calls;
};
```



#### 客户端的创建和关闭



##### lua 脚本的伪客户端

会在服务器状态结构的lua_client 属性中 关联这个伪客户端

```c
struct redisServer {
    ....
    // Lua 环境
        lua_State *lua; /* The Lua interpreter. We use just one for all clients */

        // 复制执行 Lua 脚本中的 Redis 命令的伪客户端
        redisClient *lua_client;   /* The "fake client" to query Redis from Lua */

        // 当前正在执行 EVAL 命令的客户端，如果没有就是 NULL
        redisClient *lua_caller;   /* The client running EVAL right now, or NULL */

        // 一个字典，值为 Lua 脚本，键为脚本的 SHA1 校验和
        dict *lua_scripts;         /* A dictionary of SHA1 -> Lua scripts */
        // Lua 脚本的执行时限
        mstime_t lua_time_limit;  /* Script timeout in milliseconds */
        // 脚本开始执行的时间
        mstime_t lua_time_start;  /* Start time of script, milliseconds time */

        // 脚本是否执行过写命令
        int lua_write_dirty;  /* True if a write command was called during the
                                 execution of the current script. */

        // 脚本是否执行过带有随机性质的命令
        int lua_random_dirty; /* True if a random command was called during the
                                 execution of the current script. */

        // 脚本是否超时
        int lua_timedout;     /* True if we reached the time limit for script
                                 execution. */

        // 是否要杀死脚本
        int lua_kill;         /* Kill the script if true. */
    .....
};

```



##### AOF 文件的伪客户端

服务器在载入AOF 文件时，会创建用于执行 AOF 文件包含的redis 命令的伪客户端，并在载入完成后，关闭这个伪客户端。



#### 回顾

![image-20210714101639295](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714101639295.png)





### 14、服务器

redis 服务器负责与多个客户端建立网络连接，处理客户端发送的命令请求，在数据库中保存客户端执行命令所产生的数据，并通过资源管理来维持服务器自身的运转。



#### 一个set命令的执行过程

// **set key value**



1. **客户端向服务器发送命令请求 set key value**

   ![image-20210714102345097](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714102345097.png)

2. **服务器接收并处理客户端发来的命令请求 set key value，在数据库中进行设置操作，并产生命令回复 ok**

   1. 服务器读取套接字中协议格式的命令请求，并将其保存到客户端状态的输入缓冲区里面、

      ![image-20210714103006645](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714103006645.png)

   2. 对输入缓冲区中的命令请求进行分析，提取出命令请求中包含的命令参数，以及命令参数的个数，然后分别将参数和参数个数保存到客户端状态的 argv 属性和argc 属性里面。

      ![image-20210714103117388](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714103117388.png)

   3. 调用命令执行器，执行客户端指定的命令。

      **- 查找命令**

   ![image-20210714103756104](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714103756104.png)

   ![image-20210714104004935](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714104004935.png)

   ![image-20210714104018242](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714104018242.png)

   **- 执行预备操作**

   在真正执行命令前，还需要进行一些预备操作，从而确保命令可以正确、顺利地被执行。

   ![image-20210714104603937](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714104603937.png)

   **- 调用命令的实现函数**

   ```c
   // client 是指向客户端状态的指针
   client->cmd->proc(client);
   // 等于执行 setCommand(client);
   ```

   ![image-20210714104836082](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714104836082.png)

   被调用的命令实现函数会执行指定的操作，并产生响应的命令回复，回复会保存在客户端状态的输出缓冲区中（buf属性或reply 属性），之后会返回给客户端。

   ![image-20210714105143931](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714105143931.png)

   **- 执行后续工作**

   ![image-20210714105301710](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714105301710.png)

3. **服务器将命令回复 ok发送给客户端**

   如图14-7 中，当客户端的套接字变为可写状态时，命令回复处理器会将协议格式的命令回复"+OK\r\n" 发送给客户端。

   当命令回复发送完成后，回复处理器会**清空客户端状态的输出缓冲区**，为处理下一个命令请求做好准备。

4. **客户端接收服务器返回的命令回复 ok， 并将这个回复打印给用户看。**

![image-20210714105930944](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714105930944.png)



#### serverCron 函数

Redis 服务器中的serverCron 函数 默认每隔100毫秒执行一次，这个函数**负责管理服务器的资源，并保持服务器自身的良好运转**。

##### 更新服务器时间缓存

##### 更新服务器每秒执行命令次数

##### 更新服务器内存峰值记录

##### 处理SIGTERM信号

在启动服务器时，redis会为服务器进程的SIGTERM信号关联处理器 sigtermHandler 函数，这个信号处理器负责在服务器接到 SIGTERM 信号时，打开服务器状态的 shutdown_asap 标识。

##### 管理客户端资源

![image-20210714193910536](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714193910536.png)

##### 管理数据库资源

serverCron函数每次执行都会调用databasesCron函数，这个函数会对服务器中的一部分数据库进行检查，删除其中的过期键，并在有需要时，对字典进行收缩操作。

##### 检查持久化操作的运行状态

![image-20210714194418163](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714194418163.png)

##### 将AOF缓冲区的内容写入到 AOF 文件

如果服务器开启了AOF持久化功能，并且AOF缓冲区里面还有带写入的数据，那么serverCron函数会调用响应的程序，将AOF缓冲区中的内容写入到AOF 文件里。

##### 关闭异步客户端

服务器会关闭输出缓冲区大小超出限制的客户端。

##### 增加cronloops计数器的值

cronloops 属性记录了 serverCron函数的执行次数，函数每执行一次，该属性值就增一。

cronloops 属性主要的作用：在复制模块中实现，**没执行serverCron函数N次就执行一次指定代码**：

```c
// 伪码如下：
if cronloops % N == 0
    # 执行指定代码
```



#### 初始化服务器

##### 1、初始化服务器状态结构

​	初始化服务器第一步就是创建一个 struct redisServer 类型的实例变量作为服务器的状态，并为结构中的各个属性设置默认值。

```c
void initServerConfig() {
    int j;

    // 服务器状态

    // 设置服务器的运行 ID
    getRandomHexChars(server.runid,REDIS_RUN_ID_SIZE);
    // 设置默认配置文件路径
    server.configfile = NULL;
    // 设置默认服务器频率
    server.hz = REDIS_DEFAULT_HZ;
    // 为运行 ID 加上结尾字符
    server.runid[REDIS_RUN_ID_SIZE] = '\0';
    // 设置服务器的运行架构
    server.arch_bits = (sizeof(long) == 8) ? 64 : 32;
    // 设置默认服务器端口号
    server.port = REDIS_SERVERPORT;
    server.tcp_backlog = REDIS_TCP_BACKLOG;
    server.bindaddr_count = 0;
    ...
        
};
```



##### 2、载入配置选项

在启动服务器时，用户可以通过给定的配置参数或者配置文件来修改服务器的默认配置。

如：

```bash
$ redis-server --port 233333
# 通过给定配置参数的方式，修改了服务器的运行端口号。

$ redis-server redis.conf
# 并且 redis.conf 文件中有：
# 将服务器的数据库数量设置为32个
 datebases 32
# 关闭 RDB 文件的压缩功能
rdbcompression no
```

服务器在用initServerConfig 函数初始化完server 变量之后，就会开始载入用户给定的配置参数和配置文件，并根据用户设定的配置，对server 变量相关属性的值进行修改。



##### 3、初始化服务器数据结构

之前执行initServerConfig 函数初始化 server 状态时，程序只创建了命令表一个数据结构，在载入了用户指定的配置选项后，服务器将调用 initServer函数，为其他的数据结构分配内存，并在需要的时候，为这些数据结构设置或者关联初始化值。

为什么先载入用户指定的配置选项？因为若在执行initServerConfig 函数时就对数据结构进行初始化，那么一旦用户通过配置选项来修改和数据结构相关的服务器状态属性，服务器就要重新调整和修改已创建的数据结构。

##### 4、还原数据库状态

在完成对服务器状态server变量的初始化之后，服务器需要载入RDB文件或者AOF文件，并根据文件记录的内容来还原服务器的数据库状态。

开始AOF持久化功能会使用 AOF文件进行还原，否则使用 RDB文件进行还原。

服务器还原数据库状态后，将在日志中打印出载入文件并还原数据库状态所耗费的时长。

##### 5、执行事件循环

在初始化最后一步，服务器将打出如下日志：

![image-20210714203112308](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714203112308.png)

并开始执行服务器的事件循环（loop）。

此时，服务器的初始化工作圆满完成，服务器现在开始可以接收客户端的连接请求，并处理客户端发送的命令请求。



#### 回顾

![image-20210714203244884](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210714203244884.png)

## 多机数据库的实现

### 15、复制

Redis 通过 slaveof 命令 让一个服务器去复制另一个服务器，**即 主从复制。**

```bash
127.0.0.1:12345> SLAVEOF 127.0.0.6379
OK
# 127.0.0.12345 将成为 127.0.0.1:6379 的从服务器，而服务器 127.0.0.1:6379 则会成为127.0.0.1:12345 的主服务器。
```



#### 旧版复制功能的实现

旧版本，指 Redis 2.8版本以前的版本

redis复制分为 **同步（sync）** 和 **命令传播（command propagate）**

- 同步：将从服务器的数据库状态更新至主服务器当前所处的数据库状态
- 命令传播：在主服务器的数据库状态被修改，导致主从服务器的数据库状态出现不一致时，让主从服务器的数据库重新回到一致状态。

##### 同步

![image-20210716104832363](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716104832363.png)

![image-20210716104916074](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716104916074.png)

##### 命令传播

为了让主从服务器再次回到一致状态，主服务器需要对从服务器执行命令传播操作，主服务器会将造成主从不一致的命令，发送给从服务器执行，当从服务器执行了相同的命令后，主从服务器将再次回到一致状态。



#### 旧版复制功能的缺陷

主从服务器断开后，再次发送 SYNC 命令可以让主从服务器重新回到一致状态，但是传送 rdb 文件这一步实际上不是非做不可。这样做会比较低效。

![image-20210716105641586](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716105641586.png)

#### 新版复制功能的实现

从 2.8 版本，开始使用 PSYNC 命令 替代 SYNC 命令 来执行复制时的同步操作。

PSYNC 命令具有 完整同步（full resynchronization）和 部分同步（partial resynchronization）两种模式。

#### 部分重同步的实现

部分重同步功能由三个部分构成：

- 主服务器的复制偏移量（replication offset）和从服务器的复制偏移量
- 主服务器的复制积压缓冲区（replication backlog）
- 服务器的运行ID（run ID）

##### 复制偏移量

执行复制的双方——主从服务器会分别维护一个复制偏移量。

通过对比主从服务器的复制偏移量，就会很容易的知道主从服务器是否处于一致状态。

![image-20210716110750648](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716110750648.png)

##### 复制积压缓冲区

复制积压缓冲区是由主服务器维护的一个固定长度（fixed-size）先进先出（FIFO）队列， 默认大小为1MB。

当主服务器进行命令传播时，不仅会将写命令发送给所有的从服务器，还会将写命令写入到复制积压队列中：

![image-20210716111340763](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716111340763.png)

因此，主服务器的复制积压缓冲区中会保存着一部分最近传播的写命令，并为队列中的每个字节记录对应的复制偏移量。

当从服务器重新连上主服务器时，从服务器会通过 PSYNC 命令将自己的复制偏移量 offset 发送给主服务器，主服务器根据复制偏移量来决定对从服务器执行哪种同步操作：

-  如果offset 偏移量之后的数据仍然在复制积压缓冲区中，那么主服务器就会对从服务器执行部分重同步操作。
- 否则，若不在，则对从服务器执行完整重同步操作。

![image-20210716112155441](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716112155441.png)





##### 服务器运行 ID

每个redis 服务器不论主服务器还是从服务器，都会有自己的运行 ID，在服务器启动时自动生成，由 40  个随机十六进制字符组成。

在服务器重新连接上一个主服务器时，会发送 自己保存的 运行ID（之前第一连接主服务器时，保存的主服务器运行ID），如果这个 ID 和此时的主服务器ID相同，则尝试执行部分重同步操作，若不相同，则说明主服务器变了，此时的主服务器将对从服务器进行完整重同步操作。



#### PSYNC 命令的实现



![image-20210716113502544](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716113502544.png)

#### 复制的实现

首先通过从服务器发送 slaveof 命令

```bash
SLAVEOF 127.0.0.1 6379
OK
```

##### 步骤1. 设置服务器的地址和端口

将主服务器的地址和端口保存到服务器状态的 masterhost 属性和masterport 中：

![image-20210716114259627](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716114259627.png)

##### 步骤2. 建立套接字连接

![image-20210716114513271](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716114513271.png)



##### 步骤3. 发送PING命令

从服务器成为主服务器的客户端后，第一件事就是向主服务器发送一个 PING 命令。

发送PING命令的作用：

- [ ] 通过发送PING命令检查套接字的读写状态是否正常
- [ ] 接下来的复制操作必须在主服务器在可以处理正常命令请求的状态下进行，发送PING命令可以检查主服务器是否能正常处理命令请求。

![image-20210716114910827](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716114910827.png)

##### 步骤4. 身份验证

从服务器收到主服务器的 PONG 回复后，就要决定是否进行身份验证：

​	如果从服务器设置了 masterauth 选项，则进行身份验证，否则 不进行身份验证。

![image-20210716115234610](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716115234610.png)

##### 步骤5. 发送端口信息

身份验证之后，从服务器将向主服务器发送从服务器的监听端口号，主服务器接收到后会将端口号记录在从服务器对应的客户端状态的 slave_listening_port 属性中，这个属性主要是用于在主服务器端执行 INFO replication命令时打出从服务器的端口号。

![image-20210716153653710](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716153653710.png)

![image-20210716153707667](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716153707667.png)

##### 步骤6. 同步

在这一步，从服务器向主服务器发送 PSYNC 命令，执行同步操作，并将自己的数据库更新到主服务器数据库当前所处的状态。在同步操作执行之前，只有从服务器是主服务器的客户端，但是在执行了同步操作之后，主服务器也会成为从服务器的客户端，这是主服务器对从服务器执行命令传播操作的基础。

![image-20210716153826122](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716153826122.png)

##### 步骤7. 命令传播

完成同步后，主从服务器就会进入到命令传播阶段。

#### 心跳检测

在命令传播阶段，从服务器默认会以每秒一次的频率，向主服务器发送命令：

```bash
REPLCONF ACK <replication_offset>
# replication_offset 是从服务器当前的复制偏移量
```

##### # 发送 REPLCONF ACK 命令对于主从服务器的作用：

##### 1. 检测主从服务器的网络连接状态



##### 2. 辅助实现min-slaves 选项

redis 的 min-slaves-to-write 和 min-slaves-max-log 两个选项可以防止主服务器在不安全的情况下执行写命令。如：

```bash
min-slaves-to-write 3  # 在从服务器数量少于3个时 主服务器拒绝执行写命令
min-slaves-max-lag 10  # 在3个从服务器延迟（lag）值都大于等于 10 秒时，主服务器拒绝执行写命令
```

（INFO replication 命令返回的参数中有 lag 值）

![image-20210716155624140](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716155624140.png)

##### 3. 检测命令丢失

主服务器能及时发现命令丢失，而重新发送命令给从服务器。

![image-20210716155921835](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716155921835.png)

![image-20210716155934834](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716155934834.png)

#### 回顾

![image-20210716160047899](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210716160047899.png)

### 16、Sentinel 哨兵

哨兵是 Redis 的高可用性（high availability）解决方案：

**由一个sentinel实例组成的sentinel系统可以监视任意多个主服务器，已经这些主服务器属下的所有从服务器，并在被监视的主服务器进入下线状态时，自动将下线主服务器属下的某个从服务器升级为新的主服务器，然后由新的主服务器代替已下线的主服务器继续处理命令请求。**

![image-20210719090207601](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719090207601.png)

![image-20210719090220900](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719090220900.png)

![image-20210719090237075](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719090237075.png)

![image-20210719090248904](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719090248904.png)

#### 启动并初始化 Sentinel

步骤：

##### 1、初始化服务器

##### 2、将普通redis服务器使用的代码替换成sentinel 专用的代码

##### 3、初始化sentinel状态

##### 4、初始化sentinel状态的masters属性

sentinel 状态中的masters字典记录了所有被sentinel监视的主服务器的相关信息，其中：

- 字典的键是被监视主服务器的名字

 - 而字典的值则是被监视主服务器对应的 sentinel.c/sentinelRedisInstance 结构

   ```c
   // Sentinel 会为每个被监视的 Redis 实例创建相应的 sentinelRedisInstance 实例
   // （被监视的实例可以是主服务器、从服务器、或者其他 Sentinel ）
   typedef struct sentinelRedisInstance {
       
       // 标识值，记录了实例的类型，以及该实例的当前状态
       int flags;      /* See SRI_... defines */
       
       // 实例的名字
       // 主服务器的名字由用户在配置文件中设置
       // 从服务器以及 Sentinel 的名字由 Sentinel 自动设置
       // 格式为 ip:port ，例如 "127.0.0.1:26379"
       char *name;     /* Master name from the point of view of this sentinel. */
   
       // 实例的运行 ID
       char *runid;    /* run ID of this instance. */
   
       // 配置纪元，用于实现故障转移
       uint64_t config_epoch;  /* Configuration epoch. */
   
       // 实例的地址，保存着实例的 IP 地址 和 端口号
       sentinelAddr *addr; /* Master host. */
       
       ....
       
   };
   
   
   /* Address object, used to describe an ip:port pair. */
   /* 地址对象，用于保存 IP 地址和端口 */
   typedef struct sentinelAddr {
       char *ip;
       int port;
   } sentinelAddr;
   ```

   ![image-20210719094535195](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719094535195.png)

   ![image-20210719094617830](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719094617830.png)

   ![image-20210719094630128](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719094630128.png)

   ![image-20210719094426437](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719094426437.png)

   

##### 5、创建连向主服务器的网络连接

初始化的最后一步是创建连向被监视主服务器的网络连接，sentinel将成为服务器的客户端，它可以向主服务器发送命令，并从命令回复中获取相关的信息。

对于每个对sentinel监视的主服务器来说，sentinel会创建两个连向主服务器的异步网络连接：

​	一个是命令连接，专门用于向主服务器发送命令，并接收命令回复

​	另一个是订阅连接，专门用于订阅主服务器的 ___sentinel___:hello频道

![image-20210719095115340](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719095115340.png)

#### 获取主服务器信息

sentinel 默认会以每十秒一次的频率，通过命令连接向被监视的主服务器发送INFO命令，并通过分析INFO命令的回复来获取主服务器的当前信息。

![image-20210719142546255](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719142546255.png)

![image-20210719142537554](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719142537554.png)

![image-20210719142625479](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719142625479.png)

其中 主服务器实例结构的name属性的值，是用户使用 sentinel 配置文件设置的，而 从服务器实例结构的name属性的值则是 sentinel 根据从服务器的IP地址和端口号自动设置的。



#### 获取从服务器信息

当sentinel发现主服务器有新的从服务器出现时，sentinel 除了会为这个新的从服务器创建响应的实例结构外， sentinel 还会创建连接到从服务器的命令连接和订阅连接。

![image-20210719143246902](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719143246902.png)

创建命令连接后，sentinel默认还是十秒的频率通过命令连接向从服务器发送INFO命令，并获得回复：

![image-20210719143402694](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719143402694.png)

根据这些信息，sentinel会对从服务器的实例结构进行更新：

![image-20210719143455239](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719143455239.png)

#### 向主服务器和从服务器发送信息

![image-20210719143630624](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719143630624.png)

#### 接收来自主服务器和从服务器的频道信息

对于每个于sentinel连接的服务器，sentinel 既通过命令连接向服务器的 __sentinel:hello

频道发送信息，又通过订阅连接从服务器的__sentinel:hello 频道接收信息。

![image-20210719144441368](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719144441368.png)

![image-20210719144454366](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719144454366.png)



##### 更新sentinels 字典

一个sentinel会根据收到其他sentinel发来（发送信息的sentinel为源sentinel，接收消息的为目标sentinel）的信息来保存其他的sentinel，根据信息中提取出的主服务器参数，目标sentinel会在自己的sentinel状态的masters字典中查找相应的主服务器实例结构。

![image-20210719145227688](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719145227688.png)

##### 创建连向其他sentinel的命令连接

![image-20210719145320469](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719145320469.png)

**sentinel 之间不会创建订阅连接**

#### 检测主观下线状态

默认情况，sentinel 会以每秒一次的频率向所有与它创建了命令连接的实例发送PING命令，并通过返回的PING命令回复来判断实例是否在线。



![image-20210719150643324](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719150643324.png)

sentinel 的配置文件中的 down-after-milliseconds 选项指定了sentinel 判断实例进入主观下线所需的实践长度：

如果一个实例在 down-after-milliseconds 毫秒内，连续向sentinel 返回无效回复，那么sentinel会修改这个实例所对应的实例结构，在结构的flags 属性中打开 SRI_S_DOWN 标识，以此来表示这个实例已经进入主观下线状态。

**多个sentinel 设置的主观下线时长可能不同。**

#### 检查客观下线状态

当sentinel 将一个主服务器判断为主观下线后，为确认这个主服务器是否真的下线，还会向同样监视这个主服务器的其他sentinel进行询问，看它们是否也认为主服务器已经进入了下线状态（可以是 主观下线或者客观下线）。

**当sentinel从其他sentinel 那里收到足够数量的已下线判断之后，sentinel 会将从服务器判定为客观下线，并对主服务器执行故障转移操作。**

##### 1、发送SENTNEL is-master-down-by-addr 命令

##### 2、接收SENTNEL is-master-down-by-addr 命令

##### 3、就收SENTNEL is-master-down-by-addr 命令 的回复

根据命令回复，sentinel将统计其他sentinel同意主服务器已下线的数量，当这一数量达到配置指定的判断客观下线所需的数量时，sentinel就会将主服务器实例结构flags属性的 SRI_O_DOWN 标识打开，标识主服务器已经进入客观下线状态。

![image-20210719152216313](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719152216313.png)

```c
/* A Sentinel Redis Instance object is monitoring. */
/* 每个被监视的 Redis 实例都会创建一个 sentinelRedisInstance 结构
 * 而每个结构的 flags 值会是以下常量的一个或多个的并 */
// 实例是一个主服务器
#define SRI_MASTER  (1<<0)
// 实例是一个从服务器
#define SRI_SLAVE   (1<<1)
// 实例是一个 Sentinel
#define SRI_SENTINEL (1<<2)
// 实例已断线
#define SRI_DISCONNECTED (1<<3)
// 实例已处于 SDOWN 状态
#define SRI_S_DOWN (1<<4)   /* 主观下线  Subjectively down (no quorum). */
// 实例已处于 ODOWN 状态
#define SRI_O_DOWN (1<<5)   /* 客观下线  Objectively down (confirmed by others). */
// Sentinel 认为主服务器已下线
#define SRI_MASTER_DOWN (1<<6) /* A Sentinel with this flag set thinks that
                                   its master is down. */
// 正在对主服务器进行故障迁移
#define SRI_FAILOVER_IN_PROGRESS (1<<7) /* Failover is in progress for
                                           this master. */
// 实例是被选中的新主服务器（目前仍是从服务器）
#define SRI_PROMOTED (1<<8)            /* Slave selected for promotion. */
// 向从服务器发送 SLAVEOF 命令，让它们转向复制新主服务器
#define SRI_RECONF_SENT (1<<9)     /* SLAVEOF <newmaster> sent. */
// 从服务器正在与新主服务器进行同步
#define SRI_RECONF_INPROG (1<<10)   /* Slave synchronization in progress. */
// 从服务器与新主服务器同步完毕，开始复制新主服务器
#define SRI_RECONF_DONE (1<<11)     /* Slave synchronized with new master. */
// 对主服务器强制执行故障迁移操作
#define SRI_FORCE_FAILOVER (1<<12)  /* Force failover with master up. */
// 已经对返回 -BUSY 的服务器发送 SCRIPT KILL 命令
#define SRI_SCRIPT_KILL_SENT (1<<13) /* SCRIPT KILL already sent on -BUSY */
```

**不同sentinel 判断客观下线的条件可能不同。**

#### 选举领头 Sentinel

当一个主服务器被判断为客观下线时，监视这个下线主服务器的各个sentinel会进行协商，选举出一个领头的sentinel ，并由领头sentinel 对下线主服务器执行故障转移操作。

规则是：所有sentinel都可以申请做领头，然后选择半数以上被设置为局部领头的sentinel为新的领头sentinel。

![image-20210719173147215](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719173147215.png)

选举领头sentinel 时发送的 SENTNEL is-master-down-by-addr 命令 与检查客观下线状态时的SENTNEL is-master-down-by-addr 命令稍有不同，这次发送的命令会带有 sentinel自己的运行ID。

#### 故障转移

选出领头sentinel之后，它将对已下线的主服务器执行故障转移操作：

	- 从已下线主服务器的从服务器中选出一个，将其转换为主服务器；
	- 让已下线的主服务器的所有从服务器改为复制新的主服务器
	- 将已下线主服务器设置为新的主服务器的从服务器，当这个旧的主服务器重新上线时，就会成为新的主服务器的从服务器。

##### 1、选出新的主服务器

<img src="C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719190708320.png" alt="image-20210719190708320" style="zoom:80%;" />

##### 2、修改从服务器的复制目标

![image-20210719190753768](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719190753768.png)

![image-20210719190803198](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719190803198.png)

##### 3、将旧的主服务器变为从服务器

![image-20210719190840066](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719190840066.png)

#### 回顾

- [ ] sentinel 是一个运行在特殊模式下的redis 服务器，它使用了和普通模式不同的命令表，所以，sentinel 模式能够使用的命令和普通redis服务器能够使用的命令不同。

![image-20210719191052056](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719191052056.png)



### 17、集群

redis 集群式redis 提供的分布式数据库方案，集群通过**分片（sharding）**来进行数据共享，并提供复制和故障转移功能。

#### 节点

redis 集群是由多个节点（node）组成，刚开始时，每个节点是相互独立的，都处于一个只包含自己的集群中，我们需要将各个独立的节点连接起来，构成一个包含多个节点的集群。

通过 **CLUSTER MEET <ip> <port> 命令，让 node 节点与ip 和 port 指定的节点进行握手**，握手成功时，node 节点就会将 ip和port 所指定的节点添加到node 节点当前的集群中。

![image-20210719191946494](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719191946494.png)

##### 启动节点

redis 服务器启动时会根据 cluster-enabled 配置选项是否为 yes 来决定是否开始服务器的集群模式。

![image-20210719195503463](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719195503463.png)

节点会继续使用所有在单机模式中使用的服务器组件，除此之外，节点会继续使用redisServer 结构来保存服务器状态，使用redisClient结构来保存客户端的状态，那些只有在集群模式下才会用到的数据，保存在 cluster.h / clusterNode 、 cluster.h / clusterLink、 cluster.h / clusterState 结构中。

##### 集群数据结构

cluster.h / clusterNode 结构中保存了一个节点的当前状态，如节点的创建时间、节点的名字、节点当前的配置纪元、节点的IP地址和端口号等。

```c
// 节点状态
struct clusterNode {

    // 创建节点的时间
    mstime_t ctime; /* Node object creation time. */

    // 节点的名字，由 40 个十六进制字符组成
    // 例如 68eef66df23420a5862208ef5b1a7005b806f2ff
    char name[REDIS_CLUSTER_NAMELEN]; /* Node name, hex string, sha1-size */

    // 节点标识
    // 使用各种不同的标识值记录节点的角色（比如主节点或者从节点），
    // 以及节点目前所处的状态（比如在线或者下线）。
    int flags;      /* REDIS_NODE_... */

    // 节点当前的配置纪元，用于实现故障转移
    uint64_t configEpoch; /* Last configEpoch observed for this node */

    // 由这个节点负责处理的槽
    // 一共有 REDIS_CLUSTER_SLOTS / 8 个字节长
    // 每个字节的每个位记录了一个槽的保存状态
    // 位的值为 1 表示槽正由本节点处理，值为 0 则表示槽并非本节点处理
    // 比如 slots[0] 的第一个位保存了槽 0 的保存情况
    // slots[0] 的第二个位保存了槽 1 的保存情况，以此类推
    unsigned char slots[REDIS_CLUSTER_SLOTS/8]; /* slots handled by this node */

    // 该节点负责处理的槽数量
    int numslots;   /* Number of slots handled by this node */

    // 如果本节点是主节点，那么用这个属性记录从节点的数量
    int numslaves;  /* Number of slave nodes, if this is a master */

    // 指针数组，指向各个从节点
    struct clusterNode **slaves; /* pointers to slave nodes */

    // 如果这是一个从节点，那么指向主节点
    struct clusterNode *slaveof; /* pointer to the master node */

    // 最后一次发送 PING 命令的时间
    mstime_t ping_sent;      /* Unix time we sent latest ping */

	....
            // 节点的 IP 地址
    char ip[REDIS_IP_STR_LEN];  /* Latest known IP address of this node */

    // 节点的端口号
    int port;                   /* Latest known port of this node */

    // 保存连接节点所需的有关信息
    clusterLink *link;          /* TCP/IP link with this node */
    ....

};
```

link 属性是一个 clusterLink 结构，该结构保存了连接节点所需的有关信息，如套接字描述符，输入缓冲区和输出缓冲区：

```c
/* clusterLink encapsulates everything needed to talk with a remote node. */
// clusterLink 包含了与其他节点进行通讯所需的全部信息
typedef struct clusterLink {

    // 连接的创建时间
    mstime_t ctime;             /* Link creation time */

    // TCP 套接字描述符
    int fd;                     /* TCP socket file descriptor */

    // 输出缓冲区，保存着等待发送给其他节点的消息（message）。
    sds sndbuf;                 /* Packet send buffer */

    // 输入缓冲区，保存着从其他节点接收到的消息。
    sds rcvbuf;                 /* Packet reception buffer */

    // 与这个连接相关联的节点，如果没有的话就为 NULL
    struct clusterNode *node;   /* Node related to this link if any, or NULL */

} clusterLink;
```

redisClient 结构 和 clusterLink 结构的异同：

redisClient 和 clusterLink 结构都有自己的套接字描述符和输入输出缓冲区，两个结构的区别在于，redisClient 结构中的套接字和缓冲区是用于连接客户端的，而clusterLink 结构中的套接字和缓冲区则是用于连接节点的。



每个节点还保存着一个 clusterState 结构，这个结构记录了当前节点的视角下，集群目前所处的状态，如，集群是在线还是下线，集群包含的节点数，当前的纪元等。

```c
// 集群状态，每个节点都保存着一个这样的状态，记录了它们眼中的集群的样子。
// 另外，虽然这个结构主要用于记录集群的属性，但是为了节约资源，
// 有些与节点有关的属性，比如 slots_to_keys 、 failover_auth_count 
// 也被放到了这个结构里面。
typedef struct clusterState {

    // 指向当前节点的指针
    clusterNode *myself;  /* This node */

    // 集群当前的配置纪元，用于实现故障转移
    uint64_t currentEpoch;

    // 集群当前的状态：是在线还是下线
    int state;            /* REDIS_CLUSTER_OK, REDIS_CLUSTER_FAIL, ... */

    // 集群中至少处理着一个槽的节点的数量。
    int size;             /* Num of master nodes with at least one slot */

    // 集群节点名单（包括 myself 节点）
    // 字典的键为节点的名字，字典的值为 clusterNode 结构
    dict *nodes;          /* Hash table of name -> clusterNode structures */

    // 节点黑名单，用于 CLUSTER FORGET 命令
    // 防止被 FORGET 的命令重新被添加到集群里面
    // （不过现在似乎没有在使用的样子，已废弃？还是尚未实现？）
    dict *nodes_black_list; /* Nodes we don't re-add for a few seconds. */

    // 记录要从当前节点迁移到目标节点的槽，以及迁移的目标节点
    // migrating_slots_to[i] = NULL 表示槽 i 未被迁移
    // migrating_slots_to[i] = clusterNode_A 表示槽 i 要从本节点迁移至节点 A
    clusterNode *migrating_slots_to[REDIS_CLUSTER_SLOTS];

    // 记录要从源节点迁移到本节点的槽，以及进行迁移的源节点
    // importing_slots_from[i] = NULL 表示槽 i 未进行导入
    // importing_slots_from[i] = clusterNode_A 表示正从节点 A 中导入槽 i
    clusterNode *importing_slots_from[REDIS_CLUSTER_SLOTS];

    // 负责处理各个槽的节点
    // 例如 slots[i] = clusterNode_A 表示槽 i 由节点 A 处理
    clusterNode *slots[REDIS_CLUSTER_SLOTS];

    // 跳跃表，表中以槽作为分值，键作为成员，对槽进行有序排序
    // 当需要对某些槽进行区间（range）操作时，这个跳跃表可以提供方便
    // 具体操作定义在 db.c 里面
    zskiplist *slots_to_keys;

    /* The following fields are used to take the slave state on elections. */
    // 以下这些域被用于进行故障转移选举

    // 上次执行选举或者下次执行选举的时间
    mstime_t failover_auth_time; /* Time of previous or next election. */

    // 节点获得的投票数量
    int failover_auth_count;    /* Number of votes received so far. */

    // 如果值为 1 ，表示本节点已经向其他节点发送了投票请求
    int failover_auth_sent;     /* True if we already asked for votes. */

    int failover_auth_rank;     /* This slave rank for current auth request. */
	...
};
```

![image-20210719204254012](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719204254012.png)

##### CLUSTER MEET 命令的实现

![image-20210719204414110](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719204414110.png)

之后，节点A会将节点B的信息通过Gossip协议传播给集群中的其他节点，让其他节点也与B及进行握手，经过一段时间后，节点B就会被集群中所有节点认识。

#### 槽指派

redis 集群通过分片方式保存数据库中的键值对：集群的整个数据库被分为 16384 个槽，数据库中的每个键都属于这 16384 中的一个，集群中的每个节点可以处理 0 个或 最多 16384 个槽。

只有当数据库中所有 16384 个槽都有节点在处理时，集群才处于上线状态；

否则，若数据库中有任何一个槽没有得到处理，集群都处于下线状态。

##### 记录节点的槽指派信息

```c
// clusterNode 结构体中

    // 由这个节点负责处理的槽
    // 一共有 REDIS_CLUSTER_SLOTS / 8 个字节长
    // 每个字节的每个位记录了一个槽的保存状态
    // 位的值为 1 表示槽正由本节点处理，值为 0 则表示槽并非本节点处理
    // 比如 slots[0] 的第一个位保存了槽 0 的保存情况
    // slots[0] 的第二个位保存了槽 1 的保存情况，以此类推
    unsigned char slots[REDIS_CLUSTER_SLOTS/8]; /* slots handled by this node */

    // 该节点负责处理的槽数量
    int numslots;   /* Number of slots handled by this node */
```

![image-20210719205433752](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719205433752.png)

##### 传播节点的槽指派信息

一个节点除了将自己负责处理的槽记录在 clusterNode 结构的 slots 属性和numslots 属性之外，还会将自己的slots 数组通过消息发送给集群中其他的节点，告诉其他节点自己负责处理哪些槽。

​	**集群中每个节点都会知道数据库中 16384 个槽分别被指派给了集群中的哪些节点。**



##### 记录集群所有槽的指派信息

```c

// 集群状态，每个节点都保存着一个这样的状态，记录了它们眼中的集群的样子。
// 另外，虽然这个结构主要用于记录集群的属性，但是为了节约资源，
// 有些与节点有关的属性，比如 slots_to_keys 、 failover_auth_count 
// 也被放到了这个结构里面。
typedef struct clusterState {
    ....
    clusterNode *importing_slots_from[REDIS_CLUSTER_SLOTS];

    // 负责处理各个槽的节点
    // 例如 slots[i] = clusterNode_A 表示槽 i 由节点 A 处理
    clusterNode *slots[REDIS_CLUSTER_SLOTS];
    ....
};
```

![image-20210719210239629](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719210239629.png)

​	**clusterState.slots数组记录了集群中所有槽的指派信息，而 clusterNode.slots 数组只记录了 clusterNode 结构所代表的节点的槽指派信息，**这是两个 slots 数组的关键区别。



##### CLUSTER ADDSLOTS 命令实现

CLUSTER ADDSLOTS 命令 接收一个或多个槽作为参数，并将所有的输入槽指派给接收该命令的节点负责：

```c
CLUSTER ADDSLOTS <slots> <slot ...>
```



![image-20210719210958969](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719210958969.png)

![image-20210719211336505](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719211336505.png)

![image-20210719211355629](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210719211355629.png)

在CLUSTER ADDSLOTS 命令执行完毕后，节点会通过发送消息告知集群中的其他节点，自己目前正负责处理哪些槽。



#### 在集群中执行命令

将数据库中的16384 个槽都进行了指派后，集群就会进入到上线状态，这是客户端就可以向集群中的节点发送数据命令了。

![image-20210720084744025](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720084744025.png)



##### 计算键属于哪个槽

CLUSTER KEYSLOT <key> 命令可以查看一个给定的键属于哪个槽。

![image-20210720085629945](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720085629945.png)

##### 判断槽是否由当前节点负责

计算出节点属于哪个槽之后，就判断这个槽是否有自己负责，是则执行客户端发送的命令；否则向客户端返回 MOVED 命令，指引客户端转向正在处理该槽的节点。

##### MOVED 错误

MOVED 错误的格式：

MOVED <slot> <ip> : <port>

slot 为键所在的槽，ip 和 port 则是负责处理槽slot 的节点的ip地址和端口号。

一个集群客户端通常会与集群中的多个节点**创建套接字连接**，而所谓的节点转向实际上就是**换一个套接字来发送命令。**

##### 节点数据库的实现

节点和单机服务器在数据库方面的一个区别是：节点只能使用0号数据库，而单机redis服务器则没有这一限制。

节点使用 clusterState 结构中的 slots_to_keys 跳跃表来保存 槽 和 键 之间的关系：

```c
    // 跳跃表，表中以槽作为分值，键作为成员，对槽进行有序排序
    // 当需要对某些槽进行区间（range）操作时，这个跳跃表可以提供方便
    // 具体操作定义在 db.c 里面
    zskiplist *slots_to_keys;
```

![image-20210720091618740](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720091618740.png)

![image-20210720091631873](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720091631873.png)

![image-20210720091704544](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720091704544.png)

#### 重新分片

redis 集群的重新分片可以将任意数量已经指派给某个节点（源节点）的槽改为指派给另一个节点（目标节点），并且相关槽所属的键值对也会从源节点被移动到目标节点。

**重新分片擦操作可以在线 进行，在重新分片过程中，集群不需要下线，并且源节点和目标节点都可以继续处理命令请求。**



##### 重新分片的实现原理

redis集群的重新分片操作是由redis 的集群管理软件 redis-trib 负责执行的。

![image-20210720093821823](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720093821823.png)

![image-20210720093933933](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720093933933.png)



#### ASK 错误

和moved 错误相似，当一部分的键值对在源节点，而另一个分则在目标节点中。

![image-20210720094355818](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720094355818.png)

##### CLUSTER SETSLOT IMPORTING  命令的实现

clusterState 结构的 importing_slots_from数组记录了当前节点正在从其他节点导入的槽。

CLUSTER SETSLOT <i> IMPORTING <source_id>

![image-20210720095158645](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720095158645.png)



##### CLUSTER SETSLOT MIGRATING 命令的实现

clusterState 结构的 migrating_slots_to 数组记录了当前节点正在迁移至其他节点的槽。

CLUSTER SETSLOT <i> MIGRATING <target_id>

![image-20210720095615264](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720095615264.png)

##### ASK 错误



##### ASKING 命令

ASKING 命令唯一要做的就是打开发送该命令的客户端的 REDIS_ASKING 标识。

![image-20210720100000797](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720100000797.png)



##### ASK错误和MOVED 错误的区别

![image-20210720100155478](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720100155478.png)



#### 复制与故障转移

redis 集群中的节点分为 主节点 和 从节点，其中主节点用于处理槽，而从节点则用于复制某个主节点，并在被复制的主节点下线时，代替下线主节点继续处理命令请求。

##### 设置从节点

向一个节点发送命令：

CLUSTER REPLICATE <node_id>  可以让接受命令的节点成为node_id 所指定节点的从节点，并开始对主节点进行复制。

##### 故障检测

集群中节点定期向其他的节点发送PING 信息。未在规定时间内返回PONG消息，就会在flags属性中打开 REDIS_NODE_PFAIL 标识，标识节点疑似下线状态（PFAIL）。

如果一个集群里，半数以上负责处理槽的主节点都将某个主节点x报告为疑似下线，那么这个主节点x将被标记为已下线（FAIL），将主节点x标记为已下线的节点会向集群广播一条关于主节点x的FAIL消息，所有收到这条FAIL消息的节点都会立即将主节点x标记为已下线。

![image-20210720140021450](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720140021450.png)

##### 故障转移

当一个从节点发现自己正在复制的主节点进入到了已下线状态时，从节点将开始对下线主节点进行故障转移：

1. 复制下线主节点的所有从节点里，会有一个从节点被选中
2. 被选中的从节点会执行 SLAVEOF no one 命令，成为新的主节点
3. 新的主节点会撤销所有对已下线主节点的槽指派，并将这些槽全部指派给自己
4. 新的主节点向集群广播一条 PONG 消息，这条 PONG 消息可以让集群中的其他节点立即知道这个节点已经由从节点变成主节点，并且这个主节点已经接管了原本由已下线节点负责处理的槽。
5. 新的主节点开始接收和自己负责处理的槽有关的命令请求，故障转移完成。



##### 选举新的主节点

新的主节点通过选举产生，

集群的配置纪元是个自增器，集群的某个几点开始一次故障转移操作，集群配置纪元就会增一；

对于每个配置纪元，集群里每个负责处理槽的主节点都有一次投票的机会，而第一个向主节点要求投票的从节点将获得主节点的投票；

当从节点发现自己正在复制的主节点进入已下线状态，从节点会向集群广播一条 CLUSTERMSG_TYPE_FAILOVER_AUTH_REQUEST  消息，要求所有收到这条消息、并且具有投票权的主节点向这个从节点投票；

![image-20210720142354520](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720142354520.png)



#### 消息

集群的各个节点通过发送和接收消息来进行通信。节点发送的消息主要有5种。

**MEET消息、PING消息、PONG消息、FAIL消息、PUBLISH消息**

一切消息由 消息头（header）和消息正文（data）组成。

##### 消息头

每个消息头都由一个 cluster.h/clusterMsg 结构表示：

```c
// 用来表示集群消息的结构（消息头，header）
typedef struct {
    char sig[4];        /* Siganture "RCmb" (Redis Cluster message bus). */
    // 消息的长度（包括这个消息头的长度和消息正文的长度）
    uint32_t totlen;    /* Total length of this message */
    uint16_t ver;       /* Protocol version, currently set to 0. */
    uint16_t notused0;  /* 2 bytes not used. */

    // 消息的类型
    uint16_t type;      /* Message type */

    // 消息正文包含的节点信息数量
    // 只在发送 MEET 、 PING 和 PONG 这三种 Gossip 协议消息时使用
    uint16_t count;     /* Only used for some kind of messages. */

    // 消息发送者的配置纪元
    uint64_t currentEpoch;  /* The epoch accordingly to the sending node. */

    // 如果消息发送者是一个主节点，那么这里记录的是消息发送者的配置纪元
    // 如果消息发送者是一个从节点，那么这里记录的是消息发送者正在复制的主节点的配置纪元
    uint64_t configEpoch;   /* The config epoch if it's a master, or the last
                               epoch advertised by its master if it is a
                               slave. */

    // 节点的复制偏移量
    uint64_t offset;    /* Master replication offset if node is a master or
                           processed replication offset if node is a slave. */

    // 消息发送者的名字（ID）
    char sender[REDIS_CLUSTER_NAMELEN]; /* Name of the sender node */

    // 消息发送者目前的槽指派信息
    unsigned char myslots[REDIS_CLUSTER_SLOTS/8];

    // 如果消息发送者是一个从节点，那么这里记录的是消息发送者正在复制的主节点的名字
    // 如果消息发送者是一个主节点，那么这里记录的是 REDIS_NODE_NULL_NAME
    // （一个 40 字节长，值全为 0 的字节数组）
    char slaveof[REDIS_CLUSTER_NAMELEN];

    char notused1[32];  /* 32 bytes reserved for future usage. */

    // 消息发送者的端口号
    uint16_t port;      /* Sender TCP base port */

    // 消息发送者的标识值
    uint16_t flags;     /* Sender node flags */

    // 消息发送者所处集群的状态
    unsigned char state; /* Cluster state from the POV of the sender */

    // 消息标志
    unsigned char mflags[3]; /* Message flags: CLUSTERMSG_FLAG[012]_... */

    // 消息的正文（或者说，内容）
    union clusterMsgData data;

} clusterMsg;
```

clusterMsgData.data 属性指向 联合  cluster.h/clusterMsgData，这个联合就是消息的正文：

```c
union clusterMsgData {

    /* PING, MEET and PONG */
    struct {
        /* Array of N clusterMsgDataGossip structures */
        // 每条消息都包含两个 clusterMsgDataGossip 结构
        clusterMsgDataGossip gossip[1];
    } ping;

    /* FAIL */
    struct {
        clusterMsgDataFail about;
    } fail;

    /* PUBLISH */
    struct {
        clusterMsgDataPublish msg;
    } publish;

    /* UPDATE */
    struct {
        clusterMsgDataUpdate nodecfg;
    } update;

};
```



##### MEET、PING、PONG 消息的实现

##### FAIL消息的实现

##### PUBLISH消息的实现



#### 回顾

![image-20210720144655614](C:\Users\Perry\AppData\Roaming\Typora\typora-user-images\image-20210720144655614.png)



## 独立功能的实现

### 发布与订阅

### 事务

### Lua 脚本

### 排序

### 二进制位数组

### 慢查询日志

### 监视器










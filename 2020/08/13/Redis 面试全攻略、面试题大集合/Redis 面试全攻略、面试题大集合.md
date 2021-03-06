
> 本文转载自 **后端技术指南针** 微信公众号。

# 0.前言

 通过本文你将了解到以下内容： 
 - Redis的作者、发展演进和江湖地位
 - Redis面试问题的概况
 - Redis底层实现相关的问题包括：
. **常用数据类型底层实现、SDS的原理和优势、字典的实现原理、跳表和有序集合的原理、Redis的线程模式和服务模型**

``温馨提示：``内容并不难，就怕你不看。

看不懂可以先收藏先Mark，等到深入研究的时间再翻出来看看，你就发现真是24K干货呀！停止吹嘘，写点不一样的文字吧！

# 1.Redis往事
Redis是一个使用ANSI C编写的开源、支持网络、基于内存、可选持久化的高性能键值对数据库。Redis的之父是来自意大利的西西里岛的Salvatore Sanfilippo，Github网名antirez，笔者找了作者的一些简要信息并翻译了一下，如图：
![Salvatore Sanfilippo 简介](https://img-blog.csdnimg.cn/20200727103242732.png#pic_center)
从2009年第一个版本起Redis已经走过了10个年头，目前Redis仍然是最流行的key-value型内存数据库的之一。

优秀的开源项目离不开大公司的支持，在2013年5月之前，其开发由**VMware**赞助，而2013年5月至2015年6月期间，其开发由**毕威拓**赞助，从2015年6月开始，Redis的开发由**Redis Labs**赞助。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727103446276.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMzczNjgx,size_16,color_FFFFFF,t_70#pic_center)
笔者也使用过一些其他的NoSQL，有的支持的value类型非常单一，因此很多操作都必须在客户端实现，比如value是一个结构化的数据，需要修改其中某个字段就需要整体读出来修改再整体写入，显得很笨重，但是Redis的value支持多种类型，实现了很多操作在服务端就可以完成了，这个对客户端而言非常方便。

当然Redis由于是内存型的数据库，数据量存储量有限而且分布式集群成本也会非常高，因此有很多公司开发了基于SSD的类Redis系统，比如``360开发的SSDB、Pika等数据库``，但是笔者认为从``0到1的难度是大于从1到2的难度``的，毋庸置疑Redis是NoSQL中浓墨重彩的一笔，值得我们去深入研究和使用。

# 2.Redis的江湖地位
Redis提供了Java、C/C++、C#、 PHP 、JavaScript、 Perl 、Object-C、Python、Ruby、Erlang、Golang等``多种主流语言的客户端``，因此无论使用者是什么语言栈总会找到属于自己的那款客户端，受众非常广。

笔者查了datanyze.com网站看了下Redis和MySQL的``最新市场份额和排名``对比以及``全球Top站点的部署量</font>对比(网站数据更新到写作当日2019.12.11)：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727103923668.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMzczNjgx,size_16,color_FFFFFF,t_70#pic_center)
可以看到``Redis总体份额排名第9并且在全球Top100站点中部署数量与MySQL基本持平``，所以Redis还是有一定的江湖地位的。

# 3.聊聊实战
目前Redis发布的稳定版本已经到了5.x（截止当前时间，稳定版本已经到了6.0.6）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727104200755.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMzczNjgx,size_16,color_FFFFFF,t_70#pic_center)
功能也越来越强大，从国内外互联网公司来看Redis几乎是``标配``了。作为开发人员在日常笔试面试和工作中遇到Redis相关问题的概率非常大，掌握Redis的相关知识点都十分有必要。

``学习和梳理一个复杂的东西肯定不能胡子眉毛一把抓``，每个人都有自己的认知思路，笔者认为要从充分掌握Redis需要``从底向上、从外到内``去理解Redis。

Redis的实战知识点可以简单分为``三个层次``：

 - ``底层实现``：主要是从Redis的源码中提炼的问题，包括但不限于底层数据结构、服务模型、算法设计等。
 - 
   ``基础架构``：可用概况为Redis整体对外的功能点和表现，包括但不限于单机版主从架构实现、主从数据同步、哨兵机制、集群实现、分布式一致性、故障迁移等。
 -   ``实际应用``：实战中Redis可用帮你做什么，包括但不限于单机缓存、分布式缓存、分布式锁、一些应用。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727104557708.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMzczNjgx,size_16,color_FFFFFF,t_70#pic_center)
# 4.底层实现热点题目
底层实现篇的题目主要是与Redis的源码和设计相关，可以说是Redis功能的基石，了解底层实现可以让我们更好地掌握功能，由于底层代码很多，在后续的基础架构篇中仍然会穿插源码来分析，因此本篇只列举一些热点的问题。

## Q1: Redis常用五种数据类型是如何实现的？
Redis支持的常用5种数据类型指的是value类型，分别为：**字符串String、列表List、哈希Hash、集合Set、有序集合Zset**，但是Redis后续又丰富了几种数据类型分别是Bitmaps、HyperLogLogs、GEO。

由于Redis是基于标准C写的，只有最基础的数据类型，因此Redis为了满足对外使用的5种数据类型，开发了属于自己**独有的一套基础数据结构**，使用这些数据结构来实现5种数据类型。

Redis底层的数据结构包括：简单动态数组SDS、链表、字典、跳跃链表、整数集合、压缩列表、对象。

Redis为了**平衡空间和时间效率**，针对value的具体类型在底层**会采用不同的数据结构来实现**，其中哈希表和压缩列表是复用比较多的数据结构，如下图展示了对外数据类型和底层数据结构之间的映射关系：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727104836150.png#pic_center)
从图中可以看到**ziplist压缩列表**可以作为Zset、Set、List三种数据类型的底层实现，看来很强大，压缩列表是一种为了**节约内存而开发的且经过特殊编码之后的连续内存块顺序型数据结构**，底层结构还是比较复杂的。

## Q2: Redis的SDS和C中字符串相比有什么优势？
在C语言中使用N+1长度的字符数组来表示字符串，尾部使用``'\0'``作为结尾标志，对于此种实现**无法满足Redis对于安全性、效率、丰富的功能的要求**，因此Redis单独封装了SDS简单动态字符串结构。

在理解SDS的优势之前需要先看下SDS的**实现细节**，找了github最新的src/sds.h的定义看下：

```c
typedef char *sds;

/*这个用不到 忽略即可*/
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};

/*不同长度的header 8 16 32 64共4种 都给出了四个成员
len：当前使用的空间大小；alloc去掉header和结尾空字符的最大空间大小
flags:8位的标记 下面关于SDS_TYPE_x的宏定义只有5种 3bit足够了 5bit没有用
buf:这个跟C语言中的字符数组是一样的，从typedef char* sds可以知道就是这样的。
buf的最大长度是2^n 其中n为sdshdr的类型，如当选择sdshdr16，buf_max=2^16。
*/
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};

#define SDS_TYPE_5  0
#define SDS_TYPE_8  1
#define SDS_TYPE_16 2
#define SDS_TYPE_32 3
#define SDS_TYPE_64 4
#define SDS_TYPE_MASK 7
#define SDS_TYPE_BITS 3
```

看了前面的定义，笔者画了个图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727105100555.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMzczNjgx,size_16,color_FFFFFF,t_70#pic_center)

从图中可以知道sds本质分为三部分：header、buf、null结尾符，其中header可以认为是整个sds的指引部分，给定了使用的空间大小、最大分配大小等信息，再用一张网上的图来清晰看下sdshdr8的实例：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727105141794.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMzczNjgx,size_16,color_FFFFFF,t_70#pic_center)

在sds.h/sds.c源码中可清楚地看到sds完整的实现细节，本文就不展开了要不然篇幅就过长了，快速进入主题说下**sds的优势**：

 - **O(1)获取长度**: C字符串需要遍历而sds中有len可以直接获得；
   
 - **防止缓冲区溢出bufferoverflow**:
   当sds需要对字符串进行修改时，首先借助于len和alloc检查空间是否满足修改所需的要求，如果空间不够的话，SDS会``自动扩展空间``，避免了像C字符串操作中的覆盖情况；
   
 - **有效降低内存分配次数**：C字符串在涉及增加或者清除操作时会改变底层数组的大小造成重新分配、sds使用了``空间预分配和惰性空间释放``机制，说白了就是每次在扩展时是成倍的多分配的，在缩容是也是先留着并不正式归还给OS，这两个机制也是比较好理解的；
   
 - **二进制安全**：C语言字符串只能保存ascii码，对于图片、音频等信息无法保存，sds是二进制安全的，写入什么读取就是什么，不做任何过滤和限制；

老规矩上一张黄健宏大神总结好的图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727105420771.png#pic_center)
## Q3:Redis的字典是如何实现的？简述渐进式rehash的过程。
字典算是Redis5中常用数据类型中的明星成员了，前面说过字典可以基于ziplist和hashtable来实现，我们只讨论基于**hashtable实现**的原理。

字典是个**层次非常明显的数据类型**，如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727105515993.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMzczNjgx,size_16,color_FFFFFF,t_70#pic_center)
有了个大概的概念，我们看下最新的src/dict.h**源码定义**：

```c
//哈希节点结构
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

//封装的是字典的操作函数指针
typedef struct dictType {
    uint64_t (*hashFunction)(const void *key);
    void *(*keyDup)(void *privdata, const void *key);
    void *(*valDup)(void *privdata, const void *obj);
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    void (*keyDestructor)(void *privdata, void *key);
    void (*valDestructor)(void *privdata, void *obj);
} dictType;

/* This is our hash table structure. Every dictionary has two of this as we
 * implement incremental rehashing, for the old to the new table. */
//哈希表结构 该部分是理解字典的关键
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

//字典结构
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    unsigned long iterators; /* number of iterators currently running */
} dict;
```

C语言的好处在于定义必须是由最底层向外的，因此我们可以看到一个明显的层次变化，于是笔者又画一图来展现具体的``层次``概念：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727105613601.png#pic_center)
- 关于dictEntry

dictEntry是哈希表节点，也就是我们存储数据地方，其保护的成员有：key,v,next指针。key保存着键值对中的键，v保存着键值对中的值，值可以是一个指针或者是uint64_t或者是int64_t。next是指向另一个哈希表节点的指针，这个指针可以将多个哈希值相同的键值对连接在一次，以此来**解决哈希冲突**的问题。

如图为两个冲突的哈希节点的连接关系：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727105706611.png#pic_center)
- 关于dictht

从源码看哈希表包括的成员有table、size、used、sizemask。table是一个数组，数组中的每个元素都是一个指向dictEntry结构的指针， 每个dictEntry结构保存着一个键值对；size 属性记录了哈希表table的大小，而used属性则记录了哈希表目前已有节点的数量。sizemask等于size-1和哈希值计算一个键在table数组的索引，也就是计算index时用到的。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727105753158.png#pic_center)
如上图展示了一个大小为4的table中的哈希节点情况，其中k1和k0在index=2发生了哈希冲突，进行开链表存在，本质上是先存储的k0，``k1放置是发生冲突为了保证效率直接放在冲突链表的最前面，因为该链表没有尾指针``。

- 关于dict

从源码中看到dict结构体就是字典的定义，包含的成员有type，privdata、ht、rehashidx。其中dictType指针类型的type指向了操作字典的api，理解为函数指针即可，``ht是包含2个dictht的数组``，也就是字典包含了2个哈希表，rehashidx进行rehash时使用的变量，privdata配合

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727105809212.png#pic_center)
- 字典的哈希算法

```c
//伪码：使用哈希函数，计算键key的哈希值
hash = dict->type->hashFunction(key);
//伪码：使用哈希表的sizemask和哈希值，计算出在ht[0]或许ht[1]的索引值
index = hash & dict->ht[x].sizemask;
//源码定义
#define dictHashKey(d, key) (d)->type->hashFunction(key)
```

redis使用MurmurHash算法计算哈希值，该算法最初由Austin Appleby在2008年发明，MurmurHash算法的无论数据输入情况如何都可以给出随机分布性较好的哈希值并且计算速度非常快，目前有MurmurHash2和MurmurHash3等版本。

- 普通Rehash重新散列

哈希表保存的键值对数量是**动态变化**的，为了让哈希表的负载因子维持在一个合理的范围之内，就需要对哈希表进行扩缩容。

扩缩容是通过执行rehash重新散列来完成，对字典的哈希表``执行普通rehash的基本步骤为分配空间->逐个迁移->交换哈希表``，详细过程如下：

 1. 为字典的ht[1]哈希表分配空间，分配的空间大小取决于要执行的操作以及ht[0]当前包含的键值对数量：
	 扩展操作时ht[1]的大小为第一个大于等于ht[0].used*2的2^n；
	 收缩操作时ht[1]的大小为第一个大于等于ht[0].used的2^n ；
	 **扩展时比如h[0].used=200，那么需要选择大于400的第一个2的幂，也就是2^9=512。**
 2. 将保存在ht[0]中的所有键值对重新计算键的哈希值和索引值rehash到ht[1]上；
 3. 重复rehash直到ht[0]包含的所有键值对全部迁移到了ht[1]之后释放 ht[0]， 将ht[1]设置为 ht[0]，并在ht[1]新创建一个空白哈希表， 为下一次rehash做准备。

- 渐进Rehash过程

Redis的rehash动作**并不是一次性完成的，而是分多次、渐进式地完成的**，原因在于当哈希表里保存的键值对数量很大时， 一次性将这些键值对全部rehash到ht[1]可能会**导致服务器在一段时间内停止服务**，这个是无法接受的。

针对这种情况Redis采用了**渐进式rehash**，过程的详细步骤：

 1. 为ht[1]分配空间，这个过程和普通Rehash没有区别；
 2. 将rehashidx设置为0，表示rehash工作正式开始，同时这个rehashidx是递增的，从0开始表示从数组第一个元素开始rehash。
 3. 在rehash进行期间，每次对**字典执行增删改查操作**时，``顺带``将ht[0]哈希表在rehashidx索引上的键值对rehash到 ht[1]，完成后将rehashidx加1，指向下一个需要rehash的键值对。
 4. 随着字典操作的不断执行，最终ht[0]的所有键值对都会被rehash至ht[1]，再将rehashidx属性的值设为-1来表示 rehash操作已完成。

渐进式 rehash的思想在于**将rehash键值对所需的计算工作分散到对字典的每个添加、删除、查找和更新操作上，从而避免了集中式rehash而带来的阻塞问题。**

看到这里不禁去想这种**捎带脚式**的rehash会不会**导致整个过程非常漫长***？如果某个value一直没有操作那么需要扩容时由于一直不用所以影响不大，需要缩容时如果一直不处理可能造成内存浪费，具体的还没来得及研究，**先埋个问题吧！**

## Q4:跳跃链表了解吗？Redis的Zset如何使用跳表实现的？
ZSet这种数据类型也非常有用，在做排行榜需求时非常有用，笔者就曾经使用这种数据类型来实现某日活2000w的app的排行榜，所以了解下ZSet的底层实现很有必要，之前笔者写过两篇文章介绍跳跃链表和ZSet的实现，因此查阅即可。
[深入理解跳跃链表[一]](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMTI2Ng==&mid=2247483919&idx=1&sn=1b62c9a125be1bed7970aae906639d21&scene=21#wechat_redirect)
[深入理解跳表在Redis中的应用](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMTI2Ng==&mid=2247483948&idx=1&sn=83c1ef41690a60d0b70fa4374d6e64b8&scene=21#wechat_redirect)

## Q5:Redis为什么使用单线程？讲讲Redis网络模型以及单线程如何协调各种事件运行起来的？
Redis在新版本中并不是单纯的单线程服务，一些辅助工作会有BIO后台线程来完成，并且Redis底层使用epoll来实现了基于事件驱动的反应堆模式，在整个主线程运行工程中不断协调时间事件和文件事件来完成整个系统的运行，笔者之前写过两篇相关的文章，查阅即可得到更深层次的答案。
[理解Redis单线程运行模式](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMTI2Ng==&mid=2247483755&idx=1&sn=20496502810f53409b1274afcc76a997&scene=21#wechat_redirect)
[理解Redis的反应堆模式](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMTI2Ng==&mid=2247483747&idx=1&sn=184f86ec27472736f9d0f808b54fe753&scene=21#wechat_redirect)
[浅析Redis 4.0新特性之LazyFree](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMTI2Ng==&mid=2247483717&idx=1&sn=dfa5cafa7695f2d0dfb8bc51b414d46b&scene=21#wechat_redirect)

## Q6:了解Redis的内存回收吗？讲讲你的理解
### 1.1 为什么要回收内存？
Redis作为内存型数据库，如果单纯的只进不出早晚就**撑爆**了，事实上很多把Redis当做主存储DB用的家伙们早晚会尝到这个**苦果**，当然除非你家厂子确实**不差钱**，数T级别的内存都**毛毛雨**，或者数据增长一定程度之后不再增长的场景，就另当别论了。

对于我们这种把节约成本当做**KPI**的普通厂子，还是把Redis当**缓存**用比较符合家里的经济条件，所以这么看面试官的问题还算是比较贴合实际，比起那些**手撕RBTree**好一些，如果问题刚好在你知识射程范围内，先给面试官**点个赞**再说！

为了让Redis服务**安全稳定**的运行，让使用内存保持在一定的**阈值内**是非常有必要的，因此我们就需要删除该删除的，清理该清理的，把内存留给需要的键值对，试想一条大河需要设置几个**警戒水位来确保不决堤不枯竭**，Redis也是一样的，只不过Redis只关心决堤即可，来一张图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727111416193.png#pic_center)
图中设定机器内存为128GB，占用64GB算是比较安全的水平，如果内存接近80%也就是100GB左右，那么认为Redis目前承载能力已经比较大了，具体的比例可以根据公司和个人的业务经验来确定。

笔者只是想表达出于**安全和稳定的考虑**，不要觉得128GB的内存就意味着存储128GB的数据，都是要**打折**的。

### 1.2 内存从哪里回收？
Redis占用的内存是分为两部分：**存储键值对消耗和本身运行消耗**。显然后者我们无法回收，因此只能从键值对下手了，键值对可以分为几种：**带过期的、不带过期的、热点数据、冷数据**。对于带过期的键值是需要删除的，如果删除了所有的过期键值对之后内存仍然不足怎么办？那只能把部分**数据给踢掉**了。

**人生无处不取舍**，这个让笔者脑海浮现了《泰坦尼克》，邮轮撞到了冰山顷刻间海水涌入，面临数量不足的救生艇，人们做出了抉择：让女士和孩童先走，绅士们选择留下，海上逃生场景如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727111541422.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMzczNjgx,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727111541423.png)
### 1.3 如何实施过期键值对的删除？
要实施对键值对的删除我们需要明白如下几点：

- 带过期超时的键值对**存储在哪里**？

- 如何判断带超时的键值对**是否可以被删除**了？

- 删除机制**有哪些**以及**如何选择**？

#### 1.3.1 键值对的存储
老规矩来到github看下源码，src/server.h中给的redisDb结构体给出了答案：

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
Redis本质上就是一个大的key-value，key就是字符串，value有是几种对象：字符串、列表、有序列表、集合、哈希等，这些key-value都是**存储在redisDb的dict中的**，来看下黄健宏画的一张非常赞的图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727111750875.png#pic_center)
看到这里，对于删除机制又清晰了一步，我们只要把redisDb中dict中的目标key-value删掉就行，不过貌似没有这么简单，Redis对于过期键值对肯定有自己的组织规则，让我们继续研究吧！

redisDb的``expires``成员的类型也是dict，和键值对是一样的，本质上``expires是dict的子集``，expires保存的是所有带过期的键值对，称之为``过期字典``吧，它才是我们研究的重点。

对于键，我们可以设置``绝对和相对过期时间``、以及查看``剩余时间``：

- 使用``EXPIRE和PEXPIRE``来实现键值对的秒级和毫秒级生存时间设定，这是``相对时长``的过期设置

- 使用``EXPIREAT和EXPIREAT``来实现键值对在某个秒级和毫秒级时间戳时进行过期删除，属于``绝对过期``设置

- 通过``TTL和PTTL``来查看带有生存时间的键值对的剩余过期时间

上述三组命令在``设计缓存``用处比较大，有心的读者可以留意。

过期字典expires和键值对空间dict存储的内容并不完全一样，过期字典expires的key是指向Redis对应对象的指针，其value是long long型的``unix时间戳``，前面的EXPIRE和PEXPIRE相对时长最终也会转换为时间戳，来看下``过期字典expires的结构``，笔者画了个图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727111938129.png#pic_center)
#### 1.3.2 键值对的过期删除判断
判断键是否过期可删除，需要先查``过期字典是否存在该值``，如果存在则进一步判断``过期时间戳``和``当前时间戳``的``相对大小``，做出删除判断，简单的流程如图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727112005301.png#pic_center)

#### 1.3.3 键值对的删除策略
经过前面的几个环节，我们知道了Redis的两种存储位置：键空间和过期字典，以及过期字典expires的结构、判断是否过期的方法，那么该如何实施删除呢？

先抛开Redis来想一下可能的几种``删除策略``：

- ``定时删除``：在设置键的过期时间的同时，创建定时器，让定时器在键过期时间到来时，即刻执行键值对的删除；

- ``定期删除``：每隔特定的时间对数据库进行一次扫描，检测并删除其中的过期键值对；

- ``惰性删除``：键值对过期暂时不进行删除，至于删除的时机与键值对的使用有关，当获取键时先查看其是否过期，过期就删除，否则就保留；

在上述的三种策略中定时删除和定期删除属于不同时间粒度的``主动删除``，惰性删除属于``被动删除``。

三种策略都有各自的优缺点：``定时删除``对内存使用率有优势，但是对``CPU不友好，惰性删除对内存不友好``，如果某些键值对一直不被使用，那么会造成一定量的内存浪费，``定期删除是定时删除和惰性删除的折中``。

Reids采用的是``惰性删除和定时删除的结合``，一般来说可以借助``最小堆``来实现定时器，不过Redis的设计考虑到时间事件的有限种类和数量，使用了``无序链表存储时间事件``，这样如果``在此基础上``实现定时删除，就意味着O(N)遍历获取最近需要删除的数据。

但是我觉得antirez如果非要使用定时删除，那么他``肯定不会使用原来的无序链表机制``，所以个人认为已存在的无序链表不能作为Redis不使用定时删除的``根本理由``，``冒昧猜测唯一可能``的是antirez觉得``没有必要``使用定时删除。

![在这里插入图片描述](https://img-blog.csdnimg.cn/202007271120587.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxMzczNjgx,size_16,color_FFFFFF,t_70#pic_center)
#### 1.3.4 定期删除的实现细节
定期删除听着很简单，但是如何控制执行的频率和时长呢？

试想一下如果执行``频率太少就退化为惰性删除``了，如果执行``时间太长又和定时删除类似``了，想想还确实是个难题！并且执行``定期删除的时机``也需要考虑，所以我们继续来看看Redis是如何实现定期删除的吧！笔者在``src/expire.c``文件中找到了``activeExpireCycle``函数，定期删除就是由此函数实现的，在代码中antirez做了比较详尽的注释，不过都是英文的，试着读了一下模模糊糊弄个大概，所以``学习英文并阅读外文资料是很重要的学习途径``。

先贴一下代码，核心部分算上注释``大约210行``，具体看下：

```c
#define ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP 20 /* Keys for each DB loop. */
#define ACTIVE_EXPIRE_CYCLE_FAST_DURATION 1000 /* Microseconds. */
#define ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC 25 /* Max % of CPU to use. */
#define ACTIVE_EXPIRE_CYCLE_ACCEPTABLE_STALE 10 /* % of stale keys after which
                                                   we do extra efforts. */

void activeExpireCycle(int type) {
    /* Adjust the running parameters according to the configured expire
     * effort. The default effort is 1, and the maximum configurable effort
     * is 10. */
    unsigned long
    effort = server.active_expire_effort-1, /* Rescale from 0 to 9. */
    config_keys_per_loop = ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP +
                           ACTIVE_EXPIRE_CYCLE_KEYS_PER_LOOP/4*effort,
    config_cycle_fast_duration = ACTIVE_EXPIRE_CYCLE_FAST_DURATION +
                                 ACTIVE_EXPIRE_CYCLE_FAST_DURATION/4*effort,
    config_cycle_slow_time_perc = ACTIVE_EXPIRE_CYCLE_SLOW_TIME_PERC +
                                  2*effort,
    config_cycle_acceptable_stale = ACTIVE_EXPIRE_CYCLE_ACCEPTABLE_STALE-
                                    effort;

    /* This function has some global state in order to continue the work
     * incrementally across calls. */
    static unsigned int current_db = 0; /* Last DB tested. */
    static int timelimit_exit = 0;      /* Time limit hit in previous call? */
    static long long last_fast_cycle = 0; /* When last fast cycle ran. */

    int j, iteration = 0;
    int dbs_per_call = CRON_DBS_PER_CALL;
    long long start = ustime(), timelimit, elapsed;

    /* When clients are paused the dataset should be static not just from the
     * POV of clients not being able to write, but also from the POV of
     * expires and evictions of keys not being performed. */
    if (clientsArePaused()) return;

    if (type == ACTIVE_EXPIRE_CYCLE_FAST) {
        /* Don't start a fast cycle if the previous cycle did not exit
         * for time limit, unless the percentage of estimated stale keys is
         * too high. Also never repeat a fast cycle for the same period
         * as the fast cycle total duration itself. */
        if (!timelimit_exit &&
            server.stat_expired_stale_perc < config_cycle_acceptable_stale)
            return;

        if (start < last_fast_cycle + (long long)config_cycle_fast_duration*2)
            return;

        last_fast_cycle = start;
    }

    /* We usually should test CRON_DBS_PER_CALL per iteration, with
     * two exceptions:
     *
     * 1) Don't test more DBs than we have.
     * 2) If last time we hit the time limit, we want to scan all DBs
     * in this iteration, as there is work to do in some DB and we don't want
     * expired keys to use memory for too much time. */
    if (dbs_per_call > server.dbnum || timelimit_exit)
        dbs_per_call = server.dbnum;

    /* We can use at max 'config_cycle_slow_time_perc' percentage of CPU
     * time per iteration. Since this function gets called with a frequency of
     * server.hz times per second, the following is the max amount of
     * microseconds we can spend in this function. */
    timelimit = config_cycle_slow_time_perc*1000000/server.hz/100;
    timelimit_exit = 0;
    if (timelimit <= 0) timelimit = 1;

    if (type == ACTIVE_EXPIRE_CYCLE_FAST)
        timelimit = config_cycle_fast_duration; /* in microseconds. */

    /* Accumulate some global stats as we expire keys, to have some idea
     * about the number of keys that are already logically expired, but still
     * existing inside the database. */
    long total_sampled = 0;
    long total_expired = 0;

    for (j = 0; j < dbs_per_call && timelimit_exit == 0; j++) {
        /* Expired and checked in a single loop. */
        unsigned long expired, sampled;

        redisDb *db = server.db+(current_db % server.dbnum);

        /* Increment the DB now so we are sure if we run out of time
         * in the current DB we'll restart from the next. This allows to
         * distribute the time evenly across DBs. */
        current_db++;

        /* Continue to expire if at the end of the cycle more than 25%
         * of the keys were expired. */
        do {
            unsigned long num, slots;
            long long now, ttl_sum;
            int ttl_samples;
            iteration++;

            /* If there is nothing to expire try next DB ASAP. */
            if ((num = dictSize(db->expires)) == 0) {
                db->avg_ttl = 0;
                break;
            }
            slots = dictSlots(db->expires);
            now = mstime();

            /* When there are less than 1% filled slots, sampling the key
             * space is expensive, so stop here waiting for better times...
             * The dictionary will be resized asap. */
            if (num && slots > DICT_HT_INITIAL_SIZE &&
                (num*100/slots < 1)) break;

            /* The main collection cycle. Sample random keys among keys
             * with an expire set, checking for expired ones. */
            expired = 0;
            sampled = 0;
            ttl_sum = 0;
            ttl_samples = 0;

            if (num > config_keys_per_loop)
                num = config_keys_per_loop;

            /* Here we access the low level representation of the hash table
             * for speed concerns: this makes this code coupled with dict.c,
             * but it hardly changed in ten years.
             *
             * Note that certain places of the hash table may be empty,
             * so we want also a stop condition about the number of
             * buckets that we scanned. However scanning for free buckets
             * is very fast: we are in the cache line scanning a sequential
             * array of NULL pointers, so we can scan a lot more buckets
             * than keys in the same time. */
            long max_buckets = num*20;
            long checked_buckets = 0;

            while (sampled < num && checked_buckets < max_buckets) {
                for (int table = 0; table < 2; table++) {
                    if (table == 1 && !dictIsRehashing(db->expires)) break;

                    unsigned long idx = db->expires_cursor;
                    idx &= db->expires->ht[table].sizemask;
                    dictEntry *de = db->expires->ht[table].table[idx];
                    long long ttl;

                    /* Scan the current bucket of the current table. */
                    checked_buckets++;
                    while(de) {
                        /* Get the next entry now since this entry may get
                         * deleted. */
                        dictEntry *e = de;
                        de = de->next;

                        ttl = dictGetSignedIntegerVal(e)-now;
                        if (activeExpireCycleTryExpire(db,e,now)) expired++;
                        if (ttl > 0) {
                            /* We want the average TTL of keys yet
                             * not expired. */
                            ttl_sum += ttl;
                            ttl_samples++;
                        }
                        sampled++;
                    }
                }
                db->expires_cursor++;
            }
            total_expired += expired;
            total_sampled += sampled;

            /* Update the average TTL stats for this database. */
            if (ttl_samples) {
                long long avg_ttl = ttl_sum/ttl_samples;

                /* Do a simple running average with a few samples.
                 * We just use the current estimate with a weight of 2%
                 * and the previous estimate with a weight of 98%. */
                if (db->avg_ttl == 0) db->avg_ttl = avg_ttl;
                db->avg_ttl = (db->avg_ttl/50)*49 + (avg_ttl/50);
            }

            /* We can't block forever here even if there are many keys to
             * expire. So after a given amount of milliseconds return to the
             * caller waiting for the other active expire cycle. */
            if ((iteration & 0xf) == 0) { /* check once every 16 iterations. */
                elapsed = ustime()-start;
                if (elapsed > timelimit) {
                    timelimit_exit = 1;
                    server.stat_expired_time_cap_reached_count++;
                    break;
                }
            }
            /* We don't repeat the cycle for the current database if there are
             * an acceptable amount of stale keys (logically expired but yet
             * not reclained). */
        } while ((expired*100/sampled) > config_cycle_acceptable_stale);
    }

    elapsed = ustime()-start;
    server.stat_expire_cycle_time_used += elapsed;
    latencyAddSampleIfNeeded("expire-cycle",elapsed/1000);

    /* Update our estimate of keys existing but yet to be expired.
     * Running average with this sample accounting for 5%. */
    double current_perc;
    if (total_sampled) {
        current_perc = (double)total_expired/total_sampled;
    } else
        current_perc = 0;
    server.stat_expired_stale_perc = (current_perc*0.05)+
                                     (server.stat_expired_stale_perc*0.95);
}
```
说实话这个代码``细节比较多``，由于笔者对Redis源码``了解不多``，只能做个``模糊版本的解读``，所以难免有问题，还是建议有条件的读者自行前往源码区阅读，抛砖引玉看下笔者的模糊版本：

- 该算法是个**自适应的过程**，当过期的key比较少时那么就花费很少的cpu时间来处理，如果过期的key很多就采用激进的方式来处理，避免大量的内存消耗，可以理解为判断过期键``多就多跑几次，少则少跑几次``；

- 由于Redis中有很多数据库db，该算法会逐个扫描，本次结束时继续向后面的db扫描，是个**闭环的过程**；

- 定期删除有**快速循环和慢速循环两种模式**，主要采用慢速循环模式，其循环频率主要取决于server.hz，通常设置为10，也就是每秒执行10次慢循环定期删除，执行过程中如果耗时超过25%的CPU时间就停止；

- 慢速循环的执行时间相对较长，会出现超时问题，快速循环模式的执行时间**不超过1ms**，也就是执行**时间更短**，但是执行的**次数更多**，在执行过程中发现某个db中**抽样的key**中过期key占比**低于25%**则跳过；

主体意思：定期删除是个``自适应的闭环并且概率化的抽样扫描过程``，过程中都有执行时间和cpu时间的限制，如果触发阈值就停止，可以说是尽量在不影响对客户端的响应下``润物细无声``地进行的。

#### 1.3.5 DEL删除键值对
在Redis4.0之前执行del操作时如果key-value很大，那么可能导致阻塞，在新版本中引入了BIO线程以及一些新的命令，实现了del的延时懒删除，最后会有``BIO线程``来实现内存的清理回收。

之前写过一篇4.0版本的LazyFree相关的文章，可以看下[浅析Redis 4.0新特性之LazyFree](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMTI2Ng==&mid=2247483717&idx=1&sn=dfa5cafa7695f2d0dfb8bc51b414d46b&scene=21#wechat_redirect)

### 1.4 内存淘汰机制
为了保证Redis的安全稳定运行，设置了一个max-memory的阈值，那么当内存用量到达阈值，新写入的键值对无法写入，此时就需要内存淘汰机制，在Redis的配置中有几种``淘汰策略``可以选择，详细如下：

- noeviction: 当内存不足以容纳新写入数据时，新写入操作会报错；

- allkeys-lru：当内存不足以容纳新写入数据时，在键空间中移除最近最少使用的 key；

- allkeys-random：当内存不足以容纳新写入数据时，在键空间中随机移除某个 key；

- volatile-lru：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，移除最近最少使用的 key；

- volatile-random：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，随机移除某个 key；

- volatile-ttl：当内存不足以容纳新写入数据时，在设置了过期时间的键空间中，有更早过期时间的 key 优先移除；

后三种策略都是针对过期字典的处理，但是在过期字典为空时会noeviction一样返回写入失败，毫无策略地随机删除也不太可取，所以一般选择第二种allkeys-lru基于LRU策略进行淘汰。

个人认为antirez一向都是``工程化思维``，善于使用``概率化设计``来做近似实现，LRU算法也不例外，Redis中实现了``近似LRU算法``，并且经过几个版本的迭代效果已经比较接近理论LRU算法的效果了，这个也是个不错的内容，由于篇幅限制，本文计划后续单独讲LRU算法时再进行详细讨论。

### 1.5 过期键删除和内存淘汰的关系
``过期健删除``策略``强调``的是``对过期健的操作``，如果有健过期而内存足够，Redis不会使用内存淘汰机制来腾退空间，这时会优先使用过期健删除策略删除过期健。

``内存淘汰``机制``强调``的是``对内存数据的淘汰操作``，当内存不足时，即使有的健没有到达过期时间或者根本没有设置过期也要根据一定的策略来删除一部分，腾退空间保证新数据的写入。

## Q7:讲讲你对Redis持久化机制的理解。

个人认为Redis持久化既是数据库本身的亮点，也是面试的热点，主要考察的方向包括：``RDB机制原理、AOF机制原理、各自的优缺点、工程上的对于RDB和AOF的取舍、新版本Redis混合持久化策略``等，如能把握要点，持久化问题就过关了。

之前写过一篇持久化的文章：[理解Redis持久化](https://mp.weixin.qq.com/s?__biz=MzI1MzYzMTI2Ng==&mid=2247483700&idx=1&sn=a935b0320f9daaa3d0102c8a6095fad7&scene=21#wechat_redirect),基本上也涵盖了上面的几个点，可以看一下。

## Q8:谈谈你对Redis数据同步(复制)的理解吧！
**持久化和数据同步的关系**

理解持久化和数据同步的关系，需要从``单点故障``和``高可用``两个角度来分析：

**单点宕机故障**

假如我们现在只有一台作为缓存的Redis机器，通过持久化将热点数据写到磁盘，某时刻该Redis单点机器发生``故障宕机``，此期间缓存失效，``主存储服务``将承受所有的请求``压力倍增``，监控程序将宕机Redis``机器拉起``。

重启之后，该机器可以Load磁盘RDB数据进行``快速恢复``，恢复的时间取决于数据量的多少，一般秒级到分钟级不等，恢复完成保证之前的热点数据还在，这样存储系统的CacheMiss就会降低，有效降低了``缓存击穿``的影响。

在单点Redis中持久化机制非常有用，**只写文字容易让大家睡着**，我画了张图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727113432922.png#pic_center)

**高可用的Redis系统**

作为一个``高可用``的缓存系统单点宕机是不允许的，因此就出现了``主从架构``，对主节点的数据进行``多个备份``，如果主节点挂点，可以立刻切换``状态最好``的从节点为主节点，对外提供写服务，并且其他从节点向新主节点同步数据，确保整个Redis缓存系统的高可用。

如图展示了一个``一主两从读写分离的Redis系统``主节点``故障迁移``的过程，整个过程并没有停止``正常工作``，大大提高了系统的高可用：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727113536941.png#pic_center)
从上面的两点分析可以得出个小结论**划重点**：
**持久化让单点故障不再可怕，数据同步为高可用插上翅膀。**

我们理解了数据同步对Redis的重要作用，接下来继续看数据同步的实现原理和过程、重难点等细节问题吧！

**Redis系统中的CAP理论**

对分布式存储有了解的读者一定知道``CAP理论``，说来惭愧笔者在2018年3月份换工作的时候，去Face++旷视科技面后端开发岗位时就遇到了CAP理论，除了CAP理论问题之外其他问题都在射程内，所以最终还是拿了Offer。

但是印象很深T大毕业的面试官说前面的问题答得都不错，为啥CAP问题答得这么菜…其实我当时只知道CAP理论就像高富帅一样，不那么容易达到...细节不清楚...

各位吃瓜读者，笔者前面说这个小事情的目的是想表达：``CAP理论对于理解分布式存储非常重要``，下回你们面试被问到CAP别怪我没提醒。

在理论计算机科学中，CAP定理又被称作``布鲁尔定理Brewer's theorem``，这个定理起源于加州大学``伯克利分校``的计算机科学家埃里克·布鲁尔在2000年的分布式计算原理研讨会``PODC``上提出的一个``猜想``。

在2002年``麻省理工学院``的赛斯·吉尔伯特和南希·林奇发表了``布鲁尔猜想的证明``，使之成为一个``定理。``它指出对于一个分布式计算系统来说，不可能同时满足以下三点：

- C Consistent 一致性 连贯性

- A Availability 可用性

- P Partition Tolerance 分区容忍性

来看一张``阮一峰大佬画的图``：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727113811395.png#pic_center)
举个简单的例子，说明一下CP和AP的兼容性：

**CP和AP问题**

理解CP和AP的关键在于分区容忍性P，网络分区在分布式存储中再平常不过了，即使机器在一个机房，也不可能全都在一个机架或一台交换机。

这样在局域网就会出现``网络抖动``，笔者做过1年多DPI对于网络传输中最深刻的三个名词：``丢包、乱序、重传``。所以``我们看来风平浪静的网络，在服务器来说可能是风大浪急``，一不小心就不通了，所以当网络出现断开时，这时就出现了网络分区问题。

对于Redis数据同步而言，假设从结点和主结点在两个机架上，某时刻发生网络断开，如果此时Redis读写分离，那么从结点的数据必然无法与主继续同步数据。在这种情况下，如果``继续在从结点读取数据就造成数据不一致问题，如果强制保证数据一致从结点就无法提供服务造成不可用问题``，从而看出在``P的影响下C和A无法兼顾``。
其他几种情况就不深入了，从上面我们可以得出结论：``当Redis多台机器分布在不同的网络中，如果出现网络故障，那么数据一致性和服务可用性无法兼顾，Redis系统对此必须做出选择，事实上Redis选择了可用性，或者说Redis选择了另外一种最终一致性。``

**最终一致性**

Redis选择了``最终一致性``，也就是``不保证``主从数据在``任何时刻``都是``一致``的，并且Redis主从同步默认是``异步``的，亲爱的盆友们``不要晕！不要蒙圈！``

我来一下解释``同步复制和异步复制``(注意：``考虑读者的感受 我并没有写成同步同步和异步同步 哈哈``)：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020072711403179.png#pic_center)
一图胜千言，看红色的数字就知道``同步复制和异步复制``的区别了：

- **异步复制**：当客户端向主结点写了hello world，主节点写成功之后就向客户端回复OK，这样主节点和客户端的交互就完成了，之后主节点向从结点同步hello world，从结点完成之后向主节点回复OK，整个过程客户端不需要等待从结点同步完成，因此整个过程是异步实现的。

- **同步复制**：当客户端向主结点写了hello world，主节点向从结点同步hello world，从结点完成之后向主节点回复OK，之后主节点向客户端回复OK，整个过程客户端需要等待从结点同步完成，因此整个过程是同步实现的。

Redis选择异步复制可以避免客户端的等待，更符合现实要求，不过这个复制方式可以修改，根据自己需求而定吧。

**从从复制**

假如Redis高可用系统中有``一主四从``，如果四个从同时向主节点进行数据同步，主节点的压力会比较大，考虑到Redis的最终一致性，因此Redis后续推出了``从从复制``，从而将``单层复制结构演进为多层复制结构``，笔者画了个图看下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727114219285.png#pic_center)
**全量复制和增量复制**

``全量复制``是从结点因为故障恢复或者新添加从结点时出现的``初始化阶段``的数据复制，这种复制是将主节点的数据全部同步到从结点来完成的，所以成本大但又不可避免。

``增量复制``是主从结点正常工作之后的每个时刻进行的数据复制方式，``涓涓细流同步数据``，这种同步方式又轻又快，优点确实不少，不过如果没有全量复制打下基础增量复制也没戏，所以二者不是矛盾存在而是``相互依存``的。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727114305569.png#pic_center)

**全量复制过程分析**

Redis的全量复制过程主要分**三个阶段**：

- **快照阶段**：从结点向主结点发起SYNC全量复制命令，主节点执行bgsave将内存中全部数据生成快照并发送给从结点，从结点释放旧内存载入并解析新快照，主节点同时将此阶段所产生的新的写命令存储到缓冲区。

- **缓冲阶段**：主节点向从节点同步存储在缓冲区的操作命令，这部分命令主节点是bgsave之后到从结点载入快照这个时间段内的新增命令，需要记录要不然就出现数据丢失。

增量阶段：缓冲区同步完成之后，主节点正常向从结点同步增量操作命令，至此主从保持基本一致的步调。

借鉴参考1的一张图表，写的很好，我就不再重复画图了：


考虑一个``多从并发全量复制问题``：
**如果此时有多个从结点同时向主结点发起全量同步请求会怎样？**
Redis主结点是个**聪明又诚实**的家伙，比如现在有3个从结点A/B/C陆续向主节点发起SYNC全量同步请求。
- 主节点在对A进行bgsave的同时，B和C的SYNC命令到来了，那么主节点就一锅烩，把针对A的快照数据和缓冲区数据同时同步给ABC，这样提高了效率又保证了正确性。
- 主节点对A的快照已经完成并且现在正在进行缓冲区同步，那么只能等A完成之后，再对B和C进行和A一样的操作过程，来实现新节点的全量同步，所以主节点并没有偷懒而是重复了这个过程，虽然繁琐但是保证了正确性。

再考虑一个``快照复制循环问题``：
主节点执行bgsave是``比较耗时且耗内存的操作``，期间从结点也经历``装载旧数据->释放内存->装载新数据`的过程，``内存先升后降再升的动态过程``，从而知道无论主节点执行快照还是从结点装载数据都是需要``时间和资源``的。

抛开对性能的影响，试想如果主节点快照时间是1分钟，在期间有``1w条新命令``到来，这些新命令都将写到缓冲区，如果``缓冲区比较小只有8k``，那么在快照完成之后，``主节点缓冲区也只有8k命令丢失了2k命令``，那么此时从结点进行全量同步就缺失了数据，是一次错误的全量同步。

无奈之下，``从结点会再次发起SYNC命令``，从而``陷入循环``，因此``缓冲区大小``的设置很重要，二话不说再来一张图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727114724251.png#pic_center)


**增量复制过程分析**

增量复制过程稍微简单一些，但是非常有用，试想``复杂的网络环境下，并不是每次断开都无法恢复，如果每次断开恢复后就要进行全量复制，那岂不是要把主节点搞死``，所以增量复制算是对复杂网络环境下数据复制过程的一个优化，允许一段时间的落后，最终追上就行。

增量复制是个``典型的生产者-消费者模型``，使用定长环形数组(队列)来实现，如果buffer满了那么新数据将覆盖老数据，因此从结点在复制数据的同时向主节点反馈自己的偏移量，从而确保数据不缺失。

这个过程非常好理解，``kakfa这种MQ也是这样的``，所以在合理设置buffer大小的前提下，理论上从的消费能力是大于主的生产能力的，大部分只有在网络断开时间过长时会出现buffer被覆盖，从结点消费滞后的情况，此时只能进行全量复制了。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727114804630.png#pic_center)

**无盘复制**

**理解无盘复制之前先看下什么是有盘复制呢？**
所谓盘是指磁盘，可能是机械磁盘或者SSD，但是无论哪一种相比内存都更慢，我们都知道IO操作在服务端的耗时是占大头的，因此对于全量复制这种高IO耗时的操作来说，尤其当服务并发比较大且还在进行其他操作时对Redis服务本身的影响是比较大大，之前的模式时这样的：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200727115014794.png#pic_center)

在Redis2.8.18版本之后，开发了**无盘复制**，也就是**避免了生成的RDB文件落盘再加载再网络传输的过程，而是流式的遍历发送过程，主节点一边遍历内存数据，一边将数据序列化发送给从结点**，从结点没有变化，仍然将数据依次存储到本地磁盘，完成传输之后进行内存加载，**可见无盘复制是对IO更友好**。

**小结**

时间原因只能写这么多了，``和大家一起学习不是把桶填满而是把火点燃。``

**回顾一下**：本文主要讲述了持久化和数据同步的关系、Redis分布式存储的CAP选择、Redis数据同步复制和异步复制、全量复制和增量复制的原理、无盘复制等，相信耐心的读者一定会有所收获的。

**最后可以思考一个问题**：
Redis的``数据同步仍然会出现数据丢失的情况``，比如主节点往缓冲区写了10k条操作命令，此时主挂掉了，从结点只消费了9k操作命令，那么切主之后从结点的数据就丢失了1k，即使旧主节点恢复也只能作为从节点向新主节点发起全量复制，那么我们该如何优化这种情况呢？
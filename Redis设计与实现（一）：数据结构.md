# Redis设计与实现(一)：数据结构
---
### 简单动态字符串
1. SDS用来保存数据库中的字符串值，也被用作缓冲区。AOF模块中的AOF缓冲区，以及客户端状态中的输入缓冲区等
2. 定义如下：

    ```cpp
    struct sdshdr{
        //记录buf数组中已使用字节的数量
        //等于SDS所保存字符串的长度
        int len;
        
        //记录buf数组中未使用字节的数量
        int free;
        
        //字节数组，用于保存字符串
        char buf[];
    };
    ```
SDS遵循C字符串以空字符结尾的惯例，保存空字符的1字节空间不计算在SDS的len属性里面，并且为空字符分配额外的1字节空间，以及添加空字符到字符串末尾等操作，都是由SDS函数自动完成的
3. 通过未使用的空间，SDS实现了空间预分配和惰性空间释放两种优化策略
    * 空间预分配策略，SDS将连续增长N次字符串所需的内存重分配次数从必定N次降低到最多N次。
    * 通过惰性空间释放策略，SDS避免了缩短字符串时所需的内存重分配操作，并为将来可能有的增长操作提供了优化。
4. 为了确保Redis可以适用于各种不同的使用场景，SDS的API都是二进制安全的，所有SDS API都会以二进制的方式来处理SDS存放在buf数组里的数据，程序不会对其中的数据做任何的限制、过滤、或者假设，数据在写入时是什么样的，它被读取时就是什么样。

### 链表
1. 链表在Redis中的应用非常广泛，比如列表键的底层实现之一就是链表。当一个列表键包含了数量比较多的元素，又或者列表中包含的元素都是比较长的字符串时，Redis就会使用链表作为列表键的底层实现。除了列表键之外，发布与订阅、慢查询、监视器等功能也用到了链表，Redis服务器本身还使用链表保存多个客户端的状态信息，以及使用链表来构建客户端输出缓冲区。
2. 链表节点定义如下：
    
    ```cpp
    typedef struct listNode{
        //前置节点
        struct listNode *prev;
        //后置节点
        struct listNode *next;
        //节点的值
        void *value;
    }listNode;
    ```
    
    链表(双端链表)的定义如下:
    
    ```cpp
    typedef struct list{
        //表头节点
        listNode *head;
        //表尾节点
        listNode *tail;
        //链表所包含的节点数量
        unsigned long len;
        //节点值复制函数
        void*(*dup)(void *ptr);
        //节点值释放函数
        void (*free)(void *ptr);
        //节点值对比函数
        int (*match)(void *ptr, void *key);
    }list;
    ```
3. Redis的链表实现的特性如下：双端，无环，带表头节点和表尾节点，带链表长度计数器，多态(链表节点使用void*指针来保存节点值，并且可以通过list结构中的dup,free,match三个属性为节点值设置特定函数，所以链表可以用于保存各种不同类型的值)

### 字典
1. 字典，又称符号表、关联数组或映射，是一种用于保存键值对(key-value pairs)的抽象数据结构。字典中的每个键都是独一无二的，程序可以在字典中根据键查找与之关联的值，或通过键来更新值等等
2. 字典在Redis中的应用相当广泛，比如Redis的数据库就是使用字典作为底层实现的，对数据库的增、删、改、查操作也是建立在对字典的操作之上的。字典还是哈希键的底层实现之一，当一个哈希键包含的键值比较多，又或者键值中的元素都是比较长的字符串时，Redis就会使用字典作为哈希键的底层实现。
3. Redis字典所使用给的哈希表定义如下：

    ```cpp
    typedef struct dictht{
        //哈希表数组
        dictEntry **table;
        //哈希表大小
        unsigned long size;
        //哈希表大小掩码，用于计算索引值，总是等于size-1
        unsigned long sizemask;
        //该哈希表已有节点的数量
        unsigned long used;
    }dictht;
    ```
* table属性是一个数组，数组中的每个元素是指向一个dictEntry结构的指针，每个dictEntry结构保存一个键值对。
* 哈希表节点使用dictEntry结构表示：
    ```cpp
    typedef struct dictEntry{
        //键
        void *key;
        //值
        union{
            void *val;
            uint64_t u64;
            int64_t s64;
        } v;
        //指向下个哈希表节点，形成链表
        struct dictEntry *next;
    }dictEntry;
    ```
    * next属性是指向另一个哈希表节点的指针，这个指针可以将多个哈希值相同的键值对连接在一起，以此来解决键冲突(collision)的问题。

4. Redis中的字典dict结构如下：

    ```cpp
    typedef struct dict{
        //类型特定函数
        dictType *type;
        //私有数据
        void *privdata;
        //哈希表
        dictht ht[2];
        //rehash索引，当rehash不在进行时，值为-1
        int trehashidx; 
    }dict;
    ```
    * type属性是一个指向dictType结构的指针，每个dictType结构保存了一簇用于操作特定类型键值对的函数，Redis会为用途不同的字典设置不同的类型特定函数
    * privdata属性则保存了需要传给那些类型特定函数的可选参数
    
    ```cpp
    typedef struct dictType{
        //计算哈希值的函数
        unsigned int (*hashFunction)(const void *key);
        //复制键的函数
        void *(*keyDup)(void *privdata, const void *key);
        //复制值的函数
        void *(*valDup)(void *privdata, const void *obj);
        //对比键的函数
        int (*keyCompare)(void *privdata, const void *key1, const void *key2);
        //销毁键的函数
        void (*keyDestructor)(void *privdata, void *key);
        //销毁值的函数
        void (*valDestructor)(void *privdata, void *obj);
    }dictType;
    ```
    * ht属性是一个包含两个项的数组，数组中的每个项都是一个dictht哈希表，一般情况下，字典只是用ht[0]哈希表，ht[1]哈希表只会在对ht[0]哈希表进行rehash时使用。
5. Redis计算哈希值和索引值的方法如下：
    
    ```cpp
    //使用字典设置的哈希函数，计算出key的哈希值
    hash = dict->type->hashFunction(key);
    //使用哈希表的sizemask属性和哈希值，计算出索引值
    //根据当前情况，ht[x]可以是ht[0]或者ht[1]
    index = hash & dict->ht[x].sizemask;
    ```
6. Redis的哈希表使用链地址法(separate chaining)来解决键冲突，每个哈希表节点都有一个next指针，多个哈希表节点可以用next指针构成一个单向链表，被分配到同一个索引上的多个节点可以用这个单向链表链接起来，这就解决了键冲突问题
7. 随着操作的不断执行，哈希表保存的键值对会逐渐增加或减少，为了让哈希表的负载因子(load factor)维持在一个合理的范围内，当哈希表保存的键值对数量太多或者太少时，程序需要对哈希表进行相应的扩展或收缩。
    扩展和收缩哈希表的工作可以通过执行rehash(重新散列)操作来实现，Redis对字典的哈希表指向rehash操作的步骤如下：
    * 1） 为字典ht[1]哈希表分配空间，这个哈希表的大小取决于要执行的操作，以及ht[0]当前包含的键值对数量
    * 2） 将保存在ht[0]中的所有键值对rehash到ht[1]上面：rehash指的是重新计算键的哈希值和索引值，然后将键值对放置到ht[1]的指定位置上
    * 3） 当ht[0]包含的所有键值对都迁移到ht[1]后(ht[0]变为空表)，释放ht[0]，将ht[1]设置为ht[0]，并在ht[1]新创建一个空白哈希表，为下一次rehash做准备。
8. 当以下条件的任意一个被满足时，程序会自动开始对哈希表执行扩展操作：
    * 服务器目前没有在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于1
    * 服务器目前正在执行BGSAVE命令或者BGREWRITEAOF命令，并且哈希表的负载因子大于等于5
```cpp
//负载因子 = 哈希表已保存节点的数量 / 哈希表大小
load_factor = ht[0].used / ht[0].size;
```
9. rehash并不是一次性、集中的完成的，而是分多次、渐进式地完成的。原因是当键值对数量太大时，如果一次性哈希的话，庞大的计算量可能会导致服务器在一段时间内停止服务，所以为了避免rehash对服务器性能造成影响，采用渐进式rehash
10. 渐进式rehash的好处在于它采用分而治之的方式，将rehash键值对所需的计算工作均摊到字典的每个添加、删除、查找和更新操作上，从而避免了集中式rehash而带来的庞大计算量

### 跳跃表
1. 跳跃表(skiplist)是一种有序数据结构，它通过每个节点中维持多个指向其他节点的指针，从而达到快速访问节点的目的。跳跃表支持平均O(logN)、最坏O(N)复杂度的节点查找，还可以通过顺序性操作来批量处理节点
2. Redis使用跳跃表作为有序集合键的底层实现之一，如果一个有序集合包含的元素数量比较多，又或者有序集合元素的成员是比较长的字符串时，Redis就会使用跳跃表来作为有序集合键的底层实现。注：Redis只在两个地方用到了跳跃表，一个是实现有序集合键，另一个是在集群节点中用作内部数据结构。
3. 跳跃表节点的实现由redis.h/zskiplistNode结构定义：
    
    ```cpp
    typedef struct zskiplistNode{
        //后退指针
        struct zskiplistNode *backward;
        //分值
        double score;
        //成员对象
        robj *obj;
        //层
        struct zskiplistLevel{
            //前进指针
            struct zskiplistNode *forward;
            //跨度
            unsigned int span;
        } level[];
    }zskiplistNode;
    ```
* 层：跳跃表节点的level数组可以包含多个元素，每个元素都包含一个指向其他节点的指针，程序可以通过这些层来加快访问其他节点的速度，一般来说，层的数量越多，访问其他节点的速度越快。
* 跨度：用于记录两个节点之间的距离，跨度实际上是用来计算排位的：在查找某个节点的过程中，将沿途访问过的所有层的跨度累计起来，得到的结果就是目标节点在跳跃表中的排位
4. 跳跃表：zskiplist结构如下：

    ```cpp
    typedef struct zskiplist{
        //表头节点和表尾节点
        struct skiplistNode *header, *tail;
        //表中节点的数量
        unsigned long length;
        //表中层数最大节点的层数
        int level;
    }zskiplist;
    ```
### 整数集合
1. 整数集合(intset)是集合键的底层实现之一，当一个集合只包含整数值元素，并且这个集合的元素数量不多时，Redis就会使用整数集合作为集合键的底层实现
2. 整数集合是Redis用于保存整数值的集合抽象数据结构，它可以保存类型为`int16_t`、`int32_t`或者`int64_t`的整数值，并且保证集合中不会出现重复元素
其结构定义如下：

    ```cpp
    typedef struct intset{
        //编码方式
        uint32_t encoding;
        //集合包含的元素数量
        uint32_t length;
        //保存元素的数组
        int8_t contents[];
    }intset;
    ```
contents数组是整数集合的底层实现，整数集合的元素都是contents数组中的一个数据项(item)，各个项在数组中按值的大小从小到大有序排列，并且数组中不包含任何重复项。
注意：虽然contents属性声明为int8_t类型的数组，但实际上contents数组并不保存任何 int8\_t类型的值，contents数组的真正类型取决于encoding属性的值
* 如果encoding属性的值为：INTSET_ENC_INT16，contents就是一个int16_t类型的数组(-32768 - 32767)
* 如果encoding属性的值为：INTSET_ENC_INT32，contents就是一个int32_t类型的数组(-2147483648 - 2147483647)
* 如果encoding属性的值为：INTSET_ENC_INT64，contents就是一个int64_t类型的数组(-9 223 372 036 854 775 808 - 9 223 372 036 854 775 807)
3. 升级：每当我们将一个新元素添加到整数集合里，并且新元素的类型比整数集合现有所有元素类型都要长时，整数集合需要先进行升级，然后才将新元素添加到整数集合里面。
升级过程分为三步：
* 根据新元素的类型，扩展整数集合底层数组的空间大小，并为新元素分配空间
* 将底层数组现有的所有元素都转换成与新元素相同的类型，并将类型转化后的元素放置到正确位上，而且放置元素的过程中，需要继续维持底层数组的有序性质不变。
* 将新元素添加到底层数组里面

升级的好处：(1)是提升整数集合的灵活性 (2)是尽可能地节约内存

### 压缩列表
1. 压缩列表(ziplist)是列表键和哈希键的底层实现之一。当一个列表键只包含少量列表项，并且每个列表项要么是小整数值，要么是长度比较短的字符串，那么Redis就会使用压缩列表来做列表键的底层实现。另外，当一个哈希键只包含少量键值对，并且每个键值对的键和值要么就是小整数值，要么就是长度比较短的字符串，那么Redis就会使用压缩列表来做哈希键的底层实现。
**注意：Redis3.2以后，quicklist作为列表键的底层实现之一，代替了压缩列表**

2. 压缩列表是Redis为了节约内存而开发的，是由一系列特殊编码的连续内存块组成的顺序型(sequential)数据结构，一个压缩列表可以包含任意多个节点(entry)，每个节点可以保存一个字节数组或者一个整数值
3. 下表展示了压缩列表的各个组成部分
<table>
  <tr>
    <td> zlbytes </td>
    <td> zltail </td>
    <td> zllen </td>
    <td> entry1</td>
    <td> entry2</td>
    <td> ... </td>
    <td> entryN </td>
    <td> zlend </td>
  </tr>
</table>
其中：

* zlbytes(uint32_t, 4字节)：记录整个压缩列表占用的内存字节数，在对压缩列表进行内存重分配或者计算zlend位置时使用
* zltail(uint32_t, 4字节)：记录压缩列表表尾节点距离压缩列表的起始地址有多少字节：通过这个偏移量，程序无需遍历整个压缩列表就可以确定表尾节点的地址
* zllen(uint16_t, 2字节)：记录了压缩列表包含的节点数量，当这个属性的值小于UINT16_MAX(65535)时，这个属性的值就是压缩列表包含节点的数量，当这个值等于UINT16_MAX时，节点的真实数量需要遍历整个压缩列表才能计算得出
* entryX：压缩列表包含的各个节点，节点的长度由节点保存的内容决定
* zlend(uint8_t， 1字节)：特殊值(0xff, 255)，用于标记压缩列表的末端
* redis并没有提供一个专门的结构体来表示压缩列表的信息，而是提供了一组宏来定位每个成员的地址，定义在ziplist.c中
4. 压缩列表节点的构成:

    ```cpp
    typedef struct zlentry{
        //prevrawlen 前驱节点的长度
        //prevrawlensize 编码前驱节点的长度prevrawlen所需的字节大小
        unsigned int prevrawlensize, prevrawlen;
        //len当前节点值长度
        //lensize编码当前节点长度len所需的字节数
        unsigned int lensize, len;
        //当前节点header的大小 = lensize + prevrawlensize
        unsigned int headersize;
        //当前节点的编码格式
        unsigned char encoding;
        //指向当前节点的指针，以char*类型保存
        unsigned char *p;
    }zlentry;       
    ```
虽然定义了这个结构体，但是根本没有使用zlentry结构作为压缩列表中用来存储数据节点中的结构。因为，这个结构来存小整数或短字符串实在太浪费内存了。这个结构总共在32位机上占用了28个字节，在64位机上占用了32个字节。所以它不符合压缩列表的设计目的：提高内存的利用率。因此，在redis中，并没有使用定义的结构体来操作，而是定义了宏。压缩列表的节点真正结构如下所示：

    <table>
    <tr>
    <td> previous_entry_length </td>
    <td> encoding </td>
    <td> content </td>
    </tr>
    </table>
接下来就分别讨论这三个属性
* previous_entry_length以字节为单位，记录了压缩列表中前一个节点的长度，previous_entry_length属性的长度可以是1字节或5字节
    * 如果前一个节点长度小于254字节，那么previous_entry_length属性的长度为1字节，前一节点的长度就保存在这一字节里面
    * 如果前一节点长度大于等于254字节，那么previous_entry_length属性的长度为5字节：其中属性的第一字节被置为(0xFE,254)，而后的四个字节则用于保存前一节点的长度
    * 因为节点的previous_entry_length属性记录了前一个节点的长度，所以程序可以通过指针运算，根据当前节点的起始地址来计算出前一个节点的起始地址。即使用当前节点的起始地址指针，减去当前节点previous_entry_length属性的值，可以得到一个指向前一个节点起始地址的指针

* encoding属性记录了节点的content属性所保存数据的类型以及长度
    * 一字节、两字节或者五字节长，值的最高位是00、 01或者10的是字节数组编码：这种编码表示节点的content属性保存着字节数组，数组的长度由编码出去最高两位之后的其他位记录
    * 一字节长，值的最高位以11开头的整数编码：这种编码表示节点的content属性保存着整数值，整数值的类型和长度是由编码除去最高两位之后的其他位记录

* content属性负责保存节点的值，节点值可以是一个字节数组或者整数，值的类型和长度由节点的encoding属性决定
4. 压缩列表在某些特殊情况下会产生连续多次空间扩展操作，这称为连锁更新。尽管连锁更新的复杂度很高，但是它造成性能问题的几率是很低的。
    * 首先，压缩列表要恰好有多个连续的，长度介于250字节至253字节之间的节点，连锁更新才有可能被引发，在实际中，这种情况并不多见
    * 其次，即使出现连锁更新，但只要被更新的节点数量不多，就不会对性能产生任何影响。

            
    






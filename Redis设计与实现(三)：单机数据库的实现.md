﻿# Redis设计与实现(三)：单机数据库的实现

### 数据库
1. Redis服务器将所有数据库都保存在服务器状态 server.h/redisServer结构的db数组中，db数组的每一项都是一个server.h/redisDb结构，每个redisDb结构代表一个数据库：

    ```cpp
    struct redisServer{
        //...
        //一个数组，保存着服务器中的所有数据库
        redisDb *db;
        //服务器数据库数量
        int dbnum;
        //...
    };
    ```
在初始化服务器时，程序会根据服务器状态的dbnum属性来决定应该创建多少个数据库。dbnum属性的值由服务器配置的database选项决定，默认情况下，值为16
2. 每个Redis客户端都有自己的目标数据库，每当客户端执行数据库写命令或者数据库读命令的时候，目标数据库就会成为这些命令的操作对象。默认情况下，Redis客户端的目标数据库为0号数据库，但是客户端可以通过执行`select`命令来切换目标数据库
3. 在服务器内部，客户端状态redisClient结构的db属性记录了客户端当前的目标数据库，这个属性指向一个redisDb结构的指针：

    ```cpp
    typedef struct redisClient{
        //...
        //记录客户端当前正在使用的数据库
        redisDb *db;
        //...
    }redisClent;
    ```
redisClient.db指针指向redisServer.db数组的其中一个元素，而被指向的元素就是客户端的目标数据库。

### 数据库键空间
1. Redis是一个键值对(key-value pair)数据库服务器，服务器中的每个数据库都由一个redisDb结构表示，其中redisDb结构的dict字典保存了数据库中的所有键值对，将这个字典称为键空间(key space)：

    ```cpp
    typedef struct redisDb{
        //...
        //数据库键空间，保存着数据库中的所有键值对
        dict *dict;
        //过期字典，保存键的过期时间
        dict *expires;
        //...
    }redisDb;
    ```
2. 键空间和用户所见的数据库是直接对应的：
    * 键空间的键也就是数据库的键，每个键都是一个字符串对象
    * 键空间的值也就是数据库的值，每个值可以是字符串对象、列表对象、哈希对象、集合对象和有序集合对象中的任意一种Redis对象
3. 因为数据库的键空间是一个字典，所以所有针对数据库的操作，比如添加一个键值对到数据库，或者从数据库中删除一个键值对，又或者在数据库中获取某个键值对等，实际上是通过对键空间字典进行操作来实现的。
4. 当使用Redis命令对数据库进行读写时，服务器不仅会对键空间执行指定的读写操作，还会执行一些额外的维护操作，其中包括：
    * 在读取一个键之后(读操作和写操作都要对键进行读取)，服务器会根据键是否存在来更新服务器的键空间命中(hit)次数和键空间不命中(miss)次数。
    * 在读取一个键之后，服务器会更新键的LRU(最后一次使用)时间，这个值可以用于计算键的闲置时间。使用 object idletime <key> 可以查看键key的闲置时间
    * 如果服务器在读取一个键是发现该键已经过期，那么服务器会先删除这个过期键，然后才执行余下的其他操作
    * 如果有客户端使用watch命令监视了某个键，那么服务器在对被监视的键进行修改之后，会将这个键标记为脏(dirty)，从而让事务程序注意到这个键已经被修改过
    * 服务器每修改一个键后，都会对脏(dirty)键计数器的值增1，这个计数器会触发服务器的持久化以及复制操作
    * 如果服务器开启了数据库通知功能，那么在对键进行修改之后，服务器将配置发送相应的数据库通知

### 设置键的生存时间或过期时间
1. 通过expire或者pexpire命令，客户端可以以秒或者毫秒精度为数据库中的某个键设置生存时间(Time to Live, TTL)，在经过指定的秒数或者毫秒数之后，服务器就会自动删除生存时间为0的键
2. 客户端可以通过expireat或pexpireat命令，以秒或者毫秒精度给数据库中的某个键设置过期时间(expire time)，过期时间是一个unix时间戳，当键的过期时间来临时，服务器就会自动从数据库中删除这个键
3. ttl命令和pttl命令接受一个带有生存时间或这过期时间的键，返回这个键的剩余生存时间，也就是，返回距离这个键被自动删除还有多长时间
4. redisDb结构的expires字典保存了数据中所有键的过期时间，称这个字典为过期字典
    * 过期字典的键是一个指针，这个指针指向键空间中的某个键对象
    * 过期字典的值是一个long long类型的整数，这个整数保存了键所指向的数据库键的过期时间——一个毫秒精度的unix时间戳
5. 通过过期字典，程序可以通过以下步骤检查一个给定键是否过期：
    * 检查给定键是否存在于过期字典：如果存在，那么取得键的过期时间
    * 检查当前unix时间戳是否大于键的过期时间，如果是的话，那么键已经过期，否则的话，键未过期
6. Redis对过期键采用惰性删除和定期删除两种策略。
    * 惰性删除：在所有读写数据库的命令执行之前，先判断是否过期，是否需要删除，然后再执行
    * 定期删除：在规定的时间内，分多次遍历服务器中的各个数据库，从数据的expires字典中随机检查一部分键的过期时间，并删除其中的过期键

### AOF、RDB和复制功能对过期键的处理
1. 在执行save或者bgsave命令创建一个新的RDB文件时，程序会对数据库中的键进行检查，已过期的键不会被保存到新创建的RDB文件中
2. 在启动Redis服务器时，如果服务器开启了RDB功能，那么服务器将对RDB文件进行载入：
    * 如果服务器以主服务器模式运行，那么在载入RDB文件时，程序会对文件中保存的键进行检查，未过期的键会被载入到数据库中，而过期键则会被忽略。
    * 如果以从服务器模式运行，不论是否过期，都会被载入到数据中。不过，因为主从服务器在进行数据同步的时候，从服务器的数据库就会被清空，所以一般来讲，过期键对载入RDB文件的从服务器也不会造成影响
3. 当服务器以AOF持久化模式运行时，如果数据中的某个键已经过期，但它没有被惰性删除或者定期删除，那么AOF文件不会因为这个过期键而产生任何影响。当过期键被惰性删除或者定期删除之后，程序会向AOF文件追加(append)一条DEL命令，来显示地记录该键已被删除
4. 在执行AOF重写的过程中，程序会对数据中的键进行检查，已过期的键不会被保存到重写后的AOF文件中
5. 当服务器运行在复制模式下时，从服务器的过期键删除动作由主服务器控制：
    * 主服务器在删除一个过期键后，会显示地向所有从服务器发送一个DEL命令，告知从服务器删除这个过期键
    * 从服务器在执行客户端发送的读命令时，即使碰到过期键也不会将过期键删除，而是继续像处理未过期的键一样来处理过期键
    * 从服务器只有接到主服务器发来的DEL命令之后，才会删除过期键

                        
        




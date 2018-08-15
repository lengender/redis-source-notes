### Redis zipmap内存布局分析

原贴见：[Redis zipmap内存布局分析](http://www.open-open.com/solution/view/1321697543077)

看到一篇关于zipmap的好帖子，做个搬运工，顺便过一遍。

#### zipmap
Redis被称为key/value应用中的瑞士军刀，除了其丰富的数据结构支持，更重要的是更高效的内存使用，分析源码可以发现作者使用每一个byte都精打细算。在hashtable实现中，Redis引入了zipmap数据结构，保证在hashtable刚创建以及元素较少时，用更少的内存来存储，同时对查询的效率也不会受太大的影响。下面就以源码和例子结合的方式来分析一下zipmap的内存布局。

先来看一下zipmap提供的和存储相关的3个API:
* zipmapNew：创建一个zipmap字符串。zipmap创建时只有两个字节，后面会随着set和delete操作动态扩展和收缩。
* zipmapSet：加入新的key/value或者修改zipmap中已有key对应的value。
* zipmapDel：从zipmap中删除key/value

下面给出一段伪代码并分析其内存布局的变化，如下图：
```cpp
zipmapNew();
zipmapSet(key1, value1);
zipmapSet(key2, value2);
zipmapSet(key1, value3);
zipmapDel(key2);
zipMapSet(key1, value4);
```

![zipmap变化](https://raw.githubusercontent.com/lengender/MarkdownPhotos/master/zipmap.png)


* zipmapNew()
	创建一个zipmap结构体，包含两个字节，第一个字节(zmlen)是长度为1个字节的无符号整数，用来保存zipmap当前元素个数(而非字符串长度)。当zipmap的元素个数大于等于254时，zmlen将不再起作用，zipmap需要遍历整个字符串来获取当前元素个数。最后一个字节为255,表示zipmap结束。

* zipmapSet(key1, value1)
	一个元素(key/value)在zipmap中有5个组成部分：`<len><key><len><free><value>`
	`<len>`表示紧跟其后的string(key或者value)的长度。如果string的长度小于254，`<len>`用一个字节就可以表示(254和255有特殊意义)。如果string的长度大于等于254,`<len>`需要5个字节来表示，第一个字节设置为254，紧跟其后的4个字节通过编码(按主机字节序)来表示`<len>`的值。zipmapEncodeLength和zipmapDecodeLength就是用来对`<len>`进行编码的。`<key>`和`<value>`是char型string，`<free>`在下面步骤中说明。

* zipmapSet(key2, value2)
	调用zipmapset加入新的key/value时，zipmap将更加key2/value2的长度调用zipmapResize扩展空间，并将key2/value2插入到新分配的空间。同时将zipmap元素的个数加一(如果`<zmlen>`小于254)

* zipmapSet(key1, value3)
	调用zipmapSet对已有的key修改其value，且新的value值大于现有value占有的空间时(加free的空间),zipmap将再次调用zipmapResize扩展空间，并调用memmove将key1/value1之后的字符串向后顺移。这里只调用一次memmove，不会对性能有太大影响。

* zipmapDel(key2)
	调用zipmapDel删除key2/value2时，zipmap将把key2/value2之后的字符串前移，并调用zipmapResize收缩占有的内存空间，同时zipmap元素个数减一。

* zipmapSet(key1, valu4)
	调用zipmapSet对已有的key1修改其value，且新的value值小于现有value占用的空间时，zipmap不会马上去调用zipmapResize做内存空间收缩，而是将空闲字节数存入free中，用于后面对这个key再次修改value时，避免zipmapResize(要根据新value的长度而定)。当然free的空间也不能太多，否则会造成空间的浪费。zipmap在free字节数大于等于ZIPMAX_VALUE_MAX_FREE(代码中定义为4)时，就对free的空间进行收缩。

#### 总结
以上就是zipmap内存布局和扩展收缩的过程，你可能会问zipmapGet岂不是O(n)的吗？没错。但因为key和value都是确定长度的字符串，所以这个n就是zipmap中元素的个数，而不是zipmap整个串的长度。只要使用zipmap时保证元素个数不是很多，就可以在时间复杂度和空间复杂度两方面找到很好的平衡点。redis.conf中默认配置hash-max-zipmap-entries为512。

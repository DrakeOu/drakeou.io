# Redis中的数据结构

[来源文档](https://mp.weixin.qq.com/s/MT1tB2_7f5RuOxKhuEm1vQ)



### 字符串

Redis为了极致的性能，对字符串这种简单的类型也做了丰富的优化。

Redis对其定义为**SDS（Simple Dynamic String）**

```c
/* Note: sdshdr5 is never used, we just access the flags byte directly.
 * However is here to document the layout of type 5 SDS strings. */
struct __attribute__ ((__packed__)) sdshdr5 {
    unsignedchar flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsignedchar flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsignedchar flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsignedchar flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsignedchar flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```

可以看到不同的结构都包括了**长度len**， **已分配内存alloc**，**标志位flags**，和**数据buf**。主要是在长度不同的字符串时可以使用更小的byte和short来表示长度信息，已节省空间。



### List

redis中的list相当于Java中的**LinkedList**

```c
/* Node, List, and Iterator are the only data structures used currently. */

typedefstruct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;

typedefstruct listIter {
    listNode *next;
    int direction;
} listIter;

typedefstruct list {
    listNode *head;
    listNode *tail;
    void *(*dup)(void *ptr);
    void (*free)(void *ptr);
    int (*match)(void *ptr, void *key);
    unsignedlong len;
} list;
```

可以看到其定义包含了头尾指针，方便进行O1的头尾操作。

基本操作：

- LPUSH， RPUSH
- LRANGE
- LINDEX

### 字典

相当于Java中的HashMap，通过数组+链表实现，使用拉链法解决Hash冲突。

#### 渐进式rehash

redis中hash的结构定义其实存在两个map，一般只会使用其中一个，而另一个会在**hashmap发生扩容时使用**。

因为redis是单线程，如果像map这样的大结构进行扩容一般需要

1. 申请一个更大的内存空间
2. 将原有的键值对进行迁移

而这将是一个O(n)复杂度的操作，可能造成难以接受的停顿。

所以采用的是**渐进式rehash**，操作同时保留两个hash结构，查询时两个同时查询，而逐渐将内容迁移到新的map中。

#### 扩缩容的条件

正常情况下，当 hash 表中 **元素的个数等于第一维数组的长度时**，就会开始扩容，扩容的新数组是 **原数组大小的 2 倍**。不过如果 Redis 正在做 `bgsave(持久化命令)`，为了减少内存也得过多分离，Redis 尽量不去扩容，但是如果 hash 表非常满了，**达到了第一维数组长度的 5 倍了**，这个时候就会 **强制扩容**。（这里第一维是指hashmap结构中数组的长度，因为拉链的原因整体拥有的元素数量是可能大于数组长度的，当到5倍时差不多每个位置都有元素了，查询效率也会接近链式）

当 hash 表因为元素逐渐被删除变得越来越稀疏时，Redis 会对 hash 表进行缩容来减少 hash 表的第一维数组空间占用。所用的条件是 **元素个数低于数组长度的 10%**，缩容不会考虑 Redis 是否在做 `bgsave`。



### 集合Set

特殊的字典

### 跳表

TODO



### 扩展问题

既然单线程的hashMap扩容会遇到问题而使用新的方式处理，redis中其他的数据结构有这个问题么？

理论上来说好像没有，Set和hash结构完全一样。而字符串是单个数组的重新分配，可以理解为O(1)。list本身的插入删除都是动态的，内存是一个节点一个节点分配的，链表不存在扩容问题。而跳表是多层链表，操作时比单链表麻烦，但是也是节点插入时就发生了调整也不存在扩容问题。
---
title: 关于 Redis 3.0 中的数据结构
date: 2015-05-26 00:48:00
tags:
- Redis
categories:
- 未分类
---


#### Redis 是什么？
Redis 官方网站：[http://redis.io/](http://redis.io/)

-----
#### 为什么要阅读 redis 的源代码？
等价于问题：[知乎：我们（大多数人）为什么喜欢造轮子？](http://www.zhihu.com/question/29852167)

redis 中有功能精简的模块，如 `anet`, `ae`，也有针对 redis 高度优化的数据结构模块，如 `ziplist`, `dict`, `intset` 等。这些模块相对易于学习，也非常有料。redis 本身就是一个相当优秀的开源作品，并且使用广泛，学习其中的实现，也能对 redis 有更深入的了解。

-----
#### 简单开头

Redis 对内存效率的要求常常会高于对时间效率的要求。所以接下来我们更多地会看到 redis 针对数据结构的内存上的优化。

-----
#### Data Structure #1: `adlist`

`adlist` 是 redis 实现的一个双向链表。算法原理和普通的双向链表一样。
但是由于 C 语言没有复制构造函数之类的东西，所以深复制的实现要靠函数指针如：
```c
void *(*dup) (void *ptr);
```
与之对应的还有深释放和深匹配（自己造的词）
```c
void (*free) (void *ptr);
int (*match) (void *ptr, void *key);
```

-----
#### Data Structure #2: `intset`

首先说明， `intset` 内部由数组实现。而 `intset` 顾名思义，只能够存放整数数组。但即使是整数依然有多种类型。`uint32_t`，`int16_t`，`int8_t` 等等。

由于 C 语言不具有多态，如果要一次性存放这些类型的数组，我们可以这样：
```c
typedef struct intset {
    uint32_t *ui32;
    uint16_t *ui16;
    int8_t   *i8;
    ....
} intset;
```

虽然这样可以解决问题，但这显然是一种非常麻烦的方式。而且内存开销大。

redis 提供的思路是：
可以看出，一个 `int16_t` 可以用 2 个 `int8_t` 保存，同理地，一个 `int32_t` 可以用 4 个 `int8_t` 保存。所以，我们只需要一个 `int8_t[]` 数组即可。

我们给出结构体：
```
typedef struct intset {
    uint32_t encoding;
    uint32_t length;
    int8_t contents[];
} intset;
```

* 第一个问题：**如何知道 `intset` 中存储的值的类型？**

  redis 用一个 encoding 值表示 `intset` 中值的类型。

* 第二个问题：**当值超过 `int8_t` 的范围，但在 `int16_t` 的范围内，如何对 `intset` 做出修改？**

  重新调整 `contents` 数组的大小，以容纳 `length` 个 `int16_t` 值，并修改 `encoding` ，即：
```
contents = realloc(sizeof(int16_t) * length);`
encoding = INTSET_ENC_INT16;
```

`intset` 本身还提供了普通的 set 应有的特性，比如 *有序性*，*唯一性*

-----
#### Data Structure #3: `ziplist`

`ziplist` 是一个压缩的双向链表，只能储存整数和字符串

在了解之前，我们首先要理解压缩的意义。压缩的数据一定是静态的，我们很难直接对压缩的数据进行动态操作。如果要进行动态操作，首先要进行解压(decode)，等到操作结束后，再进行压缩(encode）。

回想一下，一个双向链表节点的基本结构是
```c
typedef struct listNode {
	struct listNode *prev;
	struct listNode *next;
	void* value;
} listNode;
```
看起来无法压缩，其实仔细观察会发现，`prev` 和 `next` 指针都会占用 `8bytes` 的空间，可以从这里下手。

redis 提供的思路是：把整个 `ziplist` 放在一块连续的内存块上。这样就可以通过一些计算来获得 `next` 和 `prev`。

假设指向当前节点的指针 `p`，当前节点的内存长度 `len`，那么下一个节点的指针 `next` 显然是：
```
next = p + len;
```
如果要获取 `prev`，则还要存储前一个节点的内存长度 `prevlen`，然后计算：
```
prev = p - prevlen;
```
所以我们可以由此设计出 `listNode` 内存块的分配
```
|prevlensize|prevlen|lensize|len|value|
```
由于 `prevlen` 和 `len` 都是根据值的大小分配不同内存的（如果值小于 `int8_t` 就只会分配 `1byte`），我们就用固定长度为 `1byte` 的 `prevlensize` 和 `lensize` 来保存 `prevlen` 和 `len` 所花费的内存大小。

这样根据值的大小来动态分配值的内存空间，从而达到节约内存的目的，是 redis 常用的一种手段。

这只是大概思路，具体实现还要考虑到很多细节，比较麻烦。

-----
#### Data Structure #4: `zipmap`

与 `ziplist` 用的几乎是同样一种手段。`zipmap` 是一个压缩的 string key-value 数组。获取 value 的复杂度是 O(n) 的，因此适合数据不多的场合。

key-value 节点的结构是
```
<klen><key>|<vlen><free><value>
----key----|-------value-------
```

* Question：**free 区有什么作用？**

这要牵扯到 redis 处理内存和时间平衡的一个技巧。

我们储存一个值 `value`，并为其分配长度为 `len` 的空间，这个值在动态改变的时候，占用的内存可能会变为 `newlen`。 如果 `newlen < len` ，意味着我们会多余出一些空间，这个空间记作 `free`。我们接下来可能要做两种操作：

+ 用 `realloc` 搭配 `memmove` 等操作将内存空间减少至 `newlen`
+ 将这段多余的空间暂且放着，作为一段 `free` 空间。

从时间效率考虑，后者肯定是最优的。（感觉解释起来有点麻烦）。但从内存效率考虑，前者肯定是最优的。

根据具体情况，我们可能只考虑前者（不考虑内存），或只考虑后者（时间要求不高），也可以综合考虑，设置一个参数 `MAX_FREE_VALUE` 表示最大能够容忍的 `free` 空间。如果超过了 `MAX_FREE_VALUE` 我们就把 `free` 空间腾出来。如果没超过，就暂且放着这块 `free` 空间。

-----
#### Data Structure #5: `dict`

跟普通的 Hashtable 差不多，普通的 Hashtable 实现可参照 `java.util.Hashtable<K,V>`

redis 中的 dict 进行了优化，也更复杂：

+ 一个 dict 其实维护了两个 Hashtable

  在 rehash 操作时，我们要进行 Hashtable 的迁移工作，这时候其实可以让两个 Hashtable 一起工作。比如查找节点时，在表1中找不到，则去表2中找。这样允许我们可以不必在 rehash 的时候一次性完成全部的迁移。

+ 渐进式 rehash

  ***为什么要使用渐进式 rehash ？***

  显然，rehash 操作是整个 hashtable 的瓶颈。可以采用分摊的思想，将 rehash 操作分摊给其他操作。比如分摊给 dictAdd, dictFind, dictDelete, dictReplace。由于这些操作平均复杂度都是 O(1) 的，所以每个操作都只能分摊一次 rehash 操作。

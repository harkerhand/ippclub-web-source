---
title: 读 leveldb 的感悟 ：Cache的设计
date: 2016-05-05 21:00:13
tags:
- C++
- LevelDB
categories:
- 未分类
---


leveldb允许Cache（一个缓冲区的抽象）作为一个Option，用户可以自定义它的实现。具体来说就是这样：

```cpp
class Options {
    ...
    Cache *block_cache;
};
```

这是一个难度较高的设计需求。

表面上按照逻辑，首先我们定义好Cache的接口，用户可以自行设置Cache的实现，leveldb默认用自己的 `ShardedLRUCache`（也是LRU Cache的一种）。

然而我们会遇到几个问题：

1. Cache无法使用模板，因为它是一个Option。当然，这意味着Cache就是一个包含纯虚函数的接口类。

2. Cache中所存储的key/value的value类型不确定，key的类型已知是 string

3. Cache查询时返回一个句柄（Handle），这个Handle应该是什么

4. Handle的内容可以被用户自定义吗？如果是的话，那如何设计？

这几个问题的本质在于Option中无法得到Cache的更多信息，由于模板与继承不同，我们无法使用未指定类型的模板类的指针。简单说就是，我们不能这样写：
```cpp
Cache<K, V> *block_cache;
```

在其他一切开源的Cache实现里，Cache通常设计为模板类，他们可以这样：

```cpp
template < class TKey, class TValue, class TStrategy>
class AbstractCache
```

我们通过问题的解决慢慢来解释我们的设计。

问题2 的解决可以引入 `boost::any`（其实`void*`也可以），使用Cache的用户需要知道自己的Value是什么类型，并且要保证自己只加入了一种类型的值。这个做法让我们依然无力控制value的类型。

问题3  我们定义Handle。Handle 的定义与 Iterator 有区别，Iterator的Concept指出Iterator是一种可以用来遍历容器内每个元素的抽象，而Handle是不可遍历的，我们只能取得Handle所指的元素的值，这点和Iterator一样。

问题4 Handle 的内容。首先要知道，Handle在一个接口类中定义，那么它本身就无法包含任何信息(It doesn't carry any properties)。如果是这样，那么它只能是一个 opaque object。或许它可以包含纯虚函数，但与其在Handle中定义，不如在Cache中定义，因为这意味着Handle对象构造时会多一个vptr，而Cache本来已经有vptr了，Handle的纯虚函数定义在Cache中就无所谓，比如说leveldb这样来获取一个Handle所指元素的value的：
```cpp
virtual Cache::Value(Handle* h) const = 0;
```
其次，Handle的内容并不重要。可以说，Handle是C++的一种设计模式。Handle自身不能做任何事，它只能用来与Cache交互。具体的思路可以参照 windows 的 `HWND`。其实原理很简单，我们在Cache的子类，例如 LRUCache，实现一个 LRUHandle，它与 `Cache::Handle` 没有任何继承关系，然后我们只要用 `reinterpret_cast<Handle*>`就能将 LRUHandle 在内部转为 Handle。

按照这个思路，Cache的value类型不确定是一个隐患，这是由于Cache是一个Option所导致的。我们要如何设计Cache的抽象？

Cache的值类型不确定导致它不能像 `poco::AbstractCache` 或者 guava 的 cache 那样，定义一个全套的Cache所应该有的标准API，它只能作为一个组件来使用，我们采用组合模式来使用它。
按照这个思路设计，Cache应该是主要算法的抽象，我们需要将算法，和其他组件一起组装起来才能构成一个真正的Cache，因此，我将leveldb 的Cache改名为 CacheStrategy，由此就符合了我们的设计思路：CacheStrategy 不保证Cache里值的类型都一样，这由Cache来做。

总结一下我们的设计：
```cpp
class Options {
	…
	CacheStategy *block_cache_strategy; //
};
```
`class CacheStrategy` 代替 `leveldb::Cache`。

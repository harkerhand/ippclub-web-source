---
title: mongo 源码阅读感悟（一）
date: 2016-06-12 19:03:17
tags:
- C++
- mongo
categories:
- 未分类
---

其实开头的这一篇不完全是阅读 mongo 时的感悟，也包括了我在 leveldb 中学到的技巧。看过 leveldb 的应该都知道，这份代码的注释不算清楚，文档也少，阅读体验不算好，需要 C++ 开发者有足够的经验。因此有些东西我到了mongo才真正了解它们的作用。也是十分惭愧。

最近在阅读 mongo/bson 的源码，我试图去实现一个 bson，在造轮子的过程中学习他们的编程技巧，事实证明阅读大型且较新的项目的源码要远远好过阅读小而老的项目。

这篇文章我来介绍两个很小的类，`Status` 和 `StringData`，`StringData` 在 leveldb 中是 `Slice`，为了方便，之后我都会用 `Slice`。

### Status

许多大型 C++ 项目的编程规范要求 exception free，在实际的编程中，开发者会发现使用 error 来代替 exception 其实十分实用（golang开发中更多使用 error 而非 exception），因为很多的编程错误被强行放到了 exception 中，比如 java 的 `IndexOutOfBoundException`。再者，exception 也常常拉低了执行效率。对于不在意性能的语言（比如Java），exception 的罪状更多的在于如果不将异常及时catch，exception会蔓延到外部。

这里我们介绍一个设施，它可以保证错误只会被手动的抛到外部，甚至我们可以保存错误的状态，延迟抛出（异常能做这件事吗？）。

在 mongo 和 leveldb 中，都用了 Status 封装错误信息。

Status 封装了 enum 类型的错误码，并且可以为每个错误绑定它的具体错误信息。

```cpp
static Status FailedToParse(const Slice &msg);
```

一个 FailedToParse 错误就可以这样被抛出来。

为了节约内存，enum 被转为 `unsigned char`，用以存储至多 256 种错误。当然，大型项目可以多于这个数。

`Status::OK()` 表示这个函数调用没有错误，跟 C 语言里返回值为 0 类似，Status 通常也是通过函数返回值抛出。我们要尽量使得成功的函数调用几乎无需处理错误，只有在错误发生的时候才去处理错误。因此我们这样实现：

```cpp
class Status {
 private:
  std::unique_ptr<ErrorInfo> info_;
};
```

`Status` 内部只保存指向错误信息的指针。`Status::OK()` 抛出一个未初始化的指针（指向 null），只有在错误的信息抛出时，才会生成信息给指针。

### Slice

`Slice` 是对字符串的一种简单的 reference 的封装，它仅保存字符串的指针（`char*`）和字符串的长度。与 `std::string` 用途不同，它不会对字符串进行复制操作，在这一点上比 `std::string` 要好用许多。当然，我们也可以在整个项目中，完全使用 `std::string`，并且在函数接口中使用 `const std::string&`，这样在一个需要大量使用字符串的代码中，函数的接口会变得混乱不堪。

使用了 `Slice` 之后，我们可以这样编写一个函数的接口。

```cpp
Parser::parseInt(Slice int_string);
```
一方面这样可以简化函数接口（如果要 forward declaration 则需要引用），一方面这样可以兼容 raw string 和 `std::string`，不会出现 raw string 传参给 `const std::string &` 时，出现复制的情况。

唯一的缺点是我们需要去重新实现一些基本的工具函数。好在有 `<cstring>`，使得这一切并不算难。

### 附录
附上我的实现：

+ Slice: [github link](https://github.com/neverchanje/bson-cpp11/blob/master/src%2FSlice.h)

+ Status: [github link](https://github.com/neverchanje/bson-cpp11/blob/master/src%2FStatus.h)

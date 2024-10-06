---
title: '关于 boost::noncopyable 的一点思考'
date: 2015-10-01 21:16:00
tags:
- C++
- Boost
categories:
- 未分类
---


一般通过继承 `boost::noncopyable` 来保证类的对象不会被复制。我们会采用如下写法：

```cpp
class Buffer : boost::noncopyable {
```

注意的是，我们既可以采用公有继承 (public)，也可以采用默认继承方式 (private)，具体使用哪一种方式，需要我们的思考。

譬如，当父类拥有了 noncopyable 的特性，一般情况下，我们希望子类在继承父类的同时，也能继承其 noncopyable 的特性，于是子类就 **无需重复继承 `boost::noncoyable`** 。举个例子：

```cpp
class FixedSizeBuffer : public Buffer {
```

由于 `Buffer` 不应该被复制，在 `FixedSizeBuffer` 公有继承了 `Buffer`之后，我们希望 `FixedSizeBuffer` 能够 **同时继承 `Buffer` 的 noncopyable 的特性**。我们就不必这样写：

```cpp
class FixedSizeBuffer : public Buffer, boost::noncopyable {
```

我们可以做到吗？

首先我们来了解 boost::noncopyable 的内部实现。

```cpp
class noncopyable
{
  protected:
      noncopyable() {}
      ~noncopyable() {}
  private:  // emphasize the following members are private
      noncopyable( const noncopyable& );
      noncopyable& operator=( const noncopyable& );
  };
}
```

这是 C++99 的版本，如果是 C++11 则会使用如下写法：

```cpp
  protected:
      noncopyable( const noncopyable& ) = delete;
      noncopyable& operator=( const noncopyable& ) = delete;
```

我们知道 protected 声明的成员在公有继承下依然可见，而 private 不再可见，那是否意味着 C++99 和 C++11 的两种实现有所不同？

-------

以上是我们平时的思考。如果已知答案，这些思考只是矫情跟无聊而已。

实际上，这两种方法完全相同，使用 public 继承与使用 private 继承也 **完全没有任何区别**。

因为子类在使用 copy constructor 和 assignment operator 的时候，必须调用父类的相应函数。显然，如果把相应函数

+ 放在 private 内，或者
+ 放在 protected 块内，并用 `=delete` 标识；

那子类即使实现了相应函数也无法成功调用。


------

所以我们得出一个结论：只要父类继承了 `boost::noncopyable`，那么子类就无需重复继承。另一方面，我们一般使用 private 继承来为类增加功能，所以我们一般使用默认继承方式（private 继承）来继承 `boost::noncopyable`，即：

```cpp
class Buffer : boost::noncopyable
```

```cpp
class FixedSizeBuffer : public Buffer
```

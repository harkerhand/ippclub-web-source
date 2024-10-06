---
title: C++ 的三种实现接口的方式
date: 2016-03-11 21:45:12
tags:
- C++
categories:
- 未分类
---

## 传统 interface

想要实现 interface，在绝大多数 OOP 语言中，会被认为只有一种。如 java 和 C# 的 `interface` specifier，C++ 的 pure virtual function。通常接口类中强制的不能拥有实现，继承接口的子类至少享有**两个名字**，一个是自身定义的命名，一个是接口名。通常子类转父类（up-cast）在编译期（compile-time）决定，父类转子类（down-cast）在运行时（run-time）决定。

我们太过于习惯这种接口的方式，而这种接口方式也确实能十分灵活地应用在编程中，以至于我们常常不自觉的在设计中增加不必要的层数。如果你也喜欢 "flat is better than nested" 的理念的话，我接下来将会介绍两种 alternatives，用来代替传统的接口实现。

## Functor

使用 C 语言同样也可以进行 OOP，思路就是 function pointer。通过注册回调函数，用来替代传统的通过继承来实现接口。 C++11 所提供的`std::function`，`std::bind` 和 lambda function 为此提供了极大的便利。

#### Example

```cpp
class Comparator {
 public:

  typedef std::function<int(const Slice &, const Slice &)> CompareFunc;

  Comparator(const CompareFunc &compare_fn) :
      compare_(compare_fn)
  }

  // Three-way comparison.
  virtual int Compare(const std::string &lhs,
                      const std::string &rhs) const {
    return compare_(lhs, rhs);
  }

 private:
  const CompareFunc compare_;
};
```
每种 `Comparator` 的示例都可能实现某一种策略(Strategy)。而如果为每一个策略都定义一个子类，会将问题复杂化(flat is better than nested)。

#### 限制
+ 所有实例只能拥有父类的名字，不拥有自己的名字。当然设计上也不应该使实例拥有自己的名字。

#### 适用
+ [Strategy Pattern](https://en.wikipedia.org/wiki/Strategy_pattern)
+ 可以通过[Decorator Pattern](https://en.wikipedia.org/wiki/Decorator_pattern)来进行向下扩展。

## CRTP
所谓接口其实是一种协议，换言之就是，继承接口的子类必须**全部**实现某些功能（函数），由于子类必须不可妥协地全部实现，因此从设计的角度讲，接口的粒度也应该越细越好。

为了某一个类去实现多个接口，可能不是一种干净的方法，因为这些接口将来可能不一定会被使用。C++ 的设计者为我们提供了无比强大的 template，用来更灵活地设计模块间关系。

CRTP，即 [Curiously recurring template pattern](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern)。简单说就是将类自身作为自身模板的参数。 [boost::iterator_facade](http://www.boost.org/doc/libs/1_54_0/libs/iterator/doc/iterator_facade.html) 就利用了这种方法设计了一个 iterator 的抽象类，我们可以通过继承 `boost::iterator_facade` 轻易地实现一个 iterator。（吐槽一下，Boost.Iterator 这个库实在是太臃肿了，实际工程上完全可以写一个简易版本）

我们通过实现一个 `IteratorFacade` 来介绍这种方法，然而这种方法的实例较长，所以我们分开来讲。

#### IteratorFacade
```cpp
template<class Derived, class T>
class IteratorFacade {
```
我们自己定义的 `Iterator` 只要继承 `IteratorFacade` 就能拥有一堆跟标准库容器一样的细枝末节的功能了（这些功能确实多而且杂）。我们首先假设 `Iterator`需要是一个 [forward iterator](http://en.cppreference.com/w/cpp/concept/ForwardIterator)。
```
class Iterator:
    public IteratorFacade<
        Iterator,
        int,
        std::forward_iterator_tag> {

```

其中 `Derived` 参数就是 `Iterator` 自己了，而 `int` 对应的 `T` 参数表示这个 iterator 所指的值。

现在我们来仔细看看 `IteratorFacade` 的定义：

```cpp
template<
    class Derived,
    class T,
    class Category>
class IteratorFacade {
 public:
  typedef T& Reference;
  Derived Next() {
    Derived tmp(derived());
    IteratorCoreAccess::increment(derived());
    return tmp;
  }
  template<class Facade>
  static typename Facade::reference dereference(Facade const &f) {
    return f.dereference();
  }
  Reference Value() const {
    return IteratorCoreAccess::dereference(derived());
  }
 protected:
  Derived &derived() {
    return *static_cast<Derived *>(this);
  }
};
```

这里为了方便，我们使用 `Next` 而非 `operator++`（因为 ++ 操作符还分 postfix 和 prefix...），希望阅读到此处的读者不要奇怪。

#### 实现接口

我们要在 `IteratorFacade` 中实现的就是，用户只需实现 `increment`，`dereference` 函数，并且将 `IteratorCoreAccess` 设为友元类，即可实现一个 forward iterator（`Next()`函数可用）。如果我们想要实现一个 [bidirectional iterator](http://en.cppreference.com/w/cpp/concept/BidirectionalIterator)，我们可以实现 `decrement`，这样 `Prev()`函数即可用。

这里的秘密在于 `IteratorCoreAccess`，由于其中的成员函数用 static 声明，例如只有在 `decrement` 被调用时，编译器才会去检查 `Facade` 类型是否具有 member function `decrement`。

```
class IteratorCoreAccess {
  template<class D, class T, class C> friend
  class IteratorFacade;
 private:
  template<class Facade>
  static void increment(Facade &f) {
    f.increment();
  }
  template<class Facade>
  static void decrement(Facade &f) {
    f.decrement();
  }
  template<class Facade>
  static typename Facade::reference dereference(Facade const &f) {
    return f.dereference();
  }
  // Objects of this class are useless.
  IteratorCoreAccess() = delete;
};
```

相对于传统的 interface，这种方法（或者说设计模式）是强耦合的。它提供给用户有限的几种方案，用户根据自身需求（比如需要 forward iterator 还是 bidirectional iterator），实现相应的函数。

#### 限制
+ 这种方法几乎没有多态可言，因为父类的名字不可被使用（由于模板参数的不同，`IteratorFacade` 的两个子类不一定享有同一个父类）。这导致一定程度上扩展性降低，适用于需求极其稳定的情形。
+ 不可向下扩展。

#### 优点
+ 灵活性强。
+ 简化了接口的设计。有点像 @叛逆者 曾经在知乎上回答的，[游戏主机效率PC高的回答](https://www.zhihu.com/question/37208466/answer/71549555)中提到的，把原先积木拼装结构，变成电焊焊死的，更加结实，也失去了可扩展性。

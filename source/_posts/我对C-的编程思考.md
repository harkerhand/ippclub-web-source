---
title: 我对C++的编程思考
date: 2015-12-23 21:38:41
tags:
- C++
categories:
- 未分类
---

我最近想要这样一个分享平台：它既可以作为我的笔记工具，记录一些文字不太多又不太少的技术知识，又可以作为一个社交平台让别人认识我，让我可以装逼。微博限制字数，朋友圈也不太合适，于是我还是在博客里开一个以后可能会继续更新的文章。现在 I++ 里的博文只有我在做 C++ 的分享，以后如果有人做同样的事情，甚至写了同名的文章，那就可能要给这整个博客做点修改了，比如在博客的 timeline 上加上作者头像和信息，以前一直想做，因为比较懒所以搁置了。

### 应该使用 Returned Value Optimization 还是 `std::move`

----

分享一篇我读过的文章 [RVO V.S. std::move](https://www.ibm.com/developerworks/community/blogs/5894415f-be62-4bc0-81c5-3956e82276f3/entry/RVO_V_S_std_move?lang=zh)

这里讲到了几个以前我未曾求解的顾虑和疑惑：

+ C++11 其实已经将 RVO 写在标准中并为大多数编译器支持。这样编写合适的代码就不会再有顾虑了。

+ std::move 的使用场景。实际上 std::move 还是会造成一定的开销，只是相比于复制，移动会更快。

### C++ 使用指针还是创建对象？

------

阅读这篇文章：[C++中为什么要用指针，而不直接使用对象？](http://blog.jobbole.com/90147/)

最近写一些代码的时候，仅仅只是因为写的顺了，没经过思考地使用了 `std::unique_ptr`，现在觉得有些后悔，因为这些地方完全可以用对象代替。但是因为除了代码稍微混乱一些之外，并无伤大雅，所以还在犹豫是否要对他们进行修改。以我现在比较浅薄的经验看来，指针能够让我感觉自然的使用场景大概有这些：

+ 延迟分配内存（动态分配内存）

简单的情况是
```
Object  arr[100];
Object* arr[100];
```
对比于上下两种情况，后者在数组初始化的时候开销更小，在有些时候是更好的做法。

+ 用于 forward declaration

因为 C++ 要求声明对象必须要有类的定义，这就要求必须要 include 相关头文件，为了减少头文件依赖，有时候会用到指针。这种情况也不多。

+ shared_ptr

针对某一内存块会被多个使用者同时使用的情况。

+ 用于保留容器元素的继承属性

假如有 `B: public A` 与集合 `std::set<A>`。现在我们需要把 B 放入 A 的集合中，在 Java 中，集合内存储的元素依然是 B，而在 C++ 中，编译器会将 B 复制给类型为 A 的集合元素。此时集合内实际存储的是 A。我们希望实现 Java 的效果，就要使用指针。显然 `std::unique_ptr` 在这里可以发挥作用。

讲到这里，我想谈一谈我最近遇到的关于 `std::unique_ptr` 的几个小经历。

###有关 `std::unique_ptr` 的编程经历

-----

语法多如 C++ 的语言总有很多隐形的坑，在编写代码的过程中有时会陷入困境，不知道如何解决，但是山重水复之后，往往会发现柳暗花明的方法。

+ 在 stl 容器中存储 unique_ptr

我们以 `std::set` 来举例。

```cpp
template < class T,                        // set::key_type/value_type
           class Compare = less<T>,        // set::key_compare/value_compare
           class Alloc = allocator<T>      // set::allocator_type
           > class set;
```

这个接口告诉我们，用户需要在 `std::set` 声明处定义比较函数 comparator。换句话说，一个 `std::set` 由两个东西定义，一个是元素类型，一个是元素之间比较的方法（不考虑 allocator）。

我们其实可以将 comparator 从 set 的定义中解耦出来。实际上 C++ 有多种实现自定义比较的方法。如这篇文章所讲 [3 Ways to Define Comparison Functions in C++](http://fusharblog.com/3-ways-to-define-comparison-functions-in-cpp/)。简单说就是通过重载 `operator<` 来自定义比较方法，这样做就**无需显式定义 comparator**。当然，更多的时候，我还是喜欢显式定义 comparator，这样能更清楚的表明，**一个 set 只允许一种comparator**。

然而，这与 Java 不同。Java 的模板类型会[在运行时用具体的类型代替](http://www.infoq.com/cn/articles/cf-java-generics)。也就是说，给定集合 `set<T>` 可以代表的不只是 T 的集合，也可以是 T 的子类 SubT1 的集合，SubT2 的集合等等，并且他们之间也互相 compatible（可以直接用等号赋值）。这对面向对象来说是十分自然的一种设计，这意味着当我们将 SubT 继承自 T 之后，我们也同时将 SubT 的 container 继承自 T 的 container。

```java
//Example Code
TreeSet<E> treeSet = new TreeSet<E>();
SubE se = new SubE(1, 2);
treeSet.add(se);
// E 的 print 会输出 E
// SubE 的 print 会输出 SubE
// 这里输出的是 SubE
treeSet.first().print();
```

而在 C++ 中，我们却需要显式地声明 `set<T>` 和 `set<SubT1>`, `set<SubT2>` ... 遇到这个问题，我们可能会开始想到使用 `dynamic_cast`，然后随即又被[它的低效率](https://www.zhihu.com/question/22445339)（见 effective c++ 中的条款27）给吓跑。如果我们想要避免 `dynamic_cast`，本质上，我们就需要一些 workaround **避免 `dynamic_cast`**。

整理一下思路，我们希望能在 `std::set` 中实现多态（Polymorphism），这其中的核心问题在于实现多态的 comparator。而我们又不希望通过 RTTI 实现 downcast(dynamic_cast)，这意味着，我们**不能直接使用子类 SubT 进行自定义比较**。

```cpp
//Example Code
class A {
public:
	int a;
};

class B : public A {
public:
	int b;
};

inline bool operator<(const std::unique_ptr<A> &i1,
					  const std::unique_ptr<A> &i2) {
	return i1->a < i2->a;
}


typedef std::set<std::unique_ptr<A> > ASet;
```

上面的代码中，在 `ASet` 中插入 B 的指针时，不会对属性 `b` 的值进行比较。简单说，相同 `a` 值，不同 `b` 值的 B 会被视作相同。

排除了几种选择之后，我们可以想到用 virtual function，利用指针保留继承链。最终我使用的方法是***留后门***：既然不能直接比较 `b` 的值，就只能绕弯路，**间接地进行比较**。

```cpp
class A {
public:
	virtual boost::any Others() const {
		return 0;
	}

	virtual bool CompareOthers(const A& a) const {
		return false;
	}

	int a;
};

class B : public A {
public:
	virtual boost::any Others() const {
		return b;
	}

	virtual bool CompareOthers(const A& a) const {
		return b < boost::any_cast<int>(rhs.Others());
	}

	int b;
};
```
这里就可以看到我们留后门的两个函数 `Others` 和 `CompareOthers`，间接地利用类型转换来实现。最终的自定义比较函数就可以这么写：

```cpp
inline bool operator<(const std::unique_ptr<A> &i1,
					  const std::unique_ptr<A> &i2) {
	if(i1->a == i2->a) {
		return i1->CompareOthers(*i2);
	}
	return i1->a < i2->a;
}
```
这样我们算是基本完成了需求的实现。

还有一点。上面我们使用了 `boost::any` 表示可以接受任何值。这样写比较直观，但是[`any_cast` 的实际性能上貌似与 `dynamic_cast` 相差不大](http://www.nullptr.me/2011/05/10/boostany/)，都是使用 RTTI。在 benchmark 下，甚至可能出现 [`boost::any` 比 `void *` 性能相差近50倍的情况](https://felipedelamuerte.wordpress.com/2012/04/06/why-you-shouldnt-use-boostany-especially-not-in-time-critical-code/)。不过这里有充足的优化空间，我们可以完全不必使用 `boost::any_cast`，而是简单使用 `static_cast` 就可完成任务。所以这个方法可行。

看上去，我们好像漂亮地完成了一个（可以不使用 RTTI 的）[***Type Erasure***](https://en.wikipedia.org/wiki/Type_erasure)。然而，这意味着我们依然需要使用 downcast 来实现 ***Type inference***，否则，我们的子类 SubT 将无法在任何使用了 `std::set<std::unique_ptr<T> >` 的地方使用。除非我们可以将 T 与其子类完全地从它们的 caller 中解耦出来，这种方案才有意义。换句话说，我们需要让 caller 完全不必在乎它们使用的类型是 T，还是 SubT1，SubT2，所有的细节都由虚函数来完成。如果可以做到这样，我们就能在不使用 RTTI 的情况下，像 Java 那样完成工作。

当然，如果使用了 RTTI，那 C++ 和 Java 的区别就很小了。

多数情况下，我们没有办法不使用 downcast。那么现在问题的核心在于，有没有其他的方法，可以在不使用 RTTI 的前提下，实现 downcast。

如果在对象较小的情况下，我们可以使用 [***the clone pattern***](http://www.cplusplus.com/forum/articles/18757/) 来实现，但在这里不是 clone，我们定义一个 DownCast 函数。

```cpp
template<class T>
typename std::enable_if<std::is_same<T, B>::value, B>::type
DownCast(const std::unique_ptr<A> &pA) {
  B ret(pA->a, boost::any_cast<int>(pA->Others()));
  return ret;
}

// 我们在这里利用复制来进行 downcast，不使用 RTTI
B b = DownCast<B>(pA);
```
当然，我们这里依然假设 `boost::any_cast` 是一个 exception safe 且 without RTTI 的转换函数。因为我们在这里完全可以优化到这样。

顺便扔出一个刚刚学会的名词：[SFINAE](https://en.wikipedia.org/wiki/Substitution_failure_is_not_an_error)

-----

### 更新一下

之前在 [`boost::any` 比 `void *` 性能相差近50倍](https://felipedelamuerte.wordpress.com/2012/04/06/why-you-shouldnt-use-boostany-especially-not-in-time-critical-code/)
的链接里，博主提供了测试代码。测试程序测试对基本类型`size_t`转换所用时长。

benchmark的结果和我的差不太多。
在我的 MacBook Air (13-inch, Early 2014)，clang version 3.6.2 下

debug mode 的数据是：
```
boost::any - 18.8269
boost::spirit::hold_any - 3.50993
void ptr - 0.860663
```
release mode 的数据是
```
boost::any - 8.41195
boost::spirit::hold_any - 0.499917
void ptr - 0.043697
```

公有变量 x 也用了 `volatile` 修饰，结果应该可靠，可见，`boost::any` 确实较慢。拖累性能的地方在于exception，可能还因为一部分中间的执行代码，但应该与代码中极少数使用 `typeid` 的地方无关，知乎上 R大 也曾经解释过，[而当typeid运算符应用在一个指向多态类型对象的指针上的时候，typeid的实现才需要有运行时行为。](https://www.zhihu.com/question/38997922/answer/79179526)

所以我们在上面实现 CompareOthers 和 DownCast 的时候，完全可以不用考虑异常，类型转换失败则程序应停止运行而不是抛出异常。

------

###C++ 中指针的不便之处

跑题了很多，将这次的经历归纳在 `std::unique_ptr` 下更多的是因为，C++ 中指针与对象之间被区分开，这就意味着，
我们在判断两个 `std::set<std::unique_ptr<A> >` 是否相同时，默认对指针进行比较，而不是对对象进行比较。
这个时候，我们要为此编写 `std::set` 的自定义比较函数。

我们在使用 `std::unordered_set<std::unique_ptr<A> >` 时也会有相同的问题，同样的，我们也要编写自定义 hasher。

如果这被 C++ 标准认为是自然的行为的话，那么，我们来看看它会带来的不便之处。

不便之处在我们使用 `std::set<std::set<std::unique_ptr> >` 时尤为突出。虽然 `unique_ptr` 禁止 copy，但是 set
是允许 copy 的，这就意味这我们可能需要重新编写 set<unique_ptr> 的复制函数。这让我很疑惑为什么 unique_ptr
没有提供默认的 deep copy 接口来复制所指的对象。

我们为了使用 `std::set<std::unique_ptr>`，编写了custom comparator。在使用 `std::set<std::set<std::unique_ptr> >` 时，我们无需再次编写 comparator（stl 为提供 `std::set` 了 rational operator）。然而，如果我们使用的是 `std::unordered_set<std::set<std::unique_ptr<A> > >`，我们可能会这么写：
```cpp
struct AHasher {
  size_t
  operator()(const std::set<std::unique_ptr<A> &val) const {
    return boost::hash_value(val);
  }
};
```
这里使用了 `boost/functional/hash.hpp`。这里看起来合理，但程序运行的时候，我们会发现这与我们预期的不符。这是因为，我们忽略了为 `std::unique_ptr<A>` 自定义 hasher。这里默认地是对指针的值进行 hash，而不是对对象进行 hash。

重写一下：
```cpp
struct AHasher {
  size_t
  operator()(const std::set<std::unique_ptr<A> &val) const {
    std::size_t seed = 0;
    for (const auto &it : val) {
      boost::hash_combine(seed, it->HashValue());
    }
    return seed;
  }
};
```

原来我们在 set 中存储本来的对象，只需要定义好对象本身的 rational operator 和 copy function，亦或者是 `unordered_set`
中的 hasher，就可以很自然地使用了。使用了 `unique_ptr` 之后，我们还需要重新定义 unique_ptr 的对应函数，才能将 `unique_ptr` 完全视作对象。

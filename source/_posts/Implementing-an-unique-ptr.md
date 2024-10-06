---
title: Implementing an unique_ptr
date: 2015-10-04 18:40:00
tags:
- C++
categories:
- 未分类
---

事实证明实现一个 unique_ptr 对其原理的认知并没有什么提升。

为了简便，我们只实现 single object 版本，不实现存储 array type 的 unique_ptr。

----------

我们首先要对 UniquePtr 的定义进行声明：

```cpp
template<class T, class D>
class UniquePtr {
```

T 是 UniquePtr 所存储的指针的类型，D 是其 Deleter 的类型，UniquePtr 在析构时调用 Deleter，从而保指针离开所在域之后即被释放，这是简单原理，不详谈。

首先是一个模板类型的细节，在实现容器类的时候常常需要用到并且注意的：
UniquePtr 中所存储的类型不总是 T，而是 T 及其子类，我们记作 U。于是乎，我们需要提供这样一个构造函数：
```cpp
template<class U>
UniquePtr(U *ptr);
```

函数的原型在 [cppreference/memory/unique_ptr](http://en.cppreference.com/w/cpp/memory/unique_ptr) 中都是有提供的，所以其实实现的时候可以不用考虑设计。

然而我们要注意一个关于 Deleter 的细节：
> a) If D is non-reference type A, then the signatures are:
>
> + `unique_ptr(pointer p, const A& d);` (requires that Deleter is nothrow-CopyConstructible)
> + `unique_ptr(pointer p, A&& d);` (requires that Deleter is nothrow-MoveConstructible)
>
> b) If D is an lvalue-reference type A&, then the signatures are:
>
> + `unique_ptr(pointer p, A& d);`
> + `unique_ptr(pointer p, A&& d);`
>
> c) If D is an lvalue-reference type const A&, then the signatures are:
>
> + `unique_ptr(pointer p, const A& d);`
> + `unique_ptr(pointer p, const A&& d);`

由于每种情况提供两种 Deleter 类型不同的构造函数，我们将上面一种的 Deleter 类型记作 `del_arg1_type`，下面的记作 `del_arg2_type`。

我们可以轻易地整理出逻辑：如果 D 是 non-reference，则
```cpp
typedef typename const remove_reference<D>::type& del_arg1_type;
```
否则
```cpp
typedef D del_arg1_type;
```

另一方面，`del_arg2_type` 的逻辑就很简单了：
```cpp
typedef typename remove_reference<D>::type && del_arg2_type;
```

这样一来，我们就解决了这部分的问题。

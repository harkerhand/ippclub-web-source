---
title: 用 C++ 写一个 Inet4Address 类
date: 2015-11-01 14:12:00
tags:
- C++
categories:
- 未分类
---


`Inet4Address` 在 .net 和 facebook/folly 中也叫做 `IPAddress`。
（顺便黑一下，muduo 中居然把 `SocketAddress` 叫做 `InetAddress` 真是有失偏颇。

----

### 构造ip地址

<br/>

ip 地址的构造，比较麻烦的地方就是网络序和主机序之间的问题。

我们的 raw ipv4 address 存储在一个 `uint32_t` 的整数中，按照 [linux 官方手册](http://man7.org/linux/man-pages/man7/ip.7.html)中的描述，以下结构体表示 ipv4 address：

```c
/* Internet address. */
struct in_addr {
    uint32_t s_addr; /* address in network byte order */
};
```

`s_addr` 是按网络序存储的，然而在下面的测试中，有与我预期不符合的地方：

```cpp
BOOST_AUTO_TEST_CASE(test_byteorder_inaddr) {
  in_addr addr;
  inet_pton(AF_INET, "192.168.0.1", &addr);
  BOOST_CHECK_EQUAL(addr.s_addr, 0x0100A8C0);
}
```

我原先以为高位的 192 应该在 b3，而实际上在 b0。在 Java 中 的 `Inet4Address.getAddress()` 也有注释说明
```java
/**
 * Returns the raw IP address of this {@code InetAddress}
 * object. The result is in network byte order: the highest order
 * byte of the address is in {@code getAddress()[0]}.
 *
 * @return  the raw IP address of this object.
 */
public byte[] getAddress() {
    return null;
}
```
（一开始没有理解这一点，我在 s_addr 存储的是 little-endian 值，以至于[我引入了 redis 的 endianconv](https://github.com/neverchanje/sugen/blob/master/sugen/net/EndianConv.h) 作为工具函数，现在看来暂时没有用武之地了）

我们首先规定**我们将使用 big-endian 存储 IP 地址**，并统一使用

对于 IP 地址：**b0.b1.b2.b3**，我们用

**0x b3 b2 b1 b0 来构造大端序**

**0x b0 b1 b2 b3 来构造小端序**

1. 我们允许使用者通过 `Byte4` 类型，也就是 `std::array<uint8_t, 4>` 作为参数来构造 `Inet4Address`，前提是 `Byte4` 参数是网络序：

	**即对于 "192.168.0.1"， b0 = 192, b1 = 168, b2 = 0, b3 = 1**

2. 我们也允许使用者通过 NBO 的 `uint32_t` 类型的地址进行构造，这时候我们该用  `ntohl` 或 `htonl` 来实现。

	 ------

### 特殊的 ip 地址
<br/>

+ [loopback address](https://github.com/neverchanje/sugen/blob/master/sugen/net/Inet4Address.h#L82)
环回地址。也就是`127.0.0.1`

+ [link local address](https://github.com/neverchanje/sugen/blob/master/sugen/net/Inet4Address.h#L88)

+ [multicast address](https://github.com/neverchanje/sugen/blob/master/sugen/net/Inet4Address.h#L76)

+ [private address](https://github.com/neverchanje/sugen/blob/master/sugen/net/Inet4Address.h#L96)

这些地址在代码注释中有提供描述他们的 RFC 文档。

-----
### 总结
<br />

因为理解的偏差，造轮子的时候走了一些远路（但不是歧路）：

+ `in_addr.s_addr` 是否为 NBO

+ b0 为 ipv4 的高8位地址，而不是 b3

+ 引入 redis 的 endianconv，写了一些无用代码，希望以后会派上用场。

+ 当然还有阅读 RFC 文档，理解这些特殊 IP 地址的定义和用途。

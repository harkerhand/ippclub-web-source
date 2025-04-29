---
mermaid: true
title: Makefile 的食用方法
date: 2025-04-04 16:43:06
tags: 
- C++ 
- shell 
- 命令行
- makefile		
categories: 科普
author: D-major
index_img: Makefile的使用方法/封面.jpg
banner_img: Makefile的使用方法/封面.jpg
---
## 前置知识

C/C++ 的基础语法，Shell中命令的基本结构:`命令名 [选项] [参数]`，文件依赖关系：某个文件需要引用、包含或使用其他文件的内容才能正常工作。

---

## 第一个例子

首先，让我们来看一个简单的C++源码从预处理，编译，链接到一个完整的可执行程序在 CLI(Command Line Interface) 中需要什么命令吧。

*以 Ubuntu 上的 bash 为例*

源码：

```c++
//Hello_world.cpp
#include "Print.h"
using namespace std;

int main(){
    print_w("Hello, world.\n");
    return 0;
}
```

```c++
//Print.cpp
#include <iostream>
#include <string>

void print_w(std::string str){
    std::cout << str;
}
```

```c++
//Print.h
#include <iostream>
#include <string>

void print_w(std::string);
```



其文件依赖关系是：

![makefile_tutorial_01](https://raw.githubusercontent.com/Davidwadesmith/image-hosting/main/img/202504041719657.svg)

CLI：

```bash
g++ -c Hello_world.cpp Print.cpp
g++ Print.o Hello_world.o -o Hello_world
```

需要两行命令。

---

## Makefile 的用处

makefile 被设计出来是为了简化构建 (build) 一个项目的流程，在上面的例子中构建一个项目仅需两行命令即可，但是更可能的情况是，有一堆```.cpp```文件，一个一个的添加十分麻烦。并且代码的相互依赖关系相当复杂。比如，想象一个C++程序，它是一个模块化的应用系统，通过核心引擎 (Core Engine) 整合物理模拟、3D渲染和日志系统，使用JSON配置管理，依赖STL容器和第三方数学库实现跨模块数据交互：

![makefile_tutorial_02](https://raw.githubusercontent.com/Davidwadesmith/image-hosting/main/img/202504041720758.svg)

万一其中一个文件被改动，比如`math_utils.hpp`，全部编译将花费大量时间。而理想的状况是：

![makefile_tutorial_03](https://raw.githubusercontent.com/Davidwadesmith/image-hosting/main/img/202504041720878.svg)

*其中橙色是需要改动的文件*

因此，编写 makefile 的目的是省下一些写重复代码的工作，便于发现使用编译命令时的错误。

---

## 编写自己的第一个 Makefile

makefile 是如何减少重复的代码的呢？我们先来看看之前的工作流中有哪些重复的工作：

- 每次编译都需要输入几乎完全相同的多行命令

- 每次都需要弄清楚各个文件的依赖关系

- 每次都需要在文件系统中多次跳转

- ...

makefile是如何解决这些问题的呢？其实比较简单粗暴，它仅仅是一个自动化工具罢了，我们来看一个例子：

```makefile
#Makefile
all: 
	g++ -c Hello_world.cpp Print.cpp
	g++ Print.o Hello_world.o -o Hello_world
```

接下来在CLI中执行：

```bash
make all
```

你也许会看到：

```bash
❯ make all
g++ -c Hello_world.cpp Print.cpp
g++ Print.o Hello_world.o -o Hello_world
```

即使你完全不懂 makefile 的语法，你也大概能看出，```make```程序寻找当前目录下的```Makefile```文件并解读，还有```all```代表了它冒号后面的几行命令，以后只需要执行``make``这个程序时给它加上参数```all```便可以自动让它执行多个命令。这个以`all`开头的代码段被称为一种**规则**

发散一下，如果我们想执行不同的命令组合，可不可以通过给```make```不同的参数达到这一点呢？当然可以，我们先修改```Makefile```：

```makefile
#Makefile
all: 
	g++ -c Hello_world.cpp Print.cpp
	g++ Print.o Hello_world.o -o Hello_world
clean:
	rm -v ./Hello_world ./print.o ./Hello_world.o
```

接下来在CLI中执行：

```bash
make clean
```

你也许就能看到：

```bash
❯ make clean
rm -v ./Hello_world ./Print.o ./Hello_world.o
removed './Hello_world'
removed './Print.o'
removed './Hello_world.o'
```

*其中```-v```会让 rm 在删除时输出删除相关的信息*

于是，我们可以在 makefile 中写很多条指令，而只需要给它指定一个参数即可。

这样已经很实用了，可惜的是，这并不是 makefile 最原生的用法。

---

## 编写自己的第一个真正的  Makefile

makefile 在设计之初考虑到了代码编译的过程中形成了文件和文件之间的依赖。还记得那个```all```吗？其实```make```程序将其视为一个文件名，而```make```在当前目录中并没有找到```all```这个文件，所以```make```执行冒号之后的语句来生成这个文件。这也是 `make`如何执行的：首先执行`make`后面跟的参数（文件）对应的规则（没有参数那就是第一个规则），如果这个文件不存在就执行后面的语句，如果这个文件的依赖文件也不存在，那么就先寻找生成这个依赖文件的规则来先生成依赖文件（符合直觉的）。从这个观点来看，makefile 中冒号前的东西被称为目标文件，后面的语句是生成这个文件需要的命令。比方说：

```makefile
#Makefile
Hello_world.o : Hello_world.cpp
	g++ -c Hello_world.cpp
```

你会发现冒号后多了一个文件```Hello_world.cpp```，这个文件就是`Hello_world.o`的依赖文件，这就告诉```make```，如果`Hello_world.cpp`没有被更改（根据文件系统提供的时间戳判断）那么`Hello_world.o`也不需要更改，下面的语句可以不执行。这就省去了一部分编译时间。

## 伪目标 .Phony

既然知道了```all```被```make```程序视为一个文件，再看看`all`这个“文件名”，你可能会想，万一真有一个叫`all`的文件该怎么办，比如在之前的基础上：

```bash
touch all # 创建一个新文件叫做"all"
make all
```

你也许会看到：

```bash
❯ make all
make: 'all' is up to date.
```

这时候make发现`all`已经存在，而且没有依赖文件，那它就放心的不更新`all`也就不执行语句了。为了解决这个问题，makefile 引入了一个叫伪目标的概念，其实简单来看，就是 makefile 需要把`all`标记成一个永远都不存在的文件：

```makefile
#Makefile
.PHONY : all

all: 
	g++ -c Hello_world.cpp Print.cpp
	g++ Print.o Hello_world.o -o Hello_world
```

既然`all`永远不存在，它对应的语句就肯定能执行。这样的好处是，"目标文件" 可以不一定是文件，更像我们一开始以为的命令行参数（比如一眼便知其用途的 clean），这样就得到了更好的易读性 (readability).

---

## 变量

讲到这里你可能发现，makefile 的语法很像 shell 脚本，只不过 shell 更通用，而 makefile 对文件的依赖处理的更加好。任何一种脚本都有变量，makefile 也不例外。例如：

```makefile
SRC = Hello_world.cpp

all: $(SRC)
	g++ -c %(SRC)
```

*类似`constexpr`?*

你可以看到，这里调用变量的语法和`.sh`脚本中差不多。

既然有变量，那么没有返回变量的函数就说不过去了，makefile 中有一个名为`wildcard`的函数，其作用是将通配符 (wildcard) 展开，例如：

```makefile
#Makefile
SRC = $(wildcard *.c)
OBJS = $(patsubst %c, %o, $SRC)
TARGET = Hello_world

$TARGET: $(OBJS)
	g++ -o $(TARGET) $(OBJS)
```

我们先只看第二行，`SRC`通常是源代码 (source code) 的缩写。这里`wildcard`函数接收一个`*.c`作为参数，输出它展开后的字符串，也就是任意符合以`.c`结尾的文件名。再看第三行，`patsubst`又是一个函数，它的作用是替换字符串中的字符，它有三个参数，第一个是用来匹配原始字符串的，第二个是替换的目标，第三个就是原始字符串。例如第三行中`patsubst`的作用就是将所有`SRC`中的`.c`文件替换成同名的`.o`文件。当然它还有一种更为简单的形式：`OBJS = $(SRCS:.c=.o)` 写开来看就是：`$([原始字符串]:[匹配字串][替换字串])`

有时候，我们希望在执行语句内部指代目标文件，这时可以用**自动变量**来指代：

```makefile
#Makefile
Hello_world: Hello_world.o Print.o
	g++ -o $@ $^
```

这种写法比用普通变量来指代语义更加明确，同时也更加泛用。比如想要指代第一个依赖文件的话，可以用`@<`

---

## 模式规则

你刚才也许注意到，我们在没写生成`.o`的规则的情况下直接写了生成可执行文件的规则，这是因为 makefile 内置的一些模式匹配规则，当没有找到生成`.o`的规则时，就执行默认的规则，这个规则可能是这样的：

```makefile
#Makefile
%.o: %.c
	@echo 'compiling $<...'
	g++ -c -o $@ $<
```

*代码来自[廖雪峰的官方网站](https://liaoxuefeng.com/books/makefile/pattern-rules/index.html)*

这里`@echo`的意思是执行`echo`这个语句的时候不在CLI中显示这个语句，直接输出结果，即你的屏幕上不会显示`echo ...`。

看到这里的`%.c`和`%.o`，你可能想到了刚才`patsubst`的参数中也有这些写法，这就是 makefile 自己的模式匹配规则，完全类似`*.c`。通过这样的写法我们能够写出较少重复的代码，也就降低了出错的概率和补救的成本。

---

## 自动生成依赖

目前我们还无法解决一个问题，就是`.cpp`文件依赖`.h`的问题。因为这个依赖信息存在于`.cpp`文件的`#include`里，并不一定是同文件名的。如何才能分析文件里的内容呢？编译器非常方便的帮我们承担了这一职责，调用`g++ -MM Hello_world.cpp`将会输出一个 makefile 格式的依赖信息`Hello_world.o: Hello_world.cpp Print.h`。这就是`Hello_world.cpp`依赖的头文件的信息。我们想要把这条依赖规则添加到 makefile 中，可是当`make`运行的时候已经将`Makefile`占用了，我们不能在运行时添加这个信息，这又应该怎么办呢？解决方法是 makefile 语法中的`include`，字面意思，包含另一些文件给`make`参考，这些文件一般为`.d`文件，即依赖文件 (dependency) 。我们直接将`g++ -MM Hello_world.cpp`的输出写入`.d`文件，再在`Makefile`中添加一行`include ...`这样就能把头文件依赖问题解决了：

```makefile
# 列出所有 .c 文件:
SRCS = $(wildcard *.c)

# 根据SRCS生成 .o 文件列表:
OBJS = $(SRCS:.c=.o)

# 根据SRCS生成 .d 文件列表:
DEPS = $(SRCS:.c=.d)

TARGET = world.out

# 默认目标:
$(TARGET): $(OBJS)
	$(CC) -o $@ $^

# xyz.d 的规则由 xyz.c 生成:
%.d: %.c
	rm -f $@; \
	$(CC) -MM $< >$@.tmp; \
	sed 's,\($*\)\.o[ :]*,\1.o $@ : ,g' < $@.tmp > $@; \
	rm -f $@.tmp

# 模式规则:
%.o: %.c
	$(CC) -c -o $@ $<

clean:
	rm -rf *.o *.d $(TARGET)

# 引入所有 .d 文件:
include $(DEPS)
```

*用`;`分隔的语句会在同一个Shell环境中执行，不用的则不会。`\`和宏定义一样用来把一行语句拆成多行。*

*代码来自[廖雪峰的官方网站](https://liaoxuefeng.com/books/makefile/pattern-rules/index.html)*

这个代码中`.d`文件被`sed`（一个文字处理程序）修改了一下，将`.d`也作为了目标文件。这是因为当文件被改动时，重新编译之余，`.d`也要更新，否则要是改动把头文件改了，依赖文件就不对了。

---

## makefile 中的小技巧

想象一下`make`对上面`Makefile`的读取过程，其实分为两个阶段，一个是解析阶段，它会处理所有变量定义，规则解析，以及`include`。然后在执行阶段才会执行规则内的语句。这就导致一个问题，第一次编译时，`.d`文件还不存在，这下怎么包含呢？（虽然第一次编译肯定用不到`.d`，想想看，`.d`是为了侦测`.h`的改动而存在的）因此第一次编译会报错。这不会中断`make`程序，不过看起来不好看，这时我们可以把`include ...`换成`-include ...`来使这个报错沉默。这个技巧对规则内语句也适用。

---

## Github上的例子

如今我们很少有机会手搓 makefile 了，大部分工程也不会直接使用 makefile 。因此，我们只以 [makefile教程](https://github.com/chaselambda/makefiletutorial)为例：

```makefile
blah: blah.o
    cc blah.o -o blah # Runs third

blah.o: blah.c
    cc -c blah.c -o blah.o # Runs second

blah.c:
	echo "int main() { return 0; }" > blah.c # Runs first
```

<img src="https://raw.githubusercontent.com/Davidwadesmith/image-hosting/main/img/202504041729639.png" alt="image-20250404172903513" style="width: 20%; display: block; margin: 0 auto;" />

*其中表示箭头表示依赖关系*

这个图示已经足够清楚，一个十分简单的依赖关系。

---

### 结语

其实还有很多深层的 makefile 的语法没有在此涉及，但是希望你通过这篇文章能够理解 makefile 的设计思路，从而有足够的储备知识来在之后遇到陌生语法时不会一头雾水，而是能够仅从查阅的资料就能理解。

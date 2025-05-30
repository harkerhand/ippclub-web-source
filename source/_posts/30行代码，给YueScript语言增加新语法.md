---
title: 30 行代码，用 C++ 给 YueScript 语言增加新语法
date: 2025-05-30 18:00:00
tags:
- 'Dora SSR'
- 'YueScript'
categories:
- 编程语言
author: 李瑾
index_img: 30行代码，给YueScript语言增加新语法/封面.jpg
banner_img: 30行代码，给YueScript语言增加新语法/封面.jpg
---

&emsp;&emsp;如果你刚开始学习 C++ 并对编译器开发感兴趣，那么 YueScript 编译器项目可能是个很好的入门点。本文以一个简单的例子，教你如何为 YueScript 语言增加一个新的语法结构，并解释其中涉及的一些基础知识。

&emsp;&emsp;YueScript 的项目链接在这里：

- 官网：https://yuescript.org/zh
- GitHub：https://github.com/IppClub/YueScript
- Gitee：https://gitee.com/IppClub/YueScript

## YueScript 和转译器

&emsp;&emsp;YueScript 是一个将代码转换为 Lua 代码的转译器。简单地说，转译器就是将一种代码翻译成另一种代码的工具。目前 YueScript 的核心代码只有1万行左右的 C++ 代码。并且代码结构极为直白和简单。为了实现转译的功能，YueScript 用到了下面这些重要的概念和技术：

- **抽象语法树 (AST)**

	编译器首先会将源代码解析成树状的数据结构，树的节点代表代码的不同组成部分。

- **解析器组合 (Parser Combination)**

	YueScript 没有用传统的语法生成工具（如 yacc/bison），而是通过组合一系列小的解析器函数，来构造出完整的语言语法。这种技术叫做解析器组合，它的实现方式依赖 C++ 中对操作符（如 `>>`, `|` 等）的重载，使语法定义看起来就像在写一门迷你语言一样。比如 `key("if") >> space >> Expression` 这样的代码就可以表示一个 if 表达式的语法规则。

- **PEG 文法 (Parsing Expression Grammar)**

	这是 YueScript 所采用的语法描述模型。PEG 相较于传统的上下文无关文法（如 CFG）有几个显著优势，尤其适用于像 YueScript 这种语法糖丰富的语言。其一，PEG 具有“贪婪优先匹配”的特性，它总是选择第一个成功匹配的路径，从而消除了传统文法中的“二义性”问题；其二，PEG 支持无限长度的上下文判断，这使得它在处理一些上下文相关但又难以归入形式文法的语法糖时，特别有效。这种特性正好契合了 YueScript 所需的语法表达力，比如链式比较、嵌套赋值、匿名函数语法等结构的解析。

&emsp;&emsp;接下来，我们将具体看一下如何只用 30 行代码就给 YueScript 增加一个名为 `loop` 的新语法。

## 第一步：进行语法设计

&emsp;&emsp;首先我们先做一下新的语法设计，YueScript 已经支持了多种循环语句，比如：

- `while` 循环：带条件检查
- `for` ：基于范围和集合的循环
- `repeat ... until`：先执行，再根据条件结束

&emsp;&emsp;这些语法在 YueScript 都已经支持作为语句（statement）或表达式（expression）来使用，表达力是不错。

&emsp;&emsp;但在实际开发中，我们有时只需要一个永远执行下去，直到我们自己用 `break` 跳出的循环。虽然你可以用：

```lua
repeat
  ...
until false
```

&emsp;&emsp;或者：

```lua
while true
  ...
```

&emsp;&emsp;但是这些写法有时并不够直接，稍显有些“啰嗦”。由此，我们可以考虑加入一条更简洁的语法：

```lua
loop
  ...
```

&emsp;&emsp;然后让这条语句能被转译为：

```lua
repeat
  ...
until false
```

&emsp;&emsp;这样我们就得到一个写起来更加简洁、自然的永远运行的循环结构。

&emsp;&emsp;至此，语法设计已完成，那接下来我们就正式开工改代码来实现它吧。

## 第二步：观察 YueScript 项目的结构

&emsp;&emsp;首先我们简单了解一下 YueScript 项目的结构，浏览一下核心代码文件的部分：

```
src/yuescript
├── ast.cpp/hpp           // 基础抽象语法树（AST）实现
├── parser.cpp/hpp        // 基础解析器功能实现
├── yue_ast.cpp/h         // YueScript 具体语法的 AST 定义
├── yue_parser.cpp/h      // YueScript 的语法解析规则实现
├── yue_compiler.cpp/h    // 将 AST 编译成 Lua 代码的功能实现
├── yuescript.cpp/h       // 提供对外调用的接口
```

&emsp;&emsp;这些代码文件的功能关系可以简单地理解为：

&emsp;&emsp;用户代码 → `yue_parser` 解析为 AST → `yue_compiler` 编译为 Lua 代码。

## 第三步：定义新的 AST 节点类型

&emsp;&emsp;看好代码文件就可以开始动手了。首先，在 YueScript 中每一种语法结构都需要对应一个 AST 节点。因此，我们先要修改 `yue_ast.h` 文件。

```cpp
// 定义 loop 语法的 AST 节点类型
AST_NODE(Loop)
	ast_sel<true, Block_t, Statement_t> body;
	AST_MEMBER(Loop, &body)
AST_END(Loop)

// 修改已有的 Statement 节点，让它能包含 Loop 节点
AST_NODE(Statement)
	...
	ast_sel<true,
		Import_t, While_t, Repeat_t, For_t, ForEach_t,
		Return_t, Local_t, Global_t, Export_t, Macro_t, MacroInPlace_t,
		BreakLoop_t, Label_t, Goto_t, ShortTabAppending_t,
		Loop_t, // 在这里加入新定义的 Loop 节点
		Backcall_t, LocalAttrib_t, PipeBody_t, ExpListAssign_t, ChainAssign_t
	> content;
	...
AST_END(Statement)
```

&emsp;&emsp;可以看到新增的 Loop AST 节点可以包含一个 body 子节点，body 子节点里可以包含单条陈述语句或是一个代码块。目前定义的 AST 节点只是用来存储解析后生成的结果的数据结构。

## 第四步：创建新的语法解析规则

&emsp;&emsp;为了让解析器能够识别新的语法，我们需要修改 `yue_parser.h` 和 `yue_parser.cpp` 文件。

&emsp;&emsp;先在 `yue_parser.h` 中声明一个新规则：

```cpp
class YueParser {
	...
	// 添加 Loop 的语法规则声明
	AST_RULE(Loop);
	...
}
```

&emsp;&emsp;然后，在 `yue_parser.cpp` 中定义这个规则：

```cpp
YueParser::YueParser() {
	...
	// 使用 Parser Combination 的方式定义语法
	// key("loop") 表示匹配关键字 loop
	// space 表示匹配空白字符
	// in_block 和 Statement 表示匹配缩进块或单条语句
	Loop = key("loop") >> space >> (in_block | Statement);

    // 在原有的语句规则中添加对 Loop 的引用
	Statement =
		...
		space >> (
			Import | While | Repeat | For | ForEach |
			Return | Local | Global | Export | Macro |
			Loop | // 新增的 Loop 规则
			MacroInPlace | BreakLoop | Label | Goto | ShortTabAppending |
			LocalAttrib | Backcall | PipeBody | ExpListAssign | ChainAssign |
			StatementAppendix >> empty_block_error
		) >>
		...
	...
}
```

&emsp;&emsp;上述代码使用了 C++ 的重载操作符，如 `>>` 用来表示进行序列的匹配，和 `|` 用来表示进行可选语法的选择匹配。这样使得语法规则看起来更加直观易懂。可以看出我们的语法规则的定义和 AST 的定义是有一定的对应关系的，但是前者只是数据结构的定义，而后者是对做代码解析的处理函数做嵌套和组合。

## 第五步：实现语法转换为 Lua 代码的逻辑

&emsp;&emsp;接下来，我们要告诉 YueScript 如何把这个新语法转换成 Lua 代码。这一步涉及修改 `yue_compiler.cpp` 文件。

&emsp;&emsp;实现 loop 语法的转换逻辑，在这里我们可以偷个懒，直接复用已有的对 repeat 语法的转换，把 loop 语法的 AST 结构改造为 repeat 语法的 AST 再走转换流程：

```cpp
// 修改 YueCompiler 类，添加 Loop 语法的编译
void transformLoop(Loop_t* loopNode, str_list& out) {
	// 将 loop 转换为 repeat until false 循环结构
	auto x = loopNode;
	auto repeatNode = x->new_ptr<Repeat_t>();
	repeatNode->body.set(loopNode->body);
	repeatNode->condition.set(toAst<Exp_t>("false"s, x));
	transformRepeat(repeatNode, out);
}
```

再修改原有的 `transformStatement` 方法，增加对 Statement 的子节点，Loop 语法节点的处理：

```cpp
void transformStatement(Statement_t* statement, str_list& out) {
	...
		switch (content->get_id()) {
			...
			// 添加对 loop 语法的编译
			case id<Loop_t>(): transformLoop(static_cast<Loop_t*>(content), out); break;
			...
		}
	...
}
```

## 第六步：定义 AST 节点的格式化输出

&emsp;&emsp;因为 YueScript 语言的特殊设计，我们还需要为 AST 节点定义可以格式化为等价字符串代码的方法。在 `yue_ast.cpp` 中实现：

```cpp
std::string Loop_t::to_string(void* ud) const {
	auto info = reinterpret_cast<YueFormat*>(ud);
	if (body.is<Statement_t>()) {
		return "loop "s + body->to_string(ud);
	} else {
		str_list temp{"loop"s};
		info->pushScope();
		temp.emplace_back(body->to_string(ud));
		if (temp.back().empty()) {
			temp.back() = info->ind() + "--"s;
		}
		info->popScope();
		return join(temp, "\n"sv);
	}
}
```

## 第七步：测试你的新语法

&emsp;&emsp;最后，我们写几个示例代码，来确认新语法确实能用：

```lua
i = 1
loop
	i += 1
	print i
	break if i == 10

loop
	status = check_status!
	break if status == "done"
	sleep 0.5

loop break if success := try_connect! -- 测试 Yue 特有的陈述语句加 if 后缀的语法
loop! -- loop 目前未被定义为关键字，仍可当做函数调用
```

&emsp;&emsp;编译后得到的 Lua 代码，看起来没问题，完工：

```lua
local i = 1
repeat
	i = i + 1
	print(i)
	if i == 10 then
		break
	end
until false

repeat
	local status = check_status()
	if status == "done" then
		break
	end
	sleep(0.5)
until false

repeat
	local success = try_connect()
	if success then
		break
	end
until false

loop()
```

## 结语

&emsp;&emsp;这一条小小的 `loop` 语法背后，其实涵盖了语言设计、AST 构建、语法解析、代码生成等转译器的核心环节。

&emsp;&emsp;它是一个很适合初学者动手尝试的项目，不复杂，但足够完整。

&emsp;&emsp;如果你愿意继续深入，不妨试着来 YueScript 项目中动手尝试设计属于你自己的语法糖。你会发现，编译器开发也许并没有想象中那么遥不可及！非常欢迎你把自己发明的语法也 PR 到原项目里，让大家也一起试试看吧～


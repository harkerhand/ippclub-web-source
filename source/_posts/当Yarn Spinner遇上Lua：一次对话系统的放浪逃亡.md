---
title: 当Yarn Spinner遇上Lua：一次对话系统的放浪逃亡
date: 2025-03-11 18:53:36
tags:
- Lua
- 'Yarn Spinner'
categories:
- 游戏开发
author: 李瑾
index_img: ./当Yarn%20Spinner遇上Lua：一次对话系统的放浪逃亡/logo.png
banner_img: 
---

# 当 Yarn Spinner 遇上 Lua：一次对话系统的放浪逃亡

> *这是一场关于从 Unity 出逃，踏上 C++ & Lua 的新旅程……*

## 故事开始：当我想要一个好用的对话系统

&emsp;&emsp;在一个风和日丽的早晨，我正在做开源游戏项目《灵数奇缘》的开发，看着一堆对话选项，心想：

&emsp;&emsp;*“做这玩意儿难道要用 if-else 手撸吗？”*

&emsp;&emsp;*“不会吧不会吧，这年头居然没有一个好用的跨平台的对话系统？”*

&emsp;&emsp;于是我打开了搜索引擎，输入"开源 对话脚本 游戏"，并开始了一场游戏对话系统大比拼。



## 现有的方案不是被捆绑，就是太麻烦

&emsp;&emsp;好，来看看目前市场上的主流选手：

| **对话系统** | **语法简洁度** | **跨引擎能力** | **游戏逻辑交互** | **主要用途** | **主要支持的游戏引擎** |
|-------------|-------------|-------------|-------------|-------------|-------------|
| **Yarn Spinner** | +++++<br/>(Markdown 风) | 绑定 Unity | 需要手动绑定游戏逻辑，但自由度高 | 视觉小说、RPG、任务系统 | Unity |
| **Ink** | ++++<br/> (类似 Python 语法) | 依赖 Inkle runtime，不是纯解析 | 对游戏逻辑控制太强，影响自由度 | 互动小说、线性剧情 | Unity, Web |
| **Twine** | +++<br/>(基于 HTML 的可视化编辑) | 运行在 Web，可导出 JSON | 游戏逻辑绑定需要额外实现解析逻辑 | 交互式故事、Web 游戏 | Web, HTML5 |
| **Ren'Py** | +++<br/>(基于 Python 语言本身) | 只能运行在 Ren'Py | 只能在 Ren'Py 里操作游戏逻辑 | 视觉小说 | 仅限 Ren'Py |
| **Fungus (Unity)** | ++<br/>(可视化+脚本) | 绑定 Unity | Unity 事件系统管理 | 视觉小说、RPG 对话系统 | Unity |
| **Articy Draft** | +<br/> (AAA 级别，复杂) | 可导出 JSON，但需要自己写解析 | 提供状态管理，但与游戏逻辑耦合高 | AAA 级叙事、复杂游戏剧情 | Unity, Unreal |



## 开始比拼：谁能逃出 Unity 及其它特定生态？

### 第一轮淘汰：那些绑定引擎的对话系统

- **Ren'Py**："你必须用我自己的引擎！"（嗯，没得选，Pass）
- **Fungus**："我是 Unity 的一部分！"（Unity 才是 Fungus 的主体，而不是它自己，Pass）
- **Twine**："我是 Web 交互故事的标准！"（太偏 Web，Pass）
- **Yarn Spinner**："为行业势力 Unity 低头！"（虽然语法很香，但官方实现绑定 C#，Pass！）

&emsp;&emsp;这些方案都对 **特定引擎依赖太重**，直接出局！

------

### 第二轮筛选：那些功能集太大，干涉游戏逻辑的方案

- **Ink**：语法优雅、功能丰富，但是……
	- 需要 **Ink Runtime**，必须在它的框架内运行（自带 State Machine）。
	- 对话流程控制太强，会接管变量管理，和游戏逻辑难以分离。

- **Articy Draft**：AAA 级别的对话系统，太复杂了，感觉在填写 Excel 报表。

&emsp;&emsp;*我不想要一个庞大的剧情和游戏演出管理器，我只要一个干净的纯文字的对话脚本解析器口牙！*

&emsp;&emsp;这些方案对游戏逻辑开发的影响过大，也出局！

&emsp;&emsp;那要咋办，全部都淘汰了呢……



### 自己动手，丰衣足食！

&emsp;&emsp;经历了一番市场考察后，可以总结：

&emsp;&emsp;大多数对话系统要么绑定特定引擎，要么需要额外复杂的 runtime。

&emsp;&emsp;许多方案对游戏逻辑的干涉太强，让游戏开发变得更受限，而不是更自由。（这个功能是放在对话系统里完成能力又不够，放在游戏逻辑脚本里写似乎又绕开了对话系统功能……）

&emsp;&emsp;几乎没有一个真正跨平台、可以任意嵌入多种引擎运行环境的解决方案。

&emsp;&emsp;Yarn Spinner 算是最符合我对 IT 工程的认知和对描述性语言的审美的方案了，它的优势有：

&emsp;&emsp;**超低耦合**：不自带复杂的游戏功能框架，不会插手你的游戏逻辑。

&emsp;&emsp;**语法简单**：Markdown 风格，适合编剧直接上手。

&emsp;&emsp;**完全可以脱离游戏引擎运行**：因为几乎不包含游戏相关的功能，它可以被实现为一个纯粹的对话解析器 + 简单的运行系统就独立运行。

&emsp;&emsp;这就意味着 Yarn Spinner 有潜力可以用在任何地方，你可以用它做视觉小说，你可以用它做 RPG 任务对话，甚至你可以用它做 AI 聊天机器人！

&emsp;&emsp;这里来看看 Yarn Spinner 的语法示例，核心语言功能就是**分支**、**变量**、**命令**和**标记**：

```html
title: Start
---
// 声明变量，默认值为"玩家"
<<set $player_name to "玩家">>
// 声明变量，默认值为0
<<set $coins to 0>>

// 对话开始
叙述者: 你好，冒险者！请告诉我你的名字。

-> 艾莉
	// 设置变量，模拟玩家输入名字
	<<set $player_name to "艾莉">>
-> 哈克
	<<set $player_name to "哈克">>

叙述者: 哦，你的名字是{$player_name}，对吧？

<<if $coins == 0>>
	叙述者: 看起来你还没有金币。让我给你一些吧！
	// 更新变量值
	<<set $coins = 10>>
	// 使用标记，给程序提供信息用于做不同的文字处理
	叙述者: 现在你有 [color=red]{$coins}[/color] 个金币了！
<<else>>
	叙述者: 你已经有一些金币了！
<<endif>>

// 跳转到 Adventure 节点
<<jump Adventure>>
===

title: Adventure
---
叙述者: 让我们开始冒险吧！
===
```

&emsp;&emsp; *"就是你了！Yarn Spinner！"*

&emsp;&emsp;但是……Yarn Spinner 的官方实现是 C#，仅支持 Unity，所以我决定自己动手，移植到 C++ 和 Lua 上。



## YarnFlow：从 Yarn Spinner 到 Lua 生态的完美移植

&emsp;&emsp;YarnFlow 是什么？简单说，它是我们开发的： 一个 C++ 编写的 Yarn 编译器（把 `.yarn` 代码变成 Lua 可执行脚本）；一个 Lua 运行时，可以直接在游戏逻辑里调用 Yarn 变量、触发游戏事件； 也是一个超自由、超可移植的对话解决方案，你永远可以相信 C++ 和 Lua 稳定的跨平台能力！



### 为什么要用 C++？

&emsp;&emsp;因为我们做过的项目 YueScript 已经验证了 **C++ Parser Combinator** 的有效性！

&emsp;&emsp;在 YueScript 语言的开发过程中，我们探索过一种高效的 C++ Parser Combinator 方案，可以在性能、静态类型安全性和可维护性之间找到良好的平衡。

&emsp;&emsp;我们发现：

- **性能足够高**：C++ 在解析 Yarn 语法时比动态语言快得多，适合用来编译 Yarn 代码。
- **静态类型检查帮助维护**：相比动态语言实现（比如也使用 Lua 语言），C++ 编译器层面的静态类型检查让代码更稳定、更容易扩展。
- **YueScript 的经验可复用**：我们已经在 YueScript 项目中成功构建了一个基于 C++ 的编译器，可以直接借鉴相关技术来解析 Yarn 语法并优化编译流程。

&emsp;&emsp;所以，选择 C++ 作为 YarnFlow 的编译器实现语言，完全是有先例支撑的，而不是在盲目追求"高性能"！

&emsp;&emsp;下面节选部分 YarnFlow 编译器的 C++ 代码，可以看到通过对操作符的重载，对 Yarn Spinner 语法的描述同时也是解析器实现代码的本身，接下来直接编译为程序即可，也不需要借助第三方的工具增加生成解析器代码的构建步骤。

```cpp
// 部分构建解析器的代码
File = -Block >> white >> stop;
Block = Seperator >> line >> *(+line_break >> line);
line = (
    empty_line_break |
    check_indent >> (Command | OptionGroup | Dialog) |
    advance_match >> ensure(space >> (OptionGroup | indentation_error), pop_indent)
);
```

&emsp;&emsp;对于每一个语法规则都有对应的抽象语法树节点（AST）的静态类型，不管在运行时检查还是静态检查的辅助力都用上。未来改了语法，对应的 AST 处理部分都会及时报错，让维护者找到问题源及时调整。

```cpp
// 处理 AST 节点的代码节选，用静态类型检查减少人手工编写代码的失误
// 这样当语法规则和 AST 处理不再匹配，就会在编译器和运行时都提供错误信息
void transformAttributeValue(ast_node* value, str_list& out) {
    switch (value->get_id()) {
        case id<Value_t>(): transformValue(ast_to<Value_t>(value), out); break;
        case id<String_t>(): transformString(ast_to<String_t>(value), out); break;
        case id<AttributeValue_t>(): out.push_back(toLuaString(_parser.toString(value))); break;
        default: YUEE("AST node mismatch", value); break;
    }
}
```



### 为什么运行时要用 Lua？

&emsp;&emsp;因为 Lua 是游戏引擎里最常见的脚本语言！

&emsp;&emsp;因为 Lua 轻量、节省内存，就算单独嵌入也不会占多少资源！

&emsp;&emsp;Lua 在游戏开发中的优势：

- **几乎所有主流游戏引擎都支持 Lua**（包括 Love2D、Godot、Cocos2d-x，甚至一些 Unreal 和 Unity 项目也用 Lua 作为扩展脚本）。
- **极低的运行时开销**，内存占用极小，非常适合游戏开发。
- **即便原来的游戏引擎不支持 Lua，也可以很低成本地加上一个 Lua 解释器**，而不会显著增加二进制大小或内存消耗。

&emsp;&emsp;换句话说，哪怕你用的游戏引擎本来不支持 Lua，你也可以只为了用 YarnFlow 而加一个 Lua 运行环境，而不会让游戏包体积爆炸！



### C++ 和 Lua 的无缝对接

&emsp;&emsp;有点好奇 C++ 编译器是怎么和 Lua 语言无缝对接的吗？

&emsp;&emsp;我们可以把 C++ 想象成一个翻译官，它的任务是： 读取 Yarn Spinner 脚本 → 翻译成 Lua 代码 → 在游戏环境里运行

&emsp;&emsp;这样一来，Yarn 脚本就能直接在 Lua 环境下执行，不需要额外的跨语言的开销！

&emsp;&emsp;但是，仅仅能“跑起来”还不够，我们还需要一个可以顺畅交互的对话流程管理系统。这时候，**Lua 协程** 就派上用场了！

&emsp;&emsp;Lua 有一个语言的特性非常棒，那就是 **协程（Coroutine）**，简单来说就是可以随时 **暂停 & 继续执行的代码块**。对于 Lua 语言的协程你可以想象为一个有存档点管理的游戏，就像是：

&emsp;&emsp;**存档** → 让游戏储存进度，然后退出游戏。（储存程序状态，中止运行）

&emsp;&emsp;**读档** → 玩家重进游戏，从存档位置继续游戏。（恢复程序状态，从上个位置继续运行）

&emsp;&emsp;这个功能和对话系统的交互过程有着异曲同工之妙。比如我们有一段 Yarn Spinner 的脚本是这样：

```html
title: Yarn教学
---
Yarn教学: 例如，这里有一些选项供你选择！
<<set $choice to 0>>
-> 哇，有选项可以选！
	<<set $choice to 1>>
	Yarn教学: 选的好，朋友！
-> 不错。
	<<fade 1.0>>
===
```

&emsp;&emsp;由 YarnFlow 编译生成的 Lua 代码如下：

```lua
return function(title, state, command, yarn, gotoStory)
	-- 先暂停协程，返回对话文本，等待游戏主逻辑处理文本显示
	coroutine.yield("Dialog", {
		text = "例如，这里有一些选项供你选择！",
		title = title,
		marks = { -- 提供对话相关额外信息的标记数据
			{
				name = "Character",
				attrs = { ["name"] = "Yarn教学" },
			},
		},
	})

	-- 继续执行初始化变量
	state["choice"] = 0

	-- 暂停协程并返回选项处理
	coroutine.yield("Option", {
		{ text = "哇，有选项可以选！", title = title },
		{ text = "不错。", title = title },
	},
	{
		function() -- 可以继续执行分支对话的函数
			state["choice"] = 1
			coroutine.yield("Dialog", {
				text = "选的好，朋友！",
				title = title,
				marks = {
					{
						name = "Character",
						attrs = { ["name"] = "Yarn教学" },
					},
				},
			})
			return nil
		end,
		function() -- 执行另一个分支的函数
			command["fade"](1.0) -- 这里执行的是游戏逻辑
			return nil
		end,
	})

	return nil
end
```

&emsp;&emsp;有些 Get 到了吗？这段 Lua 代码完美地利用了协程（`coroutine.yield`）来实现对话流程的暂停和恢复。

&emsp;&emsp;每当需要等待玩家输入或者游戏事件时，对话程序就会暂停，等待游戏逻辑告诉它“可以继续了”再做继续。当操作变量时，会通过访问用户提供的 `state` 表来实现。当执行指令时，会通过提供的 `command` 表来执行。

&emsp;&emsp;这种设计让 YarnFlow 可以融入任何游戏引擎的主循环，同时不需要复杂的状态管理，协程自己就记住了执行到哪里。

&emsp;&emsp;并且实现让游戏逻辑和对话流程彻底分离，互不干扰，只通过用户定义的 state 和 comand 表来交互，同时用户也可以通过这两个自定义的接口来实现比如变量的持久化，对其他游戏系统的调用等。



### 上手使用

&emsp;&emsp;YarnFlow 已经作为一个开源软件仓库发布，可以直接本地编译为 Lua 的链接库，也可以通过 `luarocks`，通过 Lua 生态的包管理系统进行下载。有动手能力的也可以直接使用源码整合到你自己开发的引擎中提供功能。

&emsp;&emsp;仓库地址在这里：https://github.com/IppClub/YarnFlow

&emsp;&emsp;国内地址：https://atomgit.com/IppClub/YarnFlow



## Dora Story：简单点，让视觉小说开发再简单点

![demo](当Yarn%20Spinner遇上Lua：一次对话系统的放浪逃亡/demo.jpg)

&emsp;&emsp;其实在探索跨平台 Yarn Spinner 方案的过程中，我们不仅开发了 **YarnFlow**，还将它与 **Dora SSR 开源游戏引擎** 结合，打造了一个相对独立完整的视觉小说开发框架 —— **Dora Story**。

&emsp;&emsp;Dora Story 让你不依赖 Unity 也能用 Yarn Spinner 编写对话剧情，并同样以完全开源的方式增强的游戏交互功能。剧情分支、动态对话、场景音乐、角色演出？全部安排！



### 上手只要三步

1. **安装 Dora SSR：**
	下载 Dora SSR，打开浏览器。

2. **获取 Dora Story：**
	克隆或下载仓库代码，直接上传到 **Web IDE**。

3. **启动测试：**
	点击 **“启动游戏”**，立刻测试你的专属游戏！



### 强化 Yarn 扩展，让故事更自由！

&emsp;&emsp;在 **Dora Story** 里，我们基于 **YarnFlow** 提供了一系列 Yarn 扩展指令，让视觉小说的表现力更强！

#### 角色 & 场景演出

- **角色立绘登场**：
	`<<figure "Image/ch_001.png", 0, 200, 1.5>>`：控制角色位置、大小、透明度，随心调节演出效果！
- **切换背景图**：
	`<<background "Image/bg_001.png", true>>`：支持背景模糊，营造氛围！
- **使用《灵数奇缘》角色**：
	`[char id=moling name=默翎]`：直接通过特殊标记调用角色数据，让 NPC 美术更加精美！

#### 音乐 & 音效控制

- `<<BGM "Music/track.mp3">>`：播放背景音乐
- `<<stopBGM>>`：停止音乐
- `<<SE "Sound/door_knock.wav">>`：播放音效

#### 动态输入 & 游戏存档

- **输入玩家姓名**：
	`<<inputName>>`：让玩家输入自己的名字，剧情互动更沉浸！
- **变量自动存储**：
	`<<set $gold to 100>>`：Yarn 变量自动持久化到数据库，下次进入游戏时数据不会丢失！

#### 章节管理 & 资源优化

- `<<chapter "next_chapter.yarn">>`：自动切换章节，剧情推进更顺畅！

- `<<preload "Image/bg_001.png">>`：预加载资源，减少卡顿！

	

### 使用 Dora SSR 开发 Yarn Spinner 对话

&emsp;&emsp;Dora SSR 是一个开源的轻量级游戏引擎，它不仅支持多种游戏开发语言，还内置了丰富的游戏开发工具和资源。如辅助进行 Yarn 脚本编写和调试的编辑器，可以用可视化的节点来预览和管理复杂的故事线。上手试试就知道有多香了！

![Yarn Editor](当Yarn%20Spinner遇上Lua：一次对话系统的放浪逃亡/yarn-editor.jpg)



### 开源资源

&emsp;&emsp;在我们的 Dora Story 项目中也使用了 **开源游戏《灵数奇缘》** 的开源代码和授权资源**，**并采用 MIT 许可协议，开发者可**自由商用、拓展和贡献**！

&emsp;&emsp;仓库地址：https://github.com/IppClub/Dora-Story

&emsp;&emsp;国内地址：https://atomgit.com/IppClub/Dora-Story

&emsp;&emsp;从此，不用 Unity 也能写 Yarn 剧情，视觉小说开发也更自由啦！今天的故事就讲到这里，拜拜～
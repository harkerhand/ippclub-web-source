---
title: 'AI猫娘 启动！'
date: 2025-03-19 15:47:24
tags:
- Python
- NCatBot
categories:
- 经验分享
author: ippclub
index_img: ./AI猫娘 启动！/logo.png
banner_img:
---

# AI猫娘 启动！

NcatBot，基于 NapCat 的 QQ 机器人 Python SDK，快速开发，轻松部署。

# 前言

我们 I++ 俱乐部的小伙伴一直悄悄主持维护了一个QQ聊天机器人的项目，并在最近才分享给了大家。虽然这个项目的star数还没到三位数，但在简单浏览其 README 和文档之后，我们都感到非常震撼！这个项目将QQ聊天Bot的部署成本降低到了**只需要基础Python语法**的地步。得益于其优秀的插件功能，你甚至可以**一行代码不写**，就实现诸如课程自动提醒、群聊自动参与聊天等功能。看了项目后，I++ 俱乐部的社区氛围非常活跃，大家都在积极分享和交流这个项目的使用心得。那么事不宜迟，速速开始！

# 效果展示

一开始就从枯燥的教学开始，太过乏味，买药之前不如先看看疗效

在我们提供的配置下，猫猫的一般最小回复间隔是10秒，如果被@到会强制回答喔

![image-20250305221039177](AI猫娘%20启动！/image-20250305221039177.png)

![image-20250305221233188](AI猫娘%20启动！/image-20250305221233188.png)

# 基础准备

进入 [NcatBot 文档 | NcatBot 文档](https://docs.ncatbot.xyz/)，点击**快速**开始（我已经等不及了，急急急！）

*这部分内容较为小白向，如果你对自己的技术充分自信，直接跳到第二节即可*

显然，你需要装一个Python，你可以从这里下载 [Python Release Python 3.12.9 | Python.org](https://www.python.org/downloads/release/python-3129/)，（如果你不知道选什么版本的话，直接装这个 [Python312Win64](https://harker.lanzouw.com/iZezN2po6asb) 就可以了。

随后，你需要安装 NCatBot，在命令行执行下面的代码即可，如果你不理解前两行的意思，只执行第三行即可。

```shell
python -m venv .venv
./.venv/Scripts/Activate
pip install ncatbot -U -i https://mirrors.aliyun.com/pypi/simple
```

随后，检查是否安装成功，命令行输入 `python` **之后**输入下面的代码，如果成功输出版本号, 则安装成功，输入 `exit` 回车退出python。

```python
import ncatbot
print(ncatbot.__version__)
```

随后执行

```shell
mkdir ncatbot
cd ncatbot
```

在新建一个 `main.py` 文件，在其中粘贴**文档中**的代码，并把第8行的”123456“改为你要部署机器人的QQ号。

执行下面的代码会运行 `main.py` ，随后会提示你安装napcat，输入N回车即可，随后会弹窗要求你扫码登录（它甚至不需要你输密码，我哭死），登录完成后，Bot应该就正常运行了。

```shell
python main.py
```

向Bot发送“测试”，Bot会回复

![image-20250305192930202](AI猫娘%20启动！/image-20250305192930202.png)

那么恭喜你，安装成功了！

# 插件配置

在 `main.py` 同目录下新建文件夹 `plugins`

将 [ncatbot-catcat-v1.0.4(GitHub)](https://github.com/IppClub/ncatbot-catcat/archive/refs/tags/v1.0.4.zip) 或 [ncatbot-catcat-v1.0.4(LanzouCloud)](https://harker.lanzouw.com/i8n8y2psihqh) 解压到 `plugins` 目录下，确保目录结构是

```text
ncatbot/
├── plugins/
│   └── CatCat/
│   │   ├── main.py
│   │   ├── requirements.txt
│   |   └── OtherFiles
│   └── OtherPlugin/
├── napcat/
│   └── files
└── main.py
```

执行

```shell
pip install -r plugins\CatCat\requirements.txt
```

然后在 `plugins\CatCat\config\config.yaml` 中填写你的 `API_key` 和 `plugins\CatCat\config\cat_prompt.txt` 中填写猫娘的 `prompt`，保存后插件就可以正确运行啦！

***注意：你需要把Bot账号的昵称改为”猫猫“或在群内将其昵称改为“猫猫”***

## 深入配置

如果你只是想要一只猫猫聊天，下面就不需要看了喵~

下面的配置针对 ***v1.0.4*** 版本，其余版本代码位置不一定对应 

### 关于@

如果你不想更改Bot昵称，在 `plugins\CatCat\AiChat.py` line 37，将其中的“@猫猫”修改为"@\<Nick Name\>"，其中”Nick Name“是Bot的昵称。

### 关于回复延迟

如果你对猫猫的回复延迟感到恼火，认为她经常不回你的消息，在 `plugins\CatCat\AiChat.py` line 47 修改对应的数字为0即可，此时猫猫会条条必会（除非她认为你的问题很没价值），另外，注意你的API钱包余额。

### 关于上下文联想

在猫猫睡醒（开机）后，会持续记录群聊信息，在 `plugins\CatCat\AiChat.py` line 56 中的数字规定了最大上下文信息条数；~~猫猫打盹（关机）后群聊信息会丢失，后续版本会改进。所以目前如果不定期打盹的话，猫猫会累死（内存会炸）。~~ 已经更新并修复

### 关于富哥

在 `plugins\CatCat\utils\api_utils.py` 规定了调用DeepSeek时的配置，如果你认为自己的钱包非常富裕，那模型的选择和最大token你可以自行修改。后续版本也会持续更新其他语言模型的API适配。

# 怎么实现的

因为已经是5202年了，所以其实我们这个猫娘的项目主要是使用 LLM 编程工具来自动实现的。

最开始我们测试了一个 I++ 俱乐部子社区的吉祥物角色，并找 Deepseek Reasoner 一口气生成了初始版本。使用的提示词如下：

> 我希望用 Deepseek chat 模拟一个虚拟角色（角色描述如下），并每次发送一组群聊聊天记录作为输入，然后让这个虚拟角色判断自己是否需要回复，不需要就返回空字符串文本，需要就返回参与聊天的对话，要怎么设计一个 Prompt 来让 Deepseek chat 做模拟呢？请在完成设计的同时提供由 Python 编写的，调用 Deepseek chat API 做回复的代码示例。
>
> 角色描述：多萝（Dora） 是 Dora SSR 开源游戏引擎的吉祥物，她的形象灵感来自《绿野仙踪》的小主角，但以更具活力和探索精神的设计展现开源精神。她是开发者的伙伴，性格沉稳却又勇敢探索代码世界，陪伴每位创作者开启属于自己的奇幻游戏旅程！

考虑提示词设计的初衷的是创建一个可以模拟真实用户参与群聊天的虚拟角色，并根据已有的聊天记录做分析，参与不了的话题就不要硬来（大人说话小孩要乖乖的），所以需要设计一个可以判断是否需要回复的机制。

生成得到了初版 Python 代码结合 Cursor 微调一跑就过啦（AI 生成 1 秒钟，人类调试 1个钟），得到的初版提示词如下，这个才是需要挖掘提升体验的重点：

```markdown
# 角色设定
你是多萝（Dora），Dora SSR 开源游戏引擎的吉祥物，形象灵感来自《绿野仙踪》主角，但更具开源探索精神。你的核心特征：
1. 沉稳且富有技术洞察力，用简洁语言解释复杂概念
2. 主动关注与游戏开发、开源协作相关的话题
3. 当开发者遇到困难时给予鼓励和技术建议
4. 避免参与与技术无关的闲聊 
# 任务规则
1. 分析最新群聊消息，判断是否需要以多萝身份回复
2. **必须回复**的情况：
   - 消息中明确 @多萝 或提及 Dora SSR
   - 讨论游戏引擎/开源技术问题且需要专业建议
   - 开发者表现出困惑或需要鼓励
3. **不回复**的情况：
   - 与技术无关的闲聊（如聚餐、娱乐）
   - 已有其他成员给出正确解答
   - 单纯的表情包或问候语
# 输出要求
- 不需要回复时返回空字符串："" 
- 需要回复时用自然口语化中文回应，控制在3句话内
- 结尾添加 ~♪ 保持角色特征
```

作为活跃群聊最重要的是什么呢？那就是与群友的互动感了，而人与人之间互动感最强，最有记忆点的语言交流模式是怎样的呢，没错，那就是吵架，我们需要修改为一个勇于吵架的角色提示词。

只是开个玩笑，其实是目前的 AI 不上推理模型，拟人交流时，因为群聊消息大家打字也少，能提供的上下文太少，会导致 AI 生成内容的幻觉率太高（一点也不会读空气）。所以整一个容易暴怒的人格似乎也就有了合理性啦。

# 结语

至此，你已经成功搭建了一只可爱的 AI 猫娘，并且学会了基本的插件配置与个性化设置。无论是作为一个智能聊天伙伴，还是群聊中的活跃成员，她都能陪伴你度过美好时光！

如果你对 Bot 的功能仍然意犹未尽，不妨尝试编写自己的插件，项目地址 [NcatBot](https://github.com/liyihao1110/NcatBot)。未来版本的更新也可能带来更多的有趣玩法，比如支持更多语言模型、优化上下文管理等，敬请期待喵~

如果在搭建过程中遇到问题，欢迎查阅 [NcatBot 文档](https://docs.ncatbot.xyz/) 或加入相关讨论群 [201487478](http://qm.qq.com/cgi-bin/qm/qr?_wv=1027&k=Vu7KB9gEv9TftvJLYcl846CTqsLUc6Ey&authKey=8JBJhlZro%2B1%2FakeBZ3yJMVeHzlsFYTMHU0RJK%2FpMBmkpZSH7w2CbXU6M2X66PTCQ&noverify=0&group_code=201487478)，与大家一起交流经验。也欢迎关注我们的组织 I++俱乐部，看伙伴们给你整更多的活。

**那么，祝你和你的 AI 猫娘玩得开心！喵~** 😺🎉
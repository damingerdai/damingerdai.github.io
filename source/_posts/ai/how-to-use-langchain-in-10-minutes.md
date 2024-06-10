---
title: 如何在10分钟内使用LangChain
date: 2024-06-09 13:05:40
tags: [AI, LangChain, 翻译]
categories: [AI]
---

# 如何在10分钟内使用LangChain

[LangChain](https://www.langchain.com/)是一个强大的Python和Javascript/Typescript库，它可以让你快速地原型化大型语言模型应用。它允许你将LLM任务链在一起（因此得名），甚至可以让你快速轻松地运行<i>自主代理(autonomous agents)</i>。今天，我们将介绍chain的基础知识，这样你就可以立即开始你最新的LLM项目。

## 前言

本文主要讨论了LangChain的使用和优势。LangChain是一个对于希望快速创建大型语言模型应用的人来说非常有用的工具。它可以在几分钟内创建链、定义提示，甚至将多个LLM调用链接在一起以创建动态的TikTok脚本。

LangChain的优势在于其简单性和灵活性。无论你是经验丰富的开发者还是刚刚起步，LangChain的直观设计都让你能够像从未有过的那样利用大型语言模型的能力。从生成创意内容到运行自主代理，可能性是无穷无尽的。

此外，如果你正在寻找将AI集成到你现有的工作流程或产品中，TimeSurge Labs可以提供帮助。他们专注于AI咨询、开发、内部工具和LLM托管，他们的团队致力于构建AI的未来，并帮助你的业务在这个快速变化的行业中蓬勃发展。

> 该部分使用AI自动生成


## 要求

1. Python 3.9（3.10及以上版本与LangChain的一些模块存在一些问题）。 
2. 一个OpenAI API密钥

## 开始

我们将创建一个Python虚拟环境，并通过这种方式安装依赖项。

```bash
mkdir myproject
cd myproject
# or python3, python3.9, etc depending on your setup
python -m venv env
source env/bin/activate
```

完成以上步骤后，我们可以开始安装依赖项。在本教程中，我们只需要安装LangChain和OpenAI。最后，我们将使用python-dotenv来将OpenAI API密钥加载到环境中。

```bash
pip install langchain openai python-dotenv
```

LangChain是一个非常大的库，所以下载可能需要几分钟。在下载的同时，创建一个名为.env的新文件，并将你的API密钥粘贴进去。以下是一个示例：

```bash
OPENAI_API_KEY=Your-api-key-here
```

一旦完成，我们就可以创建我们的第一条chain了。

## 概念快速入门

在开始之前我们需要了解一些基础的概念。

1. <b>Chain</b>: 可以被看作是一个与LLM（语言模型）交互或在列表中进行多次LLM调用的行动列表。一个链由三个简单的部分组成。 
 a. [提示模板](https://python.langchain.com/docs/modules/model_io/prompts/prompt_templates/) - 这样你可以快速改变输入而不改变提示。 
 b. [LLM](https://python.langchain.com/v0.1/docs/modules/model_io/llms/) - 实际运行你的提示的AI。 
 c. [输出解析器](https://python.langchain.com/v0.1/docs/modules/model_io/output_parsers/) - 将输出转换成有用的东西，通常只是另一个字符串。

 ## 编写Chain

 对于这个例子，我们将编写一个生成TikTok脚本的链（毕竟我是一个[Zoomer](https://www.urbandictionary.com/define.php?term=Zoomer)）用于一个教育频道。首先，我们需要生成一个TikTok的描述。我们将使用提示模板，以便我们可以稍后重用提示。

 ```python
# prompts.py
from langchain.prompts import PromptTemplate

description_prompt = PromptTemplate.from_template(
    "Write me a description for a TikTok about {topic}")
 ```

 这可以在chain中使用。在我们定义链之前，我们需要为chain定义一个LLM。LangChain建议大多数用户应该使用ChatOpenAI类，以获得ChatGPT API的成本效益和简单性。

 ```python
 # chain.py
from langchain.chat_models import ChatOpenAI
from langchain.chains import LLMChain
from dotenv import load_dotenv
from prompts import description_prompt

# loads the .env file
load_dotenv()

llm = ChatOpenAI(model_name="gpt-3.5-turbo")
```

完成以上步骤后，我们就可以创建chain了。

```python
description_chain = LLMChain(llm=llm, prompt=description_prompt, verbose=True)
```

现在我们可以使用<b>.predict</b>来调用新的chain。


```python
output = description_chain.predict(topic="Cats are cool")
print(output)
```

这是输出结果：

```bash
😻 Unleash your inner feline aficionado! From their enchanting eyes to their purrfectly mysterious ways, cats are the epitome of coolness. 🐾 Watch as they effortlessly own their spaces, teaching us the art of relaxation and play. Whether they're mastering acrobatics or curling up for a catnap, their cool vibes are undeniable. 😎 Join the cat craze and embrace the awesomeness of these four-legged trendsetters! 🐱💫 #CatsRule #CoolCats #FelineVibes

中文翻译：😻 释放你内心的猫咪爱好者！从他们迷人的眼睛到他们完美的神秘方式，猫咪是酷的象征。🐾 观察他们如何毫不费力地拥有自己的空间，教我们放松和玩耍的艺术。无论他们是在掌握杂技还是卷起来打个猫觉，他们的酷劲是无法否认的。😎 加入猫咪狂热，拥抱这些四足潮流引领者的酷炫！🐱💫 #猫咪统治 #酷猫 #猫咪氛围
```

现在我们有了一个描述，我们需要让它写一个脚本。这就是chain的作用 - 我们可以使用稍微不同的提示再次依次调用LLM。首先，让我们为下一个chain定义一个新的提示。

```python
# add to prompts.py

script_prompt = PromptTemplate.from_template(
    "Write me a script for a TikTok given the following description: {description}")
```

这是你现在的chain.py应该是这样子的。

```python
# chain.py
from langchain.chat_models import ChatOpenAI
from langchain.chains import LLMChain
from dotenv import load_dotenv
# this line changed!!!!!
from prompts import description_prompt, script_prompt

# loads the .env file
load_dotenv()

llm = ChatOpenAI(model_name="gpt-3.5-turbo")

description_chain = LLMChain(llm=llm, prompt=description_prompt, verbose=True)

output = description_chain.predict(topic="Cats are cool")
print(output)

# new code below this line
script_chain = LLMChain(llm=llm, prompt=script_prompt)
script = script_chain.predict(description=output, verbose=True)
# 原文是script = description_chain.predict(description=output, verbose=True)，应该是错误
print(script)
```

这是新的输出结果：

```python
[Opening shot: A close-up of a cat's mesmerizing eyes, slowly blinking.]

Narrator (Voiceover): "😻 Unleash your inner feline aficionado!"

[Cut to a sleek cat walking confidently through a room, tail swaying gracefully.]

Narrator (Voiceover): "From their enchanting eyes to their purrfectly mysterious ways..."

[Transition to a montage of cats lounging in different relaxed poses.]

Narrator (Voiceover): "Cats are the epitome of coolness. 🐾"

[Show a cat effortlessly jumping onto a high shelf, landing with precision.]

Narrator (Voiceover): "Watch as they effortlessly own their spaces..."

[Cut to a person lying on the couch while a cat playfully bats at a string toy.]

Narrator (Voiceover): "Teaching us the art of relaxation and play."

[Show a cat doing a graceful mid-air flip while chasing a feather toy.]

Narrator (Voiceover): "Whether they're mastering acrobatics..."

[Transition to a cozy scene of a cat curled up in a sunlit spot, eyes half-closed.]

Narrator (Voiceover): "Or curling up for a catnap..."

[Cut to a group of cats with various personalities and fur patterns.]

Narrator (Voiceover): "Their cool vibes are undeniable."

[Show a person petting a content cat, both sharing a moment of connection.]

Narrator (Voiceover): "😎 Join the cat craze and embrace the awesomeness..."

[Cut to a playful cat chasing its tail, accompanied by a cheerful laugh.]

Narrator (Voiceover): "...of these four-legged trendsetters!"

[End with a shot of a cat sitting regally, gazing confidently into the camera.]

Narrator (Voiceover): "🐱💫 #CatsRule #CoolCats #FelineVibes"

[Fade out with a final glimpse of a cat's eyes.]

Narrator (Voiceover): "Because when it comes to cool, cats wrote the book."

[End screen: "Follow for more feline fun!"]

[Background music fades out as the TikTok video concludes.]

中文翻译：[切换到一个玩耍的猫追逐它的尾巴的镜头，伴随着一阵欢快的笑声。]

旁白（配音）："...这些四足的潮流引领者！"

[以一只猫坐得像皇族一样，自信地凝视着摄像机的镜头结束。]

旁白（配音）："🐱💫 #猫咪统治 #酷猫 #猫咪氛围"

[以猫眼的最后一瞥淡出。]

旁白（配音）："因为当谈到酷，猫咪写下了规则。"

[结束画面："关注更多的猫咪乐趣！"]

[随着TikTok视频的结束，背景音乐渐渐淡出。]
```

像这样使用它们是可以的，但是如果我们想要将它们链在一起呢？这就是[Sequential Chains](https://python.langchain.com/docs/modules/chains/foundational/sequential_chains)的作用。这些允许你将多个链绑定到一个函数调用中，它们按照定义的顺序执行。有两种类型的顺序链，我们只关注简单的顺序链。将Chain导入行编辑为以下内容：

```python
from langchain.chains import LLMChain, SimpleSequentialChain
```

将LLM链移动到文件的顶部，删除print语句和.predict调用。

```python
# chain.py
from langchain.chat_models import ChatOpenAI
from langchain.chains import LLMChain, SimpleSequentialChain
from dotenv import load_dotenv
# this line changed!!!!!
from prompts import description_prompt, script_prompt

# loads the .env file
load_dotenv()

llm = ChatOpenAI(model_name="gpt-3.5-turbo")

description_chain = LLMChain(llm=llm, prompt=description_prompt)
script_chain = LLMChain(llm=llm, prompt=script_prompt)

tiktok_chain = SimpleSequentialChain(chains=[description_chain, script_chain], verbose=True)

script = tiktok_chain.run("cats are cool")

print(script)
```

输出结果如下：

```bash
Title: #CoolCatsRule

INT. LIVING ROOM - DAY

A trendy, upbeat song begins playing as the camera pans across a stylishly decorated living room. Various cat-themed decorations can be seen, setting the perfect atmosphere for showcasing the undeniable coolness of cats.

CUT TO:

INT. BEDROOM - DAY

A YOUNG WOMAN, in her early twenties, stands in front of a full-length mirror. She wears trendy clothes and holds a CAT, who seems equally as cool, in her arms.

YOUNG WOMAN
(looking into the mirror)
Ready to show the world why cats rule!

The young woman gently places the cat on the ground as the camera zooms in on the feline.

CUT TO:

INT. KITCHEN - DAY

A CAT sits on the kitchen counter, effortlessly balancing on one paw while wearing sunglasses. The camera pans around it, capturing its cool demeanor.

CUT TO:

INT. BACKYARD - DAY

A CAT lounges in a hammock, wearing a tiny hat and reading a book. The camera captures its relaxed and sophisticated vibe.

CUT TO:

INT. LIVING ROOM - DAY

The young woman sits on the couch, surrounded by a group of COOL CATS. Each cat showcases their unique coolness, like one wearing a leather jacket and another playing a tiny electric guitar.

YOUNG WOMAN
(points to the cats)
See? Cats rule!

The camera zooms in on the cats, showing their undeniable feline awesomeness.

CUT TO:

INT. LIVING ROOM - DAY

The young woman and her cool cats gather around a table, where they enjoy a mini-cat party. There are cat-themed snacks, funky drinks, and even a DJ cat scratching vinyl records.

CUT TO:

INT. LIVING ROOM - DAY

The young woman holds up a sign saying "#CoolCatsRule" as the cats pose beside her. The camera pans out to reveal a fun, energetic dance routine as they all groove to the beat.

CUT TO:

INT. LIVING ROOM - DAY

The young woman and her cool cats strike a final pose, with the camera capturing their undeniable coolness.

YOUNG WOMAN
(looking at the camera)
Remember, folks, cats rule!

The screen fades out with the hashtag #CoolCatsRule displayed prominently.

FADE OUT.

中文翻译：室内。客厅 - 白天

年轻女子举起一个写着"#CoolCatsRule"的牌子，猫咪们在她旁边摆好姿势。摄像机拉远，展示出一个充满活力的舞蹈，他们都在随着节奏摇摆。

切换到：

室内。客厅 - 白天

年轻女子和她酷猫们摆出最后的姿势，摄像机捕捉到他们无可否认的酷劲。

年轻女子 (看着摄像机) 记住，大家，猫咪才是真正的主宰！

屏幕以#CoolCatsRule的标签显眼地淡出。

淡出
```

## 总结

LangChain对于任何希望快速原型化大型语言模型应用的人来说都是一个游戏规则改变者。在短短几分钟内，我们已经走过了创建chain、定义提示，甚至将多个LLM调用链在一起以创建动态的TikTok脚本的过程。

LangChain的力量在于其简单性和灵活性。无论你是经验丰富的开发者还是刚刚起步，LangChain的直观设计都让你能够像从未有过的那样利用大型语言模型的能力。从生成创意内容到运行自主代理，可能性是无穷无尽的。

那么为什么等待呢？立即深入LangChain，释放你的项目中的AI潜力。如果你正在寻找将AI集成到你现有的工作流程或产品中，TimeSurge Labs在这里提供帮助。我们专注于AI咨询、开发、内部工具和LLM托管，我们的热情的AI专家团队致力于构建AI的未来，并帮助你的业务在这个快速变化的行业中蓬勃发展。今天就[联系我们](https://timesurgelabs.com/#contact)!

##  声明

本文翻译自[How To Use LangChain in 10 Minutes](https://dev.to/timesurgelabs/how-to-use-langchain-in-10-minutes-56e2)
---
title: chatgpt使用体验
date: 2023-03-21 14:53:41
permalink: about_chatgpt/
tags:
- AI
---

## 序言

最近几个月chatgpt很火, 虽然我在去年12月份刚出的时候就开始试用了, 不过写篇感想什么的, 倒是确确实实是最近才想到的. 这个时候再回过头谈一谈看法什么的, 有一种蹭热度来晚了的感觉. 因此本篇我只谈使用感想, 不谈看法心得.

去年12月份的时候, 因为没有稳定的API, 所以基本只是在openai的网站上使用, 老实说还是挺受限的, 还有各种各样的问题, 回答缓慢甚至是说到一半中断了之类的. 之前`Stable Diffusion`开源模型没有出来之前, 我也不是没有逆向过novelai的api, 把画图的功能加到`Yunzai-Bot`这个聊天机器人上. 不过鉴于chatgpt热度太高, openai和逆向开发者道高一尺魔高一丈的斗智斗勇, 我实在没兴趣参与, 于是就摆烂等openai正式的api出来. 3月初有了正式的api之后, 就顺势做了一个聊天机器人用的[插件](https://github.com/xdsoar/yunzaichat). 不得不说, 接入聊天工具以后, 机器人的功能才算是真的好用了起来.

## 实用案例

### 资料检索

聊天工具也不是没有做过集成搜索引擎的事情, 依稀记得某几个版本的qq似乎就做过这样的事情, 在发出去的聊天消息里会加上搜索引擎的超链接, 点击后就会跳出搜索界面. 当然这种交互方式自然是比不上直接问机器人, "xxx是什么", "为什么yyy", 然后机器人直接告诉你答案来的自然和直接. 曾经, 搜索引擎也有过一个叫做"手气不错"的功能, 可能在搜索引擎的场景下, 这种直接给出答案的模式成功率并不高, 如果第一个链接不对, 你就只能顺着链接一个一个点开看了. 而带上下文的聊天机器人则可以在第一个答案的基础上, 一步步修正来逼近你预期的目标.

### TLDR

作为IT工作者, 我更常用的模式是用它来取代阅读一些命令行工具的说明文档. 比如`使用rsync将文件同步到目标ssh服务器命令该怎么写`, 因为chatgpt可以理解上下文, 甚至可以接着问, `我想在同步的时候删除目标服务器上多余的文件`、`同步的时候把进度信息写入到日志文件`、`把这个命令写成sh脚本, 并创建crontab任务, 每天早上2点运行一次`. 这些事情虽然自己看手册、查文档也能完成, 不过由AI来写, 确实省了不少时间. 毕竟我不是天天和这些命令打交道, 不太会把rsync每个参数都记下来的程度.

几年前我在mac上装过一个alfred插件, 叫tldr. tldr的本意是`太长不看`, 一般是指发帖或者文档内容太多, 让人不想读下去. 而这个插件的意义省去你看文档的时间, 把每个命令行工具最常用的用法的命令给你列出来. 不过实际使用上, 这个插件的体验还是有很多不足, 很多时候你对典型的命令用法会有一些小调整, 这个时候这个插件就无能为力了, 只要有一个你需要的参数它没有解释, 你就只能重新去翻手册. 在这一点上, 自然语言模式的交流, 确实要远强于说明文档. 最近几年各类**弱智**的问答机器人其实流行度很高(比如各类购物网站的客服、聊天工具上的机器人等等), 虽然很蠢, 但是又不是完全没用. 说明这种聊天问答的模式, 对比枯燥无味的手册, 确实更能让人接受.

### 生成代码

这部分功能和上一节**TLDR**部分说的有一些重合, 一些基础性质的代码生成确实很好用, 这里用我实际用过的一个例子来说明, 这部分内容包含大量代码, 不熟悉的读者可以选择跳过.

因为最近我开始大量使用openai的whisper模型(语音转文本), 所以我要编写一个脚本来读取音频文件, 这里面可以分解成几个子任务, 如下:

1. 读取一个目录下所有指定后缀的文件(在我这个场景下, 是后缀为m4a的音频文件)
2. 读取同一个目录下已经生成的文本文件(在我这个场景下, 是后缀为srt的字幕文件)
3. 计算两个列表的差集, 即真正需要调用api翻译成文本的音频文件.
4. 对于大于25MB的音频文件, 需要拆分成小于25MB的文件, 分批调用api, 再将api响应的文本做合并

这其中, 1、2两条很简单也很通用，于是我让chatgpt为我生成一段python代码来生成文件，生成的代码如下:

```python
def find_files_with_suffix(path, suffix):
    """
    遍历指定目录下的所有文件，查找指定后缀名的文件
    :param path: 指定的目录
    :param suffix: 指定的后缀名，必须包含"."
    :return: 返回满足条件的文件路径列表
    """
    file_list = []
    for root, _, files in os.walk(path):
        for file in files:
            if file.endswith(suffix):
                file_path = os.path.join(root, file)
                file_list.append(file_path)
    return file_list
```

chatgpt生成的代码不仅可用, 而且还有完备注释, 确实不错.

然后对于第3条子任务, 可以简化为计算两个集合的差集, 我的预期不高, 但是也试着让chatgpt生成代码

```python
    untranslate_list = [value for value in audio_files if value not in srt_files]
```

我不是一个正经的python主义者(pythonism), 让我写的话, 可能会写成传统的for循环, 所以这个答案多少有一点让我惊艳, 以至于后面我在写代码的过程中, 时常会尝试性的问chatgpt, 我写的这段代码还能不能更精简一点...

对于第四条, 相对比较困难, 我尝试把其中做文件拆分的部分让chatgpt来写, 很遗憾这部分的答案不太令人满意, chatgpt生成的代码如下

``` python
from pydub import AudioSegment
import math

# 定义音频文件大小的最大值为20MB
MAX_FILE_SIZE = 20 * 1024 * 1024

# 定义分割音频的函数
def split_audio(filename):
    # 加载音频文件
    audio = AudioSegment.from_file(filename)

    # 计算音频文件的总长度（单位：毫秒）
    total_length = len(audio)

    # 计算每个小于20MB的音频文件的长度（单位：毫秒）
    segment_length = math.ceil(MAX_FILE_SIZE / (total_length / 1000))

    # 定义分割音频的起始时间和结束时间
    start_time = 0
    end_time = segment_length

    # 分割音频并保存为多个小于20MB的文件
    for i, segment in enumerate(audio[::segment_length]):
        # 找到当前分段音频最后一秒的静默部分，避免将正在说话的部分切断
        last_second = segment[-1000:]
        silent_threshold = last_second.rms - 10
        silence = last_second.detect_silence(silence_threshold=silent_threshold, min_silence_len=500, seek_step=1)

        if silence:
            end_time = start_time + (silence[0][0] + silence[0][1]) * 1000
        else:
            end_time = start_time + segment_length

        # 保存当前分段音频为新的音频文件
        new_audio = audio[start_time:end_time]
        new_filename = f"output_{i}.mp3"
        new_audio.export(new_filename, format="mp3")

        # 更新下一段分割音频的起始时间
        start_time = end_time

# 调用分割音频的函数
split_audio("input.mp3")
```

这一段代码乍一看有模有样, 但是很遗憾, 它并不能达成拆分文件的目标, 这里chatgpt无法理解其中的业务逻辑(比如为了正常拆分, 我需要音频文件在静默时间段拆分, 而不是强制按照特定的时间间隔拆分, 此外在拆分的时候, 应该尽量保证每个拆分片段足够长). 更遗憾的是, 我甚至没办法在chatgpt生成的代码基础上继续改, 因为这里我完全无法理解生成的代码的内在逻辑...

另外, 根据我的理解, gpt3.5作为通用语言模型, 应该是不具备生成代码这种需要精准的思维逻辑的能力(这一点从chatgpt甚至无法顺利完成复杂的四则运算就能看出来), 生成代码的能力应该是一个独立模型完成的. 这一点我在openai的官网上查了一下, 在gpt3.5之前, openai确实有另一个专用于自然语言生成代码的模型`Codex`. 因此我猜想gpt3.5中应该是延用了`Codex`的能力, 专门训练了通过自然语言生成代码的能力.

### 角色扮演

这类玩法网上说的很多, 因为接入了聊天机器人, 所以我也简单试过几个场景, 比如让chatgpt扮演旁白, 玩文字冒险游戏之类. 这里面我觉得主要的不足在于, 因为缺乏逻辑思维, 所以chatgpt并不具备长线编排故事的能力, 所以故事的情节显得比较无趣. 倒是可以先安排好故事大纲, 让chatgpt添加细节描写. 因为不是DND玩家, 这一块就不发表深度意见了.

不过有一点倒是需要一提, 目前的对话模型(包括最新发布的GPT4), 上下文能力还是非常有限的. 虽然说和过去几乎没有上下文能力的AI(比如Siri、小爱同学等等)相比, 已经好了很多. 限于目前gpt能接受的文字内容限制(gpt3.5是4k token, gpt4宣传是扩大到了3万, 但是目前使用chatgpt plus依旧用不到这么高), 另外也是这类模型(`Generative Pre-trained Transformer`)的特点, 文本越长, 越早的内容权重会越低, 所以就容易出现比如一开始给chatgpt一个设定, 但是随着对话的进行, 这个设定就失效了.

### yet another TLDR

上文也提到了, 我有在大量使用`whisper`这个语音转文本的模型, 我经常在一些参与感不高的会议上用手机录音, 然后事后用这个模型来做语音转文本. 不过语音转出来的文本量通常非常巨大, 而且对于没有提前准备发言稿的会议发言来说, 阅读体验是非常差的, 因为这种场景下的文字经常不连贯, 有片段的重复之类, 让正常人类来阅读这些文字真的挺折磨的. 不过我意外的发现这件事情chatgpt非常适合做这件事情, 即使存在一些不通顺的句子, 这个模型也能顺利对全文进行总结摘要.

## 总结

本篇只是例举了一些我实际用到的AI对话场景, 具体对于这个模型的一些看法的内容, 以后有机会再聊吧.

## 花絮

本篇博客在写作之前, 我脑洞大开, 原本期望可以由chatgpt为我生成一篇内容, 再稍作润色修正, 遗憾的是生成的效果并不让人满意. 简单的加一些promote页并没有改善, 个人感觉对于文字风格的把控, 在缺少训练素材的情况下, 还是比较难实现的. 或许等以后openai开放模型定制后会好一些. 下文是基于`gpt-3.5-turbo`模型生成的效果, 一并记录在这里.

> ChatGPT试用评测
>
> ChatGPT是一个基于GPT-3的自然语言处理模型，可以根据输入的摘要信息生成完整的文章。在我的试用过程中，我发现ChatGPT非常好用，可以根据自己的需求输入文章的摘要信息，也可以让ChatGPT生成指令和代码，生成的代码简洁实用。
>
> 在我进行指令和代码生成的过程中，我发现ChatGPT的工作效率非常高，生成出的代码符合我的预期，而且代码的可读性也非常好。但是，正如许多人所指出的那样，ChatGPT的记忆量非常有限，目前只能保留4k token的上下文，因此记忆量很有限。
>
> 虽然ChatGPT能够用于生成代码，但是我并不认为它能够引发编程革命，因为ChatGPT本身并没有对代码做抽象的能力。因此，在目前的技术水平下，ChatGPT并不能让代码变得更少。出于这种原因，它在编程领域的应用还是受到了一些限制。
>
> 然而，从我的试用经验来看，ChatGPT大大提高了我的工作效率，尤其是在文章的生成过程中。它可以根据我的需要快速生成文章的摘要和关键信息，使我更快地完成撰写工作。
>
> 总的来说，ChatGPT是一个非常好用的自然语言处理模型，它可以帮助我们快速生成文章、指令和代码，提高工作效率。虽然它有一些限制，但在我们需要更快地完成工作的时候，ChatGPT会成为我们非常好的助手。
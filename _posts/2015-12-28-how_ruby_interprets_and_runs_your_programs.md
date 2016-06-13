---
layout: post
title:  "【翻译】Ruby是如何解释运行程序的"
date:   2015-12-28 12:55:19 +0800
categories: jekyll update
---

> 作 者：[Starr Horne](http://blog.honeybadger.io/how-ruby-interprets-and-runs-your-programs/#authordetails)
原文地址：http://blog.honeybadger.io/how-ruby-interprets-and-runs-your-programs/  
前段时间在帮一位前辈校对《Ruby原理剖析》，而这篇文章正好相关，就试着翻译一下。这是我第一次在这献丑，请大家多提建设性意见。

　　作为开发人员，对手中的工具越了解，就越利于做出准确的判断。这很有用，尤其是定位性能问题的时候，了解Ruby运行底层的运行活动就格外重要。

　　这篇文章将带你体验一段小程序的分词、解析、编译之旅。我们将用Ruby提供的工具来窥探解释器的每一步活动。

　　别担心，即使你不是专家，这篇文章对你来说也很易懂。这并非高深的技术手册，而只是一篇普通指南。

#### 初遇小程序
　　我会用if/else语句作为示例。为了节约空间，这里用三元操作符写的，别较真啦，这其实就是if/else。

```ruby
　　100 ? 'foo' : 'bar'
```

　　可别小看这段程序，随着Ruby解释处理，信息量很大的。

> **注**：本文的示例是基于Ruby（MRI）2.2的，如果你用的是其他实现（或其他历史版本），结果可能不同。

#### 分词（Tokenizing）
　　在运行程序之前，Ruby解释器得先将形式自由的程序语句转换成加更结构化的数据。

　　第一步便是将程序切分成块（chunks），被称之为词条(token)

```ruby
    # 程序字符串
    "x > 1"

    # 词条(tokens)
    ["x", ">", "1"]
```
　　Ruby标准库提供的`Ripper`模块可以帮我们模拟Ruby解释器处理代码的过程。

　　下面示例在代码上调用了`tokenize`，如你所见，返回了词条数组。
```ruby
    require 'ripper'
    Ripper.tokenize("x > 1 ? 'foo' : 'bar'")
    # => ["x", " ", ">", " ", "1", " ", "?", " ", "'", "foo", "'", " ", ":", " ", "'", "bar", "'"]
```
　　分词器(tokenizer)有点傻，即使给的是非法代码，它还是会很happy地分词。
```ruby
    # bad code
    Ripper.tokenize("1var @= \/foobar`")
    # => ["1", "var"]
```
#### 词法解析(Lexing)
　　紧随分词之后的是词法解析。虽然还是词条，但之上多了附加信息。

　　下面的示例用**Ripper**对程序进行词法解析。如你所示，这次给每个词条打上了标签，如代表identifier 的`:on_ident`、代表operator的`:on_op`、代码Integer的`:on_int`等。
```ruby
    require 'ripper'
    require 'pp'

    pp Ripper.lex("x > 100 ? 'foo' : 'bar'")

    # [[[1, 0], :on_ident, "x"],
    #  [[1, 1], :on_sp, " "],
    #  [[1, 2], :on_op, ">"],
    #  [[1, 3], :on_sp, " "],
    #  [[1, 4], :on_int, "100"],
    #  [[1, 5], :on_sp, " "],
    #  [[1, 6], :on_op, "?"],
    #  [[1, 7], :on_sp, " "],
    #  [[1, 8], :on_tstring_beg, "'"],
    #  [[1, 9], :on_tstring_content, "foo"],
    #  [[1, 12], :on_tstring_end, "'"],
    #  [[1, 13], :on_sp, " "],
    #  [[1, 14], :on_op, ":"],
    #  [[1, 15], :on_sp, " "],
    #  [[1, 16], :on_tstring_beg, "'"],
    #  [[1, 17], :on_tstring_content, "bar"],
    #  [[1, 20], :on_tstring_end, "'"]]
```
　　目前为止，还是没有实质性的语法检测，词法解析器(lexer)还是会傻傻地处理非法代码。

#### 语法解析(Parsing)
　　现在代码已被拆分了成更可操作的词条，是时候进行词法解析了。

　　词法解析阶段，Ruby会将词条文本转换成“抽象语法树”(abstract syntax tree)，简称AST。“抽象语法树”是程序在内存中的表现形式。

　　你可能会说，整体来看编程语言其实就是更友好的抽象语法树而已。
```ruby 
    require 'ripper'
    require 'pp'

    pp Ripper.sexp("x > 100 ? 'foo' : 'bar'")

    # [:program,
    #  [[:ifop,
    #    [:binary, [:vcall, [:@ident, "x", [1, 0]]], :>, [:@int, "100", [1, 4]]],
    #    [:string_literal, [:string_content, [:@tstring_content, "foo", [1, 11]]]],
    #    [:string_literal, [:string_content, [:@tstring_content, "foobar", [1, 19]]]]]]]
```
　　这些输出可能不那么容易理解。不过细看之下，相信你能看出与源码的对应关系的。
```ruby 
    # 定义程序
    [:program,
    # if比较
    [[:ifop,
    # 条件检测 (x > 100)
    [:binary, [:vcall, [:@ident, "x", [1, 0]]], :>, [:@int, "100", [1, 4]]],
    # 若为true，则返回“foo”
    [:string_literal, [:string_content, [:@tstring_content, "foo", [1, 11]]]],
    # 若为true，则返回“foobar”
    [:string_literal, [:string_content, [:@tstring_content, "foobar", [1, 19]]]]]]]
```
　　现在Ruby解释器知道你具体的目的了，可以马上运行程序了。在Ruby 1.9 及以前的版本中，确实如此，但现在（Ruby1.9以后）还有最后一步要做。

#### 编译成字节码(Compiling to bytecode)
　　不再是直接遍历抽象语法树，如今的Ruby会将抽象语法树编译成更底层的字节码。随后这些字节码会在Ruby虚拟机（Ruby virtual machine）中运行。

　　我们可以通过`RubyVM::InstructionSequence`来窥探看一下Ruby虚拟机内部的工作。下面的示例先编译了代码，然后再反编译成更可读的格式。
```ruby 
    puts RubyVM::InstructionSequence.compile("x > 100 ? 'foo' : 'bar'").disassemble
    # == disasm: <RubyVM::InstructionSequence:<compiled>@<compiled>>==========
    # 0000 trace            1                                               (   1)
    # 0002 putself
    # 0003 opt_send_without_block <callinfo!mid:x, argc:0, FCALL|VCALL|ARGS_SIMPLE>
    # 0005 putobject        100
    # 0007 opt_gt           <callinfo!mid:>, argc:1, ARGS_SIMPLE>
    # 0009 branchunless     15
    # 0011 putstring        "foo"
    # 0013 leave
    # 0014 pop
    # 0015 putstring        "bar"
    # 0017 leave
```
　　哇塞！突然看起来更像汇编而非Ruby了。下面来逐一浏览一遍，看看能否理解它们。
```ruby
    # 调用self的`x`方法，并将结果存入栈中。
    0002 putself
    0003 opt_send_without_block <callinfo!mid:x, argc:0, FCALL|VCALL|ARGS_SIMPLE>

    # 将100入栈
    0005 putobject        100

    # 进行 (x > 100)比较
    0007 opt_gt           <callinfo!mid:>, argc:1, ARGS_SIMPLE>

    # 若比较结果为false，则跳转至15行
    0009 branchunless     15

    # 若为true，则返回“foo”
    0011 putstring        "foo"
    0013 leave
    0014 pop

    # 这就是15行，如果比较为false，便跳转至此，然后返回“bar”
    0015 putstring        "bar"
    0017 leave
```
　　然后Ruby虚拟机（YARV）便会遍历并执行这些指令。就这样！

#### 小结
　　至此，这个简单好玩的Ruby运行之旅便结束了。利用本文提到的这些工具，可以验证很多关于Ruby解释程序的猜想。相信我，再也没有比抽象语法树AST更“不抽象”了（这是句玩笑话）。下次你又被某个奇葩的性能问题所困时，不妨看看对应的字节码。这可能不会直接解决你的问题，但却可能给你灵感。

> 关于作者Starr Horne：
Starr Horne，Rubyist，Honeybadger.io的Javascript主程。没沉迷于解决他人的bug时，他喜欢制作传统手工家具、阅读历史、在位于西雅图的车库中酿造啤酒。
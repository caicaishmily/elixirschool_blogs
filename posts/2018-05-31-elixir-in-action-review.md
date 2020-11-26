---
author: George Mantzouranis
author_link: https://github.com/gemantzu
categories: review
date:   2018-06-18
layout: post
title:  Reviewing Elixir in Action, Second Edition
excerpt: >
 The first installment in our new series looking at, and reviewing, different learning materials available to the Elixir community.
---

# 《Elixir in Action》回顾

欢迎来到我们新系列-回顾 Elixir 各种学习材料的第一篇文章。

当然，你可能会想："Elixir School 是最好的", 我们同意这种说法，但在我们探索其他材料以改进我们自己的材料时，我们希望与你一起分享我们的想法。

让我们开始吧，我们将以对 Saša Jurić 的新书 _《Elixir in Action, Second Edition》_ 的回顾来开启我们的系列。

_注 2018-06-18: 这本书正处于 Beta 阶段（MEAP--Manning Early Access Program），根据曼宁网站（2018年9月），非常接近出版了。你可以[在这里](https://www.manning.com/books/elixir-in-action-second-edition)找到更多关于它的细节_

## 作者

Saša Jurić 是一名拥有着十多年专业编程经验的软件开发人员。

多年来，Saša 曾使用 Elixir、Erlang、Ruby、JavaScript、C# 和 C++ 等语言，这使他拥有丰富的经验。

## 目标

这本书的目标读者是什么？ 最好的地方莫过于本书的介绍:

> 这是一本中级水平的书，教授 Elixir，Erlang，以及它们是如何帮助你开发高可用的服务器端系统。为了充分利用这本书，你应该是一个经验丰富的软件开发人员。我们不期望您知道关于 Elixir、Erlang 或函数式编程的任何东西，但你应该精通至少一门编程语言，比如 Java、C#、Ruby、JavaScript。

## 回顾

编写一本关于编程语言的书是很难的。

如果这本书不仅需要涵盖该语言，还需要涵盖它所运行的平台，那就更难了。

就像技术类书籍经常出现的情况一样，作者通常会选择两条道路中的一条。"教程" 类的作者让读者从头到尾跟着一起构建一个应用程序，而 "文档" 类作者则专注于理论、功能定义和其他技术小细节。

在这本书中，Saša Jurić 采取了另一种方式：有一个很小的项目 -- 一个 ToDoList 应用，它的存在仅仅是为了加强读者对每一章主题的理解。

他在每一节的开头都解释了语言特性，介绍了它在示例项目中的使用方法和原因，最后在结束这一节时讨论了他对特定话题的经验和观点。这样做是相当自然的，每一节（几乎）从不间断，这使我们的大脑在任何时候都专注于一件事。让我们可以和 Saša 一起经历起伏，探索内容。

## 章节分析

这本书总共有13章。

在 1-4 章中，介绍到了平台、语言基础知识，我们可以在几乎所有的语言中使用和发现（甚至更多的是函数式语言）。我们可以找到关于变量、操作符、控制流、数据抽象和协议（ Elixir 版本的多态性）的细节。在本节中，我们开始研究我们的 ToDoList 应用，到最后，我们有一个包含 ToDoList 实体的 CRUD 功能的模块。

第 5-11 章可能是本书最棒的部分（我相信这也是本书第一版在 Elixir 开发者中如此受欢迎的原因），这一节对 OTP 及其对应程序（ GenServer、Supervision、ETS 表、OTP 应用程序）和工具包（Observer、Mix、Plug）进行了非常详尽的解释。在本节中，ToDoList应用中增加了很多这些功能。

* 增加一个自定义服务器进程来维护状态
* 将状态迁移到 GenServer 上
* 增加一个缓存，以便我们开始支持多个 to-do 列表。
* 增加基本的数据持久性
* 增加一棵监督树，并引入一个基本的 workers 库；

此外，还介绍了 ETS 表、OTP 应用和配置、依赖和 Plug。在这一节中，我有一种感觉，我是以一个小学生的身份和作者结对编程，在他设计应用程序的时候，坐在他身边。当他发现自己的设计有漏洞的时候，他就会自问 "OTP 的哪一部分可以解决这个漏洞，如何解决？"。然后，他转向我，给我详细解释我们正在做的事情，继续在 todo-list 应用上实现，最后又转向我，讨论备选方案和所选方案的起伏。我喜欢这种方法，我希望在更多的语言学习书籍中看到这种方法。

In the last two chapters, we can see how to make our app run on a cluster, release it using Distillery, and maintain it (Debug, Log etc). The author follows the same approach here as well, albeit not in that much detail, as the topics discussed in this section are very broad (clustering, data replication, network partitions, releases) and would need way too much detail to cover in a language introduction book.

在最后两章，我们可以看到如何让我们的 app 在集群上运行，使用 Distillery 发布，以及使用（Debug，Log 等）维护它。作者在这里也沿用了同样的方法，不过没有那么详细，因为这部分讨论的话题非常广泛（集群、数据复制、网络分区、发布），在一本语言入门书中需要涉及的细节太多。

## 结语

最近在谈到学习新技术时，书本被推到了一边，当然这不是没有道理的，毕竟阅读和理解一本书需要投入大量的时间。

如今，大多数人似乎没有时间看书，所以取而代之的是使用视频和简短的教程来开始学习。

另一个不利于书籍的因素是，很多作者省略了他们思考过程中的重要步骤，这最终让我们这些读者无法自己产生类似的解决方案。

我们不得不去想 "我们到底是怎么想出这个解决方案的？ 是什么样的思维方式让我们走到了这一步？"。

幸运的是，Saša 很擅长帮助读者理解他的推理过程，他解释了过程中的每一步，同时在他认为应该的时候探讨了替代方案。

那么 Saša Jurić 的新作 _《Elixir in Action, Second Edition》_是否值得你投入时间呢？

如果你是一个有经验的开发人员，希望提高对 Elixir 和底层 BEAM 的理解：是的。

这不是一本适合初学者的书。

这不是一本简单的书，

它是一本强调 Elixir 和 BEAM 潜力与缺点的优秀资源。

如果你能全神贯注地阅读本书，你就会在完成本书时对这个平台有一个扎实的了解，以及如何在开发生命周期中的各个阶段最好地利用它。

期待更多评论的到来！
## 赠品

为了庆祝我们的新系列开启，我们将免费赠送 Saša Jurić 的 _《Elixir in Action, Second Edition》_!

想获得免费的机会吗？

请在 [Twitter](https://twitter.com/elixirschool) 上关注我们，并转发这篇博文公告! 我们将在月底抽取获奖者，不要错过机会!

期待更多评论的到来！
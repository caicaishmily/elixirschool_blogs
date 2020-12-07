---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
tags: ['conferences']
date: 2019-09-08
layout: post
title: Dispatch From ElixirConf 2019
excerpt: >
  We had a great time at ElixirConf 2019! Hear about Elixir School's two workshops, along with the highlight talks and activities that we enjoyed this year.
---

# 来自 ElixirConf 2019 的快讯

今年在科罗拉多州举行的 ElixirConf 真正反映了 Elixir 社区不断发展和繁荣的性质。两天的研讨会和多方会谈都安排得满满当当，展示了 Elixir 提供的日益多样化和创新的人员和技术。

## 研讨会

今年，Elixir School 很高兴能够为 ElixirConf 研讨会的合作商。这两个研讨会的目标是指导参与者使用 Phoenix 的实时功能构建他们自己的实时应用--我们称之为 Pointing Party 的票务估算工具。第一天，我们在 Phoenix Channels 和 Presence 的帮助下构建了 Pointing Party，并了解了 Phoenix 如何利用 WebSockets。第二天，我们构建了同样的功能，这次使用了 Phoenix LiveView、PubSub 和 Presence。最后，我们还学习了使用 StreamData 进行基于属性的测试。

在这两天的时间里，能与这样一群充满好奇心、敬业和友好的学生一起工作，真的是一件很开心的事情。感谢每一位报名参加的学员 我们在教学过程中度过了一段愉快的时光，每个人参与的有趣问题和方法也让我们反过来受到了挑战。特别要向我们的少数学员致敬，他们对 Elixir 来说是新人，并直接进入了复杂的 Phoenix 概念--在一天结束前成功构建出我们的应用功能，并以他们开放的态度和新鲜的视角推动和支持着周围的人。

如果你想了解更多关于这些研讨会的信息，你可以在下面查看第一天以及第二天的代码和幻灯片。

* 第一天: 使用 Phoenix、Channels 和 Presence 驾驭实时网络
  * [幻灯片](https://speakerdeck.com/sophiedebenedetto/harnessing-the-real-time-web-with-phoenix-channels-plus-presence)
  * 代码
* 第二天: 使用 Phoenix、LiveView 和 StreamData 构建防弹实时应用。
  * [幻灯片](https://speakerdeck.com/sophiedebenedetto/building-bulletproof-real-time-apps-with-phoenix-liveview-plus-stream-data)
  * [LiveView 功能完整代码](https://github.com/elixirschool/pointing-party/tree/live-view-js-hooks)
  * [基于属性测试功能的完整代码](https://github.com/elixirschool/pointing-party/tree/test-vote-calculation-with-stream-data)


## 我们喜欢的讲座

有太多精彩的讲座可以看，但由于多轨制的安排，我们很遗憾不能全部看完。这里有几个特别让我们印象深刻的。

### Jose Valim 星期四晚上的 Keynote

虽然 Elixir 现在已经功能完备，但这并不意味着核心团队在明年没有一些令人激动和雄心勃勃的目标！Jose 讨论了 `Mix.Config` 的现状和未来，包括对构建时间和运行时间配置的一些想法。他还宣布了一个令人振奋的消息-- Phoenix 1.5 将提供一个原生的 Telemetry 模块，该模块可以轻松配置应用的 Telemetry 指标，并将其报告给你选择的第三方服务（New Relic、Statsd 等）。Elixir 社区可以期待每六个月发布一个新版本的 Elixir。我们很期待的一个功能是 ExUnit 模式差异--当 ExUnit 告诉你匹配失败时，你可以很容易地看到到底什么东西不匹配。了解更多关于Elixir 未来的信息，请[在这里](https://www.youtube.com/watch?v=oUZC1s1N42Q)查看完整的演讲内容。

### Chris McCord 周五晚上的 Keynote: LiveView 在野外

在过去的一年里，Chris McCord 和其他 Phoenix 的核心维护人员为我们提供了 LiveView -- 一种使用服务器渲染的 HTML 来编写丰富的、实时的 UX 的技术。在这里，Chris 向我们介绍了 LiveView 的强大功能，并让我们看看它在 "野外" 的表现，特别是与传统的 SPA（单页应用程序）相比。LiveView 的第一个版本包含了一系列令人兴奋的强大功能--包括 JavaScript 钩子、Live 导航 和 LiveView 测试。通过[这里](https://www.youtube.com/watch?v=XhNv1ikZNLs)查看完整的演讲，了解更多关于这个强大的工具集的信息。

### 构建可靠系统的合同 - Chris Keathley

Chris Keathley 的演讲提出了一个问题。"我们如何才能构建经得起变化的系统？"，并以他的新库 [Norm](https://github.com/keathley/norm) 回答了这个问题。Norm 的开发是为了应对我们许多人面临的挑战，即协调各团队和服务，确保我们不会破坏 API，同时鼓励和允许增长。在 [ExContract](https://hexdocs.pm/ex_contract/readme.html) 测试库的帮助下，通过拥抱合同驱动的设计，并利用 Norm 来强制执行数据规范，我们可以拥抱变化并减轻破坏。了解更多关于 Elixir 工具如何帮助我们促进分布式架构的合同驱动设计的信息，请点击[这里](https://www.youtube.com/watch?v=tpo3JUyVIjQ)查看完整的演讲。
### Elixir + CQRS - 在 PagerDuty 上架构可用性、可操作性和可维护性。 - Jon Grieman

了解 Pager Duty 如何在 CQRS 模式的帮助下构建比你最可靠的应用更可靠的系统。Jon Grieman 向我们讲述了 Pager Duty 将一个 Rails 单体重构成一个由 Kafka 支持的 Elixir 伞形应用的过程，并阐述了为什么 Elixir 和 CQRS 是如此的契合。[这里](https://www.youtube.com/watch?v=-d2NPc8cEnw)可以查看完整的演讲内容

## 活动亮点

通过四天的官方和非官方活动，我们也很高兴认识了其他与会者。与会者被鼓励使用 Whova 应用来协调见面会。有些人组织了晨跑，以及在庞大的酒店周围散步，有些人则计划了从扩展 Elixir 应用的见面讨论到晚上的品酒会等各种活动。酒店本身，即 Gaylord Rockies 度假村和会议中心，提供了几家不同的餐厅和咖啡馆、步行道和一个带懒人河的室内/室外游泳池。与会者充分利用了酒店提供的所有服务，还有更多-- Elixir Outlaws 播客组织者设置了一个懒河见面会，Smart Logic 与 ClusterTruck 在附近啤酒厂的欢乐时光，以及 EMPEX 的周五晚间聚会。

总的来说，会议组织得很好，运行得很顺利，而且很有趣，很温馨，充满了令人激动的学习内容和认识的人。希望明年能在那里见到你!

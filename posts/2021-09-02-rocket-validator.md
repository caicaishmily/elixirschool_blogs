%{
author: "Jaime Iniesta",
author_link: "https://github.com/jaimeiniesta",
tags: ["Announcement"],
date: ~D[2021-09-02],
title: "Validating Accessibility and HTML with Rocket Validator",
excerpt: """
Learn how we're checking and monitoring the new Elixir School site to detect and fix accessibility and HTML issues using Rocket Validator.
"""
}

---

## 简介

Elixir School 的核心原则之一是要成为一个包容性的社区。我们希望帮助来自世界各地的人学习使用 Elixir 写代码。为了促成这一点，我们提供了多种语言的课程翻译，由世界各地的志愿者维护。通过消除语言障碍，我们正在帮助更多的人进入 Elixir 社区。

除了语言障碍，还有其他我们所知道的障碍--[网络无障碍障碍](https://www.w3.org/WAI/people-use-web/abilities-barriers/) 在万维网上随处可见，使人们更难访问那些通常被认为是理所当然的内容。

如果文字的对比度低或字体大小不容易调整，那么低视力人群可能无法阅读网页。视力不好的人可以用屏幕阅读器来听网页内容，但如果网页结构不好，不能被屏幕阅读器软件正确解读，那就不好办了。行动不便的人可能更喜欢用键盘而不是鼠标来浏览课程，但如果文件的布局不正确，这也是不行的。

好消息是，我们可以让 Elixir School 对所有人都无障碍。这需要我们注意这些障碍，并遵循网络标准，提供结构良好的网页。

## 如何发现可访问性和 HTML 的问题

万维网联盟（W3C）是一个定义和开发[Web 标准](https://www.w3.org/standards/)的国际团体，HTML 可能是这些标准中最常用的--也是最被滥用的。1997 年底，[万维网联盟](https://www.webdesignmuseum.org/web-design-history/w3c-html-validator-1997)推出了[万维网联盟 HTML 验证器](https://validator.w3.org/)，这是一项检查任何网页并发现其 HTML 结构问题的服务。二十多年过去了，这项服务仍然运行良好，许多工具都是围绕其[开源验证器](https://github.com/validator/validator)建立的。

HTML 是网页的基础，但一个完全有效的 HTML 网页仍然有可访问性问题。例如，一个网页可能有一个不好的颜色组合，导致低对比度：浅色背景上的浅色文字可能导致低视力的人无法阅读。如果一个视频没有字幕，聋哑人用户就不会明白它在说什么。对于不使用鼠标在文件中移动的用户来说，文件的某些部分可能无法访问。缩放功能可能被禁用，使用户无法放大网页以更好地观看。许多已知的问题都可以通过自动可及性检查器如[axe-core](https://github.com/dequelabs/axe-core)轻松检测出来。

## 自动化网站验证的力量

我们有工具可以帮助检测网页上的问题，这很 Nice。W3C HTML 验证器和 axe-core 都是开源的，可以被纳入我们的网络开发流程。但是，有一个重要的限制：它们一次只能检查一个 URL。所以，当你有一个只有几个网页的小网站时，使用这些工具进行手动检查可能是一个选择。但是，对于像 Elixir School 这样的大型网站，我们能做什么呢？根据最新的报告，Elixir School 有超过 1000 个网页。如果我们想用 W3C 验证器和 axe-core 来检查它们，我们就需要进行 2000 次人工检查!

## 🚀 火箭验证器来拯救我们

[Rocket Validator](https://rocketvalidator.com) 是一项对大型网站进行全站验证的在线服务。它是一个自动运行的网络爬虫，它会找到网站的内部网页，并使用 W3C 验证器和 axe-core 来验证找到的每个网页，对发现的问题产生详细的报告，并且在几秒钟内完成。

这项服务也由 Elixir、Phoenix、LiveView 和 Oban 提供，是为网络机构和自由开发者提供的基于订阅的服务。但是，对于像[Elixir School](https://github.com/elixirschool/school_house/pulls?q=is%3Apr+label%3A%22rocket+validator%22+is%3Aclosed) 和 [Docusaurus](https://github.com/facebook/docusaurus/pulls?q=is%3Apr+rocket+validator)这样的开源项目来说，它是免费的! 我们正在使用 Rocket Validator 来运行关于新的 Elixir School 的报告，并通过每周的定时任务部署触发的验证来监控网站。

## 运行一个网站验证报告

要运行一个全站的验证报告，我们需要输入一个起始 URL。这个 URL 通常是我们要验证网站的首页，但它也可以是任何内部的 URL，或者是包含要验证网页的 XML 或 TXT 网站地图。Rocket Validator 会访问这个初始 URL，然后发现内部网页，并把它们添加到报告中。

![Rocket Validator new site validation report](https://www.dropbox.com/s/amt7ilhw3b0sdh2/rocket-validator-new-report.png?raw=1)

我们还可以定义将验证速度为每秒请求的速率限制，并定义选项：

- **检查 HTML** 将对发现的每个页面运行 W3C HTML 验证器。
- **检查可及性** 将在发现的每个页面上运行 axe-core。
- **深度抓取** 将通过递归发现内部链接找到该网站的更多网页。

一旦我们点击 **开始验证**，随着网页的验证，结果将在几秒钟内出现。我们可以浏览总结报告，看到包括最重要的问题在内的全局概况（这样你就知道首先要解决什么问题），以及将网站上的常见问题分组的报告和每个网页的详细报告。

![Rocket Validator summary report](https://www.dropbox.com/s/z611nr8ofpikawx/rocket-validator-summary-report.png?raw=1)

## 分享报告

任何由 Rocket Validator 生成的网站验证报告，只要粘贴它的 URL 就可以很容易地被分享。这让我们可以在 Github issue 中引用报告中的任何部分，这样我们就可以一起修复它们。

Rocket Validator 中新的 Domain Stats 功能提供了对一个域名的报告分析，这个也很容易分享。查看更新后的[Elixir School Stats](https://rocketvalidator.com/domains/beta.elixirschool.com?auth=e536facf-2cba-4288-ba45-3e7b95addcf8)，可以得到网站上运行的最新报告，以及网页覆盖率和 HTML/Accessibility 状态随时间变化的统计。

![HTML issue density evolution for Elixir School](https://www.dropbox.com/s/m36htu85nvxmpse/rocket-validator-html-charts.png?raw=1)

上图显示了我们如何通过减少平均 HTML 问题密度（每页问题）来改善 Elixir School 的情况。

## 验证 浅色/深色 模式的对比度问题

Elixir School 有一个 浅色/深色 模式的切换，这对于选择你喜欢的用户界面是很好的，但这也意味着我们所有的 URL 都需要在浅色/深色下进行验证，以确保每个主题没有低对比度问题。

![Light and Dark modes on Elixir School](https://www.dropbox.com/s/yzzj595vteqdi69/rocket-validator-contrast.png?raw=1)

为了验证浅色和深色模式的可访问性，我们想出了在 URL 上添加一个可选参数 `?ui=dark` 的主意，如果有的话，就可以启用深色模式。因此，我们可以使用默认的[light mode XML sitemap](https://beta.elixirschool.com/sitemap.xml)来运行浅色模式的报告，而使用[dark mode XML sitemap](https://beta.elixirschool.com/sitemap_dark_mode.xml)来运行深色模式的报告，在 URLs 中加入这个参数。

有了这些 XML 网站地图，我们生成两种报告：

- 一份是针对浅色模式的报告，检查 HTML 和可访问性问题。
- 另一份报告是针对深色模式的，只检查可访问性问题，以监测可能的对比度问题。这些 URL 不需要检查 HTML，因为 HTML 标记与浅色模式相同。

## 网站定时验证以及部署钩子

网站验证报告也可以按 [计划](https://docs.rocketvalidator.com/scheduling/) 每天、每周或每月运行，从而对你的网站提供持续的监控。对于 Elixir School，我们在网站上设置了一个每周的时间表，当我们正在紧张地修复网站时，有时会改成每天运行。

我们还使用了[部署钩子](https://docs.rocketvalidator.com/deploy-hooks/) - 所以一个新的部署会触发一个网站验证报告。使用 [Heroku post-hook add-on](https://devcenter.heroku.com/articles/deploy-hooks#http-post-hook) 会很容易集成到 Heroku 中。

## 静音一些我们不会修复的问题

验证一个网站意味着检查网页是否符合当前的网络标准，但有时一个网站不能完全满足最严格的标准。

例如，一些生成的代码可能超出了你的范围。在我们的案例中，Phoenix 生成的一些 HTML 不是标准的 HTML（例如，像 `phx-click` 或 `phx-track-static` 这样的属性不是有效的 HTML）。我们还在使用 Alpine，它使用了非标准的属性，如 `x-data` 或 `x-cloak`。

一旦你意识到这些问题，并且考虑到是否可以用标准的替代方法，你可以决定将这些问题静音化。Rocket Validator 可以让你对选定的问题进行静音处理，并提示你记录静音处理的原因。

下面是[Elixir School 的静音问题列表](https://rocketvalidator.com/domains/beta.elixirschool.com?tab=mutings&auth=e536facf-2cba-4288-ba45-3e7b95addcf8).

## 超出自动验证

自动网站验证是一个强大的工具，可以在大型网站上找到众所周知的问题，它可以快速指出很多典型的问题，并提供已知的解决方案。可以把它看作是一个扫描器，它可以在你的网站上找到 "低垂的果实" 问题--Rocket Validator 可以通过 axe-core 检测到[近 100 个典型的可访问性问题](https://rocketvalidator.com/accessibility-validation)，而[HTML 验证器](https://rocketvalidator.com/html-validation)可以检测到更多可能滥用网络标准的情况。

然而，在网开发中总是需要人工检查和一些常识。一个网站可能遵循最严格的网络标准，但仍然无法访问。例如，想想图片中的 `alt` 描述--除非它们真的提供了一个[有意义的描述](https://duckduckgo.com/?q=meaningful+alt+image+description&t=opera&ia=web)，否则它们根本就没有用。

除了自动测试，你还需要在你的网站上使用屏幕阅读器做人工检查，关闭屏幕，使用键盘而不是鼠标进行导航，等等。我们意识到这一点，并且已经有一个[通过人工测试发现的问题](https://github.com/elixirschool/school_house/issues/114)的清单，很快就会得到解决。

## 结语

网页验证是网页开发中的一个重要步骤，但经常被跳过，因为手动检查每个网页是一项繁琐的工作。

幸运的是，有一些方法可以使大型网站验证自动化，因此我们可以快速检测和修复 HTML 和无障碍问题，粉碎我们网站上的许多无障碍问题，使它们可以被更多人使用。

## 致谢

这篇文章的作者是[Jaime Iniesta](https://jaimeiniesta.com)，他是一名自由网络开发者，使用 Elixir 创建了[Rocket Validator](https://rocketvalidator.com)。

通过创建一个[免费 Rocket Validator 账户](https://rocketvalidator.com/registration/new)，可以在你的网站上试用 Rocket Validator。

如果你之后想升级，请使用 **[ELIXIRSCHOOL](https://rocketvalidator.com/pricing?coupon=ELIXIRSCHOOL)** 优惠券代码，以获得任何订阅计划的 50%折扣。

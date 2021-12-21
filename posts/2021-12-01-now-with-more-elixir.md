---
%{
author: "Sean Callan",
author_link: "https://github.com/doomspork",
tags: ["Announcement"],
date: ~D[2021-12-01],
title: "Now With More Elixir!",
excerpt: """
How Elixir School migrated from Jekyll to a Phoenix powered site
"""
}

---

# 现在有了更多的灵药!

如果你在 Twitter 上关注我们，那么你可能已经在今天的大会之前偷看了新网站，但对于那些没有关注我们的人来说，我们认为应该报道一下这个历史上这个令人兴奋的新篇章。

## TL;DR 历史

Elixir 学校的第一次提交是在许多月前我生日的第二天进行的。2015 年 5 月 31 日。从那时起，这个项目已经从我开始发展到超过 500 人，他们贡献了新的内容、翻译和其他改进。随着事情的发展，我们显然需要从 repo 中的 markdown 迁移，鉴于当时 Elixir 领域缺乏选择，我们选择了 Jekyll。Jekyll 让我们上线，并为我们提供了一个静态的网站，处理全球每周成千上万的用户。

但是，学习 Elixir 的地方有一些是建立在 Ruby 工具上的。更不用说静态网站的简单性很好，但阻碍了我们的一些更酷的想法。

六年后的今天，Elixir 周围有一个充满活力的社区。有了这些丰富的经验、观点和背景，就有了一些非常棒的工具。

闲话少说，让我们来看看我们是如何从 Jekyll 到 Phoenix 的!

## Jekyll 到 Phoenix

### Nimble Publisher

在我们从 Jekyll 到 Phoenix 的旅程中，最大的障碍是找出最好的方法来迁移现有的内容，而不重写它，失去贡献者的历史，或者给贡献者增加额外的负担。进入 Dashbit 的 NimblePublisher 库!

我们将在后续的博文中介绍我们对 NimblePublisher 的使用，在此之前，请不要忘记查看我们的[NimblePublisher](https://elixirschool.com/en/lessons/misc/nimble_publisher)课程!

### 自定义 mix Tasks

我们今天主要为两个目的采用了自定义混合任务：生成网站地图和 RSS 提要。这两个任务利用了我们用 NimblePublisher 构建的内容模块。

在这两个任务中，RSS 订阅是最直接的，我们列举了我们的博客文章列表并建立了一个 XML 文档。

```elixir
defmodule Mix.Tasks.SchoolHouse.Gen.Rss do
  use Mix.Task

  alias SchoolHouse.Posts
  alias SchoolHouseWeb.{Endpoint, Router.Helpers}

  @destination "assets/static/feed.xml"

  def run(_args) do
    Mix.Task.run("app.start")

    items =
      0..(Posts.pages() - 1)
      |> Enum.flat_map(&Posts.page/1)
      |> Enum.map(&link_xml/1)
      |> Enum.join()

    document = """
    <?xml version="1.0" encoding="UTF-8" ?>
    <rss xmlns:atom="http://www.w3.org/2005/Atom" version="2.0">
      <channel>
        #{items}
      </channel>
    </rss>
    """

    File.write!(@destination, document)
  end

  defp link_xml(post) do
    link = Helpers.post_url(Endpoint, :show, post.slug)

    """
    <item>
      <title>#{post.title}</title>
      <description>#{post.excerpt}</description>
      <pubDate>#{Calendar.strftime(post.date, "%a, %d %B %Y 00:00:00 +0000")}</pubDate>
      <link>#{link}</link>
      <guid isPermaLink="true">#{link}</guid>
    </item>
    """
  end
end
```

网站地图生成器遵循类似的格式，但内容的价值和它的结构使任务变得有点儿复杂。我们将生成过程分解为几个步骤：

1. 添加博客索引和我们的隐私政策

   ```elixir
   defp all_links do
     [
       Helpers.post_url(Endpoint, :index),
       Helpers.page_url(Endpoint, :privacy)
     ] ++ post_links() ++ Enum.flat_map(supported_locales(), &locale_links/1)
   end
   ```

2. 创建一个博客文章链接的集合

   ```elixir
   defp post_links do
     0..(Posts.pages() - 1)
     |> Enum.flat_map(&Posts.page/1)
     |> Enum.map(&Helpers.post_url(Endpoint, :show, &1.slug))
   end
   ```

3. 建立每个地区的链接。这包括所有的课程和页面，如会议、播客、"为什么是 Elixir？" 以及其他

   ```elixir
   defp locale_links(locale), do: page_links(locale) ++ lesson_links(locale)

   defp page_links(locale) do
     [
       Helpers.page_url(Endpoint, :conferences, locale),
       Helpers.page_url(Endpoint, :index, locale),
       Helpers.page_url(Endpoint, :podcasts, locale),
       Helpers.page_url(Endpoint, :why, locale),
       Helpers.report_url(Endpoint, :index, locale)
     ]
   end

   defp lesson_links(locale) do
     config = Application.get_env(:school_house, :lessons)

     translated_lesson_links =
       for {section, lessons} <- config, lesson <- lessons, translated_lesson?(section, lesson, locale) do
       Helpers.lesson_url(Endpoint, :lesson, section, lesson, locale)
       end

     section_indexes =
       for section <- Keyword.keys(config) do
       Helpers.lesson_url(Endpoint, :index, section, locale)
       end

     section_indexes ++ translated_lesson_links
   end

   defp translated_lesson?(section, lesson, locale) do
     case Lessons.get(section, lesson, locale) do
     {:ok, _} -> true
     _ -> false
     end
   end

   defp supported_locales do
     :school_house
     |> Application.get_env(SchoolHouseWeb.Gettext)
     |> Keyword.get(:locales)
   end
   ```

这两个任务都在我们的发布过程中运行，并存储在静态目录中，与其他资源如 `robots.txt `一起提供。

```elixir
plug Plug.Static,
  at: "/",
  from: :school_house,
  gzip: false,
  only: ~w(css fonts images js favicon.ico robots.txt feed.xml sitemap.xml)
```

### 前进路线图

#### 新内容

通过这次发布，我们还将我们的内容重组为新的部分！这为更多的内容和更易于管理的组织和消费方式铺平了道路。这为更多的内容铺平了道路，也为组织和消费这些内容提供了更易于管理的方式。你可以留意一些新的内容：

1. **高级内容**

   1. 专门的元编程课程，探讨这一强大的功能
   2. 大大扩展了我们的规格和类型内容

2. **数据处理**

   1. 新课：Dashbit 的 [Broadway](https://github.com/dashbitco/broadway) 库
   2. 新课：Dashbit 的 [Flow](https://github.com/dashbitco/flow) 库

   这些课程给了我们对数据三要素的覆盖：GenStage、Flow 和 Broadway。

3. **Ecto**

   1. 扩展现有的 Ecto 内容
   2. 新课: 高级 Ecto 查询技术

4. **存储**

   1. 新课：Redix 库
   2. 新课：Cachex 库

5. **基础知识**

   1. 新课：函数式编程 101
   2. 新课：函数式编程 102
   3. 新课：数据结构

我们继续讨论更大量地扩展到 Phoenix 和 Erlang 内容的可能性和价值。我们希望得到您的反馈! 有兴趣为这些课程做一些贡献吗？对内容有建议吗？不要犹豫，请联系我们或参与进来。

我们的课程还不是全部，在未来的几个月里，我们有很多令人兴奋的博客内容：

1. 我们计划恢复我们的书籍、课程和会议评论，继续为社区提供公正的资源，说明哪些学习材料是值得他们花费的。

2. 与[Elixir 公司](https://elixir-companies.com/en) 一起，我们将启动一个关于使用 Elixir 公司的新博客系列。你一直在问的一些问题，我们的目的是要回答。谁在使用 Elixir？他们最终是如何选择 Elixir 的？入职情况如何？Elixir 是用来做什么的？

3. 社区亮点! 社区里有 **很多** 了不起的人，其中很多人没有得到他们应得的认可。我们希望通过选择和强调那些回馈社会和推动我们所有人的人，来解决这个问题。有一些参与 Elixir 的了不起的人被一个较小的声音所掩盖，我们想改变这种状况！我们对所有的新成员感到兴奋。

我们对所有新的可能性和储存的内容感到兴奋，我们希望你也是如此。

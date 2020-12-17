---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
date: 2020-10-06
layout: post
title: "Real-Time SVG Charts with Contex and LiveView"
excerpt: >
  Learn how to use the Contex package to render server-side SVG charts that you can update in real-time with LiveView.
---

# 使用 Context 和 LiveView 绘制实时 SVG 图表

这篇文章的灵感来自于我和 [Bruce Tate](https://twitter.com/redrapids) 一起为 Pragmatic Bookshelf 即将出版的 《LiveView》 一书所做的一些工作。很高兴能够在电子书出版后提供少量电子书的赠品给到 Elixir School，敬请关注更新!

随着 LiveView 的成熟，人们越来越清楚它不仅仅是一个在 Web 应用中构建实时功能的工具。LiveView 是你的单页面应用的状态管理系统。开发人员越来越多地接触到 LiveView，以构建和管理复杂功能的 SPA。这就是为什么在我们即将出版的 LiveView 书中，我们的目标是使用 LiveView 构建出一套多样化和强大的功能。

曾经一个常见的功能是面向管理员的实时仪表盘。这样的仪表盘往往需要数据可视化，而 LiveView 意味着我们可以实时提供数据。在这篇文章中，我们将看看如何利用 [Contex](https://github.com/mindok/contex) 库进行服务器端 SVG 图表渲染。我们将在 LiveView 中渲染一个 Contex 图表，并实时更新我们的图表。一路下来，我希望你会对使用 LiveView 可以做什么感到兴奋，如果你深入研究我们即将出版的书，你可以学到点什么。

## 问题

虽然有不少 JavaScript 图表库可供选择，但我们追求的是一个 _服务器端渲染_ 的解决方案。LiveView 在服务器上管理状态。状态变化会触发 HTML 的重新渲染，并将 HTML 推送到客户端，然后由客户端高效地更新 UI。所以，我们不希望引入一个库来在客户端上使用大量复杂的 JavaScript 来渲染图表。我们需要能够在服务器上绘制我们的图表，并将该图表的 HTML 发送到客户端。

虽然 Elixir 中并没有多少服务器渲染的图表库，但幸运的是，我们有 Contex!

## 介绍 Contex

[Contex](https://github.com/mindok/contex) 是一个用 Elixir 编写的服务器端图表库，它允许我们在服务器上构建和渲染 SVG 图表。然后我们可以使用 LiveView 在模板中渲染这些图表，并使它们能够实时更新。Contex 目前支持条形图、点图、甘特图和折线图。

值得一提的是，Elixir 中还有一些其他的服务器端图表库。[`ggity`](https://github.com/srowley/ggity) 旨在 "将 R 的 `ggplot2` 库的接口带入到用 Elixir 绘制 SVG 图的任务中"，并支持多种不同的图表类型。[`plotex`](https://github.com/elcritch/plotex) 库在服务器上构建和渲染时间图系列 SVG 图表。`ggity` 库虽然可能是探索性数据分析的一个不错的选择，但并没有针对 LiveView 的使用进行优化，目前也没有打算用于生产用途，而 `plotex` 库的目的只是为了渲染时间图系列图表。因此，`contex` 虽然还是一个新的库，有一些 [在建的 TODO](https://github.com/mindok/contex#warning)，但它最符合我们在 LiveView 中以各种格式实时执行渲染数据的需求。

在本篇文章中，我们将着重于使用 Contex 来构建一个条形图。

## 我们将构建什么

借鉴一个你将在我们即将出版的 LiveView 书中看到更深入内容的例子，我们将在 Admin Dashboard LiveView 中添加一个图表。管理员仪表盘是一个在线游戏应用的一部分，用户可以在其中玩在线版本的游戏，如乒乓球和井字游戏。我们要求用户填写一份调查表，对游戏进行 1 到 5 星的评分。我们的管理面板应该包括一个产品的图表和他们的平均星级。就像这样：

![游戏评分表](https://elixirschool.com/assets/game-ratings-chart-c97d10c38c3dc7b6c9ba91d9710f8957b4de80dab0df1218942908c028bc18b6.png)

图表也应该实时更新。换句话说，当世界各地的用户使用和评论我们的极受欢迎和超级有趣的游戏时，图表应该实时更新以反映更新的平均评分。

在这篇文章中，我们将假设我们已经有一个 LiveView `GameStoreWeb.AdminDashboardLive`，挂载在 `live '/admin-dashboard'`。

这个 LiveView 会渲染一个有状态的子组件 `GameStoreWeb.GameRatingsLive`。这个组件是我们构建和渲染条形图的地方。

简单看一下我们的 `AdminDashboardLive` LiveView，我们可以看到它建立了一个 socket，`:game_ratings_component_id`。

```elixir
# lib/game_store_web/live/admin_dashboard_live.ex
defmodule GameStoreWeb.AdminDashboardLive do
  def mount(_params, _session, socket)
    {:ok,
     socket
     |> assign(:game_ratings_component_id, "game-ratings")}
  end
end
```

然后，我们使用这个赋值来设置 `:id` 给我们有状态组件。

```elixir
# lib/game_store_web/live/admin_dashboard_live.html.leex
<%= live_component @socket,
      GameStoreWeb.GameRatingsLive,
      id: @game_ratings_component_id %>
```

我们将组件的 ID 存储在 socket assigns 中，这样我们就可以在以后使用它来通过 `send_update/2` 函数向组件发送更新。后面会有更多的介绍。

## 入门

首先，我们将在 `mix.exs` 文件中添加 Contex 包到我们应用程序的依赖关系中：

```elixir
{:contex, "0.3.0"}
```

运行 `mix deps.get` 来安装新的依赖关系。

## 查询图表所需数据

在我们渲染柱状图之前，我们需要查询并格式化我们的图表数据。我们将编写查询选择所有的游戏，以及每个游戏的平均评分。

我们将在应用核心中定义的查询构建模块中实现我们的查询。我们应用程序的功能核心是我们放置所有可预测和可靠的代码的地方。在我们的数据库中处理数据就是一个很好的例子。现在我们来看看我们的查询。

在我们的数据模型中，我们游戏有很多评级。所以，我们将查询游戏，在评级表上加入，并选择游戏名称和所有给定游戏的评级星级的计算平均值。

```elixir
# lib/catalogue/game/query.ex
defmodule GameStore.Catalogue.Game.Query do
  alias GameStore.Survey.Rating

  def base do
    Game
  end

  def with_average_ratings(query \\ base()) do
    query
    |> join_ratings
    |> average_ratings
  end

  defp join_ratings(query) do
    query
    |> join(:inner, [g], r in Rating, on: r.game_id == g.id)
  end

  defp average_ratings(query) do
    query
    |> group_by([g], g.id)
    |> select([g, r], {g.name, fragment("?::float", avg(r.stars))})
  end
end
```

_组成_ 查询的工作是可以预测和可靠的，而 _执行_ 查询的工作却不是这样。你无法确定执行数据库查询的结果是什么，而且这种工作往往依赖于用户的输入。所以，我们查询的执行将由我们应用程序的 `Catalogue` 上下文负责。上下文作为我们应用的边界，我们可以在这里定位处理不确定性和来自外界的输入的代码。

现在让我们在 `Catalogue` 上下文中用上下文函数来封装我们的查询。我们通过管道将我们的查询执行到对 `Repo.all()` 的调用。

```elixir
# lib/catalogue.ex
defmodule GameStore.Catalogue do
  alias GameStore.Catalogue.Game
  alias GameStore.Repo

  def games_with_average_ratings do
    Game.Query.with_average_ratings
    |> Repo.all()
  end
end
```

我们现在来看一下执行这个查询的结果：

```bash
# iex 环境
iex> alias GameStore.Repo
iex(4)> Catalogue.games_with_average_ratings()
[debug] QUERY OK source="games" db=6.1ms decode=2.1ms queue=7.1ms idle=1495.6ms
SELECT g0."name", avg(r1."stars")::float FROM "games" AS g0 INNER JOIN "ratings" AS r1 ON r1."game_id" = g0."id" GROUP BY g0."id" []
[
  {"Tic-Tac-Toe", 3.4285714285714284},
  {"Ping Pong", 2.5714285714285716},
  {"Pictionary", 2.625}
]
```

请注意，我们的查询选择并返回一个结果集合，每个结果都是一个有两个元素的元组。第一个元素是游戏的名称，第二个元素是平均评分。这种格式是有目的的，是为了在我们的 Contex 条形图中使用。

现在让我们把注意力转向用这些数据构建出那个图表。

## 在 LiveView 中定义你的图表

在我们认真构建出或组件之前，值得一提的是我们将应用的模式来管理该组件中的状态。我们将依靠 reducers 来连续更新 socket 状态，以初始化我们组件的起始状态并处理更新。reducers 是接受一个事物并返回一个更新后的相同类型事物的函数。它们使我们能够编写整洁、干净的管道，使我们能够轻松地构建和管理 LiveView 状态，并通过更新该状态来响应事件。这是一种模式，我们将在我们的 LiveView 书中更深入地介绍。

### 在状态中排序图表数据

首先，我们要教我们的 `GameRatingsLive` 组件查询这些有平均评分的游戏，并将它们保存在 socket assigns 中。

记得前面我们说过，我们要把 `GameRatingsLive` 组件渲染成 `AdminDashboardLive` LiveView 中的有状态组件。我们将利用有状态组件的 `update/2` 生命周期方法来获取我们的游戏和评分数据，并将其存储在 socket assigns 中。

```elixir
# lib/game_store_web/live/game_ratings_live.ex
defmodule GameStoreWeb.GameRatingsLive do
  use GameStoreWeb, :live_component
  alias GameStore.Catalogue

  def update(assigns, socket) do
    {:ok,
     socket
     |> assign(assigns)
     |> assign_games_with_average_ratings()}
  end

  defp assign_games_with_average_ratings(socket) do
    socket
    |> assign(:games_with_average_ratings, Catalogue.games_with_average_ratings())
  end
end
```

请注意，我们的 socket 是通过一组 reducer 来实现的，每一个 reducer 都会对 socket 的状态进行进一步的装饰。这些 reducer 函数接收一个 socket 的参数，对该 socket 做一些事情，然后返回一个具有更新状态的新 socket。

首先，我们将应用从父 LiveView 传来的任何赋值，然后我们将在赋值中添加一个 `:games_with_average_ratings` 的键，指向我们的查询结果的值。

现在我们已经有了状态下的查询结果，我们准备使用它们来构建我们的图表。

### 构建柱状图

构建 Contex 图表有三个阶段：

- 初始化数据集
- 初始化图表
- 将图表渲染成 SVG

我们将为我们的 `update/2` 管道添加一个 reducer，它可以更新这个进程中每一步的 socket 状态。

#### 初始化 `DataSet`

构建 Contex 图表的第一步是用 `Contex.DataSet` 模块初始化数据集。[`DataSet` 模块](https://hexdocs.pm/contex/Contex.Dataset.html) 将您的数据集包装起来，用于绘制图表。它提供了一组方便的函数，后续的图表绘制模块将利用这些函数来操作你的数据并绘制图表。`Dataset` 通过将不同的数据结构整合成一致的形式来处理它们，供图表绘制函数使用。它可以处理的数据结构有：map 的列表、列表的列表或元组的列表。回想一下，我们确保我们对具有平均评分的游戏的查询返回的是一个元组列表。

我们将首先实现一个新的 reducer 函数 `assign_dataset/1`。这个 reducer 将从 socket assigns 初始化一个新的 `DataSet`，其中包含查询结果，我们的游戏和平均评级元组列表。然后，它将把数据集添加到 socket 状态。

```elixir
# lib/game_store_web/live/game_ratings_live.ex
defmodule GameStoreWeb.GameRatingsLive do
  use GameStoreWeb, :live_component
  alias GameStore.Catalogue

  def update(assigns, socket) do
    {:ok,
     socket
     |> assign(assigns)
     |> assign_games_with_average_ratings()
     |> assign_dataset()}
  end

  # ...

  defp assign_dataset(%{assigns: %{games_with_average_ratings: games_with_average_ratings}}) do
    socket
    |> assign(:dataset, Contex.Dataset.new(games_with_average_ratings))
  end
end
```

如果我们看一下调用 `Contex.DataSet.new/1` 的输出，我们会看到以下结构：

```elixir
%Contex.Dataset{
  data: [
    {"Tic-Tac-Toe", 3.4285714285714284},
    {"Ping Pong", 2.5714285714285716},
    {"Pictionary", 2.625}
  ],
  headers: nil,
  title: nil
}
```

`DataSet` 认为给定元组的列表中第一个元素是 "category 列"，第二个元素是 "value 列"。category 列用于标注条形图类别（在我们的例子中是游戏名称），value 列用于填充该类别的值。

#### 初始化 `BarChart`

现在我们有了数据集，我们可以用它来初始化我们的 `BarChart`。我们将在后续的 `update/2` 管道中添加 `assign_chart/1` 来完成这个操作。

```elixir
# lib/game_store_web/live/game_ratings_live.ex
defmodule GameStoreWeb.GameRatingsLive do
  use GameStoreWeb, :live_component
  alias GameStore.Catalogue

  def update(assigns, socket) do
    {:ok,
     socket
     |> assign(assigns)
     |> assign_games_with_average_ratings()
     |> assign_dataset()
     |> assign_chart()}
  end

  defp assign_chart(%{assigns: %{dataset: dataset}} = socket) do
    socket
    |> assign(:chart, Contex.BarChart.new(dataset))
  end
end
```

调用 `BarChart.new/1` 将创建一个 `BarChart` 结构，描述如何绘制柱状图。现在我们来看看这个结构：

```elixir
%Contex.BarChart{
  axis_label_rotation: :auto,
  category_scale: %Contex.OrdinalScale{
    domain: ["Tic-Tac-Toe", "Ping Pong", "Pictionary"],
    domain_to_range_band_fn: #Function<2.33130404/1 in Contex.OrdinalScale.update_transform_funcs/1>,
    domain_to_range_fn: #Function<1.33130404/1 in Contex.OrdinalScale.update_transform_funcs/1>,
    padding: 2,
    range: {0, 100},
    range_to_domain_fn: #Function<4.33130404/1 in Contex.OrdinalScale.update_transform_funcs/1>
  },
  colour_palette: :default,
  custom_value_formatter: nil,
  data_labels: true,
  dataset: %Contex.Dataset{
    data: [
      {"Tic-Tac-Toe", 3.4285714285714284},
      {"Ping Pong", 2.5714285714285716},
      {"Pictionary", 2.625}
    ],
    headers: nil,
    title: nil
  },
  height: 100,
  mapping: %Contex.Mapping{
    accessors: %{
      category_col: #Function<11.109991709/1 in Contex.Dataset.value_fn/2>,
      value_cols: [#Function<11.109991709/1 in Contex.Dataset.value_fn/2>]
    },
    column_map: %{category_col: 0, value_cols: [1]},
    dataset: %Contex.Dataset{
      data: [
        {"Tic-Tac-Toe", 3.4285714285714284},
        {"Ping Pong", 2.5714285714285716},
        {"Pictionary", 2.625}
      ],
      headers: nil,
      title: nil
    },
    expected_mappings: [category_col: :exactly_one, value_cols: :one_or_more]
  },
  options: [orientation: :vertical],
  orientation: :vertical,
  padding: 2,
  phx_event_handler: nil,
  select_item: nil,
  series_fill_colours: %Contex.CategoryColourScale{
    colour_map: %{1 => "1f77b4"},
    colour_palette: ["1f77b4", "ff7f0e", "2ca02c", "d62728", "9467bd", "8c564b",
     "e377c2", "7f7f7f", "bcbd22", "17becf"],
    default_colour: nil,
    values: [1]
  },
  type: :stacked,
  value_range: nil,
  value_scale: %Contex.ContinuousLinearScale{
    custom_tick_formatter: nil,
    display_decimals: 2,
    domain: {0, 3.4285714285714284},
    interval_count: 9,
    interval_size: 0.4,
    nice_domain: {0.0, 3.6},
    range: {100, 0}
  },
  width: 100
}
```

`BarChart` 有许多可配置的默认选项，所有这些选项都列在 [文档](https://hexdocs.pm/contex/Contex.BarChart.html#summary) 中。例如，我们可以设置方向（默认为垂直）、颜色、padding 等。

我们可以利用暴露的配置函数来更新这些默认值。让我们来看看如何操作图表的颜色。

```elixir
# lib/game_store_web/live/game_ratings_live.ex
defp assign_chart(%{assigns: %{dataset: dataset}} = socket) do
  socket
  |> assign(
    :chart,
    dataset
    |> Contex.BarChart.new()
    |> Contex.BarChart.colours(:warm))
end
```

这将使我们的图表呈现为 "暖色" 配色方案。

需要注意的是，数据集的第一列被用作类别列（即条形图），第二列被用作值列（即条形图高度）。这是通过 `:column_map` 属性来管理的。我们可以看到我们的 `BarChart` 结构的 `:column_map` 值如下：

```elixir
column_map.%{category_col:}。%{category_col: 0, value_cols: [1]}
```

`0` 和 `[1]` 的值指的是我们的 `DataSet` 中元组中元素的指数。`0` 索引的元素将被视为 "类别"，而 `1` 索引的元素将被视为 "值"。我们的元组中，游戏名称位于 0 索引，平均评分位于 1 索引，因此我们的游戏名称将被视为类别，而其平均评分将被视为值。

#### 将图表渲染成 SVG

`Contex.Plot` 模块将我们的数据绘制并渲染成 SVG 标记。我们将添加另一个 reducer `assign_chart_svg` 到我们的管道中，。这个 reducer 将初始化和配置 `Contex.Plot` 并将其渲染为 SVG。然后，它将把这个 SVG 分配给 socket assigns 中的 `:chart_svg` 键。

`Plot` 模块管理图表图的布局--图表标题、轴标签、图例等。我们用图表的宽度和高度以及图表结构来初始化我们的 `Plot`。

```elixir
# lib/game_store_web/live/game_ratings_live.ex
defmodule GameStoreWeb.GameRatingsLive do
  use GameStoreWeb, :live_component
  alias GameStore.Catalogue

  def update(assigns, socket) do
    {:ok,
     socket
     |> assign(assigns)
     |> assign_games_with_average_ratings()
     |> assign_dataset()
     |> assign_chart()
     |> assign_chart_svg()}
  end

  defp assign_chart_svg(%{assigns: %{chart: chart}} = socket) do
    Plot.new(500, 400, chart)
  end
end
```

我们将用一个图表表格和一些 X 轴和 Y 轴的标签来自定义我们的图。

```elixir
defp assign_chart_svg(%{assigns: %{chart: chart}} = socket) do
  Plot.new(500, 400, chart)
  |> Plot.titles("Game Ratings", "average stars per game")
  |> Plot.axis_labels("games", "stars")
end
```

这将（你猜对了），应用标题，字幕和轴标签到我们的图表。

现在我们已经准备好借助 `Plot` 模块的 `to_svg/1` 函数将我们的图转化为 SVG。我们还要确保将生成的 SVG 添加到 socket assigns 中。

```elixir
defp assign_chart_svg(%{assigns: %{chart: chart}} = socket) do
  socket
  |> assign(
    :chart_svg,
    Plot.new(500, 400, chart)
    |> Plot.titles("Game Ratings", "average stars per game")
    |> Plot.axis_labels("games", "stars")
    |> Plot.to_svg())
end
```

现在我们准备在模板中渲染这个 SVG 标记。

#### 在模板中渲染图表

我们的 `GameRatingsLive` 模板非常简单，它渲染存储在 `@chart_svg` 赋值中的 SVG：

```html
<!-- lib/game_store_web/live/game_ratings_live.html.leex -->
<div><%= @chart_svg %></div>
```

现在，当我们导航到 `/admin-dashboard` 时，我们应该看到下面的图表：

![游戏评分图](https://elixirschool.com/assets/game-ratings-chart-c97d10c38c3dc7b6c9ba91d9710f8957b4de80dab0df1218942908c028bc18b6.png)

我要指出的一个 "疑难杂症" 是，为了让列标签（即游戏名称和星级）可见，我不得不应用从 [Contex 示例应用](https://github.com/mindok/contex-samples/blob/master/assets/css/app.css#L6-L52) 中借用自定义 CSS，并复制到下面：

```css
.exc-tick {
  stroke: grey;
}

.exc-tick text {
  fill: grey;
  stroke: none;
}

.exc-grid {
  stroke: lightgrey;
}

.exc-legend {
  stroke: black;
}

.exc-legend text {
  fill: grey;
  font-size: 0.8rem;
  stroke: none;
}

.exc-title {
  fill: darkslategray;
  font-size: 2.3rem;
  stroke: none;
}
.exc-subtitle {
  fill: darkgrey;
  font-size: 1rem;
  stroke: none;
}

.exc-domain {
  stroke: rgb(207, 207, 207);
}

.exc-barlabel-in {
  fill: white;
  font-size: 0.7rem;
}

.exc-barlabel-out {
  fill: grey;
  font-size: 0.7rem;
}
```

现在我们的图表已经呈现出漂亮的效果，我们来谈谈实时更新的问题。

## 实时更新

好消息是在 LiveView 中渲染我们的图表将获得免费的实时更新！如果服务器端发生任何状态变化，图表将自动用任何新数据重新渲染。例如，我们可以想象，利用 PubSub 在每次用户提交新的游戏评级时向父 `AdminDashboardLive` 发送一条消息。然后，`AdminDashboard` 又可以利用 `send_update/2` 函数更新子 `GameRatingsLive` 组件，使其重新渲染并从数据库中重新获取带有平均评分数据的游戏，从而渲染出带有最新游戏评分的更新图表。通过这种方式，LiveView 可以以一种反映并受分布式应用整体状态影响的方式来管理我们单页的状态。与 PubSub 和 LiveView 的合作有点超出了本篇文章的范围，但你可以在我们之前关于这个主题的 [文章](./2019-04-11-live-view-with-pub-sub.md) 中了解更多信息。

除了我们的图表将受益于免费的实时更新，仅仅凭借在 LiveView 中的渲染，Contex 库确实允许我们添加事件处理程序到图表本身。`BarChart` 模块公开了一个函数， [`event_handler/2`](https://hexdocs.pm/contex/Contex.BarChart.html#event_handler/2)，它为图表中的每个条形元素附加了一个 `phx-click` 属性。

我们将使用这个函数来实现以下功能：

> 当用户点击我们图表中的一个给定的条形元素时。
> 然后，该栏被高亮显示

类似这样：

![游戏评分表选择类别](https://elixirschool.com/assets/game-ratings-bar-chart-selected-206c41ec90ed935dbde4ce64b2bc74fcc415c364d9c04ad48663a3939c94bd3f.png)

我们将通过使用 `BarChart.event_handler/2` 函数为图表中的条形图添加 `phx-click` 事件。

```elixir
# lib/game_store_web/live/game_ratings_live.ex
defp assign_chart(%{assigns: %{dataset: dataset}} = socket) do
  socket
  |> assign(
    :chart,
    dataset
    |> Contex.BarChart.new()
    |> Contex.BarChart.colours(:warm))
    |> Contex.BarChart.event_handler("chart-bar-clicked")
end
```

现在让我们看看当我们点击图表中给定的条形图时，会发生什么，并检查我们的服务器日志。

```elixir
[error] GenServer #PID<0.617.0> terminating
** (UndefinedFunctionError) function GameStoreWeb.AdminDashboardLive.handle_event/3 is undefined or private
GameStoreWeb.AdminDashboardLive.handle_event("chart-bar-clicked", %{"category" => "Tic-Tac-Toe", "series" => "1", "value" => "3.4285714285714284"}, #Phoenix.LiveView.Socket<assigns: %{flash: %{}, live_action: :index, live_module: GameStoreWeb.AdminDashboardLive, survey_results_component_id: "survey-results"}, changed: %{}, endpoint: GameStoreWeb.Endpoint, id: "phx-FjuQogS0osBJcgnD", parent_pid: nil, root_pid: #PID<0.617.0>, router: GameStoreWeb.Router, view: GameStoreWeb.AdminDashboardLive, ...>)
```

哦，不！我们的父 LiveView 崩溃了。因为我们还没有为我们的 `"chart-bar-clicked"` 事件实现 `handle_event/3` 函数。你会注意到这个事件被发送到父 LiveView，`GameStoreWeb.AdminDashboardLive`，而不是我们的 `GameRatingsLive` 组件。这是因为，为了将一个事件发送到一个组件，而不是它的父组件，有必要将 `phx-target=<%= @myself %>` 属性添加到包含 `phx-click` 事件的元素（或其他 DOM 元素绑定）。`@myself` 赋值是指当前组件的唯一标识符。

然而，Contex 包并不允许我们通过调用 `BarChart.event_handler/2` 来指定事件目标。有一个 [open issue](https://github.com/mindok/contex/issues/29) 就是关于这个工作的，如果有读者有兴趣的话，可以贡献一下。

所以，我们需要：

- 在父 LiveView 中实现 `handle_event/3` 函数，`AdminDashboardLive`。
- 当父 LiveView 收到该事件时，获取父 LiveView 向子组件发送更新。
- 教会子组件 `GameRatingsLive` 用 "category selection" 来渲染 SVG 图表。

我们开始吧!

我们将从 `handle_event/3` 函数开始。该函数将模式匹配 `"chart-bar-clicked"` 事件名称，并使用 `send_update/2` 函数告诉 `GameRatingsLive` 组件重新渲染：

```elixir
# lib/game_store_web/live/admin_dashboard_live.ex
defmodule GameStoreWeb.AdminDashboardLive do
  alias GameStoreWeb.GameRatingsLive
  def handle_event("chart-bar-clicked", payload, socket) do
    send_update(
      GameRatingsLive,
      socket.assigns.game_ratings_component_id,
      selected_category: payload)

    {:noreply, socket}
  end
end
```

`send_update/2` 调用的是我们要更新的组件名称和一个关键字列表，这个关键字列表将作为新的赋值传递给更新的组件。关键字列表 _必须_ 包含我们要更新的 ID。在这里，我们将组件的 ID 从 socket assigns 中提取出来，我们在这篇博客文章的第一部分将其存储在那里。

通过调用 `send_update/2`，我们将使 `GameRatingsLive` 组件重新渲染并重新调用 `update/2` 回调，这次调用的 `assigns` 包括我们的`:selected_category` 键，指向点击事件的 payload。我们将在 LiveView 一书中介绍 `send_update/2` 函数，以及更多子组件和父 LiveViews 之间的通信选项。现在，我们只需要了解 `send_update/2` 可以从父 LiveView 中调用，告诉正在运行的组件更新。

现在我们已经准备好教 `GameRatingsLive` 组件如何渲染一个带有选定类别的 Contex `BarChart`。我们可以借助 [`BarChart.select_item/2`](https://hexdocs.pm/contex/Contex.BarChart.html#select_item/2) 来实现这个功能。这个函数有两个参数，一个是当前的 `BarChart` 结构，另一个是类似这样的映射：

```elixir
%{category: category, series: series}
```

幸运的是，这只是在点击事件 payload 中发送的数据，现在这些数据在 assigns 中的 `:selected_category` 键下是可用的。

让我们更新组件的 `assign_chart/1` 函数来使用 assigns 中的 `:selected_category` 信息（如果存在的话），并将其应用到条形图中。

```elixir
# lib/game_store_web/live/game_ratings_live.ex
defp assign_chart(%{assigns: %{dataset: dataset}} = socket) do
  socket
  |> assign(
    :chart,
    dataset
    |> Contex.BarChart.new()
    |> Contex.BarChart.colours(:warm))
    |> Contex.BarChart.event_handler("chart-bar-clicked")
    |> maybe_select_category()
end

defp maybe_select_category(
      chart,
      %{assigns: %{selected_category: %{"category" => category, "series" => series}}} = socket
    ) do
  chart
  |> BarChart.select_item(%{category: category, series: String.to_integer(series)})
end

defp maybe_select_category(chart, _socket) do
  chart
end
```

在这里，我们在图表创建管道中添加了一个条件性的 reducer `maybe_select_category/2`。如果它被包含 `:selected_category` 赋值的 socket 调用(如果组件是由接收到点击事件的父节点更新的情况下)，那么它将把选择的类别值应用到 `BarChart.select_item/2` 函数中。否则，它将简单地返回没有变化的图表。

现在，如果浏览器指向 `/admin-dashboard`，并点击一个给定的条形图类别栏，我们应该会看到它被适当地高亮显示了!

## 结语

我们可以看到，Contex 是 Elixir 中服务器端 SVG 图表的一个强大而灵活的工具。最重要的是，它能无缝集成到我们的 LiveView 中，适应实时更新，甚至允许我们将 `phx-click` 事件附加到图表元素上。我希望看到 Contex 库进一步发展，并鼓励任何阅读者试用它并考虑做出贡献。

除了看 Contex，我们在这里还涉及了很多 LiveView 的概念。我们看了一下核心/边界应用设计是如何在我们的 LiveView 功能中发挥作用的，我们利用了有状态的组件，并看到了父 LiveViews 如何与它们的子组件进行通信，我们还写了一些漂亮的、有组织的 LiveView 代码，利用 reducers 建立 socket 状态。如果想更深入地了解这些概念和更多的内容，不要忘了查看 Pragmatic Bookshelf 即将出版的 LiveView 书籍，并留意 Elixir School 的 LiveView 书籍赠品。

## 资源介绍

想要更近一步了解 Contex：

- [Contex 官网](https://contex-charts.org/)
- [Contex 示例应用](https://github.com/mindok/contex-samples)

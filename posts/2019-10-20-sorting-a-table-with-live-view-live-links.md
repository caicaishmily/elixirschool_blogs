---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
tags: ['live view']
date: 2019-10-20
layout: post
title: Building a Table Sort UI with Live View's `live_link`
excerpt: >
  We'll use LiveView's `live_link/2` together with the `handle_params/3` callback to allow users to sort a table in real-time.
---

# 利用 Live View 的 `live_link` 构建表格排序 UI

LiveView 可以轻松解决一些最常见的 UI 挑战，几乎不需要前端代码。它让我们可以把 JavaScript 省下来，用于复杂而精密的 UI 更改。在构建最近的一个面向管理员的视图时，包括 Flatiron 学校的学生群体表，我发现自己需要用到 LiveView。只需几行后台代码，我的可分类表格就可以运行了。继续阅读，看看你将如何利用 LiveView 的 `live_link/2` 和 `handle_params/3` 来构建这样一个功能。

## 功能

我们的视图呈现了一个学生群体的表格，看起来像这样。

![实时视图表](https://elixirschool.com/assets/live-view-table-de7e10060f002103cea071882feaa96a21c4c04777a4173a1f6953ff68340719.png)

用户需要能够按学生姓名、校区、开始日期或状态对该表进行排序。我们还希望确保 "排序" 属性包含在 URL 的查询参数中，这样用户就可以共享排序视图的链接。

下面是我们要实现的行为。请注意，当我们点击给定的列标题对表格进行排序时，URL 是如何变化的。

<iframe width="560" height="315" src="https://www.youtube.com/embed/-4VRaX1uEhk" frameborder="0" allow="accelerometer; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## 使用 `live_link/2`

LiveView 的 [`live_link/2`](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html#module-live-navigation) 函数允许使用浏览器的 [pushState API](https://developer.mozilla.org/en-US/docs/Web/API/History_API) 进行页面导航。这将确保该 URL 将改变以包含我们在给定的 `live_link/2` 调用中包含的任何参数。

然而，在我们继续之前，有一件重要的事情要注意。为了使用实时导航功能，我们的 live view 需要直接挂载在路由器中，而不是从控制器动作中渲染。

我们的路由器是这样挂载 live view 的。

```elixir
# lib/course_conductor_web/router.ex

scope "/", CourseConductorWeb do
  pipe_through([:browser, :auth])
  live "/cohorts", CohortsLive
end
```

我们已经准备好开始了！

我们先将 `"Name"` 表头变成一个实时链接。

```html
# lib/course_conductor_web/templates/cohorts/index.html.leex
<table>
  <th><%= live_link "Name", to: Routes.live_path(@socket, CourseConductorWeb.CohortsLive, %{sort_by: "name"}) %></th>
  ...
</table>
```

`live_link/2` 函数为基于 HTML5 的 pushState 导航生成一个实时链接，*不需要* 页面重新加载。

在 `Routes.live_path` 助手的帮助下，我们生成了以下的实时链接。`"/cohorts?sort_by=name"`。由于这个路由属于我们已经挂载的 `CohortsLive` live view，而且由于该 LiveView 是在我们的路由器中定义的(而不是从控制器动作中渲染的)，这意味着我们将调用我们现有的实时视图的 `handle_params/3` 函数 _而无需挂载一个新的实时视图_。非常酷!

让我们看看现在如何实现 `handle_params/3` 函数。

## 实现 `handle_params/3`
1
`handle_params/3` 回调在两种情况下被调用。

* 在 `mount/2` 被调用后（即实时视图首次渲染时）。
* 当发生实时导航事件时，比如点击实时链接。这第二种情况只有当如上所述，我们链接到的实时视图与我们当前所处的实时视图相同时，才会触发这个回调，而且实时视图是在路由器中定义的。 

`handle_params/3` 接收三个参数。
* 查询参数
* 请求的地址
* socket

我们可以使用 `handle_params/3` 来更新 socket 状态，从而触发服务器对模板的重新渲染。

鉴于 `handle_params/3` 将被我们的 liveview 调用，每当我们的 `"Name"` 实时链接被点击时，我们需要在我们的实时视图中实现这个函数，以匹配和执行我们的实时链接将发送的 `sort_by` 参数。

假设我们有下面的实时视图，它可以加载和渲染一个同学列表。

```elixir
# lib/course_conductor_web/live/cohorts_live.ex

defmodule CourseConductorWeb.CohortsLive do
  use Phoenix.LiveView

  def render(assigns) do
    Phoenix.View.render(CourseConductorWeb.CohortView, "index.html", assigns)
  end

  def mount(_, socket) do
    cohorts = Cohort.all_cohorts()
    {:ok, assign(socket, cohorts: cohorts)}
  end
end
```

像这样实现我们的 `handle_params/3`：

```elixir
#  lib/course_conductor_web/live/cohorts_live.ex

def handle_params(%{"sort_by" => sort_by}, _uri, socket) do
  case sort_by do
    sort_by
    when sort_by in ~w(name) ->
      {:noreply, assign(socket, cohorts: sort_cohorts(socket.assigns.cohorts, sort_by))}

    _ ->
      {:noreply, socket}
  end
end

def handle_params(_params, _uri, socket) do
  {:noreply, socket}
end

def sort_cohorts(cohort, "name") do
  Enum.sort_by(cohorts, fn cohort -> cohort.name end)
end
```

注意，我们已经包含了 `handle_params/3` 函数的 "catch-all" 版本，如果有人浏览到 `/cohorts`，并且包含了与我们关心的 `"sort_by"` 参数不匹配的查询参数，那么该函数将被调用。如果我们的实时视图收到这样的请求，它将不会更新状态。

现在，当用户点击 `"Name"` 实时链接时，会发生两件事。

* 浏览器的 pushState API 将被调用，将 URL 改为 `/cohorts?sort_by=name`。
* 我们已经挂载的 live view `handle_params/3` 函数将被调用，参数 `%{"sort_by" => "name"}`。

我们的 `handle_params/3` 函数将按照 cohort 名称对存储的 `socket.assigns` 的 cohort 进行排序，并根据排序后的列表更新 socket 状态。因此，模板将用排序后的列表重新渲染。

由于 `handle_params/3` 在 `mount/2` 之后 _也_ 被调用，因此我们允许用户通过浏览器直接导航到 `/cohorts?sort_by=name`，并在实时视图中看到已经按名称排序的同组表。就这样，我们让用户以零负担的代码来分享排序表视图的链接。

## 更多排序!

现在，我们的 "按名称排序" 功能已经启动并运行，让我们添加其余的实时链接，以允许用户按照我们之前列出的其他属性进行排序：校园、开始日期和状态。

首先，我们将把这些表头中的每一个都变成一个实时链接。

```html
<table>
  <th><%= live_link "Name", to: Routes.live_path(@socket, CourseConductorWeb.CohortsLive, %{sort_by: "name"}) %></th>
  <th><%= live_link "Campus", to: Routes.live_path(@socket, CourseConductorWeb.CohortsLive, %{sort_by: "campus"}) %></th>
  <th><%= live_link "Start Date", to: Routes.live_path(@socket, CourseConductorWeb.CohortsLive, %{sort_by: "start_date"}) %></th>
  <th><%= live_link "Status", to: Routes.live_path(@socket, CourseConductorWeb.CohortsLive, %{sort_by: "status"}) %></th>
</table>
```

我们将更新我们的 `handle_params/3` 函数，对描述这些属性的参数进行操作。

```elixir
def handle_params(%{"sort_by" => sort_by}, _uri, socket) do
  case sort_by do
    sort_by
    when sort_by in ~w(name course_offering campus start_date end_date lms_cohort_status) ->
      {:noreply, assign(socket, cohorts: sort_cohorts(socket.assigns.cohorts, sort_by))}

    _ ->
      {:noreply, socket}
  end
end
```

在这里，我们添加了一个检查，以查看 `sort_by` 属性是否包含在我们的可排序属性列表中。

```elixir
when sort_by in ~w(name course_offering campus start_date end_date lms_cohort_status)
```

如果是这样，我们将继续对同族进行排序。如果没有，即如果用户将浏览器指向 `/cohorts?sort_by=not_a_thing_we_support`，那么我们将忽略 `sort_by` 的值，并避免更新 socket 状态。

接下来，我们将为 `sort_cohorts/2` 函数添加必要的版本，它将针对我们新的 "排序" 选项进行模式匹配。

```elixir
def sort_cohorts(cohorts, "campus") do
  Enum.sort_by(cohorts, fn cohort -> cohort.campus.name end)
end

def sort_cohorts(cohorts, "start_date") do
  Enum.sort_by(
    cohorts,
    fn cohort -> {cohort.start_date.year, cohort.start_date.month, cohort.start_date.day} end,
    &>=/2
  )
end

def sort_cohorts(cohorts, "status") do
  Enum.sort_by(cohorts, fn cohort ->
    cohort.status
  end)
end
```

就是这样！

## 结语

LiveView 再一次让构建无缝实时用户界面变得简单。因此，虽然 LiveView 并不意味着你再也不用写 JavaScript 了，但它确实意味着我们不需要利用 JavaScript 来应对常见的日常挑战，比如在 UI 中进行数据排序。我们不需要编写复杂的 vanilla JS，也不需要使用强大的前端框架，而是能够使用大部分后端代码创建一个复杂的实时 UI，并以强大的 Elixir 容错流程作为后盾。
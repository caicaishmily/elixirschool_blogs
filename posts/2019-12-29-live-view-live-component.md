---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
tags: ['live view']
date: 2019-12-29
layout: post
title: LiveView Design Patterns - LiveComponent and the Single Responsibility Principle
excerpt: >
  It's easy to end up with an overly complex LiveView that houses lots of business rules and responsibilities. We can use `Phoenix.LiveComponent` to build a LiveView feature that is clean, maintainable and adherent to the Single Responsibility Principle.
---

# LiveView 设计模式 - LiveComponent 和单一责任原则
## LiveView 可能会变得混乱

随着 LiveView 成为一项更加成熟的技术，我们自然会发现自己使用它来支持越来越多的复杂功能。如果我们不小心，可能会导致 "控制器臃肿综合症" -- live view 中塞满了复杂的业务逻辑和不同的职责，就像经典的 "Rails 肥胖控制器"。

我们如何才能在遵守 SRP 等通用设计原则的同时，编写出易于推理和维护的实时视图呢？

实现这一目标的一种方法是利用 `Phoenix.LiveComponent` 行为。

## `Phoenix.LiveComponent` 简介

> 组件是使用 `Phoenix.LiveComponent` 行为的模块。该行为提供在 LiveView 中对状态、标记和事件进行分类的机制。--[文档](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveComponent.html)

组件通过对 `Phoenix.LiveView.live_component/3` 的调用在 live view 父进程内运行。由于它们与父组件 live view 共享一个进程，因此两者之间的通信非常简单（稍后再谈）。

组件可以是无状态或有状态的。无状态的组件除了渲染一个特定的 `leex` 模板之外，不会做更多的事情，而有状态的组件实现了一个 `handle_event/3` 函数，允许我们更新组件自己的状态。这使得组件成为从过于复杂的实时视图中剥离责任的好方法。

让我们来看看我们如何使用组件来重构现有应用程序中的一些复杂的 LiveView 代码。

## 应用程序

假设我们有一个应用程序，它使用像 RabbitMQ 这样的消息代理在系统之间发布和消费消息。我们的应用程序将这些消息持久化在 DB 中，并为用户提供一个 UI，以列出和搜索这些持久化的消息。

![live view messages index](https://elixirschool.com/assets/live-view-messages-index-cfb8d1e20fe495a1ccb66afa03b7e7a3e7ab6404ab6debd99ab631c1a69b31f6.png)

我们使用 LiveView 来实现搜索功能、分页功能，并维护当前显示哪些消息的状态。我们的 live view 模块响应搜索表单事件，并维护搜索表单的状态，处理搜索表单的提交，*并* 渲染各种搜索和分页参数的模板。

### 代码

我们的 live view 的简化版本看起来像这样：

```elixir
defmodule RailwayUiWeb.MessageLive.Index do
  def render(assigns) do
   Phoenix.View.render(RailwayUiWeb.MessageView, "index.html", assigns)
  end

  def mount(_session, socket) do
   socket =
     socket
     |> assign(:page, 1)
     |> assign(:search, %Search{query: nil, value: nil})
     |> assign(:messages, load_messages())

   {:ok, socket}
  end

  def handle_params(
       %{"page" => page_num, "search" => %{"query" => query, "value" => value}},
       _uri,
       %{assigns: %{search: search}} = socket
     ) do
   socket =
     socket
     |> assign(:page, page_num)
     |> assign(:search, Search.update(query, value))
     |> assign(:messages, messages_search(query, value, page_num))

   {:noreply, socket}
  end

  def handle_params(
       %{"page" => page_num},
       _uri,
       %{assigns: %{state: state}} = socket
     ) do
   socket =
     socket
     |> assign(:page, page_num)
     |> assign(:messages, messages_page(page_num))

   {:noreply, socket}
  end

  def handle_params(
       %{"search" => %{"query" => query, "value" => value}},
       _,
       %{assigns: %{search: search}} = socket
     ) do
   socket =
     socket
     |> assign(:search, %Search{query: query, value: value})
     |> assign(:messages, messages_search(query, value))

   {:noreply, socket}
  end

  def handle_params(_params, _, socket) do
   {:noreply, socket}
  end

  def handle_info("search", params, socket) do
   {:noreply,
    live_redirect(socket,
      to: Routes.live_path(socket, __MODULE__, params)
    )}
  end

  def handle_event(
       "search_form_change",
       %{"_target" => ["search", "value"], "search" => %{"value" => value}},
       %{assigns: %{search: search}} = socket
     ) do
   {:noreply, assign(socket, :search, %Search{query: search.query, value: value})}
  end

  def handle_event(
       "search_form_change",
       %{"_target" => ["search", "query"], "search" => %{"query" => query}},
       %{assigns: %{search: search}} = socket
     ) do
   {:noreply, assign(socket, :search, %Search{query: query, value: search.value})}
  end

  def handle_event(
       "search_form_change",
       %{"_target" => ["search", "query"], "search" => %{"value" => _value}},
       socket
     ) do
   {:noreply, socket}
  end
end
```

在状态下保持搜索表单的选择查询和输入值的表示，可以让我们确保选择正确的搜索查询单选按钮，并允许我们更新搜索表单输入字段的占位符文本。

<iframe width="560" height="315" src="https://www.youtube.com/embed/6Ta2Au-PcQI" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

保持搜索表单的状态，还可以保证用户通过一组查询参数直接导航到 `/consumed_messages` 路径，不仅可以看到正确弹出的消息，还可以看到正确配置的搜索表单。

![live component search form query params](https://elixirschool.com/assets/live-component-search-form-query-params-ffb0e6908b686be4cc42356f0ce94e5332d745442d1adf2556d093cdd090da7e.png)

### 问题

很明显，我们需要维护搜索表单的状态，但上面的 LiveView 代码太长，难以维护和推理。它管理搜索表单的状态，实现了一组 `handle_params/3` 回调来执行搜索查询和分页，并在状态中维护了一组消息。这是一个很大的工作，它违反了单一责任原则。简单地说，我们的 live view 做了太多的工作。

让我们把搜索表单的状态维护重构成有自己状态的组件吧!
## 解决方法: 搜索表单组件

我们的搜索表单组件将从父级 live view 中获取其初始搜索表单状态。这将确保用户可以直接导航到像 `/consumed_messages?search[query]=uuid&search[value]=0af71c6a-aeec-431f-83d0-ae779358b055` 这样的路由，并从 params 中看到正确配置的搜索表单。

但是，我们的搜索组件会继续保持搜索表单状态独立于父体，只有在表单提交时才会将消息转发到 live view 中。

这样一来，我们就可以将搜索表单变化事件的处理及其对搜索表单状态的后续影响移出 live view 。这将使我们在减少责任的情况下获得一个更干净的 live view。

### 定义组件
#### 从 LiveView 设置初始化状态

我们先定义我们的组件 `RailwayUiWeb.MessageLive.SearchComponent`，并从父级 live view 中的状态进行初始搜索渲染。

```elixir
defmodule RailwayUiWeb.MessageLive.SearchComponent do
  use Phoenix.LiveComponent

  def render(assigns) do
    Phoenix.View.render(RailwayUiWeb.MessageView, "search_component.html", assigns)
  end
end
```

在这一点上，我们的组件很简单。它使用 `Phoenix.LiveComponent` 行为并实现了 `render/1` 函数。这个函数渲染我们的 `search_component.html.leex` 模板（我们稍后会看一下），通过父 live view 调用 `live_component/3` 时建立的 `assigns`。

现在我们来看看这个调用。在父 live view 的模板中，我们调用。

```html
<%= live_component @socket, RailwayUiWeb.MessageLive.SearchComponent, search: @search, id: :search %>
```

这里有两件重要的事情需要指出。首先，需要注意的是，我们传递了 `:id` 属性并将其设置为 `:search` 原子的值。通过设置 `:id` 属性，组件变得有状态。如果没有这个属性，我们就无法实现 `handle_event/2` 回调。

其次，我们用 `@search` 值填充组件的 `assigns`。此时组件的 `assigns` 是这样的。

```elixir
%{search: search}
```

而来自父 live view 的 `socket.assigns` 的搜索结构将在组件自己的模板中作为 `@search`。

这使得我们可以利用父 live view 中的 `handle_params/3` 回调来建立搜索状态，然后将该搜索状态传递到组件中。让我们来仔细看看这是如何工作的。

1. 用户访问 `/consumed_messages?search[query]=uuid&search[value]=0af71c6a-aeec-431f-83d0-ae779358b055`
2. `MessageLive.Index` live view 的 `handle_params/3` 函数被调用:

```elixir
def handle_params(
      %{"search" => %{"query" => query, "value" => value}},
      _,
      %{assigns: %{search: search}} = socket
    ) do
  socket =
    socket
    |> assign(:search, %Search{query: query, value: value})
    |> assign(:messages, messages_search(query, value))

  {:noreply, socket}
end
```

3. `MessageLive.Index` live view 使用 `@search` 声明渲染模板
4. `MessageLive.Index` 的模板调用 `live_component/3`, 传递 `@search` 声明
5. `MessageLive.SearchComponent` 根据 `@search` 声明正确呈现搜索表单，以反映任何选定的搜索查询类型和输入。

现在让我们来看看组件的模板，以便了解它是如何利用搜索表单状态中的信息进行适当渲染的。
#### 构建搜索表单模板

搜索组件的模板使用 `@search` 赋值的 query 和 value 属性，以确保选择正确的单选按钮，并确保搜索表单输入正确地填充一个值（如果存在）。

```html
<!-- styling removed for brevity -->
<form>
  <div>
    <div>
      <input name="search[query]" value="uuid" type="radio" <%= if @search.query == "uuid", do: "checked" %>>
      <label class="form-check-label">message UUID</label>
    </div>
    <div>
      <input name="search[query]" value="correlation_id" type="radio" <%= if @search.query == "correlation_id", do: "checked" %>>
      <label class="form-check-label">correlation ID</label>
    </div>
    <div>
      <input name="search[query]" value="message_type" type="radio" <%= if @search.query == "message_type", do: "checked" %>>
      <label class="form-check-label">message type</label>
    </div>
  </div>
  <div>
    <input name="search[value]" value="<%= @search.value %>" type="text" placeholder="<%= "search by #{@search.query}"  %>">
  </div>
  <button type="submit" class="btn btn-primary">Submit</button>
</form>
```

这里有些事需要注意：

* `if` 条件，如下面的条件，负责确保选择正确的单选按钮。

```elixir
if @search.query == "message_type", do: "checked"
```

* 搜索表单的输入字段的 `value` 是由 `@search` 赋值的 `value` 属性填充的。

现在我们已经看到了我们的组件是如何呈现它的初始搜索表单状态的，让我们来看看我们的组件将如何处理搜索表单事件。

#### 处理表格变化事件

我们需要更新组件的 `socket.assigns` 来反映两种情况下搜索表单状态的变化。

* 用户选择一个给定的搜索查询（"消息UUID"、"相关ID"、"消息类型"）。
* 用户在搜索表格输入栏中输入一个值。

我们将在表单中添加一个 `phx-change` 事件来捕捉这些交互，并在组件中定义相应的 `handle_event/3` 回调。

```html
<form phx-change="search_form_change">
  ...
</form>
```

我们将添加下面的 `handle_event/3` 回调

```elixir
defmodule RailwayUiWeb.MessageLive.SearchComponent do
  ...

  # update search state when user inputs a search value
  def handle_event(
      "search_form_change",
      %{"_target" => ["search", "value"], "search" => %{"value" => value}},
      %{assigns: %{search: search}} = socket
    ) do
    {:noreply, assign(socket, :search, %Search{query: search.query, value: value})}
  end

  # update search state when user selects a query type radio button
  def handle_event(
        "search_form_change",
        %{"_target" => ["search", "query"], "search" => %{"query" => query}},
        %{assigns: %{search: search}} = socket
      ) do
    {:noreply, assign(socket, :search, %Search{query: query, value: search.value})}
  end
end
```

这些回调为我们确保了两件事。

* 当用户选择一个新的搜索查询类型选项时，正确的单选按钮被标记为 "选定"。
* 搜索表单输入的 `placeholder` 属性被正确更新，以反映所选的查询类型。

```html
<input name="search[value]" value="<%= @search.value %>" type="text" placeholder="<%= "search by #{@search.query}"  %>">
```

#### 处理表格提交

现在，我们表单组件的状态已经可以根据用户的交互正确更新了，我们来谈谈用户提交表单时需要发生的事情。

我们正在设计的功能需要我们在用户提交搜索表单时，在浏览器的 URL 栏中填充查询参数。这样用户就可以共享某个搜索结果与链接。

为了达到这个目的，我们可以使用 `live_redirect/2` 函数。这将利用浏览器的 `pushState` API 来改变页面导航，而不需要实际发送一个 web 请求。取而代之的是，我们的 live view 的 `handle_params/3` 回调函数将被调用，允许我们通过搜索适当的消息和更新 live view socket 的状态来响应。

但是等一下! 很遗憾，由于 `Phoenix.LiveComponent` 行为没有实现 `handle_params/3` 函数，所以 `live_redirect/2` 函数在组件内部无法使用。但幸运的是，父 live view 和组件共享一个进程。这意味着从组件内部调用 `self()` 会返回一个 PID，*这个 PID 与父 live view 进程* 是相同的。因此，在我们的组件中，我们可以 `send` 一个消息到 `self()`，并在父 live view 中处理该消息。

我们将利用这个功能，让我们的组件通过向父 live view 发送消息来处理提交事件中的搜索，指示该 live view 执行实时重定向。

我们首先在组件模板中为搜索表单添加一个 `phx-submit` 绑定事件：

```html
<form phx-submit="search" phx-change="search_form_change">
  ...
</form>
```

然后我们需要为 `"search"` 事件实现一个 `handle_event/3` 函数

```elixir
defmodule RailwayUiWeb.MessageLive.SearchComponent do
  ...

  def handle_event("search", params, socket) do
    send self(), {:search, params}
    {:noreply, socket}
  end
end
```

我们函数中最重要的一部分是这一行：

```elixir
send self(), {:search, params}
```

在这里，我们将发送一个消息 `{:search, params}` ，让父级 live view 可以响应。

最后，我们将在父 live view 中实现一个 `handle_info/2` 回调，它将负责用搜索表单中的 params 执行实时重定向：

```elixir
defmodule RailwayUiWeb.MessageLive.Index do
  ...

  def handle_info({:search, params}, socket) do
    {:noreply,
     live_redirect(socket,
       to: Routes.live_path(socket, __MODULE__, params)
     )}
  end
end
```

这将反过来导致 live view 的 `handle_params/3` 回调被调用，从而正确更新 live view 的状态。

```elixir
defmodule RailwayUiWeb.MessageLive.Index do
  ...

  def handle_params(
       %{"search" => %{"query" => query, "value" => value}},
       _,
       %{assigns: %{search: search}} = socket
     ) do
   socket =
     socket
     |> assign(:search, %Search{query: query, value: value})
     |> assign(:messages, messages_search(query, value))

   {:noreply, socket}
  end
end
```

## 结语

作为这次重构的结果，我们有了一个更干净的 live view 模块，更遵守单一责任原则。我们的 live view 可以专注于给定一组 params 的正确状态的设置。同时，维护搜索表单的状态和适当地呈现搜索表单属性所需的逻辑可以放在一个专门的组件中。

当我们发现自己无法在组件中使用 `live_redirect/2` 时，我们确实遇到了一个障碍。然而，由于组件和 live view 共享一个流程，我们发现很容易在两者之间实现通信。

不过，这种方法还是不能让我们建立一个完全不知道搜索表单状态的 live view 。为了让用户直接导航到带有查询参数的路线，我们的父级 live view 确实设置了搜索表单的初始状态，并将其传递到组件中。不管这个缺点如何，在这里组件已经达到让我们能够编写和维护一个更纤细的 live view。

要想了解 LiveView 提供的其他一些状态、标记和事件处理隔离选项，请查看[文档](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html#module-compartmentalizing-markup-and-events-with-render-live_render-and-live_component)。
---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
tags: ['phoenix']
date: 2019-04-11
layout: post
title:  Building Real-Time Features with Phoenix Live View and PubSub
excerpt: >
  Integrate Phoenix PubSub with LiveView to build real-time features capable of broadcasting updates across a set of clients.
---
# 使用 Phoenix LiveView 和 PubSub 构建实时功能

<!-- TODO: replace href link -->
在一篇 [早期文章](./2019-03-18-phoenix-live-view.md) 中，我们使用了全新的（在写文章时还未发布）Phoenix LiveView 库来构建一个实时功能，只需要很少的后台代码和更少的 JavaScript。LiveView 允许我们通过套接字轻松地将客户端连接到服务器，并将更新推送到客户端。在一个允许用户将 repo "部署" 到 GitHub 的应用中，我们实现了以下实时功能。

<div class="responsive-embed">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/8M-Hjj7IBu8" frameborder="0" allowfullscreen></iframe>
</div>
<br />

但是，当我们有一组客户 _他们所有人_ 都需要看到 _相同的_ 实时更新时，会发生什么？Phoenix Channels 可能看起来很合适，但如果我们能让现有的 LiveView 简单地将更新广播到一组订阅的客户端，那不是很好吗？我们可以使用Phoenix 的 PubSub 模块来实现这一点!

在这篇文章中，我们将学习如何使用 PubSub 将实时更新展示给所有的 LiveView 客户端，而不仅仅是点击 "部署到 GitHub" 按钮的人。我们的成品将是这样的。

<div class="responsive-embed">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/QLwfYNgVuu0" frameborder="0" allowfullscreen></iframe>
</div>
<br />

让我们开始吧！

## PubSub 是什么 & 为什么我们需要它?

PubSub（"publish/subscribe"，即："发布/订阅"）描述了一种模式，在这种模式中，我们将消息发布到一个 "主题"，这样这些消息就可以被任何数量的订阅者消费。在我们的网络应用中，一组连接到我们服务器的客户端成为一个特定主题的订阅者。其中一个特定的客户端（点击 "部署到 GitHub" 按钮的客户端）将向该主题发布或广播消息，以便被其他订阅客户端接收和操作。

[Phoenix PubSub 库](https://hexdocs.pm/phoenix/1.1.0/Phoenix.PubSub.html) 允许我们设置自己的发布/订阅流程。需要注意的是，Phoenix 的 PubSub 库利用了 Elixir 分布式的优势--我们应用的分布式节点上的客户端可以订阅一个共享话题，并向该共享话题广播，因为 PubSub 在配置使用 `Phoenix.PubSub.PG2` 适配器时，可以直接在服务器之间交换通知（后面会详细介绍）。

首先，我们将我们的 LiveView 进程订阅一个共享主题。然后，我们将使用每个 live view 的套接字，在每个订阅者收到该主题的广播时，将变化推送给他们。通过这种方式，我们将结合 LiveView 提供的实时能力，以及 PubSub 提供的在分布式客户端上传递消息的能力。

## 配置 Phoenix PubSub

我们使用 `Phoenix.PubSub.PG2` 适配器配置应用程序的端点。如果我们以这种方式部署的话，这样一来，我们就能够在应用的分布式节点上订阅客户端。在 `config/config.exs` 中以下配置将确保 pubsub 在后台启动，并通过端点模块暴露其功能。

```elixir
config :my_app, MyAppWeb.Endpoint,
  pubsub: [name: MyApp.PubSub, adapter: Phoenix.PubSub.PG2]
  ...
```

接下来，我们将在 LiveView 中教客户端订阅一个共享的主题。
## 订阅一个主题

客户端啥时候应该订阅一个主题呢？我们已经有了一个负责渲染视图，接收点击事件并将更改推送到前端的 LiveView。这个 LiveView 还应该订阅一个共享的主题，并向该主题广播，以便实时更新可以在 LiveView 的所有实例中共享。

当 LiveView 挂载时，每个 LiveView 进程都应该订阅该主题。我们可以通过 `Phoenix.PubSub.subscribe/3` 函数来实现。

```elixir
defmodule MyAppWeb.GithubDeployView do
  use Phoenix.LiveView

  @topic "deployments"

  def render(assigns) do
    MyAppWeb.PageView.render("index.html", assigns)
  end

  def mount(_session, socket) do
    MyAppWeb.Endpoint.subscribe(@topic)
    {:ok, assign(socket, text: "Ready!", status: "ready")}
  end
  ...
end
```

现在我们的 LiveView 实例已经订阅了该主题，我们准备好开始广播。

## 广播至订阅者

### 一个 LiveView 刷新器

什么时候我们需要给订阅客户端广播呢？ 在回答这个问题之前，让我们看一下我们的（稍微重构的）LiveView 代码。 回想一下，我们正在开发一个应用程序，该应用程序允许用户将包含某些内容的存储库 “部署” 到 GitHub。 用户单击一个按钮即可启动几步部署过程（创建组织，创建存储库，推送一些内容）。

因此，当用户单击 LiveView 模板上的 “部署到 GitHub” 按钮时：

```elixir
<div class="">
  <div class="bar">
    <button phx-click="deploy">Deploy to GitHub</button>
    <div class="github-deploy">
      Status: <span class=<%= @status %>><%= @text %></span>
    </div>
  </div>
</div>
```

它将调用 `MyAppWeb.GithubDeployView.handle_event`，第一个参数是我们的 `phx-click` 事件，`"deploy"`。

然后，我们的 live view 将调用一些代码，通过在 `@deployment_steps` 模块属性中查找下一步，并将下一条消息传递给 live view，从而执行部署过程中的每一步。

因此，当用户点击 `deploy` 按钮触发事件时，我们的 `handle_event/3` 函数将通过以下方式进行响应： 

* 查找下一步，`"create-org"`。
* 查找我们想显示的文本，`"创建org"`。
* 向自己发送 `"create-org"` 信息。
* 更新套接字的状态，将 `step` 键指向 `"create-org"`，将 `text` 键指向 `"create org"`。这将导致 live view 的模板用新的文本重新渲染。

向自己发送 `"create-org"` 消息将导致 live view 的 `handle_info/2` 函数被调用。live view 又会查找下一步，将下一步消息传递给自己，并再次更新 socket。一直到消息 `"done"` 为止。

```elixir
defmodule MyAppWeb.GithubDeployView do
  use Phoenix.LiveView
  @deployment_steps %{
    "deploy" => %{next_step: "create-org", text: "Creating org"},
    "create-org" => %{next_step: "create-repo", text: "Creating repo"},
    "create-repo" => %{next_step: "push-contents", text: "Pushing contents"},
    "push-contents" => %{next_step: "done", text: "Done!"}
  }
  @topic "deployments"

  def render(assigns) do
    MyAppWeb.PageView.render("index.html", assigns)
  end

  def mount(_session, socket) do
    {:ok, assign(socket, text: "Ready!", status: "ready")}
  end

  def handle_event(step, _value, socket) do
    text = @deployment_steps[step][:text]
    next_step = @deployment_steps[step][:next_step]
    state = %{text: text, status: step}
    send(self(), next_step)
    {:noreply, assign(socket, state)}
  end

  def handle_info("done", socket) do
    IO.puts "Done!"
    {:noreply, assign(socket, text: "Done!", status: "done")}
  end

  def handle_info(step, socket) do
    IO.puts "HANDLE INFO FOR #{step}..."
    MyApp.GitHubClient.do(step) # our app doing some work, details omitted.
    text = @deployment_steps[step][:text]
    next_step = @deployment_steps[step][:next_step]
    state = %{text: text, status: step}
    send(self(), next_step)
    {:noreply, assign(socket, state)}
  end
end
```

当我们只关心将更新推送到一个 LiveView 进程的套接字上时，这个方法很好用。但是，其他所有加载了我们的 Github Deploy 页面并在自己的 LiveView 进程上操作的用户呢？如果我们希望所有这样的用户都能看到由一个人的按钮点击引起的更新呢？这时我们的 PubSub 代码就能派上用场了。

## 发布广播

每次一个 `GithubDeployView` 实例挂载，我们就订阅 _相同的_ 主题：

```elixir
@topic "deployments"

def mount(_session, socket) do
  MyAppWeb.Endpoint.subscribe(@topic)
  {:ok, assign(socket, text: "Ready!", status: "ready")}
end
```

因此，如果一个给定的 LiveView 进程 *广播* 该主题，我们所有的订阅者都会收到该消息。我们希望我们的 LiveView 每更新其 socket 的状态时，就进行广播。这样，我们就可以告诉所有订阅的 LiveView 进程更新自己的 socket 的状态，然后会导致该 LiveView 的模板重新渲染。流程将像这样工作。

![live_view_pub_sub](https://elixirschool.com/assets/live_view_pub_sub-34d2a313aa84af95d437c7120197633465900ee7a4cdabc49232d7c5c8396c0e.png)

让我们在 LiveView 第一次接收到 `"deploy"` 事件时，以及在它接收到每个后续部署步骤事件时，添加一个广播。

```elixir
defmodule MyAppWeb.GithubDeployView do
  use Phoenix.LiveView
  @deployment_steps %{
    "deploy" => %{next_step: "create-org", text: "Creating org"},
    "create-org" => %{next_step: "create-repo", text: "Creating repo"},
    "create-repo" => %{next_step: "push-contents", text: "Pushing contents"},
    "push-contents" => %{next_step: "done", text: "Done!"}
  }
  @topic "deployments"

  def render(assigns) do
    MyAppWeb.PageView.render("index.html", assigns)
  end

  def mount(_session, socket) do
    {:ok, assign(socket, text: "Ready!", status: "ready")}
  end

  def handle_event(step, _value, socket) do
    text = @deployment_steps[step][:text]
    next_step = @deployment_steps[step][:next_step]
    state = %{text: text, status: step}
    MyAppWeb.Endpoint.broadcast_from(self(), @topic, step, state)
    send(self(), next_step)
    {:noreply, assign(socket, state)}
  end

  def handle_info("done", socket) do
    IO.puts "Done!"
    {:noreply, assign(socket, text: "Done!", status: "done")}
  end

  def handle_info(step, socket) do
    IO.puts "Processing #{step}..."
    MyApp.GitHubClient.do(step) # our app doing some work, details omitted.
    text = @deployment_steps[step][:text]
    next_step = @deployment_steps[step][:next_step]
    state = %{text: text, status: step}
    MyAppWeb.Endpoint.broadcast_from(self(), @topic, step, state)
    send(self(), next_step)
    {:noreply, assign(socket, state)}
  end
end
```

通过使用 `Phoenix.PubSub.broadcast_from/4` 函数，我们向一个主题的所有订阅者广播一条描述新套接字状态的消息，*不包括我们调用广播* 的进程。我们不需要收到点击事件的 live view 向自己广播，因为它已经通过 `send(self(), next_step)` 向自己发送了下一条消息，并且已经通过 `assign(socket, state)` 更新了自己 socket 的状态。

现在我们已经成功地广播了消息，我们需要教 LiveView 如何处理接收的消息。我们可以通过定义另一个 `handle_info/2` 函数来实现，该函数将对广播结构进行模式匹配。

```elixir
def handle_info(%{topic: @topic, payload: state}, socket) do
  IO.puts "HANDLE BROADCAST FOR #{state[:status]}"
  {:noreply, assign(socket, state)}
end
```

当我们的 LiveView 订阅者接收到广播时，将会调用这个 `handle_info/2` 函数。每个订阅者将通过 `assign(socket, state)` 更新自己的 socket，导致每个订阅者的模板重新渲染。

如果我们启动应用程序，打开两个浏览器窗口，然后点击 "部署到 GitHub"，我们应该看到两个浏览器窗口都更新了。

而且我们可以通过 `puts` 语句看到，两个客户端中只有一个客户端接收到了广播，而另一个客户端（发起点击事件的那个客户端）则直接向自己发送消息。

```bash
[info] GET /
[debug] Processing with MyAppWeb.PageController.index/2
  Parameters: %{}
  Pipelines: [:browser]
[info] Sent 200 in 33ms
[info] CONNECT Phoenix.LiveView.Socket
  Transport: :websocket
  Connect Info: %{}
  Parameters: %{"vsn" => "2.0.0"}
[info] Replied Phoenix.LiveView.Socket :ok
[info] Replied phoenix:live_reload :ok
[info] Replied phoenix:live_reload :ok
HANDLE BROADCAST FOR deploy
HANDLE INFO FOR create-org
HANDLE BROADCAST FOR create-org
HANDLE INFO FOR create-repo
HANDLE BROADCAST FOR create-repo
HANDLE INFO FOR push-contents
HANDLE BROADCAST FOR push-contents
Done!
```

## 结语

另一种构建这种广播功能的方法是使用一个 Elixir [Registry](https://hexdocs.pm/elixir/master/Registry.html)。不过它不会给我们提供像 PubSub 那样轻松地跨分布式节点广播的能力，但我很想看到它解决这个问题的实现。

Phoenix PubSub 库允许我们构建一个实时功能，只需额外的五行代码就能将共享更新广播给一组用户。我们的 Phoenix 应用已经被配置为使用 Phoenix PubSub，并且由于一些开箱即用的配置，已经有了 pubsub 后台。事实证明，将它与我们现有的 LiveView 代码集成是非常简单直接的，短时间内就拥有了更多先进的实时功能。
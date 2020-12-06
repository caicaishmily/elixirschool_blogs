---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: til
tags: ['phoenix']
date: 2019-03-19
layout: post
title:  Walk-Through of Phoenix LiveView
excerpt: >
  Learn how to use Phoenix LiveView for real-time features without complicated JS frameworks.
---
# Walk-Through of Phoenix LiveView

它来了！利用服务器渲染的 HTML 和 Phoenix 原生 WebSocket 工具的 Phoenix LiveView 来了，您可以在不使用复杂的 JavaScript 情况下构建花哨的实时功能。如果你厌倦了写JS（我今天和 Redux 的相处的不好，别问了），这就是你要的库。

Phoenix LiveView 是一个全新的产品，所以我想我会提供一个简短的 demo，为任何想要开始尝试它的人构建一个超级简单的演示。请记住，该库仍然是一个候选版本，因此可能会发生变化。

## 什么是 LiveView?

Chris McCord 在 12 月的 [公告](https://dockyard.com/blog/2018/12/12/phoenix-liveview-interactive-real-time-apps-no-need-to-write-javascript) 中已经阐述的很好了。

> Phoenix LiveView 是一个令人兴奋的新库，它通过服务器渲染的 HTML 实现了丰富的实时用户体验。LiveView 驱动的应用程序在服务器上是有状态的，并通过 WebSockets 进行双向通信，与 JavaScript 替代品相比，它提供了一个非常简化的编程模型。

## 杀掉你的 JavaScript

如果您经历了过于复杂的 SPA（例如，Redux 的所有东西），那么您会感觉到所有花哨的 JavaScript 常常伴随着维护和迭代成本。

Phoenix LiveView 感觉非常适合。 90％ 的时间里，您确实希望进行一些实时更新，但实际上并不需要许多现代 JS 框架的破坏球。

让我们启动并运行 LiveView，以支持一项功能，该功能在我们的服务器执行创建 GitHub 存储库的分步过程时推送实时更新。

下面是我们正在构建的功能。

<div class="responsive-embed">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/8M-Hjj7IBu8" frameborder="0" allowfullscreen></iframe>
</div>
<br />

## 起步

Phoenix LiveView 的 [Readme](https://github.com/phoenixframework/phoenix_live_view) 中详细介绍了以下步骤：

* 在你的 `mix.exs` 文件中安装依赖:

```elixir
def deps do
  [
    {:phoenix_live_view, "~> 0.2.0"}
  ]
end
```

* 使用签名盐更新应用程序的端点配置，以供实时查看连接使用：

```elixir
# Configures the endpoint
config :my_app, MyApp.Endpoint,
  ...
  live_view: [
    signing_salt: "YOUR_SECRET"
  ]
```
*注: 你可以在命令行使用 `mix phx.gen.secret` 生成一个秘钥*

* 更新您的配置，以启用扩展名为 `.leex` 的 LiveView 模板。

```elixir
config :phoenix,
  template_engines: [leex: Phoenix.LiveView.Engine]
```

* 在 `：fetch_flash` 之后，将 LiveView.Flash 插件添加到浏览器管道中。

```elixir
pipeline :browser do
  ...
  plug :fetch_flash
  plug Phoenix.LiveView.Flash
end
```

* 将以下内容导入您的 `lib/app_web.ex` 文件中：

```elixir
def view do
  quote do
    ...
    import Phoenix.LiveView, only: [live_render: 2, live_render: 3]
  end
end

def router do
  quote do
    ...
    import Phoenix.LiveView.Router
  end
end
```

* 在端点模块中暴露一个 socket，以供 LiveView 使用：

```elixir

defmodule MyAppWeb.Endpoint do
  use Phoenix.Endpoint

  socket "/live", Phoenix.LiveView.Socket

  # ...
end
```

* 在你的 NPM 依赖中添加 LiveView

```elixir
# assets/package.json
{
  "dependencies": {
    ...
    "phoenix_live_view": "file:../deps/phoenix_live_view"
  }
}
```

在这一步之后你需要运行 `npm install`

* 在 `app.js` 中使用 LiveView JavaScript 库连接到 LiveView socket

```javascript
import LiveSocket from "phoenix_live_view"

let liveSocket = new LiveSocket("/live")
liveSocket.connect()
```

* 你的 live views 应保存在 `lib/my_app_web/live/` 目录中。 为了支持页面实时重新加载，请将以下正则添加到您的 `config/dev.exs`中：

```elixir
config :demo, MyApp.Endpoint,
  live_reload: [
    patterns: [
      ...,
      ~r{lib/my_app_web/live/.*(ex)$}
    ]
  ]
```

现在我们已经准备好构建和渲染一个 live view 了

### 从控制器渲染一个 Live View

您可以 [直接从路由器提供实时视图](https://github.com/phoenixframework/phoenix_live_view/blob/master/lib/phoenix_live_view.ex#L86)。 但是，在此示例中，我们将让我们的控制器渲染 live view。 让我们来看看我们的控制器：

```elixir
defmodule MyApp.PageController do
  use MyApp, :controller
  alias Phoenix.LiveView

  def index(conn, _) do
    LiveView.Controller.live_render(conn, MyAppWeb.GithubDeployView, session: %{})
  end
end
```

我们调用 `live_render/3` 函数，该函数接受 `conn` 的参数，要呈现的 live view 以及我们要发送到实时视图中的任何会话信息。

现在，我们准备定义自己的 live view。

### 定义 Live View

我们的第一个 live view 位于 `my_app_web/live/github_deploy_view.ex` 中。该视图负责处理用户将某些内容 “部署” 到 GitHub 的交互。 此过程涉及创建 GitHub 组织，创建存储库并向该存储库推送一些内容。 就本示例而言，我们将不在乎此过程的实现细节。

我们的 live view 将使用 `Phoenix.LiveView`，并且必须实现两个功能：`render/1` 和 `mount/2`。

```elixir
defmodule MyAppWeb.GithubDeployView do
  use Phoenix.LiveView

  def render(assigns) do
    ~L"""
    <div class="">
      <div>
        <%= @deploy_step %>
      </div>
    </div>
    """
  end

  def mount(_session, socket) do
    {:ok, assign(socket, deploy_step: "Ready!")}
  end
end
```

现在我们已经有了基本的部件，让我们来分析一下 live view 的过程。

### 它如何工作

live view 的连接过程是这样的:

![live_view](https://elixirschool.com/assets/live_view-6a1ff8ddee59b55d1ee0b72dc8d47c55e55bdcaf6b788cc65af31afec66836d3.png)

当我们的应用程序接收到 index 路由的 HTTP 请求时，它将通过渲染 live view 的 `render/1` 函数中定义的静态 HTML 来进行响应。它将通过首先调用我们视图的 `mount/2` 函数来实现这一目的，然后再渲染由 `mount/2` 分配给 socket 的任何默认值填充的 HTML。

渲染的 HTML 将包括已签名的会话信息。会话将使用我们在 `config.exs` 中提供给 live view 配置的签名盐进行签名。当客户端打开 live view socket 连接时，签名的会话将被送回服务器。如果你在浏览器中检查渲染 live view 的页面，你会看到那个签名的会话。

```html
<div
  id="phx-20gvOvqvFMA="
  data-phx-view="MyApp.GithubDeployView"
  data-phx-session="SFMyNTY.g3QAAAACZAAEZGF0YW0AAACQZzNRQUFBQUVaQUFDYVdS">
  ...
</div>
```

一旦渲染了静态 HTML，由于以下代码段，客户端将发送实时 socket 连接请求：

```javascript
import LiveSocket from "phoenix_live_view"

let liveSocket = new LiveSocket("/live")
liveSocket.connect()
```

这将启动一个有状态连接，该状态连接将导致 socket 更新时重新渲染视图。 由于页面首先呈现为静态 HTML，因此即使在浏览器中禁用了JavaScript，我们也可以放心，页面将始终为用户呈现。

现在，我们了解了如何首先呈现 live view 以及如何建立实时视图 socket 连接，让我们渲染一些实时更新。

### 渲染实时更新

LiveView 监听套接字的更新，并且只重新渲染页面中需要更新的部分。 仔细研究我们的 `render/1` 函数，我们看到它渲染了分配给套接字的键的值。

在 `mount/2` 赋值 `:deploy_step` 的情况下，我们的 `render/1` 函数将其渲染为：

```elixir
def render(assigns) do
  ~L"""
  <div class="">
    <div>
      <%= @deploy_step %>
    </div>
  </div>
  """
end
```

*注意: [`~L`](https://github.com/phoenixframework/phoenix_live_view/blob/master/lib/phoenix_live_view.ex#L159) 魔符代表 Live EEx。这是内置的 LiveView 模板。与 `.ex` 模板不同的是，LEEx 模板只能够跟踪和渲染必要的变化。因此，如果 `@deploy_step` 的值发生变化，我们的模板将只重新渲染页面的那一部分。*

让我们给用户提供一种方法来启动 "deploy to GitHub" 的过程，并在每个部署步骤被执行时看到页面更新。

LiveView 支持 DOM 元素绑定，让我们有能力响应客户端事件。我们将创建一个 "deploy to GitHub" 按钮，并使用 phx-click 事件绑定监听该按钮的点击事件。

```elixir
def render(assigns) do
  ~L"""
  <div class="">
    <div>
      <div>
        <button phx-click="github_deploy">Deploy to GitHub</button>
      </div>
      Status: <%= @deploy_step %>
    </div>
  </div>
  """
end
```

`phx-click` 绑定将会把我们的点击事件发送到服务器，由 `GithubDeployView` 处理。在我们的实时视图中，事件由 `handle_event/3` 函数处理。这个函数会接收一个参数，即事件名称、一个值和 socket。

有 [许多方法可以填充 `value` 参数](https://github.com/phoenixframework/phoenix_live_view/blob/master/lib/phoenix_live_view.ex#L228)，但我们在这个例子中不会使用这个数据点。

让我们为 "github_deploy" 事件构建 `handle_event/3` 函数。

```elixir
def handle_event("github_deploy", _value, socket) do
  # do the deploy process
  {:noreply, assign(socket, deploy_step: "Starting deploy...")}
end
```

我们的函数负责两件事。首先，它将启动部署过程（coming soon）。然后，它将更新套接字中 `:deploy_step` 键的值，这将导致我们的模板重新渲染页面中 `<%= @deploy_step %>` 的部分。所以用户将看到 `Status: Ready！` 改为 `Status: Starting deploy...`。

接下来，我们需要 "部署到 GitHub" 的过程能够在每一次更新 socket 的 `:deploy_step`。我们将让视图的 `handle_event/3` 函数向自己发送一条消息，以推送过程中的每一个连续步骤。

```elixir
def handle_event("github_deploy", _value, socket) do
  :ok = MyApp.start_deploy()
  send(self(), :create_org)
  {:noreply, assign(socket, deploy_step: "Starting deploy...")}
end

def handle_info(:create_org, socket) do
  {:ok, org} = MyApp.create_org()
  send(self(), {:create_repo, org})
  {:noreply, assign(socket, deploy_step: "Creating GitHub org...")}
end

def handle_info({:create_repo, org}, socket) do
  {:ok, repo} = MyApp.create_repo(org)
  send(self(), {:push_contents, repo})
  {:noreply, assign(socket, deploy_step: "Creating GitHub repo...")}
end

def handle_info({:push_contents, repo}, socket) do
  :ok = MyApp.push_contents(repo)
  send(self(), :done)
  {:noreply, assign(socket, deploy_step: "Pushing to repo...")}
end

def handle_info(:done, socket) do
  {:noreply, assign(socket, deploy_step: "Done!")}
end
```

*这段代码很简单--我们并不担心部署 GitHub repo 的实现细节，但我们可以想象如何在这个代码流中添加错误处理和其他责任。*

我们的 `handle_event/3` 函数通过发送 `:create_org` 消息给视图本身来启动部署过程。我们的视图通过调用执行该步骤的代码和更新套接字来响应此消息。这将导致我们的模板再次重新渲染，所以用户会看到 `Status: Starting deploy...` 改为 `Status: Creating GitHub org...`。这样，视图就会执行 GitHub 部署过程中的每一步，更新 socket，并使模板每次都重新渲染。

现在我们的实时更新已经开始工作了，让我们把 HTML 代码从 `render/1` 函数中重构一下，放到自己的模板文件中。

### 渲染一个模板文件

我们将模板定义在 `lib/my_app_web/templates/page/github_deploy.html.leex`:

```html
<div>
  <div class>
    <button phx-click="github_deploy">Deploy to GitHub</button>
    <div class="github-deploy">
      Status: <%= @deploy_step %>
    </div>
  </div>
</div>
```

接下来，我们只需要简单地让 live view 的 `render/1` 函数告诉我们的 `PageView` 去渲染这个模板。

```elixir
defmodule MyApp.GithubDeployView do
  use Phoenix.LiveView

  def render(assigns) do
    MyApp.PageView.render("github_deploy.html", assigns)
  end
  ...
end
```

现在我们的代码更有条理了。

## 结语

从这个有限的例子中，我们可以看到这是一个多么强大的产品。我们只用服务器端的代码就能实现这个实时功能，而且服务器端的代码还不多。我真的很喜欢玩 Phoenix LiveView，我很高兴看到其他开发人员用它构建的东西。祝你编码愉快!
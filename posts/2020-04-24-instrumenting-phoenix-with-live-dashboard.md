---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
date: 2020-04-24
layout: post
title: "Instrumenting Phoenix with Telemetry and LiveDashboard"
excerpt: >
  The recent release of the [LiveDashboard](https://github.com/phoenixframework/phoenix_live_dashboard) library allows us to visualize our application metrics, performance and behavior in real-time. In this post, we'll add LiveDashboard to our Phoenix app, examine the out-of-the-box features and take a look under the hood to understand how LiveDashboard hooks into Telemetry events in order to visualize them.
---

# 使用 Telemetry 和 LiveDashboard 对 Phoenix 进行仪表化

最近发布的 [LiveDashboard](https://github.com/phoenixframework/phoenix_live_dashboard) 库允许我们实时地可视化应用程序的指标、性能和行为。在这篇文章中，我们将把 LiveDashboard 添加到我们的 Phoenix 应用中，检查开箱即用的功能，并在了解 LiveDashboard 背后是如何钩入 Telemetry 事件以实现可视化。

## 应用

我们将使用我们为 Telemetry 系列博客文章建立的 Phoenix 应用，[Quantum](https://github.com/elixirschool/telemetry-code-along/tree/live-dashboard)。Quantum 应用并没有做太多事情--它的存在其实只是为了被测量。在这篇文章中，我们将设置一个 Telemetry supervisor，为 Telemetry 事件实现一套度量定义。我们将定义的指标与 Phoenix 和 Ecto 发出的一些开箱即用的 Telemetry 事件相匹配。要深入了解 Telemetry 事件，请查看我们的系列文章 [Instrumenting Phoenix with Telemetry](https://elixirschool.com/blog/instrumenting-phoenix-with-telemetry-part-one/)。本系列即将发布的文章将仔细研究 Phoenix 和 Ecto 提供的开箱即用的 Telemetry 事件，演练如何添加我们自己的 Telemetry 事件等。

### 代码

本演示的最终代码可以在[这里](https://github.com/elixirschool/telemetry-code-along/tree/live-dashboard)找到。

## 添加 LiveDashboard

首先，我们将把 LiveDashboard 依赖添加到我们的 Phoenix 应用中。

```elixir
# mix.exs
def deps do
  [
    {:phoenix_live_dashboard, "~> 0.1"}
  ]
end
```

接下来，我们将确认 LiveView 已经配置了：

```elixir
# config/config.exs
config :quantum, QuantumWeb.Endpoint,
  live_view: [signing_salt: "SECRET_SALT"]
```

然后呢，我们确认我们应用的 `Endpoint` 已经声明了 LiveView 的 socket

```elixir
# lib/quantum_web/endpoint.ex
socket "/live", Phoenix.LiveView.Socket
```

最后，我们将设置从 `/dashboard` 端点到路由器 LiveDashboard 的请求转发。

```elixir
use MyAppWeb, :router
import Phoenix.LiveDashboard.Router

...

if Mix.env() == :dev do
  scope "/" do
    pipe_through :browser
    live_dashboard "/dashboard"
  end
end
```

就是这样! 如果我们运行 `mix deps.get`，然后运行 `mix phx.server`，我们将看到我们的 LiveDashboard 和它开箱即用的监控可视化。

## LiveDashboard 开箱即用

### 首页

![live dashboard home](https://elixirschool.com/assets/ld-home-2d61dc0ffcb8963f2b65e58ca56806b9dbe785ce182b0caaddcfe8eec779d2d4.png)

LiveView 为我们显示了一些开箱即用的监控功能。

在 `Home` 页面，我们可以看到系统信息，比如我们的 Erlang/OTP 版本，Phoenix 版本和 Elixir 版本。我们还可以看到我们的应用所负责的端口、进程和原子的数量等一些信息。

### 进程

![live dashboard processes](https://elixirschool.com/assets/ld-processes-7ca35aed133cdf8783bf2d2a5b4c8c4214ed944a9cec80e1c72b3d499d86b7c8.png)

`Processes` 页面允许我们对应用程序中运行的进程进行检查。我们可以看到一些有用的信息，比如每个进程占了多少内存，甚至某个进程当前正在执行的功能。检查一个给定的进程，我们可以看到更多的信息，包括它的状态，初始函数和堆栈跟踪。

### 端口

![live dashboard ports](https://elixirschool.com/assets/ld-ports-76d718b3ed6d6aeedac08e445387ac547e10ada36e39fadfac545f2748d4c8da.png)

在 `Ports` 页面，我们可以看到我们的应用程序所暴露的端口（负责应用程序的 I/O）。

检查一个给定的端口，我们可以看到哪个进程负责暴露该端口并管理该端口的输入/输出。

![live dashboard port-detail](https://elixirschool.com/assets/ld-port-detail-5f1bda010dde6141a05027693671a301f699c9b0953fad07baecce28e8f93a26.png)

### Sockets

`Sockets` 页面公开了当前由应用程序管理的所有 socket 的信息。Phoenix 应用中的 socket 负责所有 UDP/TCP 流量。在这里，我们甚至可以看到负责监听端口 `:4000` 的 socket 连接。

![live dashboard port-4000-detail]({% asset ld-port-4000-detail.png @path %})

### ETS

LiveDashboard 为我们提供的最后一个开箱即用的页面是 `ETS` 页面。ETS（Erlang Term Storage）是我们的内存存储。我们甚至可以看到 Telemetry handler 表的条目。

![live dashboard ets-detail](https://elixirschool.com/assets/ld-ets-detail-b88172846267d88d9432518fe7fc6b0c454ce3932f8e5e2e8846810b1ec0e388.png)

## LiveDashboard 指标

### 建立指标

有两个 LiveDashboard 功能，我们要做一点工作才能启用。我们将从 [LiveDashboard Metrics](https://hexdocs.pm/phoenix_live_dashboard/metrics.html#content) 开始，它利用了 `telemetry_metrics` 库。

首先，我们将把 `telemetry_metrics` 添加到我们应用程序的依赖中：

```elixir
# mix.exs
{:telemetry_metrics, "~> 0.4"},
```

然后运行 `mix deps.get`

[`Telemetry.Metrics`](https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.html) 库提供了一个接口，用于将 Telemetry 事件转换为度量指标。我们将在稍后的一篇博文中更深入地了解这个库，现在只关注一个高级别的理解。

现在我们已经安装了这个库，我们准备好定义 Telemetry 监督器了。监督器将实现一个 `metrics/0` 函数，声明我们要处理的 Telemetry 事件集，并指定要为这些事件构建哪些度量指标。

```elixir
# lib/quantum/telemetry.ex
defmodule Quantum.Telemetry do
  use Supervisor
  import Telemetry.Metrics

  def start_link(arg) do
    Supervisor.start_link(__MODULE__, arg, name: __MODULE__)
  end

  def init(_arg) do
    Supervisor.init([], strategy: :one_for_one)
  end

  def metrics do
    [
      # Erlang VM Metrics - Formats `gauge` metric type
      last_value("vm.memory.total", unit: :byte),
      last_value("vm.total_run_queue_lengths.total"),
      last_value("vm.total_run_queue_lengths.cpu"),
      last_value("vm.system_counts.process_count"),

      # Database Time Metrics - Formats `timing` metric type
      summary(
        "quantum.repo.query.total_time",
        unit: {:native, :millisecond},
        tags: [:source, :command]
      ),

      # Database Count Metrics - Formats `count` metric type
      counter(
        "quantum.repo.query.count",
        tags: [:source, :command]
      ),

      # Phoenix Time Metrics - Formats `timing` metric type
      summary(
        "phoenix.router_dispatch.stop.duration",
        unit: {:native, :millisecond}
      ),

      # Phoenix Count Metrics - Formats `count` metric type
      counter(
        "phoenix.router_dispatch.stop.count"
      ),

      counter(
        "phoenix.error_rendered.count"
      )
    ]
  end
end
```

在这里，我们使用 `Telemetry.Metrics` 的度量函数(`counter`、`summary` 和 `last_value`)来指定哪些 Telemetry 事件要被视为哪种度量指标。这些函数中的每一个都以 Telemetry 事件为参数，例如，`quantum.repo.query.tquery.total_time`。我们的 `metrics/0` 函数中列出的事件都是由 Phoenix 或 Ecto 源码为我们免费执行的。

`metrics/0` 函数之后会传递给 LiveDashboard LiveView。LiveDashboard 使用这个 Telemetry 事件- metrics 列表，为每个事件附加适当的处理程序。关于如何在 ETS 的帮助下执行和处理 Telemetry 事件，请查看我们的 Telemetry 介绍博客[文章](./2020-04-22-instrumenting-phoenix-with-telemetry-part-one.md)。

我们稍后将重新审视它是如何工作的。首先，让我们将新的监督器添加到我们的应用程序的监督树中。

```elixir
# lib/quantum/application.ex

children = [
  Quantum.Repo,
  Quantum.Telemetry,
  Quantum.Endpoint,
  ...
]
```

现在，我们已经准备好用我们新定义的指标来配置 LiveDashboard。

### 配置 LiveDashboard 指标

我们将在 `live_dashboard` 路由调用中添加以下选项。

```elixir
# lib/quantum_web/router.ex
live_dashboard "/dashboard", metrics: Quantum.Telemetry
```

现在，如果我们访问浏览器中的 `/dashboard`，并点击 `Metrics` 标签，我们将看到我们的指标：

![live dashboard metrics](https://elixirschool.com/assets/ld-metrics-46aeb268368afabb5b7f1d9a66874e6b3d88f16b784a4a2ce5e8dece506f865d.png)

让我们来一探究竟，以便更好地了解这个配置是如何工作的。

### LiveDashboard Metrics 的背后

`live_dashboard` 宏包含下面这一行，它将实时 `/metrics` 请求路由到 `Phoenix.LiveDashboard.MetricsLive` LiveView，会话 payload 包含我们传递的 `metrics.Quantum.Telemetry` 选项：

```elixir
# live_dashboard/router.ex
live "/:node/metrics", Phoenix.LiveDashboard.MetricsLive, :metrics, opts
```

当 `Phoenix.LiveDashboard.MetricsLive` LiveView 启动时，它会使用我们定义在 `Quantum.Telemetry` 中指标相关的参数调用 `Phoenix.LiveDashboard.TelemetryListener.listen/2` 函数。

`Phoenix.LiveDashboard.TelemetryListener` 模块通过在 ETS 中存储事件/处理程序组合，负责将一组处理程序附加到 Telemetry 事件。`Phoenix.LiveDashboard.TelemetryListener` 的 `init/1` 函数对我们在`Quantum.Telemetry` 中定义的指标进行迭代，并在 ETS 中存储每个度量的 Telemetry 事件名称及其自己的 `handle_metrics/4` 函数的处理程序。

```elixir
# live_dashboard/telemetry_listener.ex
def init({parent, metrics}) do
  metrics = Enum.with_index(metrics, 0)
  metrics_per_event = Enum.group_by(metrics, fn {metric, _} -> metric.event_name end)

  for {event_name, metrics} <- metrics_per_event do
    id = {__MODULE__, event_name, self()}
    :telemetry.attach(id, event_name, &handle_metrics/4, {parent, metrics})
  end

  {:ok, %{ref: ref, events: Map.keys(metrics_per_event)}}
end
```

回顾我们之前关于 Telemetry 的文章，`:telemetry.attach/4` 函数在 ETS 中存储了将 Telemetry 事件映射到处理程序的条目。之后，当给定的 Telemetry 事件被执行时（例如，当 Ecto 源代码执行 `"quantum.repo.query.tquery.total_time"` 事件时），Telemetry 将在 ETS 中查找这个名称的事件，并调用存储的处理函数，在本例中 `Phoenix.LiveDashboard.TelemetryListener.handle_metrics/4`。

看看 `Phoenix.LiveDashboard.TelemetryListener.handle_metrics/4` 函数，我们可以看到它做了一些格式化事件度量的工作，然后发送消息到它的 `父亲` -- `Phoenix.LiveDashboard.MetricsLive` LiveView。

```elixir
# live_dashboard/telemetry_listener.ex
def handle_metrics(_event_name, measurements, metadata, {parent, metrics}) do
  time = System.system_time(:second)

  entries =
    for {metric, index} <- metrics do
      if measurement = extract_measurement(metric, measurements) do
        label = tags_to_label(metric, metadata)
        {index, label, measurement, time}
      end
    end

  send(parent, {:telemetry, entries})
end
```

`Phoenix.LiveDashboard.MetricsLive` LiveView 为这个 `{:telemetry, entries}` 事件实现了一个 `handle_info/2` 方法，通过更新 `ChartComponent` 作为响应，导致 UI 更新。

```elixir
# live_dashboard/live/metrics_live.ex
def handle_info({:telemetry, entries}, socket) do
  for {id, label, measurement, time} <- entries do
    data = [{label, measurement, time}]
    send_update(ChartComponent, id: id, data: data)
  end

  {:noreply, socket}
end
```

### 放在一起

让我们来回顾一下所有这些活动部件是如何连接的。

1. 我们定义了一个 Telemetry supervisor，`Quantum.Telemetry`，它实现了一个 `metrics/0` 函数。这个函数负责将一组 Telemetry 事件映射到各种类型的度量。
2. 我们将 Telemetry 监督器 `Quantum.Telemetry` 作为一个选项传递给路由器的 LiveDashboard。
3. LiveDashboard 启动一个 LiveView，`Phoenix.LiveDashboard.MetricsLive`，里面有我们在 `Quantum.Telemetry` 中定义的指标。
4. `Phoenix.LiveDashboard.MetricsLive` 调用 `Phoenix.LiveDashboard.TelemetryListener`，它用自己的 `handle_metrics/4` 处理函数将我们在 `Quantum.Telemetry` 中定义的 Telemetry 事件作为指标存储在 ETS 中。
5. 之后，当这些 Telemetry 事件之一被执行时，Telemetry 调用存储的处理函数 `Phoenix.LiveDashboard.TelemetryListener.handle_metrics/4`。
6. `handle_metrics/4` 函数将事件格式化为指定的度量，并向 `Phoenix.LiveDashboard.MetricsLive` LiveView 发送一条消息，然后更新 UI!

这个过程如果很酷，但这里有很多需要解读的地方。如果想深入了解 ETS 中如何存储 Telemetry 事件、执行和处理 Telemetry 事件，请查看[本帖](./2020-04-22-instrumenting-phoenix-with-telemetry-part-one.md)。

## LiveDashboard 请求记录

最后一个我们要设置的 LiveDashboard 功能是 RequestLogger。这个 LiveDashboard 功能可以让我们记录 LiveDashboard 中所有传入的请求。

### 添加 RequestLogger

我们需要做的就是将以下内容添加到我们的应用程序的 `Endpoint` 模块中，就在 `Plug.RequestId` 之前：

```elixir
# lib/quantum_web/endpoint.ex
plug Phoenix.LiveDashboard.RequestLogger,
  param_key: "request_logger",
  cookie_key: "request_logger"
```

现在我们可以访问浏览器中的 `/dashboard`，点击 `Request Logger` 标签。我们将点击 "enable cookie" 来启用请求日志流的 cookie。现在我们应该看到我们的请求日志流：

![live dashboard request-logger](https://elixirschool.com/assets/ld-request-logger-d47411c199d38e7709e0a9d528c9a9ed1b8d93900a91cdd55e5bc541b0517a8f.png)

让我们简单了解一下 LiveDashboard 背后 RequestLogger 的工作原理。

### LiveDashboard RequestLogger 一探究竟

LiveDashboard 路由挂载一个 LiveView, `Phoenix.LiveDashboard.RequestLoggerLive`:

```elixir
# live_dashboard/router.ex
live "/:node/request_logger",
     Phoenix.LiveDashboard.RequestLoggerLive,
     :request_logger,
     opts
```

`RequestLoggerLive` LiveView 从端点抓取主应用程序的 PubSub 服务器并订阅 "请求记录器" 主题：

```elixir
# live_dashboard/live/request_logger_love.ex

def mount(%{"stream" => stream} = params, session, socket) do
  %{"request_logger" => {param_key, cookie_key}} = session

  if connected?(socket) do
    endpoint = socket.endpoint
    pubsub_server = endpoint.config(:pubsub_server) || endpoint.__pubsub_server__()
    Phoenix.PubSub.subscribe(pubsub_server, Phoenix.LiveDashboard.RequestLogger.topic(stream))
  end

  socket =
    socket
    |> assign_defaults(params, session)
    |> assign(
      stream: stream,
      param_key: param_key,
      cookie_key: cookie_key,
      cookie_enabled: false,
      autoscroll_enabled: true,
      messages_present: false
    )

  {:ok, socket, temporary_assigns: [messages: []]}
end
```

这意味着 `RequestLoggerLive` LiveView 进程订阅了通过 "请求记录器" 主题广播的任何事件。它为 `{:logger, level, message}` 事件实现了一个 `handle_info/2` 函数，并通过更新 socket 和 UI 做出响应：

```elixir
# live_dashboard/live/request_logger_live.ex
def handle_info({:logger, level, message}, socket) do
  {:noreply, assign(socket, messages: [{message, level}], messages_present: true)}
end
```

什么时候 `{:logger, level, message}` 事件会通过 PubSub 广播到这个主题？

当 LiveDashboard 启动时，会给应用程序添加一个 _新_ 的日志后台。

```elixir
# live_dashboard/application.ex
defmodule Phoenix.LiveDashboard.Application do
  @moduledoc false
  use Application

  def start(_, _) do
    Logger.add_backend(Phoenix.LiveDashboard.LoggerPubSubBackend) # HERE!

    children = [
      {DynamicSupervisor, name: Phoenix.LiveDashboard.DynamicSupervisor, strategy: :one_for_one}
    ]

    Supervisor.start_link(children, strategy: :one_for_one)
  end
end
```

`Phoenix.LiveDashboard.LoggerPubSubBackend` 广播了 log 信息到 RequestLogger PubSub topic 无论何时它收到一条 log 事件：

```elixir
# live_dashboard/logger_pubsub_backend.ex
def handle_event({level, gl, {Logger, msg, ts, metadata}}, {format, keys} = state)
    when node(gl) == node() do
  with {pubsub, topic} <- metadata[:logger_pubsub_backend] do
    metadata = take_metadata(metadata, keys)
    formatted = Logger.Formatter.format(format, level, msg, ts, metadata)
    Phoenix.PubSub.broadcast(pubsub, topic, {:logger, level, formatted}) # HERE!
  end

  {:ok, state}
end
```

就是这样！

## 结语

LiveDashboard 为 Phoenix 开发者的工具箱增加了一个强大的工具。现在，我们可以轻松地可视化我们应用程序，并在开发过程中实时监控其性能和行为。LiveDashboard 代表了 "让开发者幸福" 工具这个不断增长的神殿中的又一个入口，它使 Phoenix 和 Elixir 越来越成为 Web 开发的一个引人注目的选择。

我希望这次功能之旅能让你对 LiveDashboard 感到兴奋，同时我们对它的窥探再次说明了 ETS 和消息传递等 Elixir 语言功能是如何让我们有可能构建强大的系统，并且仍然保持简单而优雅。

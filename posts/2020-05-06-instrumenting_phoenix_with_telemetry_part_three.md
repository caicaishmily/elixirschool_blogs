---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
date: 2020-05-06
layout: post
title: "Instrumenting Phoenix with Telemetry Part III: Phoenix + Ecto Telemetry Events"
excerpt: >
  In this series, we're instrumenting a Phoenix app and sending metrics to StatsD with the help of Elixir and Erlang's Telemetry offerings. In Part III we'll examine Phoenix and Ecto's out-of-the-box Telemetry events and use `Telemetry.Metrics` to observe a wide-range of such events.
---

## 目录

In this series, we're instrumenting a Phoenix app and sending metrics to StatsD with the help of Elixir and Erlang's Telemetry offerings.

在这个系列中，我们将借助 Elixir 和 Erlang 的 Telemetry 产品，对一个 Phoenix 应用进行仪表化，并将指标发送到 StatsD。

- [第一部分: Telemetry 的背后](./2020-04-22-instrumenting-phoenix-with-telemetry-part-one.md)
- [第二部分: 用 `TelemetryMetrics` + `TelemetryMetricsStatsd` 处理 Telemetry 事件](./2020-04-29-instrumenting_phoenix_with_telemetry_part_two.md)
- 第三部分: 观测 Phoenix + Ecto Telemetry 事件
- [第四部分: 用 `telemetry_poller`、`TelemetryMetrics` + `TelemetryMetricsStatsd` 对 Erlang VM 进行测量](./2020-05-13-instrumenting-phoenix-with-telemetry-part-four.md)

## 简介

在 [上一篇](./2020-04-29-instrumenting_phoenix_with_telemetry_part_two.md) 中，我们看到了 `Telemetry.Metrics` 和 `TelemetryMetricsStatsd` 库是如何抽象出定义自定义处理程序，将这些处理程序附加到事件上，并实现我们自己的度量报告逻辑。但是我们的 Telemetry 管道仍然需要一点点的工作--我们仍然需要发送所有 Telemetry 事件!

为了真正能够观察我们生产的 Phoenix 应用的状态，我们需要报告的不仅仅是一个端点的请求持续时间和计数。我们需要报告信息丰富的指标，描述整个应用程序的网络请求、数据库查询、Erlang 虚拟机的行为和状态、应用程序中任何工作者的行为和状态等等。

通过在我们需要的地方执行自定义的 Telemetry 事件来手工测量所有这些指标，将是非常繁琐和耗时的。最重要的是，在整个应用程序中标准化事件命名约定、测量和元数据将是一个挑战。

在这篇文章中，我们将研究 Phoenix 和 Ecto 的开箱即用的 Telemetry 事件，并使用 `Telemetry.Metrics` 来观察广泛的此类事件。

## 通过 Phoenix 和 Ecto Telemetry 事件实现可观测性。

为了实现可观察性，我们知道我们需要跟踪以下内容：

- 所有端点的所有请求数量和持续时间，并能通过以下方式查看这些信息：
  - 路由
  - 回复状态
- 所有 Ecto 查询的次数和持续时间，并能按以下方式查看这些信息：
  - 查询命令（如 `SELECT`、`UPDATE`）。
  - 表（如 `Users`）

幸运的是，几乎 _所有_ 这些事件都已经由 Phoenix 和 Ecto 直接发出了。

在下面的教程中，我们将教 `Telemetry.Metrics` 来观察这些开箱即用的事件，并为每个事件格式化适当的指标集，并加上信息丰富的标签。

## 关于格式化指标的说明

在上一篇文章中，我们使用 `TelemetryMetricsStatsd` 报告库来格式化指标，并通过 UDP 将其发送到 StatsD。我们可以用标准格式化或 DogStatsD 格式化来配置这个报表。标准格式化程序可以构建和发送与 StatsD 的 Etsy 实现兼容的度量。这个实现不支持标签，所以 `TelemetryMetricsStatsd` 通过在度量名称中包含标签值来适应我们分配给度量的标签。

例如，如果我们指定以下计数器度量指标：

```elixir
counter(
  "phoenix.request",
  tags: [:request_path]
)
```

执行一个 Telemetry 事件，其中我们传递进来的元数据参数 `conn` 包含 `%{request_path: "/register/new"}`，那么 `TelemetryMetricsStatsd` 将构建一个度量。

```
"phoenix.request.-register-new"
```

如果我们最终想把 StatsD 度量发送到 Datadog，而 Datadog 支持度量标记呢？在这种情况下，我们可以将 `TelemetryMetricsStatsd` 报告器配置为使用 DogStatsD 格式化，它将为上述事件（包括标签）发出以下计数器指标：

```
"phoenix.request:1|c|#request_path:/register/new"
```

在本教程中，我们将使用 DogStatsD 格式化，以便于阅读和理解我们正在构建并发送至 StatsD 的指标和标签。

```elixir
defmodule Quantum.Telemetry do
  use Supervisor
  import Telemetry.Metrics

  def start_link(arg) do
    Supervisor.start_link(__MODULE__, arg, name: __MODULE__)
  end

  def init(_arg) do
    children = [
      {TelemetryMetricsStatsd, metrics: metrics(), formatter: :datadog} # Add the formatter here!
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end

  defp metrics do
    [
      # coming soon!
    ]
  end
end
```

## 起步

你可以克隆下来的 [repo](https://github.com/elixirschool/telemetry-code-along/tree/part-3-start)，跟着这个教程。

- 在分支 [part-3-start](https://github.com/elixirschool/telemetry-code-along/tree/part-3-start) 上查看我们代码的起始状态。
- 在分支 [part-3-solution](https://github.com/elixirschool/telemetry-code-along/tree/part-3-solution) 上找到解题代码。

## Phoenix Telemetry 事件

### `[:phoenix, :router_dispatch, :stop]` 事件

首先，我们将利用一个开箱即用的 Phoenix 事件来帮助我们跟踪和报告 Web 请求数量和持续时间的指标 -- `[:phoenix, :router_dispatch, :stop]` 事件。

`Phoenix.Router` 模块在请求被 Plug 管道和控制器处理后，但在响应呈现之前执行这个事件。在 Phoenix 的源代码中，我们可以 [在这里](https://github.com/phoenixframework/phoenix/blob/d4596650df21e7e0603debcb5f2ad25eb9ac082d/lib/phoenix/router.ex#L357) 看到该事件被发出。

```elixir
# phoenix/lib/phoenix/router.ex

duration = System.monotonic_time() - start
metadata = %{metadata | conn: conn}
:telemetry.execute([:phoenix, :router_dispatch, :stop], %{duration: duration}, metadata)
```

在这里，Phoenix 正在计算持续时间，从当前时间中减去在请求处理管道开始时设置的开始时间。然后，它在更新元数据 map，以包含 `conn`。最后，它在用这些信息执行 Telemetry 度量。

现在我们知道了我们关心的是哪个 Telemetry 事件，让我们的 `Telemetry.Metrics`、`Quantum.Telemetry` 模块也知道它。

```elixir
# lib/quantum/telemetry.ex

def metrics do
  [
    summary(
      "phoenix.router_dispatch.stop.duration",
      unit: {:native, :millisecond},
      tags: [:plug, :plug_opts]
    ),

    counter(
      "phoenix.router_dispatch.stop.count",
      tags: [:plug, :plug_opts]
    )
  ]
end
```

现在，每当 `Phoenix.Router` 为给定的 Web 请求执行 `[:phoenix, :router_dispatch, :stop]` Telemetry 事件时，我们将看到向 StatsD 发出的计数器和定时器指标，这些指标被事件元数据中的 `:plug` 和 `:plug_opts` 值标记。

通过运行应用程序并访问登陆页面，我们会看到向 StatsD 报告的以下内容：

```
# timing
"phoenix.router_dispatch.stop.duration:5.411|ms|#plug:Elixir.QuantumWeb.PageController,plug_opts:index"

# counter
"phoenix.router_dispatch.stop.count:1|c|#plug:Elixir.QuantumWeb.PageController,plug_opts:index"
```

与我们以前从应用程序中的每个控制器中手动执行 Telemetry 事件的方法相比，这对我们来说是一个巨大的胜利。通过在我们的 `Quantum.Telemetry` 模块中定义该事件的指标，并将这些指标提供给 `TelemetryMetricsStatsd` 报告器，我们能够报告我们的应用程序在所有端点上收到的每个网络请求的指标。

### 从标签中获取更多信息

我们还可以看到度量函数的 `:tags` 选项是多么有用。这些标签确保了我们向 StatsD 报告的指标是信息丰富的--它们包含了来自网络请求上下文的数据。从上一篇文章中可以回想一下，`TelemetryMetricsStatsd` 报告器将为给定的度量应用标签，这些标签作为键存在于事件元数据中。因此，当 `Phoenix.Router` 执行以下 Telemetry 事件时：

```elixir
# phoenix/lib/phoenix/router.ex

duration = System.monotonic_time() - start
metadata = %{metadata | conn: conn}
:telemetry.execute([:phoenix, :router_dispatch, :stop], %{duration: duration}, metadata)
```

它向 `:telemetry.execute/3` 调用提供 `metadata`，其中包括 `:plug` 和 `:plug_opts` 的顶层键。

它 _还_ 在元数据映射中包含了请求 `conn`，键为 `:conn`。如果我们想从 `conn` 中抓取一些数据来包含在我们的度量标签中呢？

如果我们能用响应状态来标记这些 web 请求计数器指标，那就太好了--这样我们就可以汇总成功和失败的 web 请求的数量。

响应状态存在于 `conn` 中，键为 `:status`。但是 `Telemetry.Metrics.counter/2` 函数只知道如何处理所提供的 `metadata` 中的顶级标签。如果有办法告诉计数器度量如何应用嵌套在所提供元数据中的数据中的标签就好了。

这就是度量函数的 `:tag_values` 选项的用武之地。我们可以使用 `:tag_values` 选项来存储一个函数，该函数将在稍后的 Telemetry 事件处理过程中被调用，以从嵌套的元数据信息中构建额外的标签。

_我们_ 所要做的就是实现一个函数，期望接收事件元数据，并返回一个包含所有我们想要应用到度量的标签的映射。

```elixir
# lib/quantum/telemetry.ex

def endpoint_metadata(%{conn: %{status: status}, plug: plug, plug_opts: plug_opts}) do
  %{status: status, plug: plug, plug_opts: plug_opts}
end
```

然后，当我们调用一个给定的度量函数时，例如 `counter/2`，我们将 `:tag_values` 选项设置为这个函数，`:tags` 设置为我们完整的标签列表。

```elixir
# lib/quantum/telemetry.ex

def metrics do
  [
    counter(
      "phoenix.router_dispatch.stop.count",
      tag_values: &__MODULE__.endpoint_metadata/1,
      tags: [:plug, :plug_opts, :status]
    )
  ]
end
```

现在，当我们运行 Phoenix 服务器并访问登陆页面时，我们看到以下计数器指标被发送到 StatsD：

```
"phoenix.router_dispatch.stop.count:1|c|#plug:Elixir.QuantumWeb.PageController,plug_opts:index,status:200"
```

请注意，现在指标被标记为响应状态。这将使我们很容易在 Datadog 中可视化失败和成功请求的计数。

### 更多 Phoenix Telemetry 事件

到目前为止，我们只是利用了 Phoenix 源代码执行的几个 Telemetry 事件中的一个。我们可以让我们的 Telemetry 管道处理一些有用的事件。现在让我们简单地看看其中的一些事件。

#### `[:phoenix, :error_rendered]` Telemetry 事件

`Phoenix.Endpoint.RenderErrors` 模块在渲染错误视图后执行一个 Telemetry 事件。我们可以在 [源代码](https://github.com/phoenixframework/phoenix/blob/00a022fbbf25a9d0845329161b1bc1a192c2d407/lib/phoenix/endpoint/render_errors.ex#L81) 中看到执行该事件的调用。

```elixir
# phoenix/lib/phoenix/endpoint/render_errors.ex

defp instrument_render_and_send(conn, kind, reason, stack, opts) do
  start = System.monotonic_time()
  metadata = %{status: status, kind: kind, reason: reason, stacktrace: stack, log: level}

  try do
    render(conn, status, kind, reason, stack, opts)
  after
    duration = System.monotonic_time() - start
    :telemetry.execute([:phoenix, :error_rendered], %{duration: duration}, metadata)
  end
end
```

我们可以告诉我们的 Telemetry 管道把这个事件作为一个计数器来处理，并在我们的 `Quantum.Telemetry.metrics/0` 函数中用请求路径和响应状态来标记它，像这样：

```elixir
# lib/quantum/telemetry.ex

def metrics do
  [
    counter(
      "phoenix.error_rendered.count",
      tag_values: &__MODULE__.error_request_metadata/1,
      tags: [:request_path, :status]
    )
  ]
end

def error_request_metadata(%{conn: %{request_path: request_path}, status: status}) do
  %{status: status, request_path: request_path}
end
```

现在，我们会看到当用户访问 `/blah` ，一个不存在的路径时，StatsD 中的以下计数器指标会递增：

```
"phoenix.error_rendered.count:1|c|#request_path:blah,status:404"
```

#### `Phoenix.Socket` Telemetry 事件

Phoenix 还为 Socket 和 Channel 交互提供了一些开箱即用的工具。

每当连接到 Socket 时，`Phoenix.Socket` 模块就会执行一个 Telemetry 事件。我们可以在 [源代码](https://github.com/phoenixframework/phoenix/blob/e83b6291cb4ed7cd6572b7af274842910667ade3/lib/phoenix/socket.ex#L450) 中看到该事件。

```elixir
# phoenix/lib/phoenix/socket.ex
def __connect__(user_socket, map, socket_options) do
  %{
    endpoint: endpoint,
    options: options,
    transport: transport,
    params: params,
    connect_info: connect_info
  } = map

  start = System.monotonic_time()

  case negotiate_serializer(Keyword.fetch!(options, :serializer), vsn) do
    {:ok, serializer} ->
      result = user_connect(user_socket, endpoint, transport, serializer, params, connect_info)

      metadata = %{
        endpoint: endpoint,
        transport: transport,
        params: params,
        connect_info: connect_info,
        vsn: vsn,
        user_socket: user_socket,
        log: Keyword.get(options, :log, :info),
        result: result(result),
        serializer: serializer
      }

      duration = System.monotonic_time() - start
      :telemetry.execute([:phoenix, :socket_connected], %{duration: duration}, metadata)
      result

    :error ->
      :error
  end
end
```

我们可以看到，该事件的执行带有持续时间测量和元数据映射，其中包括连接参数和其他上下文信息。我们可以通过在我们的 `Quantum.Telemetry.metrics/0` 列表中添加 `"phoenix.socket_connected"` 事件的度量来告诉我们的 Telemetry 管道处理这个事件：

例如：

```elixir
# lib/quantum/telemetry.ex

def metrics do
  [
    counter(
      "phoenix.socket_connected.count",
      tags: [:endpoint]
    )
  ]
end
```

现在，我们将在每次加入 socket 时增加一个 StatsD 指标。

### `Phoenix.Channel` Telemetry 事件

`Phoenix.Channel.Server` 模块会执行两个 Telemetry 事件 -- 一个是当加入 Channel 时，另一个是当 channel 调用 `handle_info/2` 时。

我们可以在 [源码](https://github.com/phoenixframework/phoenix/blob/8a4aa4eed0de69f94ab09eca157c87d9bd204168/lib/phoenix/channel/server.ex#L302) 中看到 `[:phoenix, :channel_joined]` Telemetry 事件。

```elixir
# phoenix/lib/phoenix/channel/server.ex

def handle_info({:join, __MODULE__}, {auth_payload, {pid, _} = from, socket}) do
  %{channel: channel, topic: topic, private: private} = socket

  start = System.monotonic_time()
  {reply, state} = channel_join(channel, topic, auth_payload, socket)
  duration = System.monotonic_time() - start
  metadata = %{params: auth_payload, socket: socket, result: elem(reply, 0)}
  :telemetry.execute([:phoenix, :channel_joined], %{duration: duration}, metadata)
  GenServer.reply(from, reply)
  state
end
```

我们可以 [在这里](https://github.com/phoenixframework/phoenix/blob/8a4aa4eed0de69f94ab09eca157c87d9bd204168/lib/phoenix/channel/server.ex#L319) 看到 `[:phoenix, channel_handled_in]` 事件

```elixir
# phoenix/lib/phoenix/channel/server.ex

def handle_info(
    %Message{topic: topic, event: event, payload: payload, ref: ref},
    %{topic: topic} = socket
  ) do
  start = System.monotonic_time()

  result = socket.channel.handle_in(event, payload, put_in(socket.ref, ref))
  duration = System.monotonic_time() - start
  metadata = %{ref: ref, event: event, params: payload, socket: socket}

  :telemetry.execute([:phoenix, :channel_handled_in], %{duration: duration}, metadata)

  handle_in(result)
end
```

我们可以通过在 `Quantum.Telemetry` 中为 `"phoenix.channel_joined"` 和 `"phoenix.channel_handled_in"` 事件中的任何一个定义指标，为这些事件添加一些指标报告。

现在我们已经简单了解了 Phoenix Telemetry 事件，让我们为 Ecto 事件挂上一些报告。

## Ecto Telemetry 事件

Ecto 为查询提供了一些开箱即用的可视化。现在让我们来看看并为其中的一些 Telemetry 事件定义指标。

Ecto 将为每个发送到 Ecto 适配器的查询执行一个 Telemetry 事件，[`[:my_app, :repo, :query]`](https://github.com/elixir-ecto/ecto/blob/2aca7b28eef486188be66592055c7336a80befe9/lib/ecto/repo.ex#L120)。它将发出这个事件，并附上一个测量 map 和一个元数据 map。

测量 map 将包括：

```
* `:idle_time` - 连接在被查出查询结果前的等待时间。
* `:queue_time` - 等待检查数据库连接的时间。
* `:query_time` - 执行查询的时间 the time spent executing the query
* `:decode_time` - 解码从数据库收到的数据所花费的时间。
* `:total_time` - 总计时间。
```

元数据 map 将包括:

```
* `:type` - Ecto 查询的类型。例如，对于 Ecto.SQL 数据库，它将是 `:ecto_sql_query`
* `:repo` - Ecto 储存库
* `:result` - 查询结果
* `:params` - 查询参数
* `:query` - 以字符串形式发送到数据库的查询
* `:source` - 查询的源头(可能为零)
* `:options` - `:telemetry_options` 下 repo 操作的额外选项。
```

如果我们想建立按表和命令汇总的 Ecto 查询次数的度量，我们可以在 `Quantum.Telemetry` 度量列表中建立以下度量。

```elixir
def metrics do
  [
    counter(
      "quantum.repo.query.count",
      tag_values: &__MODULE__.query_metatdata/1,
      tags: [:source, :command]
    )
  ]
end

def query_metatdata(%{source: source, result: {_, %{command: command}}}) do
  %{source: source, command: command}
end
```

这将在 StatsD 中为给定命令对给定表的每次查询增加一个计数器。例如：

```
"quantum.repo.query:1|c|#source:users,command:select"
```

我们还可以利用 `summary` 度量函数建立一个时序度量：

```elixir
def metrics do
  [
    summary(
      "quantum.repo.query.total_time",
      unit: {:native, :millisecond},
      tag_values: &__MODULE__.query_metadata/1,
      tags: [:source, :command]
    )
  ]
end

def query_metadata(%{source: source, result: {_, %{command: command}}}) do
  %{source: source, command: command}
end
```

这将向 StatsD 报告每一个对给定表执行给定命令的查询的时间指标。例如：

```
"quantum.repo.query.total_time:1.7389999999999999|ms|#source:users,command:select"
```

## 更多度量指标

本文主要介绍了 `counter/2` 和 `summary/2` 以及 `Telemetry.Metrics` 函数，分别对应 StatsD "count" 和 "timing" 度量类型。`Telemetry.Metrics` 实现了五个度量函数，每个函数都映射到一个特定的度量类型。要了解如何定义和报告这些不同的度量类型，请查看[metrics 文档](https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.html#module-metrics) 和 [TelemetryMetricsStatsd 文档](https://hexdocs.pm/telemetry_metrics_statsd/TelemetryMetricsStatsd.html)。

## 结语

通过利用 Phoenix 和 Ecto 为我们执行的 Telemetry 事件，对我们的 Phoenix 应用进行仪表化，使我们能够在不编写大量自定义代码的情况下实现高度的可观察性。

我们简单地定义了我们的 `Telemetry.Metrics` 模块，配置它来启动 `TelemetryMetricsStatsd` 报告器，并定义了现有的 Telemetry 事件列表，作为指标进行观察。现在，我们向 StatsD 报告了一组有价值的信息丰富的指标，格式化的 Datadog，而无需手动执行一个 Telemetry 事件或定义任何我们自己的事件处理程序。

## 下一个

在这个系列中，我们还将探索一种开箱即用的度量报告的味道。在我们的 [下一篇](./2020-05-13-instrumenting-phoenix-with-telemetry-part-four.md) 中，我们将使用 Erlang `telemetry_poller` 库将 Erlang 虚拟机的测量结果作为 Telemetry 事件发出，我们将使用 `Telemetry.Metrics` 和 `TelemetryMetricsStatsd` 来观察和报告这些事件作为度量。

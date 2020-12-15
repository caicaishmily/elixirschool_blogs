---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
date: 2020-04-29
layout: post
title: "Instrumenting Phoenix with Telemetry Part II: Telemetry Metrics + Reporters"
excerpt: >
  In this series, we're instrumenting a Phoenix app and sending metrics to StatsD with the help of Elixir and Erlang's Telemetry offerings. In Part II we'll use Elixir's `Telemetry.Metrics` and `TelemetryMetricsStatsd` libraries to define and send metrics to StatsD for a given Telemetry event.
---

## 目录

在这个系列中，我们将借助 Elixir 和 Erlang 的 Telemetry 产品，对一个 Phoenix 应用进行仪表化，并将指标发送到 StatsD。简要介绍一下我们将涉及的内容：

- [第一部分: Telemetry 的背后](./2020-04-22-instrumenting-phoenix-with-telemetry-part-one.md)
- 第二部分: 用 `TelemetryMetrics` + `TelemetryMetricsStatsd` 处理 Telemetry 事件
- [第三部分: 观测 Phoenix + Ecto Telemetry 事件](./2020-05-06-instrumenting_phoenix_with_telemetry_part_three.md)
- [第四部分: 用 `telemetry_poller`、`TelemetryMetrics` + `TelemetryMetricsStatsd` 对 Erlang VM 进行测量](./2020-05-06-instrumenting_phoenix_with_telemetry_part_three.md)

## 简介

在本系列的 [第一部分](./2020-04-22-instrumenting-phoenix-with-telemetry-part-one.md) 中，我们了解了为什么可观测性很重要，并介绍了 Erlang 的 Telemetry 库。我们用它为我们的 Phoenix 应用手工轧制了一些可视化仪表盘，但它给我们留下了一些额外的问题需要解决。在这篇文章中，我们将使用 Elixir 的 `Telemetry.Metrics` 和 `TelemetryMetricsStatsd` 库来定义一个给定的 Telemetry 事件的度量指标并将其发送到 StatsD。

## 回顾

在 [我们的上一篇文章](./2020-04-22-instrumenting-phoenix-with-telemetry-part-one.md), 我们添加了一些 Telemetry 监测指标给 Phoenix 应用, [Quantum](https://github.com/elixirschool/telemetry-code-along/tree/part-1-solution)。你可以 [在这里](https://github.com/elixirschool/telemetry-code-along/tree/part-1-solution) 查看我们之前文章中的最终代码。概括地说，我们建立了一个 Telemetry 事件，`[:phoenix, :request]`，将其附加到一个处理模块 `Quantum.Telemetry.Metrics` 上。我们只从一个控制器动作-- `UserController` 的 `new` 动作中执行这个事件。

从该控制器动作中，我们执行 Telemetry 事件，测量 map 结构包括 web 请求的持续时间和请求 `conn`。

```elixir
# lib/quantum_web/controllers/user_controller.ex
def new(conn, _params) do
  start = System.monotonic_time()
  changeset = Accounts.change_user(%User{})
  :telemetry.execute([:phoenix, :request], %{duration: System.monotonic_time() - start}, conn)
  render(conn, "new.html", changeset: changeset)
end
```

我们在模块 `Quantum.Telemetry.Metric` 中用 `handle_event/4` 回调函数处理这个事件。在这个函数中，我们使用事件数据，包括持续时间和 `conn` 中的信息，在 `Statix` Elixir StatsD 客户端库的帮助下，向 StatsD 发送一组指标。

```elixir
# lib/quantum/telemetry/metrics.ex
defmodule Quantum.Telemetry.Metrics do
  require Logger
  alias Quantum.Telemetry.StatsdReporter

  def handle_event([:phoenix, :request], %{duration: dur}, metadata, _config) do
    StatsdReporter.increment("phoenix.request.success", 1)
    StatsdReporter.timing("phoenix.request.success", dur)
  end
end
```

## 这有什么不妥？

Telemetry 让我们很容易发出一个事件并对其进行操作，但我们目前对 Telemetry 库的使用还有很多需要改进的地方。

我们目前的方法有一个缺点是，它让我们对 Telemetry 的事件处理和指标报告束手无策。我们必须定义我们自己的自定义事件处理模块，手动将该模块附加到给定的 Telemetry 事件上，并定义处理者的回调函数。

为了让回调函数向 StatsD 报告给定事件的指标，我们必须创建我们自己的自定义模块，使用 `Statix` 库，_并_ 编写代码，为给定的 Telemetry 事件格式化指标以发送给 StatsD。将 Telemetry 事件数据翻译成适当的 StatsD 度量的心理开销是昂贵的，而且我们执行和处理每一个新的 telemetry 事件都必须进行这种努力。

## 我们需要帮助

如果我们不需要定义我们自己的处理模块或度量报告逻辑，那不是很好吗？如果有某种方法可以简单地列出我们关心的 telemetry 事件，并将它们作为正确的格式化指标自动报告给 StatsD...就好了。

这正是 `Telemetry.Metrics` 和 `TelemetryMetricsStatsd` 库的作用所在。

## 介绍 `Telemetry.Metrics` 和 `TelemetryMetricsStatsd`

[`Telemetry.Metrics`](https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.html) 库提供了一个共同的接口，用于定义基于 Telemetry 事件的度量。它允许我们声明我们要处理的一组 Telemetry 事件，并指定为这些事件构建哪些度量。它还允许我们指定一个开箱即用的报告器来处理和向第三方报告我们的事件。

这意味着我们 _不_ 需要定义自己的处理模块和函数，也不需要编写任何代码来负责向 StatsD 等常见的第三方工具报告事件的度量。我们将使用 `TelemetryMetricsStatsd` 报告库向 StatsD 报告我们的指标，但是 Elixir 的 Telemetry 系列库还包括一个针对 Prometheus 的报告器，或者你也可以自己开发。

在上一篇文章中，我们添加了代码，从我们的 `UserController` 的 `new` 动作中执行以下 Telemetry 事件：

```elixir
:telemetry.execute([:phoenix, :request], %{duration: System.monotonic_time() - start}, conn)
```

现在，不用我们的自定义处理程序和 `Statix` 报告来处理这个事件，而是用 `Telemetry.Metrics` 和 `TelemetryMetricsStatsd` 报告来为我们做所有的工作。

## 它如何工作

在开始写代码之前，我们先来了解一下 `Telemetry.Metrics` 和 `TelemetryMetricsStatsd` 报告器是如何与 Erlang 的 Telemetry 库一起工作来处理 Telemetry 事件的。

`Telemetry.Metrics` 库负责指定我们要处理的 Telemetry 事件作为度量。它定义了我们关心的事件列表，并指定哪些事件应该作为哪种类型的度量(例如，计数器、定时、仪表等)被发送到 StatsD。它把这个作为度量的事件列表交给 Telemetry 报告客户端 `TelemetryMetricsStatsd`。

`TelemetryMetricsStatsd` 库负责获取该事件列表，并通过调用 `:telemetry.attach/4` 将自己的事件处理模块 `TelemetryMetricsStatsd.EventHandler` 附加到每个事件。从我们的第一篇文章中回想一下，`:telemetry/attach/4` 将事件及其相关的处理程序存储在一个 ETS 表中。

之后，当通过调用 `:telemetry.execute/3` 执行 Telemetry 事件时，Telemetry 会在 ETS 表中为给定的事件查找事件处理程序 `:elemetryMetricsStatsd.EventHandler`，并调用它。事件处理模块将把事件、元数据和任何相关的标签格式化为适当的 StatsD 度量，并通过 UDP 把产生的度量发送给 StatsD。

大部分这些都是在引擎盖下发生的。我们只需要定义一个 `Telemetry.Metrics` 模块，并把我们想处理的 Telemetry 事件列为哪种类型的度量。就是这样！

## 起步

你可以通过克隆下来的 [repo](https://github.com/elixirschool/telemetry-code-along/tree/part-2-start) 来跟进这个教程。

- 在分支 [part-2-start](https://github.com/elixirschool/telemetry-code-along/tree/part-2-start) 上查看我们代码的起始状态。
- 在分支 [part-2-solution](https://github.com/elixirschool/telemetry-code-along/tree/part-2-solution) 上查找解题代码。

## 概述

为了让我们的 Telemetry 管道启动和运行，我们不需要写太多代码。

我们将：

1. 定义一个监督模块并且引入 `Telemetry.Metrics`
2. 使用 `Telemetry.Metrics` 度量定义函数为我们要观察的 Telemetry 事件定义一组度量指标。
3. 告诉监督器运行 `TelemetryMetricsStatsd` GenServer，其中包含我们在上一步定义的度量列表

让我们开始吧！

## 设置 `Telemetry.Metrics`

首先, 我们将 `Telemetry.Metrics` 库和 `TelemetryMetricsStatsd` 报告库添加到我们应用的依赖中然后运行 `mix deps.get`:

```elixir
# mix.exs
defp deps do
  [
    {:telemetry_metrics, "~> 0.4"},
    {:telemetry_metrics_statsd, "~> 0.3.0"}
  ]
end
```

现在我们准备定义一个模块，导入 `Telemetry.Metrics`。

## 步骤 1: 定义一个 Metrics 模块

我们将定义一个模块，导入 `Telemetry.Metrics` 库作为一个监督器。我们的 Supervisor 将启动由 `TelemetryMetricsStatsd` 报告者提供的子 GenServer。它将通过 `:metrics` 选项启动 GenServer，同时提供一个参数，即要监听的 Telemetry 事件列表，结构为 metrics。

我们将把 metrics 模块放在 `lib/quantum/telemetry.ex` 中

```elixir
defmodule Quantum.Telemetry do
  use Supervisor
  import Telemetry.Metrics

  def start_link(arg) do
    Supervisor.start_link(__MODULE__, arg, name: __MODULE__)
  end

  def init(_arg) do
    children = [
      {TelemetryMetricsStatsd, metrics: metrics()}
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

我们一会儿再来讨论指标列表。首先，让我们教我们的应用程序在启动时启动这个 Supervisor，但在 `Quantum.Application.start/2` 函数中把它添加到我们应用程序的监督树中。

```elixir
# lib/quantum/application.ex
def start(_type, _args) do
  children = [
    Quantum.Repo,
    QuantumWeb.Endpoint,
    Quantum.Telemetry
  ]

  opts = [strategy: :one_for_one, name: Quantum.Supervisor]
  Supervisor.start_link(children, opts)
end
```

现在我们已经准备好指定哪些 Telemetry 事件作为指标来处理。

## 步骤 2: 将事件指定为指标

我们的 `Telemetry.Metrics` 模块, `Quantum.Telemetry`, 负责告诉 `TelemetryMetricsStatsd` GenServer 对哪些 Telemetry 事件作出反应，以及如何将每个事件视为特定类型的度量指标。

我们要处理上面描述的 `[:phoenix, :request]` 事件。首先，让我们考虑一下我们要为这个事件报告什么类型的指标。比方说，我们想为每个这样的事件递增一个计数器，从而跟踪我们的应用程序收到的对端点的请求数量。我们还可以发送一个定时度量来报告一个给定的网络请求的持续时间。

现在，我们对我们要为我们的事件构建什么样的度量并发送给 StatsD 有了一个基本的概念，让我们看看 `Telemetry.Metrics` 是如何让我们定义这些度量的。

### 定义我们的度量指标

`Telemetry.Metrics` 模块提供了一套 [五个度量函数](https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.html#module-metrics)。这些函数负责将 Telemetry 事件数据格式化为给定的度量指标。

我们将使用 `Telemetry.Metrics.counter/2` 和 `Telemetry.Metrics.summary/2` 函数来定义我们对给定事件的度量。

在导入 `Telemetry.Metrics` 的 `Quantum.Telemetry` 模块中，我们将在 `metrics` 函数中添加以下内容：

```elixir
# lib/quantum/telemetry.ex
defp metrics do
  [
    summary(
      "phoenix.request.duration",
      unit: {:native, :millisecond},
      tags: [:request_path]
    ),

    counter(
      "phoenix.request.count",
      tags: [:request_path]
    )
  ]
end
```

每个度量函数需要两个参数：

- 事件名
- 一个 options 列表

并返回一个描述给定度量指标类型的结构。例如，`counter/2` 函数返回一个 [`%Telemetry.Metrics.Counter{}`](https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.Counter.html#content) 结构，看起来像这样：

```elixir
%Telemetry.Metrics.Counter{
  description: Telemetry.Metrics.description(),
  event_name: :telemetry.event_name(),
  measurement: Telemetry.Metrics.measurement(),
  name: Telemetry.Metrics.normalized_metric_name(),
  reporter_options: Telemetry.Metrics.reporter_options(),
  tag_values: (:telemetry.event_metadata() -> :telemetry.event_metadata()),
  tags: Telemetry.Metrics.tags(),
  unit: Telemetry.Metrics.unit()
}
```

现在我们已经定义了我们的指标列表，我们已经准备好进入下一步了。

## 步骤 3: 启动 `TelemetryMetricsStatsd` GenServer 的指标列表。

当 `TelemetryMetricsStatsd` GenServer 被启动时，指标结构的列表会被传递给它。

```elixir
defmodule Quantum.Telemetry do
  use Supervisor
  import Telemetry.Metrics

  def start_link(arg) do
    Supervisor.start_link(__MODULE__, arg, name: __MODULE__)
  end

  def init(_arg) do
    children = [
      {TelemetryMetricsStatsd, metrics: metrics()} # here!
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end

  defp metrics do
    [
      summary(
        "phoenix.request.duration",
        unit: {:native, :millisecond},
        tags: [:request_path]
      ),

      counter(
        "phoenix.request.count",
        tags: [:request_path]
      )
    ]
  end
end
```

这就启动了下面的过程：

- 当 `TelemetryMetricsStatsd` 启动时，它在 ETS 中存储了事件和它们的处理程序 _和_ 一个配置 map，包括这个度量结构的列表。
- 之后，当 `TelemetryMetricsStatsd` 对执行的事件做出响应时，它在 ETS 中查找事件，并使用存储在该配置图中的度量结构来格式化适当的度量以发送给 StatsD。

### 见证它的行动

请注意，在对 `counter/2` 和 `summary/2` 的调用中，我们使用 `:tag` 选项来指定当指标被发送到 StatsD 时，哪些标签将被应用到指标中。当 `TelemetryMetricsStatsD` 报告器收到我们的 `[:phoenix, :request]` 事件时，将抓取事件元数据中存在的标签键的任何值，并使用它们来构建度量。

所以，当我们执行我们的 Telemetry 时，将 `conn` 作为元数据参数传递进来。

```elixir
# lib/quantum_web/controllers/user_controller.ex
def new(conn, _params) do
  :telemetry.execute([:phoenix, :request], %{duration: System.monotonic_time() - start}, conn)
end
```

`TelemetryMetricsStatsD` 将格式化一个计数器和摘要指标，标记为在 `conn` 中找到的 `:request_path` 键的值。

因此，如果我们运行我们的应用程序并发送一些 Web 请求，我们将看到以下指标报告给 StatsD。

```
{
  counters: {
    'phoenix.request.count.-register-new': 2,
  },
  timers: {
    'phoenix.request.count.-register-new': [ 0, 0 ],
  },
  timer_data: {
    'phoenix.request.duration.-register-new': {
      count_90: 2,
      mean_90: 0,
      upper_90: 0,
      sum_90: 0,
      sum_squares_90: 0,
      std: 0,
      upper: 0,
      lower: 0,
      count: 2,
      count_ps: 0.2,
      sum: 0,
      sum_squares: 0,
      mean: 0,
      median: 0
    }
  }
}
```

## 揭秘

不管你信不信，`Quantum.Telemetry` 模块是我们必须编写的唯一代码，以便将这些指标发送到 StatsD 的 `"phoenix.router_dispatch.stop"` 事件。Telemetry 库为我们处理了其他所有的事情。

让我们仔细看看它是如何工作的。

1. 我们在 `Quantum.Telemetry` 中定义的 `Telemetry.Metrics` 监督器定义了一个我们想要为给定的 Telemetry 事件向 StatsD 发射的度量列表。
2. 当监督器启动时，它启动 `TelemetryMetricsStatsd` GenServer，并给它传递一个列表。
3. 当 `TelemetryMetricsStatsd` GenServer 启动时，它为每个列出的事件调用 `:telemetry.attach/4`，将其与处理程序回调和包括度量定义的配置图一起储存在 ETS 表中。它给 `:telemetry.attach/4` 的处理程序回调是自己的 `TelemetryMetricsStatsd.EventHandler.handle_event/4` 函数。
4. 之后，当一个 Telemetry 事件通过调用 `:telemetry.execute/3` 被执行时，Telemetry 在 ETS 中查找给定事件的处理回调和配置（包括度量定义）。
5. 然后 `:telemetry.execute/3` 函数调用这个处理程序回调 `TelemetryMetricsStatsd.EventHandler.handle_event/4`，其中包括事件名称、事件测量 map 结构、事件元数据和指标配置。
6. `TelemetryMetricsStatsd.EventHandler.handle_event/4` 函数使用所有这些信息格式化适当的指标，并通过 UDP 将其发送到 StatsD。

Phew!

让我们通过查看一些源代码来深入了解这个过程。

### `TelemetryMetricsStatsd` 将事件附加到处理程序和配置数据

当我们的监督器启动 `TelemetryMetricsStatsd` GenServer 时，GenServer 的 `init/1` 函数调用 [`TelemetryMetricsStatsd.EventHandler.attach/7`](https://github.com/beam-telemetry/telemetry_metrics_statsd/blob/master/lib/telemetry_metrics_statsd/event_handler.ex#L24)，其参数集包括我们提供的指标列表。这反过来又会执行对 `:telemetry.attach/4` 的调用。

```elixir
# telemetry_metrics_statsd/lib/telemetry_metrics_statsd/event_handler.ex

def attach(metrics, reporter, mtu, prefix, formatter, global_tags) do
  metrics_by_event = Enum.group_by(metrics, & &1.event_name)

  for {event_name, metrics} <- metrics_by_event do
    handler_id = handler_id(event_name, reporter)

    :ok =
      :telemetry.attach(handler_id, event_name, &__MODULE__.handle_event/4, %{
        reporter: reporter,
        metrics: metrics,
        mtu: mtu,
        prefix: prefix,
        formatter: formatter,
        global_tags: global_tags
      })
  end
end
```

对 `:telemetry.attach/4` 的调用将创建一个 ETS 条目，该条目存储了事件名称以及处理程序回调函数 `&TelemetryMetricsStatsd.EventHandler.handle_event/4`，以及一个包含该事件的度量定义的配置图。

### `TelemetryMetricsStatsd.EventHandler` 处理执行的事件

随后，`[:phoenix, :request]` 事件在我们的 `UserController` 中被执行：

```elixir
# lib/quantum_web/controllers/user_controller.ex
def new(conn, _params) do
  :telemetry.execute([:phoenix, :request], %{duration: System.monotonic_time() - start}, conn)
end
```

`:telemetry.execute/3` 函数在 ETS 中查找事件。它获取处理程序的回调函数，以及为该事件存储的配置，包括度量定义的列表。

然后，Telemetry 将调用回调函数 `TelemetryMetricsStatsd.EventHandler.handle_event/4`，并提供测量 map 结构和元数据，以及它在 ETS 中为该事件查找的存储配置。

`TelemetryMetricsStatsd.EventHandler.handle_event/4` 将根据 ETS 中存储的事件的度量定义来格式化度量，并将得到的度量发送给 StatsD。

在这里我们可以看到，[`TelemetryMetricsStatsd.EventHandler.handle_event/4`](https://github.com/beam-telemetry/telemetry_metrics_statsd/blob/master/lib/telemetry_metrics_statsd/event_handler.ex#L46) 会对事件的度量定义进行迭代，并使用给定的测量和元数据映射以及配置中存储的度量列表中的度量结构，从事件数据中构建合适的度量。然后通过调用 `publish_metrics/2`，通过 UDP 将度量值发布到 StatsD。

```elixir
# telemetry_metrics_statsd/lib/telemetry_metrics_statsd/event_handler.ex

def handle_event(_event, measurements, metadata, %{
      reporter: reporter,
      metrics: metrics,
      mtu: mtu,
      prefix: prefix,
      formatter: formatter_mod,
      global_tags: global_tags
    }) do
  packets =
    # iterate over the stored metric definitions
    for metric <- metrics do
      # get the measurement for the metric type from the measurements map
      case fetch_measurement(metric, measurements) do
        {:ok, value} ->
          # collect metric tags specified in the metric struct
          tag_values =
            global_tags
            |> Map.new()
            |> Map.merge(metric.tag_values.(metadata))
          tags = Enum.map(metric.tags, &{&1, Map.fetch!(tag_values, &1)})
          # format the metric given the metric type, value and tags
          Formatter.format(formatter_mod, metric, prefix, value, tags)

        :error ->
          :nopublish
      end
    end
    |> Enum.filter(fn l -> l != :nopublish end)

  case packets do
    [] ->
      :ok

    packets ->
      # publish metrics to StatsD
      publish_metrics(reporter, Packet.build_packets(packets, mtu, "\n"))
  end
end
```

## 结语

`Telemetry.Metrics` 和 `TelemetryMetricsStatsd` 库使我们更容易处理 Telemetry 事件并根据这些事件报告指标。我们所要做的就是定义一个使用 `Telemetry.Metrics` 的 Supervisor，并告诉这个 Supervisor 用一个度量定义的列表来启动 `TelemetryMetricsStatsd` GenServer。

就是这样! `TelemetryMetricsStatsd` 库将负责调用 `:telemetry.attach/3` 在 ETS 中存储事件以及处理回调函数和该事件的度量列表。之后，当 Telemetry 事件被执行时，Telemetry 将查找该事件及其相关的处理函数和指标列表，并利用这些数据调用处理函数。处理函数 `TelemetryMetricsStatsd.EventHandler.handle_event/4` 将迭代存储在 ETS 中的事件的度量结构列表，并根据度量类型和标签、事件测量 map 结构和元数据构建相应的 StatsD 度量。所有这些都是免费的。

## 下一个

在这篇文章中，我们看到了 `Telemetry.Metrics` 和 `TelemetryMetricsStatsd` 是如何抽象出自定义处理程序和回调函数的需求，将这些处理程序附加到事件上，并实现我们自己的度量报告逻辑。但是我们的 Telemetry 管道仍然需要一点工作。

我们仍然需要发送 _所有_ 的 Telemetry 事件。

为了能够真正观察我们生产环境的 Phoenix 应用的状态，我们需要报告的不仅仅是一个端点的请求持续时间和计数。我们希望能够处理信息丰富的事件，这些事件描述了整个应用中的 Web 请求、数据库查询、Erlang 虚拟机的行为和状态、应用中任何工作者的行为和状态等等。

通过手动在我们需要的地方执行自定义的 Telemetry 事件来完成可视化指标，它们将是乏味和耗时的。最重要的是，在整个应用程序中标准化事件命名约定、测量和元数据将是一个挑战。

在[下周的文章](./2020-05-06-instrumenting_phoenix_with_telemetry_part_three.md)中，我们将研究 Phoenix 和 Ecto 开箱即用的 Telemetry 事件，并使用 `Telemetry.Metrics` 来观察广泛的此类事件，从而使我们无需为大多数观察性用例执行我们自己的自定义事件。

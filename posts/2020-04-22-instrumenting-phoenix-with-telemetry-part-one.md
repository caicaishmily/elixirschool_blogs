---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
date: 2020-04-22
layout: post
title: "Instrumenting Phoenix with Telemetry Part I: Telemetry Under The Hood"
excerpt: >
  In this series, we're instrumenting a Phoenix app and sending metrics to StatsD with the help of Elixir and Erlang's Telemetry offerings. In Part I we'll start out by setting up a basic, DIY Telemetry pipeline and examining how Erlang's Telemetry library works under the hood
---

## 目录

在这个系列中，我们将借助 Elixir 和 Erlang 的 Telemetry 产品，对一个 Phoenix 应用进行仪表化，并将指标发送到 StatsD。简要介绍一下我们将涉及的内容：

- 第一部分: Telemetry 的背后
- [第二部分: 用 `TelemetryMetrics` + `TelemetryMetricsStatsd` 处理 Telemetry 事件](./2020-04-29-instrumenting_phoenix_with_telemetry_part_two.md)
- [第三部分: 观测 Phoenix + Ecto Telemetry 事件](./2020-05-06-instrumenting_phoenix_with_telemetry_part_three.md)
- [第四部分: 用 `telemetry_poller`、`TelemetryMetrics` + `TelemetryMetricsStatsd` 对 Erlang VM 进行测量](./2020-05-13-instrumenting-phoenix-with-telemetry-part-four.md)

在第一部分中，我们将从建立一个基本的、DIY 的 Telemetry 管道开始，并了解 Erlang 的 Telemetry 库背后是如何工作的。然后，在第二部分中，我们将利用 `TelemetryMetrics` 和 `TelemetryMetricsStatsd` 库来响应 Telemetry 事件，将它们格式化为度量并将这些度量报告给 StatsD。在第三部分，我们将通过执行 Telemetry 事件来使用 Phoenix 和 Ecto 提供的开箱即用的仪表。最后，在第四部分，我们将利用 `telemetry_poller` Erlang 库来采集 Erlang 虚拟机的测量结果，并将其作为 Telemetry 事件发出，然后我们的 Telemetry 管道可以观察和报告这些事件。

## 简介

在这篇文章中，我们将讨论为什么可观测性很重要，以及 Telemetry 如何帮助我们在 Elixir 项目中把可观测性当作一等公民对待。然后，我们将使用 Telemetry 和 StatsD 手绘我们自己的仪表管道。最后，我们将看看 Telemetry 库的外壳，并为本系列的第二部分做准备，在这部分中，我们将利用 `Telemetry.Metrics` 库来实现更简单的仪器和报告。

## 可观测性的重要性

用 [Charity Majors](https://charity.wtf/2020/03/03/observability-is-a-many-splendored-thing/) 的不朽名言来说，可观察性意味着问自己：

> 你能不能理解系统内部发生的事情--你能不能理解系统可能陷入的任何内部状态？

> 你能理解系统内部发生的事情吗？ 你能理解系统可能陷入的任何内部状态吗？ 仅仅通过从外部提出问题？

任何花了几个小时(几天?)调试一个他们无法在本地复现的生产环境问题，主要依靠猜测和机构知识的人都知道缺乏这种能力的代价是什么。

我们中的许多人已经将这种情况视为非常自然的事情--我们已经接受了这种挫折感，将其视为软件工程师工作的一部分。我们把可观察性看成是我们无法掌控的东西，或者是事后的想法--在建立新功能或发布 MVP 的主要目标达到之后，"最好有的东西"。

传统的 "web 开发者" 和 "开发-运维工程师" 之间的分工让我们误以为可观察性不是 web 开发者的责任。在过去，确保系统的可观察性可能需要一套专门的技能，但我们所处的世界越来越不是这样。

有了像 Datadog、Splunk 和 Honeycomb 这样的第三方工具，以及像 Telemetry 这样的库，Web 开发人员有能力把可观察性当作头等公民来对待。套用 Charity Majors 的话说，在今天的世界里，我们可以检测我们的代码，观察它的部署，并回答这样的问题："它是否在做我所期望的事情？" 通过这种方式，我们可以构建 "既能理解又好理解" 的系统。

## Telemetry 为我们提供可视性

[Telemetry](https://www.erlang-solutions.com/blog/introducing-telemetry.html) 是一套开源的库，旨在统一和规范 BEAM 上的库和应用程序的检测和监控方式。它是如何工作的？

> Telemetry 的核心是事件。事件表明发生了一些事情：一个 HTTP 请求被接受，一个数据库查询返回了一个结果，或者用户已经登录。每个事件可以有许多处理程序连接到它，当事件被发布时，每个处理程序执行一个特定的动作。例如，一个事件处理程序可能会更新一个指标，记录一些数据，或者丰富分布式跟踪的上下文。

Ecto 和 Phoenix 中都已经包含了 Telemetry 功能，发出的事件我们可以选择接收并采取相应的行动。我们也可以从我们的应用代码中发出自己的自定义 Telemetry 事件，并定义处理程序来响应它们。

从 Ecto 和 Phoenix 免费发出的 Telemetry 事件意味着 _所有_ 使用这些库的应用程序的标准化仪表。同时，Telemetry 提供的处理这些事件和发送额外的自定义事件的接口使每个 Elixir 开发者都能通过轻松地编写和发布完全检测的代码来实现可观察性目标。

现在我们了解了为什么可观察性很重要以及 Telemetry 如何支持可观察性，让我们深入了解一下 Telemetry 库吧!

## 在 Elixir 中使用 Telemetry

首先，我们先看一下如何在我们的 Phoenix 应用中为自定义 Telemetry 事件设置一个简单的报告管道。然后，我们将深入 Telemetry 库的背后，以了解我们的管道是如何工作的。

### 起步

你可以通过克隆下来的 [repo](https://github.com/elixirschool/telemetry-code-along/tree/part-1-start) 来跟进这个教程。

- 在分支 [part-1-start](https://github.com/elixirschool/telemetry-code-along/tree/part-1-start) 上查看我们代码的起始状态。
- 在分支 [part-1-solution](https://github.com/elixirschool/telemetry-code-along/tree/part-1-solution) 上查找解题代码。

我们的 Phoenix 应用 Quantum (懂了吗?)，非常简单--用户可以注册、登录并点击一些按钮。厉害吧？真的，这个虚拟应用只是为了被工具化而存在，所以它并没有什么用，抱歉。

### 概述

为了让我们的管道启动和运行，我们将：

- 安装 Telemetry 库
- 执行 Telemetry 事件
- 定义一个事件处理模块和回调函数。
- 将事件附加到处理程序上

让我们开始吧!

### 我们的工具是什么？

我们首先要选择一个工作流程来制作工具。但首先...

#### 关于衡量标准的说明

为了使我们真正具有可观察性，我们需要的不仅仅是对预定义指标的可见性。指标对于创建静态仪表盘、监测一段时间内的趋势和针对这些趋势建立警报阈值是有用的。指标对于我们监控我们的系统是必要的，但由于它们是预先聚集和预先定义的，所以它们不能实现真正的可观察性。为了实现真正的可观察性，我们需要能够提出和回答我们运行系统的任何问题。所以，我们要跟踪和发出具有丰富上下文的事件。例如，我们不需要为一个特定的 Web 请求建立一个指标，并跟踪它的次数和持续时间，而是希望能够发出描述任何 Web 请求的信息，并包含该请求上下文的丰富描述符--它的持续时间、它被发送到的端点等。

尽管我们希望为每个 web 请求发出一个事件，但我们将从选择一个端点开始。

#### 我们自定义的 Telemetry 事件

让我们在每次用户点击 `/register` 路径并访问注册页面时发出一个 Telemetry 事件。

### 步骤 1: 安装 Telemetry

首先，我们通过运行 `mix deps.get` 安装 Telemetry：

```elixir
# add the following to the `deps` function of your mix.exs file
{:telemetry, "~> 0.4.1"}
```

现在，我们准备发出一个事件！

### 步骤 2: 执行一条 Telemetry 事件

要发出一个 Telemetry 事件，我们调用 [`:telemetry.execute/3`](https://hexdocs.pm/telemetry/index.html#execute)。是的，没错，我们直接从 Elixir 代码中调用 Erlang 的 `:telemetry` 模块。Elixir/Erlang 互操作 FTW!

`execute/3` 函数需要三个参数--事件的名称、我们用来描述该事件的测量值以及描述事件上下文的任何元数据。

我们将从我们的 `UserController` 的 `new` 函数中发出以下事件，名称为 `[:phoenix, :request]`。

我们的事件：

```elixir
defmodule QuantumWeb.UserController do
  use QuantumWeb, :controller

  alias Quantum.Accounts
  alias Quantum.Accounts.User

  def new(conn, _params) do
    start = System.monotonic_time()
    changeset = Accounts.change_user(%User{})

    :telemetry.execute([:phoenix, :request], %{duration: System.monotonic_time() - start}, conn)

    render(conn, "new.html", changeset: changeset)
  end
end
```

在这里，我们发出一个事件，由 `conn` 结构描述，其中包括持续时间测量--跟踪 web 请求的持续时间--以及 web 请求的上下文，。

### 步骤 3: 定义和添加 Telemetry 处理程序

我们需要定义一个处理程序，实现一个回调函数，当我们的事件被执行时，该函数将被调用。

回调函数将使用 [function arity pattern matching](https://medium.com/flatiron-labs/how-functions-pattern-match-in-elixir-12a44a51c6ad) 来匹配我们发出的特定事件。

让我们定义一个模块 `Quantum.Telemetry.Metrics`，实现一个函数 `handle_event/4`。

```elixir
# lib/quantum/telemetry/metrics.ex
defmodule Quantum.Telemetry.Metrics do
  require Logger
  alias Quantum.Telemetry.StatsdReporter

  def handle_event([:phoenix, :request], %{duration: dur}, metadata, _config) do
    # do some stuff like log a message or report metrics to a service like StatsD
    Logger.info("Received [:phoenix, :request] event. Request duration: #{dur}, Route: #{metadata.request_path}")
  end
end
```

### 步骤 4: 将事件附加到处理程序

为了让这个模块的 `handle_event/4` 函数在我们的 `[:phoenix, :request]` 事件被执行时被调用，我们需要将处理程序 "附加" 到事件上。

我们借助于 Telemetry 的 [`attach/4`](https://hexdocs.pm/telemetry/index.html#attach) 函数来实现。

`attach/4` 函数有四个参数。

- 一个独特的 "处理程序 ID"，将用于以后查找该事件的处理程序。
- 事件名称
- 处理器回调函数
- 任何处理程序配置(在本例中我们不需要利用它)

我们将在应用程序的 `start/2` 函数中调用这个函数。

```elixir
# lib/quantum/application.ex

def start(_, _) do
  :ok = :telemetry.attach(
    # unique handler id
    "quantum-telemetry-metrics",
    [:phoenix, :request],
    &Quantum.Telemetry.Metrics.handle_event/4,
    nil
  )
  ...
end
```

Now that we've defined and emitted our event, and attached a handler to that event, the following will occur:

现在我们已经定义并发出了我们的事件，并为该事件附加了一个处理程序，下面将发生：

- 当一个用户访问 `/register` 路由触发 `UserController` 的 `new` 动作
- 我们发出 Telemetry 事件，包含请求持续时间和 `conn` 就像这样：`:telemetry.execute([:phoenix, :request], %{duration: System.monotonic_time() - start}, conn)`
- 然后我们的 `Quantum.Telemetry.Metrics.handle_event/4` 函数将被调用，参数为事件名称、测量 map（包括请求持续时间）和测量元数据，我们为其传递了 `conn`。

因此，如果我们使用 `mix phx.server` 运行服务端，然后访问 `http://localhost:4000/register/new`，我们将会在命令行看到下面的 log

```
[info] Received [:phoenix, :request] event. Request duration: 18000, Route: /register/new
```

这个日志语句只是我们可以对 Telemetry 事件进行响应的一个例子。稍后，我们将使用这个事件中的信息向 StatsD 报告一个指标。

接下来，我们将看看 Telemetry 库的内部，以了解我们发出的事件是如何调用我们的处理程序的。

## Telemetry 揭秘

当一个事件发出时，Telemetry 如何调用处理程序的回调函数？它利用了 ETS! 当我们调用 `:telemetry.attach/4` 时，Telemetry 将我们的事件和相关的处理程序存储在 ETS 表中。当我们调用 `:telemetry.execute/3` 时，Telemetry 会在 ETS 表中查找给定事件的处理函数并执行它。

在接下来的章节中，我们将通过一些 Telemetry 的源码，以便更好地理解这个过程是如何工作的。如果你是 Erlang 的新手（比如我！），没问题。只要尽力读完代码，就能有一个高层次的理解。

### 为事件附加处理程序

`:telemetry.attach/4` 函数将处理程序及其相关事件存储在我们提供的唯一处理程序 ID 下的 ETS 表中。

如果我们偷看一下 `attach/4` 的源代码，我们可以[在这里](https://github.com/beam-telemetry/telemetry/blob/master/src/telemetry.erl#L89)看到插入 ETS 的调用

```erlang
% telemetry/src/telemetry.erl
% inside the attach/4 function:

telemetry_handler_table:insert(HandlerId, EventNames, Function, Config).
```

再看一下 [`telemetry_handler_table` 源代码](https://github.com/beam-telemetry/telemetry/blob/master/src/telemetry_handler_table.erl#L65), 我们可以看到，处理程序是这样存储在 ETS 表中的：

```erlang
% telemetry/src/telemetry_handler_table.erl
% inside the insert/5 function:

Objects = [#handler{id=HandlerId,
            event_name=EventName,
            function=Function,
            config=Config} || EventName <- EventNames],
            ets:insert(?MODULE, Objects)
```

所以，每个 handler 都以这样一个格式存储在 ETS 中：

```erlang
{
  id=HandlerId,
  event_name=EventName,
  function=HandlerFunction,
  config=Config
}
```

其中 `HandlerId` `EventName`、`HandlerFunction` 和 `Config` 设置为我们在调用 `:telemetry.attach/4` 时传递的任何内容。

### 执行事件

When we call `:telemetry.execute/3`, Telemetry will look up the handler by the event name and invoke its callback function. Let's take a look at the source code for `:telemetry.execute/3` [here](https://github.com/beam-telemetry/telemetry/blob/master/src/telemetry.erl#L108):

当我们调用 `:telemetry.execute/3` 时，Telemetry 会根据事件名称查找处理程序，并调用其回调函数。让我们看看` :telemetry.execute/3` [源代码](https://github.com/beam-telemetry/telemetry/blob/master/src/telemetry.erl#L108)。

```erlang
% telemetry/src/telemetry.erl

-spec execute(EventName, Measurements, Metadata) -> ok when
     EventName :: event_name(),
     Measurements :: event_measurements() | event_value(),
     Metadata :: event_metadata().
execute(EventName, Value, Metadata) when is_number(Value) ->
   ?LOG_WARNING("Using execute/3 with a single event value is deprecated. "
                "Use a measurement map instead.", []),
   execute(EventName, #{value => Value}, Metadata);
execute(EventName, Measurements, Metadata) when is_map(Measurements) and is_map(Metadata) ->
   Handlers = telemetry_handler_table:list_for_event(EventName),
   ApplyFun =
       fun(#handler{id=HandlerId,
                    function=HandlerFunction,
                    config=Config}) ->
           try
               HandlerFunction(EventName, Measurements, Metadata, Config)
           catch
               ?WITH_STACKTRACE(Class, Reason, Stacktrace)
                   detach(HandlerId),
                   ?LOG_ERROR("Handler ~p has failed and has been detached. "
                              "Class=~p~nReason=~p~nStacktrace=~p~n",
                              [HandlerId, Class, Reason, Stacktrace])
           end
       end,
   lists:foreach(ApplyFun, Handlers).
```

我们来分析一下这个过程:

#### 首先, 在 ETS 中查找该事件的处理程序。

```erlang
% telemetry/src/telemetry.erl

Handlers = telemetry_handler_table:list_for_event(EventName)
```

看一下 [`telemetry_handler_table.list_for_event/1` 源代码](https://github.com/beam-telemetry/telemetry/blob/master/src/telemetry_handler_table.erl#L45)，我们可以看到，在 ETS 中，处理程序是通过给定的事件名称来查找的，比如这样。

```erlang
% telemetry/src/telemetry_handler_table.erl
% inside list_for_event/1

ets:lookup(?MODULE, EventName)
```

这将返回存储的事件处理程序列表，其中每个处理程序将知道它的句柄 ID、句柄函数和任何配置。

```erlang
{
  id=HandlerId,
  function=HandlerFunction,
  config=Config
}
```

#### 然后, 建立一个 `ApplyFun`，以便为每个处理程序调用。

`ApplyFun` 将调用给定处理程序的 `HandleFunction`，其中包括通过调用 `:telemetry.execute/3` 传入的事件、测量和元数据，以及存储在 ETS 中的任何配置。

```erlang
% telemetry/src/telemetry.erl

ApplyFun =
  fun(#handler{id=HandlerId,
               function=HandlerFunction,
               config=Config}) ->
      try
          HandlerFunction(EventName, Measurements, Metadata, Config)
      catch
          ?WITH_STACKTRACE(Class, Reason, Stacktrace)
              detach(HandlerId),
              ?LOG_ERROR("Handler ~p has failed and has been detached. "
                         "Class=~p~nReason=~p~nStacktrace=~p~n",
                         [HandlerId, Class, Reason, Stacktrace])
      end
  end
```

#### 最后, 遍历 `Handlers` 然后为每个 handler 调用 `ApplyFun`

```erlang
lists:foreach(ApplyFun, Handlers).
```

就是这样！

#### 把它们放在一起

总之，当我们将一个事件 "附加" 到一个处理程序时，我们将处理程序及其回调函数存储在该事件名称下的 ETS 中。之后，当一个事件被 "执行" 时，Telemetry 会查找该事件的处理程序并执行回调函数。很简单。

现在我们了解了我们的 Telemetry 管道是如何工作的，我们准备好考虑拼图的最后一块：事件报告。

### 报告事件到 StatsD

如何处理事件取决于你，但一个常见的策略是向 StatsD 报告指标。在这个例子中，我们将使用 [`Statix`](https://github.com/lexmag/statix) 库来向 StatsD 报告描述我们事件的指标。

首先，我们运行 `mix deps.get` 来安装 Statix

```elixir
# add the following to the `deps` function of your mix.exs file
{:statix, ">= 0.0.0"}
```

接下来，我们定义一个模块使用 `Statix`:

```elixir
# lib/quantum/telemetry/statsd_reporter.ex
defmodule Quantum.Telemetry.StatsdReporter do
  use Statix
end
```

我们需要在我们的应用的 `start` 函数中启动 `StatsdReporter`

```elixir
# lib/quantum/application.ex
def start(_, _) do
  :ok = Quantum.Telemetry.StatsdReporter.connect()
  :ok = :telemetry.attach(
    # unique handler id
    "quantum-telemetry-metrics",
    [:phoenix, :request],
    &Quantum.Telemetry.Metrics.handle_event/4,
    nil
  )
  ...
end
```

现在我们可以在事件处理函数中调用 `Quantum.Telemetry.StatsdReporter` 将指标发送到 StatsD：

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

这里，我们报告了请求事件的总数，以及跟踪 web 请求的持续时间。

现在，如果我们运行我们的应用并访问 `/register` 很多次，我们应该可以在 StatsD 里面看到如下：

```bash
{
  counters: {
    'quantum.phoenix.request': 7
  },
  timers: {
    'quantum.phoenix.request': [
      18000, 18000,
      19000, 19000,
      20000, 22000,
      24000
    ]
  },
  timer_data: {
    'quantum.phoenix.request': {
      count_90: 6,
      mean_90: 19333.333333333332,
      upper_90: 22000,
      sum_90: 116000,
      sum_squares_90: 2254000000,
      std: 2070.1966780270627,
      upper: 24000,
      lower: 18000,
      count: 7,
      count_ps: 0.7,
      sum: 140000,
      sum_squares: 2830000000,
      mean: 20000,
      median: 19000
    }
  },
  counter_rates: {
    'quantum.phoenix.request': 0.7
  }
}
```

_注: 在本地安装运行 StatsD 可以跟着这个简单的 [指导](https://anomaly.io/statsd-install-and-config/index.html)_

这个报告还有一些需要改进的地方。我们目前没有利用作为 `metadata` 参数传递到 `handle_event/4` 函数中的请求上下文。从可观察性的角度来看，这个指标本身并没有什么帮助，因为它并没有告诉我们关于哪个端点收到了请求，是谁发送的，以及响应是什么。

我们在这里有两个选择。我们可以从我们的控制器中发出一个更具体的事件，比如说：

```elixir
:telemetry.execute([:phoenix, :request, :register], %{duration: System.monotonic_time() - start}, conn)
```

这让我们不得不从每个控制器的动作中定义和发出自定义事件。很快就很难跟踪和标准化这些事件了。

我们可以在事件处理程序中为我们发送给 StatsD 的指标添加一些标签。

```elixir
StatsdReporter.increment("phoenix.request.success", 1, tags: [metadata.request_path])
StatsdReporter.timing("phoenix.request.success", dur, tags: [metadata.request_path])
```

标准 StatsD 代理不支持标签，如果我们在这里使用标签会导致出错。但是，如果你向 DogStatsD 代理报告，目标是向 Datadog 发送指标，你的标签将被成功应用，就像这样：

```
quantum.phoenix.request:1|c|#/register/new
quantum.phoenix.request:21000|ms|#/register/new
```

我们现在不会深入解决这个问题。相反，我们要强调的是，度量报告是复杂的。这是一个很难解决的问题，我们可以很容易地投入许多小时和大量的代码来解决这个问题。

### 结语

这看起来有点难，这太难了吗？

Telemetry 提供了一个简单的仪表化接口，但我们的裸例还有很多需要改进的地方。早些时候，我们确定了对应用程序收到的所有网络请求进行检测和报告的需求。我们希望能够汇总和分析描述整个 Web 应用的请求时间和计数的指标，我们希望我们发出的描述这些数据点的数据是信息丰富的，这样我们就可以按端点、响应状态等进行划分。这样一来，我们的应用就变成了 _可观察的_，也就是说，它的输出可以告诉使用它的状态。

然而，在我们当前的方法中，我们是为 _一个特定的端点_ 手动发出 _一个 Telemetry 事件_。这种方法让我们不得不为来自 _每个端点_ 的 _每个请求_ 手动发送 Telemetry 事件。

我们的报告机制，目前设置为向 StatsD 发送指标，也有点问题。我们不仅要在 `Statix` 库的帮助下设置我们自己的报告器，我们也没有正确地利用标签或我们丰富的事件上下文。我们将不得不做额外的工作来利用我们的事件上下文，要么通过标记与 DogStatsD 报告（甚至更多的工作来设置一个全新的报告！）或通过更新事件本身的名称。

"等一下"，你可能会想，"我想 Telemetry 应该是对仪表化事件进行标准化，并使其快速、简单地操作和报告这些事件。为什么我 _还_ 得发出自己的所有事件，并为我 _所有_ 的报告需求提供钩子？"

### 下一个

好吧，答案是，你 _不_ 需要发送所有事件，_也_ 不需要负责所有的报告！现在我们了解了 *如何*设置 Telemetry 管道，以及如何使用 ETS 为事件存储和执行回调，我们已经准备好依靠一些方便的抽象来实现。

令人惊讶的是，Phoenix 和 Ecto _都是_ 的抽象库。Phoenix 和 Ecto 已经可以从源代码中发出常见的事件，包括请求计数和持续时间。`Telemetry.Metrics` 库将使我们超级容易地钩入这些事件，而无需定义和附加我们自己的处理程序。

此外，Telemetry 提供了许多报告客户端，包括 StatsD 报告器，我们可以把它插入到我们的 `Telemetry.Metrics` 模块中，允许我们利用事件元数据和标签，免费向 StatsD 或 DogStatsD 报告。

在[下一篇](./2020-04-29-instrumenting_phoenix_with_telemetry_part_two.md)，我们将利用 `Telemetry.Metrics` 和 `TelemetryStatsdReporter` 来观察、格式化和报告我们在这里建立的 Telemetry 事件。通过这样做，我们将抽象出我们的自定义处理程序 _和_ 我们的自定义 StatsD 报告器的需求。

回头见！

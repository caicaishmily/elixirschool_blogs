---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
date: 2020-05-13
layout: post
title: "Instrumenting Phoenix with Telemetry Part IV: Erlang VM Measurements with `telemetry_poller`"
excerpt: >
  In this series, we're instrumenting a Phoenix app and sending metrics to StatsD with the help of Elixir and Erlang's Telemetry offerings. In Part III we'll incorporate Erlang's [`telemetry_poller` library](https://github.com/beam-telemetry/telemetry_poller) into our Phoenix app so that we can observe and report on Erlang VM Telemetry events.
---

## 目录

在这个系列中，我们将借助 Elixir 和 Erlang 的 Telemetry 产品，对一个 Phoenix 应用进行仪表化，并将指标发送到 StatsD。简要介绍一下我们将涉及的内容：

- [第一部分: Telemetry 的背后](./2020-04-22-instrumenting-phoenix-with-telemetry-part-one.md)
- [第二部分: 用 `TelemetryMetrics` + `TelemetryMetricsStatsd` 处理 Telemetry 事件](./2020-04-29-instrumenting_phoenix_with_telemetry_part_two.md)
- [第三部分: 观测 Phoenix + Ecto Telemetry 事件](./2020-05-06-instrumenting_phoenix_with_telemetry_part_three.md)
- 第四部分: 用 `telemetry_poller`、`TelemetryMetrics` + `TelemetryMetricsStatsd` 对 Erlang VM 进行测量

## 简介

在 [上一篇](./2020-05-06-instrumenting_phoenix_with_telemetry_part_three.md) 中，我们使用 `Telemetry.Metrics` 来定义一些开箱即用的 Phoenix 和 Ecto Telemetry 事件的度量，并使用 `TelemetryMetricsStatsd` 来处理和报告这些事件到 StatsD。

在这篇文章中，我们将把 Erlang 的 [`telemetry_poller`](https://github.com/beam-telemetry/telemetry_poller) 库整合到我们的 Phoenix 应用中，这样我们就可以观察和报告 Erlang VM Telemetry 事件。

## 起步

你可以通过克隆下来的 [repo](https://github.com/elixirschool/telemetry-code-along/tree/part-4-start) 来跟进这个教程。

- 在分支 [part-4-start](https://github.com/elixirschool/telemetry-code-along/tree/part-4-start) 上查看我们代码的起始状态。
- 在分支 [part-4-solution](https://github.com/elixirschool/telemetry-code-along/tree/part-4-solution) 上找到解题代码。

## 概述

为了报告 Erlang 虚拟机测量的指标，我们将：

- 安装 `telemetry_poller` 依赖
- 使用 `Telemetry.Metrics` 定义 `telemetry_poller` Telemetry 事件的指标
- 就这些！

## 步骤 1: 安装 `telemetry_poller`

首先，我们将在我们的应用程序中加入 `telemetry_poller` 依赖，并运行 `mix deps.get`

```elixir
# mix.exs
def deps do
  {:telemetry_poller, "~> 0.4"}
end
```

## 步骤 2: 为 `telemetry_poller` 事件定义指标

### `telemetry_poller` Telemetry 事件

当我们的应用程序启动时，`telemetry_poller` 应用程序也将开始运行。这个应用程序将对 Erlang VM 进行轮询，以进行以下测量，并将这些测量作为 Telemetry 事件来执行。

- 内存 - 测量 Erlang VM 使用的内存。
- 总运行队列长度 -- 测量 Erlang 调度器要调度的任务队列。这个事件将通过测量 map 来执行，测量 map 描述了：

  - `total` -- 所有运行队列长度的总和。
  - `cpu` -- CPU 调度器运行队列长度的总和。
  - `io` -- dirty IO 运行队列的长度。如果运行在 20 之前的 Erlang 版本上，它总是 0。

- System count - 测量本地节点当前存在的进程数、本地节点当前存在的原子数和本地节点当前存在的端口数。
- 进程信息 - 含有给定进程信息的测量，例如您的应用程序中的 worker。

让我们在 `Quantum.Telemetry` 模块中定义其中一些事件的指标。

### 定义我们的指标

[`Telemetry.Metrics.last_value/2`](https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.html#last_value/2) 函数定义了一个度量指标，用于保存最近事件中选定的测量值。`TelemetryMetricsStatsd` 报告者将把这样的度量作为 "度量" 指标发送给 StatsD。让我们为上面提到的一些 Telemetry 事件定义一组度量指标：

```elixir
# lib/quantum/telemetry.ex

defp metrics do
  [
    # VM Metrics - gauge
    last_value("vm.memory.total", unit: :byte),
    last_value("vm.total_run_queue_lengths.total"),
    last_value("vm.total_run_queue_lengths.cpu"),
    last_value("vm.system_counts.process_count")
  ]
end
```

现在，当 `telemetry_poller` 执行相应的事件时，我们会看到以下指标发送到 StatsD。

```
gauges: {
  'vm.memory.total': 49670008,
  'vm.total_run_queue_lengths.total': 0,
  'vm.total_run_queue_lengths.cpu': 0,
  'vm.system_counts.process_count': 366
}
```

这就是了! 在我们结束之前，让我们来看看 `telemetry_poller` 库的内部。

## 揭秘 `telemetry_poller`

看一些源代码，我们可以了解 `telemetry_poller` 到底是如何执行这些事件的。

### `memory/0` 函数

[`memory/0`](https://github.com/beam-telemetry/telemetry_poller/blob/master/src/telemetry_poller_builtin.erl#L22) 函数通过调用 `erlang:memory/0` 来抓取内存测量值，并将这些测量值作为测量 map 传递给 `telemetry:execute/3` 。

```erlang
% telemetry_poller/src/telemetry_poller_builtin.erl

memory() ->
    Measurements = erlang:memory(),
    telemetry:execute([vm, memory], maps:from_list(Measurements), #{}).
```

让我们进一步分析一下。我们可以通过自己在 `iex` 中的尝试来检查调用 [`erlang:memory()`](http://erlang.org/doc/man/erlang.html#memory-0) 所返回的测量值。

```elixir
iex(1)> :erlang.memory()
[
  total: 28544704,
  processes: 5268240,
  processes_used: 5267272,
  system: 23276464,
  atom: 339465,
  atom_used: 317752,
  binary: 58656,
  code: 5688655,
  ets: 515456
]
```

我们可以看到，它包含一个 `:total` 的键，指向分配给 Erlang VM 的内存总量的值。

这样，一个名为 `[vm，memory]` 的 Telemetry 事件就会被执行，并得到一组包括这个总数的测量值。当我们调用我们的 `Telemetry.Metrics.last_value/2` 函数时，我们是在告诉我们的报告者 `TelemetryStatsD` 为这个事件附加一个处理程序，并通过构建一个包含在所提供的测量值中的 `:total` 键值的测量值来响应它。

```elixir
# lib/quantum/telemetry.ex

defp metrics do
  [
    last_value("vm.memory.total", unit: :byte)
  ]
end
```

### `total_run_queue_lengths/0` 函数

[`total_run_queue_lengths/0`](https://github.com/beam-telemetry/telemetry_poller/blob/master/src/telemetry_poller_builtin.erl#L27) 函数测量 VM 运行队列的总长度，以及 CPU 调度器运行队列的总长度，并将这些测量结果传递给对 `telemetry:execute/3` 的调用。

```erlang
% telemetry_poller/src/telemetry_poller_builtin.erl

total_run_queue_lengths() ->
    Total = cpu_stats(total),
    CPU = cpu_stats(cpu),
    telemetry:execute([vm, total_run_queue_lengths], #{
        total => Total,
        cpu => CPU,
        io => Total - CPU},
        #{}).
```

为了观察这个事件，我们指定 Telemetry 管道为 `[vm, total_run_queue_lengths]` 事件附加一个处理程序，并为每个被执行的此类事件构建两个测量指标--一个是 `total` 测量值，一个是 `cpu` 测量值。

```elixir
# lib/quantum/telemetry.ex

defp metrics do
  [
    last_value("vm.total_run_queue_lengths.total"),
    last_value("vm.total_run_queue_lengths.cpu")
  ]
end
```

### `system_counts/0` 函数

[`system_counts/0`](https://github.com/beam-telemetry/telemetry_poller/blob/master/src/telemetry_poller_builtin.erl#L42) 函数通过调用 `telemetry:execute/3` 进行测量，包括总进程数，并利用这些测量结果执行 Telemetry 事件。

```erlang
system_counts() ->
    ProcessCount = erlang:system_info(process_count),
    PortCount = erlang:system_info(port_count),
    telemetry:execute([vm, system_counts], #{
        process_count => ProcessCount,
        port_count => PortCount
    }).
```

为了观察这一事件，我们指定我们的 Telemetry 管道为 `[vm, system_counts]` 事件附加一个处理程序，并为每一个这样的事件构建一个具有 `process_count` 测量值的度量指标：

```elixir
# lib/quantum/telemetry.ex

defp metrics do
  [
    last_value("vm.system_counts.process_count")
  ]
end
```

## 轮询自定义测量

你也可以使用 `telemetry_poller` 库来发射描述你的应用程序中运行的自定义进程或 worker 的测量值，或者发射自定义测量值。更多信息请参见[文档](https://hexdocs.pm/telemetry_metrics/Telemetry.Metrics.html#module-vm-metrics)。

## 结语

我们再一次看到，Erlang 和 Elixir 的 Telemetry 系列库让我们很容易用很少的手工代码实现相当全面的测量。通过将 `telemetry_poller` 库添加到我们的依赖关系中，确保我们的应用程序将定期执行一组描述 Erlang VM 测量的 Telemetry 事件。我们正在观察这些事件，对它们进行格式化，并在 `Telemetry.Metrics` 和 `TelemetryMetricsStatsd` 的帮助下将它们发送到 StatsD，使我们能够在任何给定的时间内更全面地描绘我们 Phoenix 应用的状态。

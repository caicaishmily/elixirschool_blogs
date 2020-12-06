---
author: Bobby Grayson
date: 2019-02-21
layout: post
title:  Elixir Supervisor Strategies
excerpt: >
  Learn the ins and outs of Elixir's 3 supervisor strategies
---

# ELixir 监督策略

OTP 和 Elixir 的独特之处在于，应用程序可以采取与它们启动的不同进程的监督者行为模式。

在这篇文章中，我们将通过一个监督应用来研究 Elixir 中可用的三种模式。

首先，我们创建一个监督应用程序。

```sh
mix new counter --sup
cd counter
```

现在我们有了一个应用程序，我们将创建 3 个模块。

它们都是 GenServers，并且与应用程序一起启动，每隔一秒向自己发送一条消息，以增加自己的状态。

其中一个将始终工作，一个每发送 6 条消息后失败，一个每 20 条消息后失败。

开始的时候，它会有 `application.ex` 中默认的监督策略 `one_for_one`。

这个策略是，如果一个进程死亡，它的兄弟进程应该保持工作不受影响。

**注意：无论你的监督策略如何，如果你的应用程序中的子程序在 ```start_link``` 上没有成功返回一个 ```{:ok, pid}``` 元组，那么应用程序作为一个整体将无法启动，你的监管策略也就无所谓了。**

我们一开始就坚持这样做。

让我们从 `lib/counter/one.ex` 中的第一个模块开始。

如果它的状态是 22，它就会失败。

```elixir
defmodule Counter.One do
  use GenServer

  def start_link(_state \\ 0) do
    IO.inspect("starting", label: "Counter.One")
    success = GenServer.start_link(__MODULE__, 0)
    IO.inspect("started", label: "Counter.One")
    success 
  end

  @impl true
  def init(state) do
    work(state)
    # Schedule work to be performed on start
    schedule_work()
    {:ok, state}
  end

  @impl true
  def handle_info(:work, state) do
    work(state)
    # Reschedule once more
    schedule_work()
    {:noreply, state + 1}
  end

  defp schedule_work() do
    Process.send_after(self(), :work, 1000)
  end

  def work(state) do
    case state do
      22 -> raise "I'm Counter.One and I'm gonna error now"
      _ -> IO.inspect("working and my state is #{state}", label: "Counter.One")
    end
  end
end
```

注意:

这是对 [GenServer 文档中的不错的例子](https://hexdocs.pm/elixir/GenServer.html#module-receiving-regular-messages) 的轻微修改。

关于 `Process.send_after/3` 的更多内容，也可以参考[过时的 Elixir School 的博客文章](http://elixirschool.com/blog/til-send-after/)。

现在，如果我们打开 `lib/counter/application.ex` 并将其添加到 children 中，我们就可以让它跟随我们的 app 一起启动。

```elixir
defmodule Counter.Application do
  # See https://hexdocs.pm/elixir/Application.html
  # for more information on OTP Applications
  @moduledoc false

  use Application

  def start(_type, _args) do
    # List all child processes to be supervised
    children = [
      Counter.One
    ]

    # See https://hexdocs.pm/elixir/Supervisor.html
    # for other strategies and supported options
    opts = [strategy: :one_for_one, name: Counter.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

现在假如我们启动应用，我们将会看到它开始工作并且在状态 22 的时候失败：

```sh
Counter.One: "working and my state is 18"
Counter.One: "working and my state is 19"
Counter.One: "working and my state is 20"
Counter.One: "working and my state is 21"
Counter.One: "starting"
Counter.One: "working and my state is 0"
Counter.One: "started"

18:27:42.566 [error] GenServer #PID<0.119.0> terminating
** (RuntimeError) I'm Counter.One and I'm gonna error now
    (one) lib/counter/one.ex:33: Counter.One.work/1
    (one) lib/counter/one.ex:21: Counter.One.handle_info/2
    (stdlib) gen_server.erl:616: :gen_server.try_dispatch/4
    (stdlib) gen_server.erl:686: :gen_server.handle_msg/6
    (stdlib) proc_lib.erl:247: :proc_lib.init_p_do_apply/3
Last message: :work
State: 22
Counter.One: "working and my state is 0"
Counter.One: "working and my state is 1"
```

这个失败是因为我们做了一个特定的子句，当我们的计数器中的状态达到 `22` 时，通过引发一个错误来让应用出现失败。

失败后会以状态 0（默认）重新启动。

现在让我们再搞一个永远不会失败的模块。

```elixir
defmodule Counter.Two do
  use GenServer

  def start_link(_state \\ 0) do
    IO.inspect("starting", label: "Counter.Two")
    success = GenServer.start_link(__MODULE__, 0)
    IO.inspect("started", label: "Counter.Two")
    success 
  end

  @impl true
  def init(state) do
    work(state)
    # Schedule work to be performed on start
    schedule_work()
    {:ok, state}
  end

  @impl true
  def handle_info(:work, state) do
    work(state)
    # Reschedule once more
    schedule_work()
    {:noreply, state + 1}
  end

  defp schedule_work() do
    Process.send_after(self(), :work, 1000)
  end

  def work(state) do
    IO.inspect("working and my state is #{state}", label: "Counter.Two")
  end
end
```

我们照样可以把它添加到 `lib/counter/application.ex` 中去

```elixir
  # ...
  def start(_type, _args) do
    # List all child processes to be supervised
    children = [
      Counter.One,
      Counter.Two
    ]
  end
  # ...
```

现在，对于我们的第三个也是最后一个模块，它将会在状态为 `5` 的时候失败。

```elixir
defmodule Counter.Three do
  use GenServer

  def start_link(_state \\ 0) do
    IO.inspect("starting", label: "Counter.Three")
    success = GenServer.start_link(__MODULE__, 0)
    IO.inspect("started", label: "Counter.Three")
    success 
  end

  @impl true
  def init(state) do
    work(state)
    # Schedule work to be performed on start
    schedule_work()
    {:ok, state}
  end

  @impl true
  def handle_info(:work, state) do
    work(state)
    # Reschedule once more
    schedule_work()
    {:noreply, state + 1}
  end

  defp schedule_work() do
    Process.send_after(self(), :work, 1000)
  end

  def work(state) do
    case state do
      5 -> raise "I'm Counter.Three and I'm gonna error now"
      _ -> IO.inspect("working and my state is #{state}", label: "Counter.Three")
    end
  end
end
```

我们可以把它添加到 `lib/counter/application.ex` 的  children 中，放在其他两个子程序之后。

```elixir
  # ...
  def start(_type, _args) do
    # List all child processes to be supervised
    children = [
      Counter.One,
      Counter.Two,
      Counter.Three
    ]
  end
  # ...
```

## One for One

现在，让我们启动我们的应用程序，看看每个 GenServer 的失败行为和状态。

这些日志只截取到有趣的部分。

```sh
Counter.One: "working and my state is 4"
Counter.Two: "working and my state is 4"
Counter.Three: "working and my state is 4"
Counter.One: "working and my state is 5"
Counter.Two: "working and my state is 5"
Counter.Three: "starting"
Counter.Three: "working and my state is 0"
Counter.Three: "started"

18:11:37.495 [error] GenServer #PID<0.130.0> terminating
** (RuntimeError) I'm Counter.Three and I'm gonna error now
    (counter) lib/counter/three.ex:33: Counter.Three.work/1
    (counter) lib/counter/three.ex:21: Counter.Three.handle_info/2
    (stdlib) gen_server.erl:616: :gen_server.try_dispatch/4
    (stdlib) gen_server.erl:686: :gen_server.handle_msg/6
    (stdlib) proc_lib.erl:247: :proc_lib.init_p_do_apply/3
Last message: :work
State: 5
Counter.One: "working and my state is 6"
Counter.Two: "working and my state is 6"
Counter.Three: "working and my state is 0"
Counter.One: "working and my state is 7"
Counter.Two: "working and my state is 7"
Counter.Three: "working and my state is 1"
```

所以，我们可以看到第一次崩溃。

`Counter.Three` 的进程因我们的抛出错误而失败，并被重新启动。

因为在 Elixir 中的默认策略是 `one_for_one`，所以这是预料之中的。

在默认配置中，我们不希望一个子进程的失败影响到其他进程。

如果我们让 `Counter.One` 继续到 22，我们会看到同样的行为（允许崩溃而不影响任何兄弟姐妹，因为它是一对一）。

## Rest for One

现在让我们试试 `rest_for_one`.

Rest for one 作为一个策略，按顺序启动子程序，如果前面的子程序失败，后面的子程序也会失败。

我们要将 `lib/counter/application.ex` 中分配 `opts` 的那一行改成这样。

```elixir
# ...
    children = [
      Counter.One,
      Counter.Two,
      Counter.Three
    ]

    opts = [strategy: :rest_for_one, name: Counter.Supervisor]
# ...
```

现在，让我们重启应用。

这些日志也只是截取到有趣的部分。

```sh
Counter.One: "working and my state is 3"
Counter.Two: "working and my state is 3"
Counter.Three: "working and my state is 3"
Counter.One: "working and my state is 4"
Counter.Two: "working and my state is 4"
Counter.Three: "working and my state is 4"
Counter.One: "working and my state is 5"
Counter.Two: "working and my state is 5"
Counter.Three: "starting"

18:30:56.925 [error] GenServer #PID<0.134.0> terminating
** (RuntimeError) I'm Counter.Three and I'm gonna error now
    (counter) lib/counter/three.ex:33: Counter.Three.work/1
    (counter) lib/counter/three.ex:21: Counter.Three.handle_info/2
    (stdlib) gen_server.erl:616: :gen_server.try_dispatch/4
    (stdlib) gen_server.erl:686: :gen_server.handle_msg/6
    (stdlib) proc_lib.erl:247: :proc_lib.init_p_do_apply/3
Last message: :work
State: 5
Counter.Three: "working and my state is 0"
Counter.Three: "started"
Counter.One: "working and my state is 6"
Counter.Two: "working and my state is 6"
Counter.Three: "working and my state is 0"
Counter.One: "working and my state is 7"
Counter.Two: "working and my state is 7"
```

这里的关键启示是 _顺序很重要_。

因为 `Counter.One` 直到它的状态是 `22` 时才会失败，而 `Counter.Three` 失败时的状态是 `5`，所以 `Counter.One` 将强制从其第三个子项开始重新启动，但 `Counter.Three` 的失败对它的兄弟姐妹没有影响。

## One For All

现在让我们通过 `one_for_all` 启用它。

在这种监督模型中，如果一个子进程失败，则必须重新启动所有子进程。

为此，我们再更改一下 `lib/counter/application.ex`。

```elixir
    opts = [strategy: :one_for_all, name: Counter.Supervisor]
```

如果我们用 `iex -S mix` 重启应用程序，我们可以看到 `Counter.Three` 状态一到 5 就会出现行为，但再次确认到 22 的时候又会出现同样的情况。

```sh
Counter.Two: "working and my state is 4"
Counter.One: "working and my state is 5"
Counter.Two: "working and my state is 5"
Counter.One: "starting"
Counter.One: "working and my state is 0"
Counter.One: "started"
Counter.Two: "starting"
Counter.Two: "working and my state is 0"
Counter.Two: "started"
Counter.Three: "starting"
Counter.Three: "working and my state is 0"
Counter.Three: "started"

18:34:56.122 [error] GenServer #PID<0.121.0> terminating
** (RuntimeError) I'm Counter.Three and I'm gonna error now
    (counter) lib/counter/three.ex:33: Counter.Three.work/1
    (counter) lib/counter/three.ex:21: Counter.Three.handle_info/2
    (stdlib) gen_server.erl:616: :gen_server.try_dispatch/4
    (stdlib) gen_server.erl:686: :gen_server.handle_msg/6
    (stdlib) proc_lib.erl:247: :proc_lib.init_p_do_apply/3
Last message: :work
State: 5
Counter.One: "working and my state is 0"
Counter.Two: "working and my state is 0"
Counter.Three: "working and my state is 0"
```

我们也可以改变 `children` 中变量匹配中的顺序，同样的事情也会照样发生。

这是一个很大的问题，但希望现在 Elixir 应用的监督策略更清晰一些!
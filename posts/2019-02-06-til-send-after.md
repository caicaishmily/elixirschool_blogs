---
author: Sean Callan
author_link: https://github.com/doomspork
categories: til
date: 2019-02-07
layout: post
title:  TIL about `Process.send_after/4`
excerpt: >
  Want to schedule something to run later? Need a reoccurring task? Today we learn how!
---

# til send_after

稍后执行代码或创建循环任务可能很棘手，但你知道我们可以在 Elixir 中只用一个进程就能完成这个任务吗？

有了 GenServer、`Process.send_after/4` 和 `handle_info/2` 回调，我们就有了所需的一切。

让我们看看 `Process.send_after/4` 和预期的参数。

```elixir
send_after(dest, msg, time, opts \\ [])
```

- `dest` 参数采用进程的 `pid` 或名称，我们将使用一个命名的 GenServer 示例。
- 我们要发送给进程的 `msg`，这几乎可以是任何数据结构，但我们将坚持用一个简单的原子类型。
- 以毫秒为单位，`time` 是我们想要过多久发送消息的时间。
- 最后但并非最不重要的，options。

一切都很好，但是 `time` 过去后，`msg` 到底在哪里？

好问题！

比较少使用的 `handle_info/2` 回调函数是用来处理这些消息的。

就像 `handle_cast/2` 一样，`handle_info/2` 需要两个参数：第一个是上面的 `msg`，第二个是当前状态。

这足以使我们继续前进，但是如果您有兴趣了解有关 `Process.send_after/4` 的更多信息，请务必查看[官方文档](https://hexdocs.pm/elixir/Process.html#send_after/4)。

为了演示如何使用上述工具执行重复工作，我们构建一个简单的模块，每 10 秒输出一次当前时间。

由于我们在使用 GenServer，因此我们可以依靠 `init/1` 来作为开始使用 `Process.send_after/4` 和 `:tick` 消息进行重复工作的好地方：

```elixir
@ten_seconds 10000

def init(opts) do
  Process.send_after(self(), :tick, @ten_seconds)

  {:ok, opts}
end
```

接下来我们需要为我们的 `:tick` 消息定义 `handle_info/2` 回调。

对于这个函数，我们将获取并格式化当前时间，然后输出它，最重要的是使用 `Process.send_after/4` 触发 10 秒后的另一个 `:tick`。

```elixir
def handle_info(:tick, state) do
  time =
    DateTime.utc_now()
    |> DateTime.to_time()
    |> Time.to_iso8601()

  IO.puts("The time is now: #{time}")

  Process.send_after(self(), :tick, @ten_seconds)

  {:noreply, state}
end
```

当我们把所有的内容汇集到我们的 `Example` 模块中时，我们的代码应该像这样：

```elixir
defmodule Example do
  use GenServer

  @ten_seconds 10000

  def init(opts) do
    Process.send_after(self(), :tick, @ten_seconds)

    {:ok, opts}
  end

  def handle_info(:tick, state) do
    time =
      DateTime.utc_now()
      |> DateTime.to_time()
      |> Time.to_iso8601()

    IO.puts("The time is now: #{time}")

    Process.send_after(self(), :tick, @ten_seconds)

    {:noreply, state}
  end
end
```

不要再耽误时间了，让我们把我们的新代码投入使用吧！

打开 `iex`，复制并粘贴我们的新模块。

现在我们启动 `GenServer.start/3`，这将反过来启动我们的时钟信息。

```shell
iex> GenServer.start(Example, [])
{:ok, #PID<0.134.0>}
iex>
The time is now: 02:22:04.900603
The time is now: 02:22:14.904617
The time is now: 02:22:24.905600
The time is now: 02:22:34.906790
The time is now: 02:22:44.907672
The time is now: 02:22:54.908688
The time is now: 02:23:04.909642
The time is now: 02:23:14.910623
```

Tada！

每隔10秒我们就会看到一个更新的时间。

没有CRON，没有后台任务框架，没有外部依赖，只有Elixir。
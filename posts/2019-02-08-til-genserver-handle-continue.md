---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: til
date: 2019-02-15
layout: post
title:  TIL GenServer's `handle_continue/2`
excerpt: >
  Support non-blocking, async GenServer initialization callbacks with OTP 21's nifty `handle_continue/2`!
---

# GenServer handle_continue

当你启动 GenServer 时需要执行一个长期运行的进程会发生什么？我们 _不_ 希望该进程的执行阻止 GenServer 完成启动。我们也不想在进程的运行和 GenServer 的收件箱中到达的其他消息之间创造一个竞赛条件从而异步执行那个进程。在这篇文章中，我们将仔细研究这两个问题，并了解 OTP 21 的 `GenServer.handle_continue/2` 如何成为完美的解决方案。

## 启动 GenServers 而不被阻断

比方说，我们正在为一个购物清单履行应用程序构建一个 GenServer。我们的 GenServer 将持有描述一个杂货店购物清单的状态，_并_ 知道与该购物清单相关的可用库存。当我们的 GenServer 启动时，它将接收一个购物单并将其放入状态。但是请稍等一下！我们的 GenServer 初始化过程 _然后_ 需要从另一个来源获取该购物清单和检索相关库存。

我们解决这个问题的第一次尝试可能是这样的。

```elixir
defmodule ShoppingListFulfillment do
  use GenServer

  def start_link(shopping_list) do
    GenServer.start_link(__MODULE__, shopping_list)
  end

  def init(shopping_list) do
    state = %{
      shopping_list: shopping_list,
      inventory: get_inventory_for(shopping_list)
    }
    {:ok, state}
  end

  defp get_inventory_for(shopping_list) do
    # something that could be time consuming!
    # like a web request or a database call
    # returns some inventory info for each item on the shopping list
  end
end
```

这里，我们在 `init/1` 回调中调用了 "get inventory for shopping list item" 代码。当 `start_link/1` 被调用时，回调会被触发。

这种方法的问题是 `start_link/1` 会阻塞，直到 `init/1` 返回 `{:ok, state}`。我们不会从 `init/1` 返回，直到库存获取代码运行 _之后_。这可能会很耗时。我们不希望我们的 GenServer 被这个阻塞。

让我们探索一种异步的方法。

## 异步回调和竞赛条件

我们可以在我们的 `init` 回调中使用 `Kernel.send/2` 来启动一些异步工作，而不会阻塞 `GenServer.start_link/1`。当我们使用 `send/2` 并传给它第一个参数`self`，即我们 GenServer 的 PID 时，我们的 GenServer 将用一个与我们发送的消息相匹配的 `handle_info/2` 函数来处理该消息。

```elixir
defmodule ShoppingListFulfillment do
  use GenServer

  def start_link(shopping_list) do
    GenServer.start_link(__MODULE__, shopping_list)
  end

  def init(shopping_list) do
    state = %{
      shopping_list: shopping_list,
      inventory: []
    }
    send(self, :get_inventory)
    {:ok, state}
  end

  def handle_info(:get_inventory, %{shopping_list: shopping_list}) do
    inventory = get_inventory_for(shopping_list)
    state = %{
      shopping_list: shopping_list,
      inventory: inventory
    }
    {:noreply, state}
  end

  defp get_inventory_for(shopping_list) do
    # something that could be time consuming!
    # like a web request or a database call
    # returns some inventory info for each item on the shopping list
  end
end
```

这种方法解除了 `GenServer.start_link/1` 的阻塞。它不再需要 _等待_ 获取库存的工作。现在，一旦我们获取完库存信息，就会异步更新状态。

不过这种方法也有一个缺点。因为我们在 `init` 函数中发送 `:get_inventory` 消息，这并不意味着 `:get_inventory` 是 GenServer 将接收和处理的第一个消息。这可能会导致一个竞赛条件！

如果我们的 GenServer 收到一个消息，询问购物清单上的一个项目是否有货，在它收到并完成处理消息以获得库存 _之前_，会发生什么？这可能会导致一个假否定! 我们会看到状态中的 `inventory` 是空的，并告诉发送者他们的物品不可用。哦不!

如果有什么方法可以异步获取库存，而不阻塞 `start_link/1`，_同时_ 确保它在 GenServer 收到的任何其他消息被响应 _之前_ 执行......

## 使用 `handle_continue/2`

几个月前发布的 OTP 21 给我们提供了一个解决这个问题的方法。每当前一个回调返回 `{:continue, :message}` 时，GenServer 进程就会调用 [`GenServer.handle_continue/2`](https://hexdocs.pm/elixir/GenServer.html#c:handle_continue/2) 回调。

> `handle_continue/2` 在前一个回调之后立即被调用，这使得它对于在初始化之后执行工作或者将回调中的工作分成多个步骤，沿途更新进程状态非常有用。[*](http://erlang.org/doc/man/gen_server.html#Module:handle_continue-2)

这种方法确保我们的 GenServer 不会处理任何其他消息，直到 `handle_continue/2` 完成。没有更多的竞赛条件!

让我们来看看。

```elixir
defmodule ShoppingListFulfillment do
  use GenServer

  def start_link(shopping_list) do
    GenServer.start_link(__MODULE__, shopping_list)
  end

  def init(shopping_list) do
    state = %{
      shopping_list: shopping_list,
      inventory: []
    }
    {:ok, state, {:continue, :get_inventory}}
  end

  def handle_continue(:get_inventory, %{shopping_list: shopping_list}) do
    inventory = get_inventory_for(shopping_list)
    state = %{
      shopping_list: shopping_list,
      inventory: inventory
    }
    {:noreply, state}
  end

  defp get_inventory_for(shopping_list) do
    # something that could be time consuming!
    # like a web request or a database call
    # returns some inventory info for each item on the shopping list
  end
end
```

现在，`init/2` 返回 `{:ok, state, {:continue, :get_inventory}}`。这将立即触发回调 `handle_continue(:get_inventory, state)`。这个回调保证在我们的 GenServer 继续处理任何其他消息之前完成运行。

## 结语
OTP 21 的 `handle_continue/2` 回调允许我们以非阻塞、异步的方式处理昂贵的 GenServer 初始化工作，避免了竞赛条件。如果你正在构建一个需要处理初始化回调的GenServer，可以考虑使用 `handle_continue/2`。
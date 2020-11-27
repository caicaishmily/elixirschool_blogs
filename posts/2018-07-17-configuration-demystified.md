---
author: Sean Callan
author_link: https://github.com/doomspork
categories: general
date: 2018-07-17
layout: post
tags: ['configuration', 'software design']
title: Configuration Demystified
excerpt: >
  We attempt to clear up some confusion around configuration by looking at the different types, the roles they play, and a different approach we could take.
---

# 揭秘配置

There's been a lot of discussion about configuration in the community lately.
We thought this would be an opportune time to discuss configuration and how best to handle it within an Elixir application.
It is surprising to see how a small change to our applications configuration can eliminate much of the headaches others are experiencing.

最近社区里有很多关于配置的讨论。

我们认为这是一个很好的时机去讨论配置以及如何在 Elixir 应用程序中最好地处理配置。

令人惊讶的是，我们的应用程序配置的一个小小的改变，就可以消除许多其他人正在经历的头疼的问题。

### 配置类型

在我们进一步讨论之前，我们先来看看这两种配置类型和它们所扮演的角色。

__运行时配置__

如果你曾经使用系统环境变量来配置应用程序的某些部分，那么你就会熟悉运行时配置。

顾名思义，这是应用程序在运行时的配置。

当我们将构建的文件部署到不同的系统时，我们可以预期这些值会发生变化。

__构建时配置__

构建时配置，有时也被称为 Application 配置，这是一些不同的东西，虽然这种差异很微妙，但在某些情况下可能是一个陷阱。

当我们考虑到代码和它的配置被编译成可以发布的构建文件时，这种差异就会显现出来。

可以肯定地说，无论应用程序在哪里运行，我们都希望某些东西保持不变；无论在哪里部署，我们都打算使用相同的 `Logger` 配置。

如果依靠依赖注入来进行测试，那么就可以肯定地知道，我们不想在最终的可交付产品中使用这些依赖。

它们配置了我们代码的功能。

### 它是如何做到的

许多人的挫折感来自于 `Application.get_env/2` 和 `System.get_env/1` 的使用。

我们先看一下许多 Elixir 项目中常见的配置。

```elixir
use Mix.Config

config :example_app, Data.Repo,
  adapter: Ecto.Adapters.MySQL,
  username: System.get_env("EXAMPLE_APP_USERNAME"),
  password: System.get_env("EXAMPLE_APP_PASSWORD"),
  hostname: System.get_env("EXAMPLE_APP_HOSTNAME"),
  database: System.get_env("EXAMPLE_APP_DATABASE"),
  pool_size: 10
```

看上去很简单，对不对？

错!

我们将应用程序的配置定义在 `config.exs` 文件和它的朋友中，当我们生成构建工件时，就会被编译，就像 Distillery 产生的那些工件一样。

这意味着那些 `System.get_env/1` 函数需要在编译时处理。

看到问题了吗？

应用程序的代码编译是和系统配置编译耦合在一起的。

如果我们想在本地生成构建工件然后在其他地方运行该怎么办？

如果出现紧急情况，`EXAMPLE_APP_HOSTNAME` 的值被更新了怎么办？

在这种配置下，我们的应用程序需要重新编译才能使更改生效。

让我们用颜色来区分变化来说明这个概念。

![elixir-config-recompile](https://user-images.githubusercontent.com/73386/41503026-d8a66ef4-7185-11e8-95fa-37598f6a56ff.png)

这里可以看到我们的运行时的值是不同的，这就需要我们重新编译我们的代码。

这导致了一个新的构建工件和运行时 B 的配置更新。

我们已经成功地将运行时配置和代码耦合在一起。

为了看到环境的变化在我们的代码中的映射，重新编译是不可避免的。

对于那些使用版本的人来说，这种配置类型的组合往往需要额外的库来弥补这个空隙。

从根本上来说，这个问题是将两个独立的概念混为一谈：构建时配置和运行时配置。

### 新方法

现在让我们看看另一种配置方法，它可以对运行时进行修改，而不需要重新编译我们的代码。

```elixir
defmodule ExampleApp.Repo do
  use Ecto.Repo, otp_app: :data

  def init(_, opts) do
    {:ok, build_opts(opts)}
  end

  defp build_opts(opts) do
    system_opts = [
      database: System.get_env("EXAMPLE_APP_DATABASE"),
      hostname: System.get_env("EXAMPLE_APP_HOSTNAME"),
      password: System.get_env("EXAMPLE_APP_PASSWORD"),
      username: System.get_env("EXAMPLE_APP_USERNAME")
    ]

    Keyword.merge(opts, system_opts)
  end
end
```

在这里，一开始就调用了 `Repo.init/2`, 用当前系统环境的值来更新我们的配置，而不需要重新编译任何东西。

现在我们可以在本地生成我们的构建工件，并在其他地方运行它们。

前面提到的 `EXAMPLE_APP_HOSTNAME` 变化的情况会怎样呢？

应用程序重启以后会拉出最新的值，不需要编译。

让我们更新一下之前图上的说明，以反映这种新的方法。

![elixir-config](https://user-images.githubusercontent.com/73386/41503027-d8b8ecc8-7185-11e8-8284-73d417fea6dc.png)

我们的运行时环境已经发生了变化，但我们的应用程序的配置和构建工件没有，也不应该发生变化。

我们已经成功地将我们的代码与运行时配置解耦了，而且配置是显式的，与代码并存。

### 把它整合起来

In our last example we see the benefit to separating our configuration into two distinct parts.
An easy way to avoid the confusion and pitfalls of configuration is to remember this simple rule: `System.get_env/1` should never be used to populate our application's configuration, the values defined in `config.exs`.

在上一个例子中，我们看到了将配置分成两个不同部分的好处。

避免配置混乱和陷阱的一个简单方法是记住这个简单的规则：永远不应该使用 `System.get_env/1` 读取`config.exs` 中定义的值来填充我们应用程序的配置。

这个担心对本地开发和测试意味着什么？

不用担心!

我们可以将这两种配置类型结合起来，以保持本地开发的简单和方便。

让我们更新一下我们的 `Repo.init/2` 函数，保证其在运行时拒绝任何解析为 `nil` 的值，失败后返回 `opts` 提供的应用程序配置（在 `config.exs`、`dev.exs` 和 `test.exs` 中设置的值）。

```elixir
defmodule ExampleApp.Repo do
  def init(_, opts) do
    {:ok, build_opts(opts)}
  end

  defp build_opts(opts) do
    system_opts = [
      database: System.get_env("EXAMPLE_APP_DATABASE"),
      hostname: System.get_env("EXAMPLE_APP_HOSTNAME"),
      password: System.get_env("EXAMPLE_APP_PASSWORD"),
      username: System.get_env("EXAMPLE_APP_USERNAME")
    ]

    system_opts
    |> remove_empty_opts()
    |> merge_opts(opts)
  end

  defp merge_opts(system_opts, opts) do
    Keyword.merge(opts, system_opts)
  end

  defp remove_empty_opts(system_opts) do
    Enum.reject(system_opts, fn {_k, value} -> is_nil(value) end)
  end
end
```

当我们的应用程序启动时，它会尝试检索这些系统变量，去掉 `nil` 值，最后将我们的应用程序配置所定义的选项与运行时配置合并，优先考虑运行时选项。

现在就可以使用我们已经习惯的 `dev.exs` 和 `test.exs` 文件了，同时也保证了我们最终的构建工件会被正确设置，从而使部署的配置变得轻而易举。

你对这种方法有什么看法？

我们很想听听你的想法！

在下一篇配置文章中，我们将探讨如何设计我们的库，以消除对 `Application.get_env/2` 的需求，同时允许多个独立配置的实例存在于同一个应用程序中。
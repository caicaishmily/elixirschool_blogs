---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: til
date: 2018-12-17
layout: post
tags: ['umbrella applications', 'testing']
title:  TIL How to Run Tests for One Child App in an Umbrella
excerpt: >
  Run all of the tests, or just a specific tests, for a given child app in an umbrella application with this handy command.
---

# TIL 在 Umbrella 子应用中如何运行测试

如果你正在开发一个有多个子应用的 Elixir umbrella 应用，那么你就会知道，为整个 umbrella 应用运行测试并不总是那么理想。它可能需要一段时间才能运行，而且在仅对子应用进行开发时，很难对某一特定的故障集进行归类。

那么，如何为一个特定的子应用运行测试呢？

### 不像这样

我第一次尝试只运行一个子应用的测试，结果有点像这样。

```bash
mix test test/github_client_test.exs
==> learn_client
Paths given to `mix test` did not match any directory/file: test/github_client_test.exs
==> course_suite_client
Paths given to `mix test` did not match any directory/file: test/github_client_test.exs
==> github_client
....

Finished in 0.2 seconds
4 tests, 0 failures

Randomized with seed 395181
==> deployer
Paths given to `mix test` did not match any directory/file: test/github_client_test.exs
==> deployer_web
Paths given to `mix test` did not match any directory/file: test/github_client_test.exs
```

这在 _技术上_ 是可行的 -- 它 _确实_ 运行了我们指定的测试--但它并不是我们想要的。我们从 umbrella app 的根目录下运行的任何 `mix` 命令都会从 `apps/` 目录下的每个子 app 的根目录下递归运行。因此，虽然当命令在包含该测试的子应用中执行时，这确实运行了 `github_client_test.exe` 测试，但它也对 *所有其他子应用* 执行了该命令。导致了这些不太好的错误信息。

```bash
Paths given to `mix test` did not match any directory/file:
test/github_client_test.exs
```

不太理想。

### 像这样

要运行一个特定子应用的 _所有_ 测试，我们可以从式形应用的根目录运行以下内容。

```elixir
mix cmd --app child_app_name mix test --color
```

我们使用 `mix cmd --app` 来表示我们要在其中运行给定的 `mix` 命令的应用程序。之后我们再使用 `mix test` 命令。此外我们还可以指定要运行的测试文件甚至行号。

```elixir
mix cmd --app child_app_name mix test test/child_app_name_test.exs:8 --color
```

其中 `mix test` 命令后面是你要运行的测试文件的路径，_从该子应用的根目录开始_。

`--color` 标志很重要。如果没有这个标志，我们就不能得到高亮的红/绿颜色，这会使我们的测试输出非常难读。

#### 奖励

每次我们想要运行一个给定子应用的测试时，输入这个命令都是挺麻烦的。我们可以定义一个 mix 别名来让我们的生活更轻松一些。[mix alias](https://hexdocs.pm/mix/Mix.html#module-aliases)为我们提供了一种定义自定义 mix 任务的方法，该任务仅在本地可用，而不是在我们应用程序的打包版本中可用，也就是说，不会提供给安装我们的应用程序作为依赖的开发人员。

我们将为我们的子应用程序测试命令定义一个混合别名。

```elixir
# mix.exs for root of umbrella app

def project do
    [
      aliases: aliases(),
      ...
    ]
end

def aliases do
  [
    child_app_name_test: "cmd --app child_app_name mix test --color"
  ]
end
```

然后在 umbrella app 的根目录，我们可以运行：

```bash
mix child_app_name_test
```

或者，运行一个特殊的测试

```bash
mix child_app_name_test test/child_app_name_test.exs
```

就是这样! 一个用于运行子应用的规范的、漂亮的、易用的命令。你可以为你的 umbrella 应用中的每个子应用定义一个这样的别名，并轻松运行这些测试。
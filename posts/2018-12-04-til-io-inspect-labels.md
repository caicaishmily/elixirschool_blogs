---
author: Sean Callan
author_link: https://github.com/doomspork
categories: til
date: 2018-12-04
layout: post
title:  TIL about `IO.inspect/2`'s `:label` opt
excerpt: >
  Did you know you could label your output?  Neither did we!  Check out today's TIL to learn more.
---

# til io inspect labels

如果你曾经发现自己在调试 Elixir，那么你可能很熟悉 `IO.inspect/2`，但为了以防万一，让我们看看一个如何使用它的例子。

```elixir
defmodule Example do
  def sanitize_params(params) do
    params
    |> IO.inspect()
    |> Map.take(["quantity", "price"])
    |> IO.inspect()
    |> Enum.into(%{}, fn {k, v} -> {k, String.to_integer(v)} end)
  end
end
```

我们的函数很简单。给定一个 map，取一些键，并将它们转换为整数；在这个例子中，我们不考虑无效输入和错误处理。

让我们看看 `IEx` 里面的输出

```elixir
iex> params = %{"price" => "100", "quantity" => "1", "onsale" => true}
iex> Example.sanitize_params(params)
%{"onsale" => true, "price" => "100", "quantity" => "1"}
%{"price" => "100", "quantity" => "1"}
%{"price" => 100, "quantity" => 1}
```

并排地看代码和输出，我们可以很容易地跟上，但如果不这样做，我们必须记住我们的 `IO.inspect/2` 调用的内容和位置，输出才有意义。

你能想象屏幕上有 _更多_ 的输出，而代码又不那么简单的情况吗？

请允许我向你介绍我们的新朋友 `:label` 选项!

让我们重温一下之前的代码，介绍一下一直以来很有用的 `:label` 选项，然后直接跳到 `IEx` 输出。

```elixir
defmodule Example do
  def sanitize_params(params) do
    params
    |> IO.inspect(label: "input")
    |> Map.take(["quantity", "price"])
    |> IO.inspect(label: "Map.take/2")
    |> Enum.into(%{}, fn {k, v} -> {k, String.to_integer(v)} end)
  end
end
```

```elixir
iex> params = %{"price" => "100", "quantity" => "1", "onsale" => true}
iex> Example.sanitize_params(params)
input: %{"onsale" => true, "price" => "100", "quantity" => "1"}
Map.take/2: %{"price" => "100", "quantity" => "1"}
%{"price" => 100, "quantity" => 1}
```

哇哦！

我们的调试输出现在有了标签，使我们的代码追踪变得更加容易。

它还可以变得更好!

标签不需要预设，我们可以使用我们捕获的值来制作动态标签。

```elixir
iex> Enum.each(%{"a" => 1, "b" => 2, "c" => 3}, fn {k, v} -> IO.inspect(v, label: k) end)
a: 1
b: 2
c: 3
```

这多酷啊!

你以前知道 `:label` 吗?

如果知道，你是否发现它和我们刚刚使用的一样有用？
---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
date: 2019-01-19
layout: post
title: Understanding Recursion with Elixir
excerpt: >
  De-mystify the concept of recursion and gain a deeper understanding of how and why to use it by writing our very own recursive function in Elixir.
---

# 理解 Elixir 中的递归

"递归" 对于我们这些不熟悉其应用的人来说，可能是一个可怕的词。在这篇文章中，我们将通过在 Elixir 中编写我们自己的递归函数，来消除递归概念的神秘感，并深入了解如何以及为什么要使用它。

## 什么是递归

简而言之，“递归” 就是一个函数调用它自己。

简而言之，"递归" 就是一个函数调用自己。首先我们来看一个人为的例子。在本篇文章的后面，我们将建立一个更实用的递归函数。

下面我们定义了一个函数 `RecursionPractice.hello_world/0`，这个函数会调用自己。

```elixir
defmodule RecursionPractice do
  def hello_world do
    IO.puts("Hello, World!")
    hello_world()
  end
end
```

如果你认为调用我们的 `RecursionPractice.hello_world/0` 函数会导致 `"Hello, World!"` 被无限次地输出到终端上--你是对的! `hello_world` 函数有两个作用。

1. 输出 "Hello, World!"。
2. 调用 `hello_world/0`（再次）。

当再次调用 `hello_world/0` 时，它会做两件事。

1. 输出 "Hello, World!"。
2. 调用 `hello_world/0`（再次）。

虽然 "一个调用自身的函数" 是递归的基本定义，但它 _并不是_ 我们想要实现递归函数的方式。

任何递归函数都需要在一定的条件下 _停止调用自己_ 的方法。这个条件通常被称为 **基例**。让我们为我们的 `RecursionPractice.hello_world/0` 函数创建一个基例。我们将计算我们调用函数的次数，一旦达到 10 次就停止调用。

```elixir
def hello_world(count \\ 0) do
  IO.puts("Hello, World!")
  if count < 10 do
    new_count = count + 1
    hello_world(new_count)
  end
end
```

`if` 条件控制我们的递归函数。如果计数小于 10，将计数增加 1，然后再次调用 `hello_world/1`。否则，_不做任何事情_，即停止调用递归函数!

我们可以借助 [guard 子句](https://elixirschool.com/en/lessons/basics/functions/#guards) 重构这段代码。我们不在函数里面写一个 `if` 条件，而是定义另一个版本的 `RecursionPractice.hello_world/1` 函数来处理我们的基本情况。这个版本将在计数大于或等于 10 的时候运行。

```elixir
defmodule RecursionPractice do
  def hello_world(count \\ 0)
  def hello_world(count) when count >= 10, do: nil

  def hello_world(count) do
    IO.puts("Hello, World!")
    new_count = count + 1
    hello_world(new_count)
  end
end
```

*请注意，我们已经将默认参数定义移到了一个函数头中。如果你正在定义一个有多个子句和默认值的函数，那么默认值的定义应该放在函数头中。在 [Elixir School 的这节课](https://elixirschool.com/en/lessons/basics/functions/#default-arguments) 中了解更多关于默认参数、函数头和函数子句的信息。*

## 为什么它有用?

当我们需要在某个条件下重复一个动作时，递归在任何时候都很有用。任何时候，只要你想使用 `while` 或 `until` 循环，你都可能用递归来实现。

如何决定到底是使用递归方法而不是 `while` 循环这样的迭代方法？编写递归函数时，达到递归比循环方式产生的代码更简单，更容易阅读。不过要注意，如果你写一个递归函数时没有 "基例"，或者没有停止点，你会以堆栈溢出错误告终--你会 _永远地_ 调用这个函数。

## 使用 Elixir 构建递归函数

现在我们对递归是什么以及它是如何工作的有了更好的理解，让我们建立一个更实用的递归函数。

Elixir 的 `List` 模块为我们提供了许多方便的函数，用于对列表进行操作，其中包括一个 `List.delete/2` 函数，它的工作原理如下。

给定一个列表和该列表中的一个元素，返回一个新的列表，该列表不包含给定元素的 _第一次出现_。例如：

```elixir
List.delete(["Apple", "Pear", "Grapefruit"], "Pear")
=> ["Apple", "Grapefruit"]
```

然而，我们将看到，如果给定的列表中包含一个以上的 `"Pear"`, `List.delete/2` 只删除 _第一个_ `"Pear"`

```elixir
List.delete(["Apple", "Pear", "Grapefruit", "Pear"], "Pear")
["Apple", "Grapefruit", "Pear"]
```

如果我们想从列表中删除所有出现的某个元素，该怎么办？`List` 模块没有实现这样的功能。让我们建立我们自己的函数吧!

我们想要的行为是这样的。

```elixir
List.delete(["Apple", "Pear", "Grapefruit", "Pear"], "Pear")
["Apple", "Grapefruit"]
```

在开始构建我们的函数之前，我们先来看看如何使用递归和模式匹配来操作 Elixir 列表。

### 在一个列表上使用递归

> Elixir 中的列表是非常有效的链表，这意味着它们在内部以包含链表头和尾的对来表示。- Hex Docs

这意味着我们可以使用 [模式匹配](https://elixirschool.com/en/lessons/basics/pattern-matching/) 来抓取第一个元素，也就是列表的 "head"。

```elixir
iex> [head | tail] = [1,2,3]
iex> head
1
iex> tail
[2,3]
```

利用这种模式匹配的方法，我们可以对列表的每个成员进行操作:

```
iex> list = [1,2,3,4]
[1, 2, 3, 4]
iex> [head | tail] = list
[1, 2, 3, 4]
iex> head
1
iex> tail
[2, 3, 4]
iex> [head | tail] = tail
[2, 3, 4]
iex> head
2
iex> tail
[3, 4]
iex> [head | tail] = tail
[3, 4]
iex> head
3
iex> tail
[4]
iex> [head | tail] = tail
[4]
iex> head
4
iex> tail
[]
```

使用这种方法，让我们定义一个自定义函数，以递归一个列表中的每个元素。

我们的函数将抓取列表中的 `head`，并将其 `puts` 到终端。然后我们将 `tail` 分割成它自己的 `head` 和 `tail`。我们会一直这样做，直到列表为空。

```elixir
defmodule MyList do
  def my_each([head | tail]) do
    IO.puts(head)
    if tail != [] do
      my_each(tail)
    end
  end
end
```

我们的 **基例** 发生在 `tail` 为空时，即列表中没有更多元素时。我们可以利用 [Elixir 的模式匹配函数 arity 的能力](https://elixirschool.com/en/lessons/basics/functions/#functions-and-pattern-matching) 来清理一下这个问题。

我们不在递归函数中实现 `if` 条件，而是定义另一个版本的函数，当调用 `my_each` 时，其参数为空列表，该函数将被运行。所以如果  `my_each` 被调用时的参数是一个非空的列表，第一个版本的函数将被运行。它将抓取列表的 `head` 并 `puts` 出来。然后它将用列表的 `head` 作为参数再次调用 `my_each`。如果尾部是空的，函数的第二个版本就会运行。在这种情况下，我们将不会再次调 用`my_each`。

```elixir
defmodule MyList do
  def my_each([head | tail]) do
    IO.puts(head)
    my_each(tail)
  end

  def my_each([]), do: nil
end
```

我们来看看运行结果：

```elixir
iex> MyList.my_each([1,2,3,4])
1
2
3
4
```

现在我们已经掌握了如何使用递归和 Elixir 列表的模式匹配，让我们回到我们的 "全部删除" 递归函数。

### 定义一个 `delete_all/2` 递归函数
#### 期望的行为

在我们开始编码之前，让我们先规划一下我们的函数需要如何表现。由于 Elixir 是一种函数式语言，我们不会对原始列表进行改变。取而代之的是，我们将建立一个由原始列表中的所有元素组成的新列表，减去与我们想要排除的元素相匹配的所有元素。

我们的方法是这样的。

* 看看列表的头部 如果该元素等于我们要移除的元素的出现值，我们将 *不* 抓取该元素添加到新列表中。
* 如果该元素不等于我们要删除的值，我们将把它添加到新的列表中。
* 无论在哪种情况下，我们都会抓取列表的尾部，然后重复上一步。
* 一旦尾部为空，即我们已经查看了列表中的每个元素，就停止递归。

#### 让我们创建它!

首先，我们要定义一个 `MyList.delete_all/2` 函数，它接收两个参数：原始列表和我们想要删除的元素

```elixir
defmodule MyList
  def delete_all(list, el) do
    # coming soon!
  end
end
```

然而，我们需要访问一个新的空列表，我们将用我们不删除的原始列表中的元素来填充。所以，我们将定义一个版本的 `delete_all` ，它将接受三个参数：原始列表、我们要删除的元素和新的空列表。

`MyList.delete_all/2` 将调用 `MyList.delete_all/3` 函数。这就省去了用户必须用空列表的第三个参数来调用 `delete_all`，并允许我们提供一个漂亮的整洁的 API。

```elixir
defmodule MyList
  def delete_all(list, el) do
    delete_all(list, el, [])
  end

  def delete_all([head | list], el, new_list) do
  end
end
```

`MyList.delete_all/3` 函数的第一项工作是确定当前列表中的第一个元素，即列表的 `head` 是否与我们要删除的元素值相同。

如果是，我们 *不* 会将它添加到我们的新列表中。相反，我们将再次调用 `MyList.delete_all/3` 与当前列表的剩余部分，即 `tail`，并传入我们的没改变的 `new_list`。我们可以通过一个卫兵子句来实现。

```elixir
def delete_all([head | tail], el, new_list) when head === el do
  delete_all(tail, el, new_list)
end
```

如果当前列表的头部 *不* 等于我们要删除的值，但是，我们 *希望* 在继续前进之前将其添加到 `new_list` 中。

我们将定义另一个 `delete_all/3` 函数，这次不使用卫兵子句，以满足这个条件。

```elixir
def delete_all([head | tail], el, new_list) do
  delete_all(tail, el, [head | new_list])
end
```

我们将当前 `head` 添加至我们新的列表中。像这样：

```elixir
[ head | new_list ]
```

然后我们再次调用 `delete_all/3`，将列表的剩余部分（`tail`）、要删除的元素和更新后的 `new_list` 传递给它。

什么时候我们应该停止递归？换句话说，什么情况下会导致我们停止调用 `delete_all/3` 呢？当我们已经递归了原始列表中的所有元素，使 `tail` 为空时，我们将停止调用 `delete_all/3` 并返回新的列表。让我们定义一个最后的 `delete_all/3` 函数来匹配这个条件。

```elixir
def delete_all([], el, new_list) do
  new_list
end
```

这种方法唯一的问题是，它建立并返回一个新的列表，在这个列表中，我们从原始列表中保留的所有元素都以相反的顺序填充。这是因为通过这样构建出我们的新列表。

```elixir
[ head | new_list ]
```

我们将我们想保留的元素添加到新列表的前面，而不是最后。

一旦我们达到空列表的基本情况，我们可以通过在 `new_list` 上使用 `Enum.reverse` 来解决这个问题。

```elixir
def delete_all([], el, new_list) do
  Enum.reverse(new_list)
end  
```

如果我们把所有的代码放在一起，我们就有了：

```elixir
defmodule MyList do
  def delete_all(list, el) do
    delete_all(list, el, [])
  end

  def delete_all([head | tail], el, new_list) when head === el do
    delete_all(tail, el, new_list)
  end

  def delete_all([head | tail], el, new_list) do
    delete_all(tail, el, [head | new_list])
  end

  def delete_all([], el, new_list) do
    Enum.reverse(new_list)
  end  
end
```

我们甚至可以更进一步，用 Elixir 的模式匹配函数 arity 的能力来替换我们的守护子句。当 `head === el` 时，我们可以像这样写函数，而不是使用卫兵子句来运行某个版本的函数。

```elixir
def delete_all([el | tail], el, new_list) do
  delete_all(tail, el, new_list)
end
```

现在我们可以调用我们的函数：

```elixir
iex> MyList.delete_all(["Apple", "Pear", "Grapefruit", "Pear"], "Pear")
["Apple", "Grapefruit"]
```

就是这样！
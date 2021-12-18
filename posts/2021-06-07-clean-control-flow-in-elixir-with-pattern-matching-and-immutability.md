---
%{
    author: "Cristine Guadelupe",
    author_link: "https://github.com/cristineguadelupe",
    tags: ["general"],
    date: ~D[2021-06-07],
    title: "Clean Control Flow in Elixir with Pattern Matching and Immutability",
    excerpt: """
    Learn how to use pattern matching instead of guard clauses to implement really clean control flow in Elixir.
    """
}
---

Elixir 最让我着迷的功能之一就是模式匹配。我总是想知道它是不是我所需要的那个用来解决问题的东西，我喜欢探索它。当你把模式匹配的美丽和不可变的力量结合起来时，有些事情几乎看起来很神奇，但它们并不神奇!

我的重点不是介绍关于模式匹配和不可变特性的所有内容，而是展示在 Elixir 中我们如何使用模式匹配而不是守卫子句来实现干净的控制流。

在这篇文章中，我们将专注于实现桌面游戏《战舰》的逻辑。我们要实现的第一条规则很简单：一个玩家不能在一行里面连续走两次。解决这个问题的方法之一是跟踪最后一个移动的玩家。

有了这些信息，我们现在有两种可能性：如果即将下棋的棋手与上一次下棋的棋手相同，我们就会忽略这步棋。否则我们就可以计算这步棋。

根据我们对 Elixir 的经验，我们可能会选择一个条件作为第一个解决方案，比如说：

```elixir
def maybe_move(player, last_player) do
    if player != last_player do
        player
        |> make_a_move()
        |> set_as_last_player()
    else
      :ignored
    end
end
```

或者使用带有守卫子句的模式匹配

```elixir
def maybe_move(player, last_player) when player == last_player do
    :ignored
end

def maybe_move(player, last_player) do
    player
    |> make_a_move()
    |> set_as_last_player()
end
```

但是我们有可能将之前在守卫子句解决方案中已经使用的模式匹配与不可变性的力量结合起来，得出一个更加炼金主义的解决方案！

```elixir
def maybe_move(last_player, last_player) do
    :ignored
end

def maybe_move(player, last_player) do
    player
    |> make_a_move()
    |> set_as_last_player()
end
```

等一下，我们在这里做了什么？

我们定义了第一版的 `maybe_move` 函数，它的第一和第二个参数都是 `last_player`。这意味着该函数只有在第一个参数所代表的玩家与第二个参数代表的玩家相匹配时才会执行。由于不可变性，当我们以相同的名字调用两个参数时，Elixir 会检查它们是否真的相同！我们可以很容易地把两个参数都称为 player 或者甚至命名像 player_is_the_last_player 这样的参数。这并不重要! 规则是：如果我们想确保相同，我们就用相同的名字调用这两个参数。

**好了，现在是使用我们漂亮的小代码进行游戏的时候了!**

假设我们有 `player1` 和 `player2`，`player1` 走了最近的一步棋，因此 player1 是我们的最后一名玩家，现在 `player2` 将尝试移动！

所以我们将调用函数 `maybe_move(player2, player1)`，其中 `player2` 是想走棋的玩家，`player1` 是 last_player。

我们有两个需要传递 2 个参数的 `maybe_move` 函数，所以 Elixir 将尝试从上到下进行模式匹配，即它将尝试匹配的第一个函数是：

```elixir
def maybe_move(last_player, last_player) do
    :ignored
end
```

我们的第一个参数是 `player2`，Elixir 将用 `last_player = player2` 来绑定它，由于第二个参数也是 `last_player`，此时 Elixir 将使用 `^`（pin 运算符）来检查之前的绑定对第二个参数是否生效，而不是试图重新绑定它。

```elixir
last_player = player2
ˆlast_player = player1
```

由于 player2 与 player1 不同，当前没有一个有效的模式匹配，因此 Elixir 将继续尝试与下一个函数匹配!

**我们尝试下一个匹配!**

```elixir
def maybe_move(player, last_player) do
    player
    |> make_a_move()
    |> set_as_last_player()
end
```

现在的行为将是不同的，我们要求 Elixir 匹配两个不同的参数。也就是说，为每个参数做一个绑定。

```elixir
player = player2
last_player = player1
```

有了有效的匹配，我们的函数就会运行! Player2 将开始下一步棋，然后被注册为我们的新的 `last_player`！

如果 player2 连续尝试另一步棋怎么办？

那么，我们将再次调用第一个 `maybe_move` 函数，并尝试进行匹配。Player2 想要下棋，并且 Player2 也是 `last_player`，所以我们得到以下调用:

`maybe_move(player2, player2)`

试图将其与第一个 maybe_move 函数相匹配，我们得到以下匹配:

```elixir
def maybe_move(last_player, last_player) do
    :ignored
end
```

```elixir
last_player = player2
ˆlast_player = player2
```

这是一个有效的匹配! 由于我们的函数只是忽略了下棋的尝试，所以在另一个棋手尝试下棋之前，什么都不会发生!
就是这样！我们已经了解到模式匹配和数据不可变性如何共同为控制流提供一个优雅的解决方案，这是我们 Elixir 工具箱中的另一个工具!

## 资源

如果你想了解更多关于模式匹配的信息，你可以在 ElixirSchool 找到令人惊奇的资料。

- [模式匹配](https://elixirschool.com/en/lessons/basics/pattern-matching/)
- [函数和模式匹配](https://elixirschool.com/en/lessons/basics/functions/#functions-and-pattern-matching)

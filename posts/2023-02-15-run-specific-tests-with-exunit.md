%{
author: "Kevin Mathew",
author_link: "https://github.com/kevinam99",
tags: ["testing", "ExUnit"],
date: ~D[2023-03-19],
title: "Run specific test cases with ExUnit",
excerpt: """
Learn how you can easily run one specific test case or a group test cases in Elixir without having to run through the whole test suites and custom workarounds.
"""
}

---

当开始使用 Elixir 时，我很惊讶在 Elixir 中用 ExUnit 编写测试是多么容易，并且能够进行真正的 TDD。这在我做自己的项目时很好，因为项目规模小，也没有很多组件。但当我开始全职工作时，就很难测试我在大代码库中写的特定功能了。

我发现自己把那些想跳过的测试都注释掉了，这使得整个过程很乏味，我又回到了 iex 中手动测试功能。当我发现这两种运行特定测试案例的方法时，我的苦恼就结束了，我希望这对我的帮助和对你的帮助一样大。

想象一下，一个测试文件有几个测试，但你只想运行 `add/2` 函数的测试用例，该函数只是简单地将两个数字相加并给出一个结果。

```elixir
test "it adds two numbers correctly" do
  assert add(2, 2) == 4
end
```

有三种方法可以解决这个问题：

### 1. 在 `mix test` 命令中指定测试的行号

如果上述测试案例在测试文件中的第 100 行，那么运行你的命令为：

```shell
$ mix test ./path/to/test/file/math_test.exs:100
```

这将运行第 100 行的测试用例，跳过所有其他测试。当然，基本的设置将被完成。轰隆隆! 节省了这么多时间，又回到了高效的状态。

现在，虽然这是一个很好的解决方案，但它是暂时的，并且依赖于行号保持不变。如果测试用例移动到不同的行，要么预期的测试用例不会运行，要么所有的测试都会被跳过，因为在那一行没有找到测试用例。让我们来谈谈如何解决这些问题。

### 2. 使用 `@tag` 属性

考虑与上述相同的测试案例，但有一个小变化

```elixir
@tag run: true
test "it adds two numbers correctly" do
  assert add(2, 2) == 4
end
```

在这种情况下，要运行的命令是

```shell
$ mix test ./path/to/test/file/math_test.exs --only run:true
```

这个命令会找到你指定的标签并运行相关的测试。这种方法是不分行的，只需要正确的标签。标签可以是任何在你的项目中最有意义的键值对，但要确保使用 `@tag`。这可以进一步扩展并添加到你想一起运行的其他测试。当这些测试用例可能散落在测试文件中时，这很有帮助。

但是，如果你想运行的测试已经先后在一起，你不想重复标记它们，怎么办？这个时候 `describe` 宏的就可以帮助你了。

### 3. `describe` 宏

使用 `describe `宏将测试相同功能的测试用例分组是一个好的做法。

例如:

```elixir
describe "math functions" do
  test "it adds two numbers correctly" do
    assert add(2, 2) == 4
  end

  test "it returns an error tuple when dividing by 0" do
    assert divide(2, 0) == {:error, "Cannot divide by 0"}
  end

  test "it computes the max value correctly" do
    assert max(4, 2) == 4
  end
end

```

在这里，你只想测试那里的数学函数，但不需要麻烦地运行 `mix test` 命令，把行号写三次，也不需要写三次标签。你要做的与我们在第一种方法中讨论的类似，行号，希望这次是`describe` 的行号。

你可以使用 `@tag` 在一个测试中运行不同的特定 `describe`。`mix test` 命令仍将与前两种方法中看到的一样。

这就是全部了! 希望这对你有帮助，使你应用程序的测试更加容易。

## 结语

ExUnit 是一个强大而简单的工具，使你的 Elixir 应用程序的测试变得容易和强壮。它所提供的选项在处理功能测试时提供了恒定的生产力，不会浪费任何时间。

我们讨论的两件事是使用测试案例的行号或 `describe` 和使用 `@tag` 属性。请记住，你可以使用任何标签，只要你在 `mix test` 命令中指定 `@tag` 和正确的名称。

另外，提醒一下关于 doctests。编写 doctests 可以为你的团队和你自己省去记住一个函数做什么和返回的痛苦。Doctests 还提供了一个免费的测试；)

额外提示：如果你正在运行一个 umbrella 项目，或者只是想运行所有具有相同标签的测试，只需运行相同的 `mix test` 命令，使用 `--only` 标志，但不指定测试文件的路径。

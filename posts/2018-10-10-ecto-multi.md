---
author: Svilen Gospodinov
author_link: https://github.com/svileng
categories: general
tags: ['ecto', 'software design']
date:   2018-11-07
layout: post
title:  A brief guide to Ecto.Multi
excerpt: Learn how to compose and execute batches of queries using Ecto.Multi.
---

# Ecto.Multi 简单指导

Ecto.Multi 是一套旨在合成和执行原子操作的实用程序，通常（但并不总是，你很快就会看到）针对数据库执行。此外，它还能处理回滚，提供成功或错误的结果，扁平化嵌套代码，并节省多次往返数据库的时间。

如果你发现自己正在运行和管理许多数据库查询（和其他操作），那么继续阅读，你可能会发现一些有用的工具然后把它们加入到你的 Elixir/Ecto 工具箱。
## 创建一个 Multi

一切都是从 `%Multi{}` 结构开始，你可以调用 `Ecto.new()` 函数创建一个新的 Multi。

```elixir
iex> Ecto.Multi.new()
%Ecto.Multi{names: %MapSet<[]>, operations: []}
```
## 执行 Multi 操作

要运行一个 Multi，你必须把它扔给 `Repo.transaction/1`：

```elixir
iex> Ecto.Multi.new() |> Repo.transaction()
{:ok, %{}}
```

很明显，我们刚刚运行了一个空的 Multi，这显然会成功，因为没有执行任何操作（在 {:ok, return} 元组的第二个元素中也没有返回任何内容。为了使 Multis 有用，你需要向它们添加操作。

接下来，我们将介绍你可能最终要做的一些最常见的操作。

## 跟单个 changesets 一起工作

当使用多个 `%Ecto.Changeset{}` 时，通常你会多次调用 `Repo.insert/1` / `update/1` 等来运行这些操作。切换到 `Ecto.Multi` 就像将 `Repo.update/1` 替换成对应的 `Ecto.Multi.update/3` 一样简单。

举个例子。

假设你已经事先创建了 `team_changeset`、`user_changeset` 和 `foo_changeset`，那么就会是这样的。

```elixir
Ecto.Multi.new()
|> Ecto.Multi.insert(:team, team_changeset)
|> Ecto.Multi.update(:user, user_changeset)
|> Ecto.Multi.delete(:foo, foo_changeset)
|> Repo.transaction()
```

使用的原子-- `:user`, `:team` 和 `:foo` --由你决定。你可以传递任何东西（也可以使用字符串，而不是原子），只要它是当前 Multi 的唯一值。

## 上一次操作的结果
Operations will be run in the order they’re added to the Multi. Often you need the result of a previous operation, which you can get by running a custom Multi operation, like so:

Multi 中的操作将按照添加它们的顺序运行。通常您需要前一个操作的结果，您可以通过运行自定义的 Multi 操作来获得，比如这样：

```elixir
Ecto.Multi.new()
|> Ecto.Multi.insert(:team, team_changeset)
|> Ecto.Multi.run(:user, fn repo, %{team: team} ->
  # Use the inserted team.
  repo.update(user_changeset)
end)
```

`Ecto.Multi.run/3` 的第一个参数需要一个名字，就像 Multi insert/delete/update 等函数一样，我把它叫做 `:user`；第二个参数是一个函数，它为你提供当前 Ecto Repo 的情况以及之前的操作结果。这个结果是一个 map 结构，你可以使用唯一的键进行模式匹配得到特定操作的结果，本例中是 `:team`。

注意这里我们调用的是 `repo.update(user_changeset)` (这和 `Ecto.Repo.update/1` 是同一个函数)；你需要从你传递给 `Multi.run/3` 的函数中返回一个 `{:ok, val}` 或 `{:error, val}` 元组。使用 `Repo.update` 就能得到我们需要的东西。

## 自定义操作
实际上, `Multi.run/3` 几乎可以用于很多情况。只要你返回一个 成功/错误 元组，它就会成为同一个原子事务的一部分。

```elixir
Ecto.Multi.new()
|> Ecto.Multi.insert_all(:users, MyApp.User, users)
|> Ecto.Multi.run(:pro_users, fn _repo, %{users: users} ->
  result = Enum.filter(users, & &1.role == "pro")
  {:ok, result}
end)
```

这里 `Repo.transaction/1` 返回的结果 `:pro_users` 可以用于后续的操作。这是一个确保代码与其他数据库操作一起运行的好方法。如果 `:users` 操作失败或在这之前发生其他事情，这个潜在的昂贵的过滤功能将永远不会被执行。
## 跟多个 Mutils 和动态数据一起使用

Ecto.Multi 的好处是它只是一个数据结构，你可以把它传来传去。它很容易动态地生成数据并将不同的 multis 组合在一起，然后再将所有的事情作为一个事务来执行。

```elixir
posts
|> Stream.filter(fn post ->
  # Filter old posts...
end)
|> Stream.map(fn post ->
  # Create changesets.
  Ecto.Changeset.change(post, %{category: "new"})
end)
|> Stream.map(fn post_cs ->
  # Create a Multi with a single update
  # operation, generating a unique key for the op.
  key = "post_#{post_cs.data.id}"
  Ecto.Multi.update(Ecto.Multi.new(), key, post_cs)
end)
|> Enum.reduce(Multi.new(), &Multi.append/2)
```

多亏了 `Multi.append/2`，我们现在有了一个单一的 Multi 可以让所有的更新操作都按顺序进行。如果你需要的话，还有 `merge` 和 `prepend`。
## 处理返回结果

一旦你调用 `Repo.transaction/1`，你就可以对返回的结果元组进行模式匹配。

在成功的情况下，你将收到所有的 `{:ok, result}`，结果是一个 map；操作及其成功的结果将在返回结果的 map 中，在你选择的唯一键下。

如果出现错误，所有的数据库操作将被回滚，你将得到 `{:error，failed_operation，failed_value，changes_so_far}`，它允许你单独处理特定操作的错误，并检查它们。请注意，`changes_so_far` 的意思只是 "在这一次失败之前的操作都很顺利"，实际上数据库中没有留下任何数据。

```elixir
Ecto.Multi.new()
|> Ecto.Multi.insert(:team, team_changeset)
|> Ecto.Multi.update(:user, user_changeset)
|> Ecto.Multi.delete(:foo, foo_changeset)
|> Repo.transaction
|> case do
  {:ok, %{user: user, team: team, foo: foo}} ->
    # Yay, success!
  {:error, :foo, value, _} ->
    # It seems that :foo failed!
  {:error, op, res, others} ->
    # One of the others failed!
end
```

## 结语

这个简短的指南试图涵盖最常见的用例和功能。希望你觉得有用。

如果你想了解更多--请前往[Ecto.Multi官方文档](https://hexdocs.pm/ecto/Ecto.Multi.html)，在那里你可以探索所有可用的东西。
---
author: Prince Wilson
date: 2019-03-25
layout: post
categories: til
tags: ['ecto']
author_link: https://github.com/maxcell
title:  TIL Ecto Constraints and Validations
excerpt: >
  Let's take a look at how Ecto handles these two ways of ensuring data integrity
---

# TIL Ecto 约束和验证

开发人员希望为用户创造最好的应用。在这个过程中，他们希望确保在数据没有被保存到数据库中时，能够给用户提供良好的反馈。

在 Elixir 中，数据库之上有一个很好的工具可以帮助他们-- Ecto! 它可以对特定字段进行验证和约束，以确保数据的完整性。

然而，你知道验证和约束之间有区别吗？我不知道。在建立一个副项目的过程中，我遇到了几次问题！我想说的是，这两种方法都是为了确保数据的完整性。让我们讨论一下每种方法的目的，看看它们之间的区别。我们会在最后深入探讨**为什么我们需要它们**。

## 数据完整性是规则 #1

> 数据完整性是指在数据的整个生命周期中维护和保证数据的准确性和一致性。 -- [维基百科](https://en.wikipedia.org/wiki/Data_integrity)

因此，我们正在构建一个超级酷的应用程序，用户可以登录和登出。我们可能会有这样的一些模式。

```elixir
# Using Phoenix 1.4 with Contexts but still applies all the same
defmodule MyCoolWebApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :display_name, :string
    field :email, :string
    field :password_hash, :string
    field :password, :string, virtual: true

    timestamps()
  end
end
```

在我们的 schema 里面，我们要描述 `changeset`。Ecto.Changesets 允许我们过滤、铸造、验证和约束由我们的模式创建的代表数据库记录的结构。让我们来看看一个 _只_ 铸造的 changeset。

```elixir
def changeset(user, attrs) do
  user
  |> cast(attrs, [:display_name, :email, :password])
end
```

如果我们只有这些，可能就会有一些让人头疼的事情发生。很容易在没有填写所有字段的情况下过早地提交表格。潜在地，现在数据库中用户的个人资料里电子邮件或密码拥有一个可爱的 `nil`，或者更糟的 `""`。这对用户和开发者来说都是很糟糕的事情。

因此，为了解决这个问题，我们将使用一个验证!

### 验证

Ecto 中的许多验证将在不需要与数据库交互的情况下被执行，这意味着验证将在尝试插入或更新数据库之前被执行。如果我们想在数据库中插入一个新的用户，我们首先要确保 changeset 里面有数据。

让我们往 `changeset` 添加 [`validate_required/3`](https://hexdocs.pm/ecto/Ecto.Changeset.html#validate_required/3):

```elixir
def changeset(user, attrs) do
  user
  |> cast(attrs, [:display_name, :email, :password])
  |> validate_required([:display_name, :email, :password])
end
```

而且，Ecto 免费为我们的 changeset 增加了描述性错误。

```elixir
iex> %User{} |> User.changeset(%{})
%Ecto.Changeset<
  action: nil,
  changes: %{},
  errors: [
    display_name: {"can't be blank", [validation: :required]},
    email: {"can't be blank", [validation: :required]},
    password: {"can't be blank", [validation: :required]}
  ],
  data: %MyCoolWebApp.Accounts.User<>,
  valid?: false
>
```

有很多验证可以增强你的应用程序！请看一下 [Ecto.Changeset](https://hexdocs.pm/ecto/Ecto.Changeset.html#summary) 的文档。在下一节，我们将看看为什么需要应用约束，以及如何使用它们。
### 约束

如果我们使用了验证，那为什么还需要约束呢？让我们想想我们之前使用的很多应用。当我们注册一个应用程序时，我们是否可以用与另一个用户相同的电子邮件来注册？（提示：答案应该永远是否定的）。

那么，为什么我们就不能有一个唯一性的验证呢？记住，根据定义，验证是 _在检查数据库之前_ 执行的。如果我们有一个唯一性的验证，那就意味着所有的东西都是唯一的，即使你添加了重复的内容，因为它不检查数据库。

约束是一个由 **数据库** 执行的规则。你的应用程序将首先运行 Ecto.Changeset 的验证，而不是和数据库交互。然后它将通过检查数据库来执行任何约束。

让我们添加我们的第一个约束来强制用户使用唯一的电子邮件!

```elixir
user
|> cast(attrs, [:display_name, :email, :password])
|> validate_required([:display_name, :email, :password])
|> unique_constraint(:email)
```

如果我们试着往数据库中添加用户，第一次看上去都一切都很好：

```elixir
iex> user = %User{}
iex> attrs = %{display_name: "prince", email: "prince@test.com", password: "super_secret"}
iex> user |> User.changeset(attrs) |> Repo.insert()
{:ok,
 %MyCoolWebApp.Accounts.User{
   __meta__: %Ecto.Schema.Metadata<:loaded, "users">,
   display_name: "prince",
   email: "prince@test.com",
   id: 1,
   inserted_at: ~N[2019-03-18 01:41:34],
   password: "super_secret",
   password_hash: "$argon2i$v=19$m=65536,t=6,p=1$bhjgmBs9/gYcM2L5Z5sL/g$Z+4D7NIaauU+jwhdYRY4hz0adUdhjAJK6CwYk1AOJdE",
   updated_at: ~N[2019-03-18 01:41:34]
 }}
```

我们要确保没有重复的东西被保存下来，所以我们再试着发送同样的东西。

```elixir
iex> user = %User{}
iex> attrs = %{display_name: "prince", email: "prince@test.com", password: "super_secret"}
iex> user |> User.changeset(attrs) |> Repo.insert()
{:ok,
 %MyCoolWebApp.Accounts.User{
   __meta__: %Ecto.Schema.Metadata<:loaded, "users">,
   display_name: "prince",
   email: "prince@test.com",
   id: 2,
   inserted_at: ~N[2019-03-18 01:43:57],
   password: "super_secret",
   password_hash: "$argon2i$v=19$m=65536,t=6,p=1$H+Fq/IPW+M0YPHOZxMs13Q$ne+jDkwfcOigT8TKDIBYJjVwNdaNkzF/hc7YcRXRItY",
   updated_at: ~N[2019-03-18 01:43:57]
 }}
```

这很奇怪。它没有显示错误？那是因为事实上 Ecto 不知道这是一个错误。

它认为数据是安全的，然后把记录保存到数据库中! 如果你在 `changeset/3` 中添加了一个约束，你 **必须** 在数据库级别强制执行这个约束，这样它才能正确地抛出唯一性错误。所以对于我们的 `unique_constraint`，我们需要确保为 email 字段创建一个 `unique_index`。即使你在 `changeset/3` 中写了 `unique_constraint`，除非有一个 `unique_index` 约束应用到数据库中，否则它不会检查该约束。

所以我们需要创建并运行 migration：

```elixir
defmodule MyCoolWebApp.Repo.Migrations.UpdateUniqueEmailsToUsers do
  use Ecto.Migration

  def change do
    create unique_index(:users, [:email])
  end
end
```

```bash
$ mix ecto.migrate
```

现在如果我们再试一遍，我们将会看到记录没有被保存：

```elixir
iex> user = %User{}
iex> attrs = %{display_name: "prince", email: "prince@test.com", password: "super_secret"}
iex> user |> User.changeset(attrs) |> Repo.insert()
{:error,
 %Ecto.Changeset<
   action: :insert,
   changes: %{
     display_name: "prince",
     email: "prince@test.com",
     password: "super_secret",
     password_hash: "$argon2i$v=19$m=65536,t=6,p=1$b6gWjyTiL+JGV6Gz3DjE6A$5m67mfrU/y9YV7adpJ5GXb4+Uh7ley1H3Dz88gCJ4K8"
   },
   errors: [
     email: {"has already been taken",
      [constraint: :unique, constraint_name: "users_email_index"]}
   ],
   data: %MyCoolWebApp.Accounts.User<>,
   valid?: false
 >}
```

现在，我们有了一个确保没有两个用户共享相同电子邮件的应用程序! 约束是很重要的，可以确保在数据库层面上，数据仍然具有完整性。

不过这里要谈一个注意事项，在验证可以同时检查的情况下，约束会逐一失效。如果你的表有好几个约束，每个约束都被违反，你的数据库只会给你一个它发现的第一个约束错误。最好是在应用约束之前，先在验证中尽可能多的捕获错误。

### 验证, 约束, 还是两者兼而有之?

验证和约束有同样的目标，即确保你的数据具有完整性。在考虑应用哪一个时，我们应该问自己两个问题。

1. 你是想防止脏数据写入数据库吗？那你**必须**要有一个约束。
2. 你是想要让用户在应用中发现他们自己可以修复的错误吗？这时你可以使用验证。

当我们需要以不同的方式检查数据以确保数据的完整性时，我们需要这两者。在这个例子中，我们需要在保存到数据库之前检查用户是否发送了非空数据，我们还想确保他们没有任何重复的数据。他们不能知道在我们的数据库中已经有一个他们的电子邮件，所以我们有一个约束。然而，他们可以修正他们忘记填写的字段，所以我们有一个验证。
### 结语

Ecto 对于开发人员来说是一个强大的工具，它可以让他们更容易地与数据库对接。然而，了解它的重要性，这样我们才能正确使用它。当你在考虑你的数据库设计时，一定要先考虑一下你需要执行的验证和约束!
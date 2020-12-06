---
author: Kate Travers
author_link: https://github.com/ktravers
categories: til
tags: ['ecto']
date: 2019-08-23
layout: post
title: TIL How to Select Merge with Ecto.Query
excerpt: >
  Before you reach for adding another association to your schema, consider using `Ecto.Query#select_merge/3` with a virtual field instead.
---

# TIL 如何使用 Ecto.Query 选择合并

在使用 Elixir 和 Ecto 的过程中，我遇到过这样的情况：我需要从一个表中检索数据，也许还要从一个未关联的表中检索一两个字段。过去，每当发生这种情况时，我通常会做一些我不是很满意的事情--也许是更新模式，把它分成多个查询，或者是建立一个多查询语句，如果我觉得很有创意的话。

值得高兴的是，今天我知道了有一个更好的方法。你可以用 `Ecto.Query#select_merge/3` 用一个查询表达式来实现同样的最终结果。

让我们通过一个例子来看看它的操作。

## 设置

假设你在一所有招生部门的学校工作，你的任务是显示与某一招生有关的所有事件，组织成三列。1) 日期 2) 采取的行动 3) 谁采取的行动。

| Date     | Action           | Taken By         |
|----------|------------------|------------------|
| 8/1/2019 | Student Admitted | Albus Dumbledore |


首先，我们有一个 `AdmissionEvent` schema，看起来像这样。

```elixir
defmodule Registrar.Tracking.AdmissionEvent do
  use Ecto.Schema

  schema "admission_events" do
    field(:action, :string)
    field(:admission_id, :integer)
    field(:admitter_uuid, Ecto.UUID)
    field(:occurred_at, :naive_datetime)
  end
end
```

...然后 User schema 看起来像这样:

```elixir
defmodule Registrar.User do
  use Ecto.Schema

  schema "users" do
    field(:uuid, Ecto.UUID)
    field(:full_name, :string)
  end
end
```

...为了完整起见，还提供了一个超级简单的 `Admission` schema:

```elixir
defmodule Registrar.Admission do
  use Ecto.Schema

  schema "admission" do
    field(:admittee_uuid, Ecto.UUID)
    field(:admitter_uuid, Ecto.UUID)
    field(:admitted_at, :naive_datetime)
  end
end
```

这里的问题是，录取者的全名存在于 `User` 中，而用户目前并没有与 `AdmissionEvent` 关联。因此，如果我们直接执行选择查询，我们最终会得到我们需要填充表的录取事件，但不会得到录取者的全名。

```elixir
defmodule Registrar.Tracking.AdmissionEvent do
  use Ecto.Schema
  import Ecto.Query, only: [from: 2]
  alias Registrar.Admission
  alias Registrar.Tracking.AdmissionEvent

  schema "admission_events" do
    field(:action, :string)
    field(:admission_id, :integer)
    field(:admitter_uuid, Ecto.UUID)
    field(:occurred_at, :naive_datetime)
  end

  def for_admission(query \\ AdmissionEvent, %Admission{} = admission) do
    from(ae in query,
      where: ae.admission_id == ^admission.id,
      order_by: [desc: ae.occurred_at]
    )
  end
end
```

```elixir
# Taking our query function for a spin...

iex> admission = Repo.get(Admission, 1)
iex> AdmissionEvent.for_admission(admission) |> Repo.all()
[
  %Registrar.Tracking.AdmissionEvent{
    __meta__: %Ecto.Schema.Metadata<:loaded, "admission_events">,
    action: "Student Admitted",
    admission_id: 3,
    admitter_uuid: "7edd4d7f-a790-41f9-b4ef-16f1dc3b33ea",
    id: 1,
    occurred_at: ~N[2019-07-29 02:22:18]
  }
]
```

那么，我们要如何去获取全名呢？我们有很多选项可以选择，但在这篇文章中，我们将比较两个选项：一个是大家可能会首先使用的选项（添加关联和预加载数据），另一个是我们希望今后更经常使用的选项（`Ecto.Query#select_merge/3`）。
## 选项 1: 添加关联和预加载数据

如果我们把 User 和 AdmissionEvent 关联起来，那么我们就可以预先加载关联的 User 记录，直接从那里读取全名。

```elixir
defmodule Registrar.Tracking.AdmissionEvent do
  use Ecto.Schema
  import Ecto.Query, only: [from: 2]
  alias Registrar.Tracking.AdmissionEvent
  alias Registrar.User

  schema "admission_events" do
    field(:action, :string)
    field(:admission_id, :integer)
    field(:admitter_uuid, Ecto.UUID)
    field(:occurred_at, :naive_datetime)

    # New association
    belongs_to(:admitter, User, foreign_key: :uuid)
  end
end

defmodule Registrar.User do
  use Ecto.Schema
  alias Registrar.Tracking.AdmissionEvent

  schema "users" do
    field(:uuid, Ecto.UUID)
    field(:full_name, :string)

    # New association
    has_many(:admission_events, AdmissionEvent)
  end
end
```

```elixir
# Trying out our new association...

iex> admission = Repo.get(Admission, 1)
iex> events = AdmissionEvent.for_admission(admission) |> Repo.all() |> Repo.preload(:admitter)
[
  %Registrar.Tracking.AdmissionEvent{
    __meta__: %Ecto.Schema.Metadata<:loaded, "admission_events">,
    action: "Student Admitted",
    admission_id: 3,
    admitter: %Registrar.User{
      __meta__: %Ecto.Schema.Metadata<:loaded, "users">,
      id: 1,
      name: "Albus Dumbledore",
      uuid: "7edd4d7f-a790-41f9-b4ef-16f1dc3b33ea"
    },
    admitter_uuid: "7edd4d7f-a790-41f9-b4ef-16f1dc3b33ea",
    id: 1,
    occurred_at: ~N[2019-07-29 02:22:18]
  }
]
iex> event = List.first(events)
iex> event.admitter.name
"Albus Dumbledore"
```

这种方法可以完成工作，但有点沉重。我们只需要录取者的全名，为什么要检索整个 `User` 结构？你也可以看到这种模式如何导致一个超级混乱的 User schema。现在它有很多 `admission_events` ，但很快它可能有很多`application_events`，`interview_events`，`billing_events` 等等。
## 选项 2: ✨ 查询 合并 ✨

`Ecto.Query#select_merge/3` 给了我们一个更简洁、更精确的选择。看看这个丝滑的例子。

```elixir
defmodule Registrar.Tracking.AdmissionEvent do
  use Ecto.Schema
  import Ecto.Query, only: [from: 2]
  alias Registrar.Admission
  alias Registrar.User
  alias Registrar.Tracking.AdmissionEvent

  schema "admission_events" do
    field(:action, :string)
    field(:admission_id, :integer)
    field(:admitter_uuid, Ecto.UUID)
    field(:occurred_at, :naive_datetime)

    ###### STEP ONE #######
    #  Add Virtual Field  #
    #######################
    field(:admitter_name, :string, virtual: true)
  end

  def for_admission(query \\ AdmissionEvent, %Admission{} = admission) do
    from(ae in query,
      where: ae.admission_id == ^admission.id,
      order_by: [desc: ae.occurred_at],

      #### STEP TWO ####
      #  Join on User  #
      ##################
      join: u in User,
      on: ae.admitter_uuid == u.uuid,

      ############ STEP THREE #############
      #  Select Merge into Virtual Field  #
      #####################################
      select_merge: %{admitter_name: u.full_name}
    )
  end
end
```

```elixir
# Trying out select merge...

iex> admission = Repo.get(Admission, 1)
iex> AdmissionEvent.for_admission(admission) |> Repo.all()
[
  %Registrar.Tracking.AdmissionEvent{
    __meta__: %Ecto.Schema.Metadata<:loaded, "admission_events">,
    action: "Student Admitted",
    admission_id: 3,
    admitter_name: "Albus Dumbledore",
    id: 1,
    occurred_at: ~N[2019-07-29 02:22:18]
  }
]
iex> event = List.first(events)
iex> event.admitter_name
"Albus Dumbledore"
```

通过添加一个虚拟字段并将其填充到 `select_merge` 中，我们最终得到了一个更轻量级的解决方案。我们不需要添加任何新的关联就能获得我们所需要的数据，保持我们的 schema 解耦。另外，如果我们需要为不同类型的事件引入事件日志，我们有一个可遵循的模式，这个模式的扩展性更强。

## 总结

`Ecto.Query#select_merge/3` 允许我们在选择查询中直接填充一个虚拟字段，使我们在设计 shcema 和组成查询时具有各种灵活性。

100% 会再次编写。
## 资源

- [文档](https://hexdocs.pm/ecto/Ecto.Query.html#select_merge/3)
- [源代码](https://github.com/elixir-ecto/ecto/blob/master/lib/ecto/query.ex#L1168-L1209)

---
%{
author: "Yuri Oliveira",
author_link: "https://github.com/yuriploc",
tags: ["ecto"],
date: ~D[2021-11-22],
title: "TIL: Cleaner queries with Ecto `map`",
excerpt: """
Today I learned how to write cleaner `Ecto` select queries with the help of `Ecto.Query.map`.
"""
}

---

# TIL: 使用 Ecto 的 `map` 进行更干净的查询

当处理一张大表时，一个常见的做法是避免使用 `SELECT *` 以更好地利用数据库索引和资源。

因此，不要这样写：

```elixir
query = from p in Post
Repo.all(query)
```

为了避免得到比我们需要的更多的数据，我们可以明确地告诉 Ecto（和 DB）我们希望它返回哪些列：

```elixir
query = from p in Post, select: %{id: p.id, title: p.title, category_id: p.category_id}
Repo.all(query)
```

但为什么我们要如此明确地重复键和值呢？难道没有一个更好的方法吗？

事实证明，Ecto.Query 已经用 `map/2` 函数为我们解决了这个问题。所以像这样：

```elixir
query = from p in Post, select: %{id: p.id, title: p.title, category_id: p.category_id}
Repo.all(query)
```

变成这样:

```elixir
query = from p in Post, select: map(p, [:id, :title, :category_id])
Repo.all(query)
```

或者，在 Pipeline 中:

```elixir
Post
|> select([p], %{id: p.id, title: p.title, category_id: p.category_id})
|> Repo.all()
```

```elixir
Post
|> select([p], map(p, [:id, :title, :category_id]))
|> Repo.all()
```

当在一个函数中使用它时，我们甚至可以有动态字段，比如：

```elixir
def filter_posts_by_id(posts_ids, fields \\ [:id, :title, :category_id]) do
  Post
    |> where([p], p.id in ^posts_ids)
    |> select([p], map(p, ^fields))
    |> Repo.all()
end
```

由于 `Ecto.Query.map/2` 函数和管道的使用，我们最终得到了干净、可组合和高度可读的代码。

享受 Ecto 吧!

_感谢 Groxio 导师们的支持_

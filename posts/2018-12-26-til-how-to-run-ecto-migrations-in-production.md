---
author: Kate Travers
author_link: https://github.com/ktravers
categories: til
tags: ['ecto']
date: 2018-12-16
layout: post
title: TIL How to Run Ecto Migrations on Production
excerpt: >
  What to do when you can't use `mix ecto.migrate`
---

# TIL 如何在生产环境运行 Ecto Migrations

你可能认为这个问题的答案只需要简单的谷歌搜索就能找到。不幸的是，情况并非如此，今天下午我正在处理一个带有新添加的 Ecto 后台的 Phoenix 项目。为了让其他人（说实话，未来的我）免于同样的挫折，这里是我找到的最直接的解决方案。

## 什么是不 work 的

[`Mix`](https://elixirschool.com/lessons/basics/mix/). `Mix` 任务没有被编译到你的已部署的版本中，正如这个[令人激动的讨论](https://github.com/bitwalker/exrm/issues/67) 所证明的那样。没有计划在短时间内改变这种情况。

所以不要尝试在生产环境上使用你信任的本地 `mix ecto.migrate` 任务。在这里它帮不了你。
## 什么可以 Work

### 1. [Ecto.Migrator](https://hexdocs.pm/ecto/Ecto.Migrator.html)

Ecto 提供了 [Ecto.Migrator](https://hexdocs.pm/ecto/Ecto.Migrator.html)，这是 Ecto 迁移 API 的一流模块。通过 ssh 到你的应用服务器上，连接你的Phoenix 应用，然后手动运行以下内容。

```elixir
iex> path = Application.app_dir(:my_app, "priv/repo/migrations")
iex> Ecto.Migrator.run(MyApp.Repo, path, :up, all: true)
```

理想的情况下，你会把上面的内容包在自己的任务中，在构建和部署过程中被调用。请看 Plataformatec 的博客，里面有一个[很好的例子](http://blog.plataformatec.com.br/2016/04/running-migration-in-an-exrm-release/)。

### 2. eDeliver

我们的应用使用 [`edeliver`](https://github.com/edeliver/edeliver) 来进行部署，它有一个超级方便的命令来手动运行迁移。

```shell
mix edeliver migrate production
```

如果我们看一下源码，这个命令其实只是 [为你包装 `Ecto.Migrator`](https://github.com/edeliver/edeliver/blob/963610a90f67fc3671127e64df37a67ec365ef5b/lib/edeliver.ex#L124) 保存一些宝贵的按键。

要想成功运行，你需要在你的 `.delivery/config` 文件中添加 `ECTO_REPOSITORY="MyApp.Repo"`。

同样，Plataformatec 有一篇很好的博客文章 [用 eDeliver 部署你的 Elixir 应用](http://blog.plataformatec.com.br/2016/06/deploying-elixir-applications-with-edeliver/)。

## 结语

嗨，未来的我！ 希望这篇文章对第 n 次仍然有帮助。

### 参考文献:

- [Phoenix Ecto Integration](https://github.com/phoenixframework/phoenix_ecto)
- [Ecto.Migrator](https://hexdocs.pm/ecto_sql/Ecto.Migrator.html)
- [eDeliver](https://github.com/edeliver/edeliver)

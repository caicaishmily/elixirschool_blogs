---
author: Bobby Grayson
categories: general
tags: ['plug']
date: 2019-02-12
layout: post
title: Deploying our `Plug.Router` application to Heroku
excerpt: >
  Want to put your app in the real world? Today we do it with Heroku!
---

# putting a plug app on heroku

在上一篇文章 [用 Plug.Router 构建 Web 应用](https://elixirschool.com/blog/building-apps-with-plug-router/)中，我们只用 `Plut.Router` 构建了一个网站。

今天，我们将探讨如何将之前构建的应用在 Heroku 上启动并运行；这篇文章不会关注 Phoenix 的部署。

在这篇(简短)的文章中，我们将展示如何在 [Heroku](http://heroku.com) 上启动和运行一个 vanilla `Plug` 应用。

这真的很简单，但今天当我开始做的时候，我发现的资源要么集中在 Phoenix 上，要么有一些漏洞，所以我想我会在这里写一个简单的。

在我们开始之前，先去 Heroku 注册一个账号，以及安装他们的 CLI 工具。

## 我们开始吧!

要将我们的应用程序部署到 Heroku，我们需要做一些事情：

* 添加一个 `Procfile` 来告诉 Heroku 我们的服务器进程要做些什么
* 将构建包添加到我们的应用程序中。 这些指导 Heroku 如何构建我们的 Elixir 代码。
* 添加我们的环境变量
* 最后，更新我们的 `application.ex` 代码以使用 Heroku 提供的` PORT` 变量

我们不会在 `Procfile` 详细信息上浪费太多时间，而只是在它里面定义我们的服务器进程将要执行的操作。 在我们的例子中，我们要在 `web` 进程中运行我们的应用程序。 为此，让我们在应用程序根目录中创建并打开 `Procfile`：

```
web: mix run --no-halt
```

我们已经告诉 Heroku 我们希望我们的 `web` 进程执行 `mix run --no-halt` 命令。

就是这样！

现在我们可以运行 `heroku create` 来使其成真。

不过，在准备好让它动起来之前，我们还有一些步骤。

使用 Heroku 的乐趣在于，它们使用出色的内置工具为您处理了很多工作。

这些功能之一是 `buildpacks`，其中包括用于构建（在我们的情况下为编译）我们的应用程序代码的说明。

对于 Elixir，我们需要添加一个专门的 `buildpack`，它知道如何获取 Elixir 依赖项并构建该代码。

```
heroku buildpacks:set https://github.com/HashNuke/heroku-buildpack-elixir
```

现在我们需要设置环境变量

对于我们的应用来说，唯一需要我们担心的就是：`MIX_ENV`

```
heroku config:set MIX_ENV=prod
```

我们还需要给 buildpack 添加一些配置，以便它知道我们的 Elixir/Erlang 环境是什么样的。

我们将其放在名为 `elixir_buildpack.config` 的文件中。

```
# Erlang version
erlang_version=21.1

# Elixir version
elixir_version=1.8
```

现在我们可以在真实世界中使用它了：

```
$ git push heroku master && heroku open
```

该推送成功后，我们就可以看到我们的组合网站在现实世界中上线运行了。

黑客快乐!

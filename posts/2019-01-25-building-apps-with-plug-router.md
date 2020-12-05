---
author: Sean Callan
categories: general
tags: ['plug', 'software design']
date: 2019-01-25
layout: post
title: Building web apps with Plug.Router
---

# 使用 Plug.Router 构建 web 应用

当谈到用 Elixir 构建 Web 应用程序时，许多人都会立即想到 Phoenix。

然而，你知道 `Plug.Router` 也是一种可行的选择吗？

有时，它甚至可以更快。

## 项目

For this project we'll build a simple single page portfolio site.
We can expect our site to load and display our portfolio from a file, database, or somewhere else dynamically. As well as allowing users to submit contact information via a web form.

在这个项目中，我们将建立一个简单的单页投资组合网站。

我们希望我们的网站能从文件、数据库或其他地方动态地加载和显示我们的投资组合。以及允许用户通过网页表单提交联系信息。

__请注意__: 为了保持应用和教程的简洁性，我们不考虑后端数据库的相关内容，只专注于网络部分。

想跳过阅读，只看一段代码？

请到 [elixirschool/router_example](https://github.com/elixirschool/router_example)

## 起步

一开始我们需要做以下的事情：

1. 使用 `mix new --sup` 生成一个新项目
2. 往 `mix.exs` 中添加 `plug_cowboy` 依赖
3. 将我们的路由放入应用程序的监管树中

事不宜迟，让我们开始吧，生成我们的新项目。

```shell
$ mix new router_example --sup

* creating README.md
* creating .formatter.exs
* creating .gitignore
* creating mix.exs
* creating lib
* creating lib/router_example.ex
* creating lib/router_example/application.ex
* creating test
* creating test/test_helper.exs
* creating test/router_example_test.exs

Your Mix project was created successfully.
You can use "mix" to compile it, test it, and more:

    cd router_example
    mix test

Run "mix help" for more commands.
```

让我们进入我们的新目录，在你选择的编辑器中打开 `mix.exs`。

在这里，我们将通过添加 `plug_cowboy` 依赖关系来做一个小小的改变。

```elixir
defp deps do
  [
    {:plug_cowboy, "~> 2.0"}
  ]
end
```

有了这个变化，我们就可以用 `mix deps.get` 来获取我们的依赖关系，然后继续。

虽然我们还没有创建我们的路由，但我们还是先把 supervisor 设置好吧。

接下来我们需要打开 `lib/router_example/application.ex`，这样我们就可以更新我们的 supervisor 的子程序。

`plug_cowboy` 包中包含的 `Plug.Cowboy.child_spec/3` 函数让这一步变得简单。

让我们更新一下我们应用程序的 `start/2` 函数。

```elixir
def start(_type, _args) do
  children = [
    Plug.Cowboy.child_spec(scheme: :http, plug: RouterExample.Router, options: [port: 4001])
  ]

  opts = [strategy: :one_for_one, name: RouterExample.Supervisor]
  Supervisor.start_link(children, opts)
end
```

我们的起步设置完成了！

在运行我们的应用程序之前，我们接下来需要创建我们的 `RouterExample.Router` 模块。

## 存根

正如前言中提到的，为了使教程简短而有针对性，我们将对那些从我们的数据存储中检索数据并将其持久化的函数进行存根处理。

我们不会在这个文件上浪费太多时间，我们只需要两个函数：一个是给我们结果集合（存根数据库查询），第二个是将我们传递的参数写入终端（而不是将其持久化到存储中）。

让我们创建一个新的文件 `lib/router_example/stubs.ex`，并将下面的代码复制到其中。

```elixir
defmodule RouterExample.Stubs do
  def portfolio_entries do
    for x <- 1..10, do: %{name: "Project #{x}", image: "https://picsum.photos/400/300/?random?t=#{x}"}
  end

  def submit_contact(params) do
    IO.inspect(params, label: "Submitted contact")
  end
end
```

_注_: 如果你对 `IO.Inspect/2` 中的 `:label` 选项不熟悉，请查看我们的另一篇博文[TIL IO.Inspect labels](https://elixirschool.com/blog/til-io-inspect-labels/).

## 路由

一行小小的代码，`use Plug.Router`，将 `Plug.Router` 的力量带入我们的应用中，释放出巨大的潜力。

需要复习一下 `use/1` 吗？

请前往 Elixir School 的 [关于 `use` 的部分](https://elixirschool.com/en/lessons/basics/modules/#use)。

那么 `Plug.Router` 到底 _是_ 什么呢？

简单地说，`Plug.Router` 是一个宏的集合，它使请求路径和它们的类型很容易匹配。

如果你熟悉 Ruby 的 [Sinatra](http://sinatrarb.com/)，Python 的 [Flask](http://flask.pocoo.org/)，Java 的[Jersey](https://jersey.github.io/)，或者任何其他的 "微框架"，那么这看起来会很熟悉。

为我们的路由创建一个新文件并且打开它。`lib/router_example/router.ex`。

让我们把下面的基本路由代码复制到我们的新文件中，然后看看各个部分。

```elixir
defmodule RouterExample.Router do
  use Plug.Router

  plug :match
  plug :dispatch

  get "/ping" do
    send_resp(conn, 200, "pong")
  end
end
```

我们首先注意到的是在 `use` 之后的一系列插件：`match` 和 `dispatch`。

这些都是为我们所包含的，分别将传入的请求匹配到一个函数并调用它。

接下来我们看到的是我们的第一个路由!

这是一个简单的健康检查，但它还是很重要的，它向我们展示了我们所有的路由将遵循的格式。HTTP 动词，路径和代码块。

正如人们所想象的那样，不止有 `get/1`，我们还有 `post/1`、`patch/1`、`put/1`、`delete/1`、`option/1` 和 `match/1`。

在接下来的几节中，我们将探索一些其他的宏，但首先让我们看看如何处理渲染 EEx 模板和 JSON。

关于 `Plug.Router` 的更多内容请查看我们的[Plug 课程](https://elixirschool.com/en/lessons/specifics/plug/#plugrouter)中的专门章节。

### 渲染模板

为了我们的目的，我们将专注于 `EEx.eval_file/2`，并将其作为我们自己的 `render/3` 函数的基础。

对于我们的 `render/3` 函数，我们将传递我们的连接结构、一个没有 ".eex" 扩展名的模板，以及我们可能想要提供给嵌入式 Elixir 的任何变量绑定。

我们希望调用的函数看起来像这样。

```elixir
render(conn, "index.html", portfolio: [])
```

现在我们知道了我们想要的东西，是时候创建它了。

`EEx.eval_file/2` 函数获取我们模板的文件路径以及变量绑定，并返回计算出的字符串，即我们的响应体。

由于 EEx 做了所有繁重的工作，我们的 `render/3` 函数需要做的只是建立完整的文件路径，并通过 `send_resp/3` 发送响应。

```elixir
@template_dir "lib/router_example/templates"
...

defp render(%{status: status} = conn, template, assigns \\ []) do
  body =
    @template_dir
    |> Path.join(template)
    |> String.replace_suffix(".html", ".html.eex")
    |> EEx.eval_file(assigns)

  send_resp(conn, (status || 200), body)
end
```

我们已经设置好了让 EEx 在 `lib/router_example/templates` 目录下查找我们的模板，所以我们来创建这个目录。

接下来我们将创建两个模板 `index.html.eex` 和 `contact.html.eex`。

你可以 [在这里](https://raw.githubusercontent.com/elixirschool/router_example/master/lib/router_example/templates/index.html.eex) 找到 `index.html.eex` 的代码和 `contact.html.eex` 的 [代码](https://raw.githubusercontent.com/elixirschool/router_example/master/lib/router_example/templates/contact.html.eex)，我们今天不重点讨论 HTML 和 CSS。

### 发送 & 接收 JSON

在 Plug.Router 中构建 JSON 端点，比我们刚才介绍的模板渲染要省事得多。

首先我们需要一个库来解析和编码 JSON，出于这个目的，我们将使用 `jason`。

```elixir
{:jason, "~> 1.1"}
```

在避免我们忘记之前，现在是运行 `mix deps.get` 的好时机。

一旦我们完成了这些，我们就可以继续更新我们的路由器，通过 `Plug.Parsers` 插件来处理传入的 JSON。

让我们打开 `lib/router_example/router.ex` 并更新我们的插件，以 `Jason` 作为我们的解码器包含 `Plug.Parsers`。

```elixir
plug Plug.Parsers, parsers: [:json],
                   pass: ["text/*"],
                   json_decoder: Jason
plug :match
plug :dispatch
```

这就是我们需要为 JSON 做的所有事情。

如果我们想保持简单，我们可以利用 `Jason.encode/1` 或 `Jason.encode!/1` 以及 `send_resp/3` 就可以了。

```elixir
{:ok, json} = Jason.encode(result)
send_resp(conn, 200, json)
```

或者，如果我们想要更精致一点，我们可以做一个 `render_json/2`。

```elixir
defp render_json(%{status: status} = conn, data) do
  body = Jason.encode!(data)
  send_resp(conn, (status || 200), body)
end
```

在本篇的其余部分，我们将使用 `render_json/2` 方法。

_注_：如果你打算使用类似 [JSON:API 规范](https://jsonapi.org) 的东西，你可能需要一个额外的依赖，比如 [jsonapi](https://github.com/jeregrine/jsonapi) 来帮忙。
### 定义路由

现在我们已经有了路由代码、渲染模板的代码和渲染 JSON 的代码，剩下的就是定义我们的路由了！

我们之前创建了两个 EEx 模板，`index.html.eex`和`contact.html.eex`，所以我们将首先创建路由来处理这些请求。

从我们的 healthcheck 端点，我们知道了我们期望的格式，但是我们可以使用我们新的 `render/3` 函数以及我们的存根数据。

如果你看了一个 `index.html.ex`，那么你就知道我们的 EEx 期望的是捕获一个 `portfolio`，其中包含 `:name` 和 `:image` 的地图列表，我们在我们的存根模块中方便地定义了格式。

让我们在 `lib/router_example/router.ex` 中把所有的部分整合在一起。

```elixir
get "/" do
  render(conn, "index.html", portfolio: Stubs.portfolio_entries())
end

get "/contact" do
  render(conn, "contact.html")
end
```

我们正在实现，但还没有完成。

我们必须处理联系表单的 AJAX 请求。

为了集中精力，我们今天不会在验证方面走弯路，而是利用 `Stubs.submit_contact/1` 函数。

让我们用 `post/1` 宏创建一个新的路由，使用上述函数，并使用我们的 `render_json/2` 函数发送一个愉快的 JSON 消息。

```elixir
post "/contact" do
  Stubs.submit_contact(conn.params)
  render_json(conn, %{message: "Thank you! We will get back to you shortly."})
end
```

就这样，我们完成了，对吗？

好吧--我们可能还想处理对我们还没有定义的路由的请求。

让我们接下来做这个，然后我们就可以称之为完成了。

### 丢失的路由

由于 Elixir 强大的模式匹配功能，处理丢失的路由是很直接的。

通过 `match/3` 宏和 `_`，我们可以对所有请求进行匹配。

通过将其放置在路由器的底部，我们可以确保如果一个请求之前没有被匹配，它将被捕获并处理。

现在，我们将实现一个简单的消息，类似于我们如何用 `send_resp/3` 实现我们的 "/ping" 端点。


```elixir
defmodule RouterExample.Router do
  use Plug.Router

  ...

  match _ do
    send_resp(conn, 404, "Oh no! What you seek cannot be found.")
  end
end
```

哒哒哒哒！

我们的应用已经完成了，该收尾了。

### 结束语

在这段很小的代码行中，我们已经走了很长的路，让我们看看我们的应用程序的全部内容。

```elixir
defmodule RouterExample.Router do
  use Plug.Router

  alias RouterExample.Stubs

  @template_dir "lib/router_example/templates"

  plug Plug.Parsers, parsers: [:urlencoded, :json],
                   pass: ["text/*"],
                   json_decoder: Jason
  plug :match
  plug :dispatch

  get "/ping" do
    send_resp(conn, 200, "pong")
  end

  get "/" do
    render(conn, "index.html", portfolio: Stubs.portfolio_entries())
  end

  get "/contact" do
    render(conn, "contact.html")
  end

  post "/contact" do
    Stubs.submit_contact(conn.params)
    render_json(conn, %{message: "Thank you! We will get back to you shortly."})
  end

  match _ do
    send_resp(conn, 404, "Oh no! What you seek cannot be found.")
  end

  defp render(%{status: status} = conn, template, assigns \\ []) do
    body =
      @template_dir
      |> Path.join(template)
      |> String.replace_suffix(".html", ".html.eex")
      |> EEx.eval_file(assigns)

    send_resp(conn, (status || 200), body)
  end

  defp render_json(%{status: status} = conn, data) do
    body = Jason.encode!(data)
    send_resp(conn, (status || 200), body)
  end
end
```

我们渲染 EEx 模板，接收和发送 JSON，_整个_ app (`RouterExample.Router` + `RouterExample.Stubs`) 只有 58 行代码！

剩下要做的就是运行它，享受我们的新网站。

我们将使用 `mix run --no-halt` 运行我们的应用，应用可以在 [localhost:4001](localhost:4001) 找到。

毫无疑问，这是一个非常基本的实现，但它让我们开始了。

有了这些简单的部分，我们就有了我们所需要的东西，可以建立一些重要的东西。

我们希望你能喜欢它

在未来的文章中，我们将探索一些改进，比如将路由组合成模块，支持 Webpack，其他重构等。

我们应用的代码可以在 [elixirschool/router_example](https://github.com/elixirschool/router_example) 找到。
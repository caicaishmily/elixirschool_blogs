---
author: Sean Callan
author_link: https://github.com/doomspork
categories: general
tags: ['admissions']
date: 2019-10-23
layout: post
title: Building Elixir School's Admissions portal
excerpt: >
  Follow along as we use build Elixir School's Slack invite portal: Admissions.
---

# 构建 Elixir School 的招生门户网站

如果你不知道，Elixir School 有自己的 Slack，贡献者可以聚集在这里讨论我们组织的内容和项目，但最重要的是，在我们的 Elixir 旅程中相互支持。当我们开始创建我们自己的 Slack 时，我们想解决许多公共 Slack 的一个大问题：信噪比不好，垃圾信息太多。

> 你是否为 Elixir School 的项目做出过贡献，但没有加入我们的 Slack？现在就去 https://admissions.elixirschool.com 获得邀请吧！

那么，我们如何才能既保持 Slack 的公开性，又能防止垃圾邮件发送者加入，并且不给我们的维护者增加工作？我们的解决方案是：要求任何一方对我们的任何一个项目至少有一次贡献。

要做到这一点，需要一个使用 GitHub 来验证用户资格的应用程序。这个应用后来被称为：资格认证

> 想跳过前面看最终产品吗？在这里可以找到代码 https://github.com/elixirschool/admissions。

在这篇文章中，我们将探讨 Admissions 是如何工作的，以及我们如何使用 Elixir 和 Phoenix 实现我们的目标。首先，让我们看看预期的流程，然后从那里开始工作。

![image](https://user-images.githubusercontent.com/73386/67163591-ff12a280-f32d-11e9-83f9-f033345f559b.png)

除了告诉我们应用程序应该如何运作，这个图还将流程分解成方便的开发任务。根据这个图，我们来探讨一下为了实现我们的高层需求，我们需要的各个子任务。

1. 允许用户使用 GitHub 登录，并获取其访问令牌。我们可以利用 [Ueberauth](https://github.com/ueberauth/ueberauth) 和它的 [GitHub 策略](https://github.com/ueberauth/ueberauth_github) 来为我们完成繁重的工作。
2. 有了用户的访问令牌后，使用 GitHub API 查看用户是否对组织的项目做出了贡献。为了避免花时间编写自己的 GitHub API 客户端，我们要利用[Tentacat](https://github.com/edgurgel/tentacat)。
3. 利用 API 搜索的结果，对用户的结果进行处理。
   1. 在用户 **是** 贡献者的情况下，让他们确认要使用 Slack 的电子邮件地址，使用 Slack API 发送邀请，最后祝贺他们。
   2. 如果他们 **没有** 贡献，我们需要通知他们不符合条件

### 使用 GitHub 登录

我们将从一个新的 Phoenix 项目(`mix phx.new admissions`)开始，研究如何支持 GitHub 登录。为此我们需要一个新的依赖：`ueberauth_github`。

```elixir
  defp deps do
    [
      {:gettext, "~> 0.11"},
      {:phoenix, "~> 1.4.0"},
      {:phoenix_html, "~> 2.11"},
      {:plug_cowboy, "~> 2.0"},
      {:ueberauth_github, "~> 0.7.0"},

      {:phoenix_live_reload, "~> 1.2", only: :dev}
    ]
  end
```

> 我们不需要包含 `ueberauth` 本身，作为 `ueberauth_github` 的依赖，它已经为我们包含了。

> 有用的提示：你知道你可以使用 `mix hex.info <package name>` 来获取最新版本吗？试试吧!

有了新的依赖关系，我们的应用还剩下什么呢？有很多事情要做 为了完成我们与 Ueberauth 的整合，我们有几个子任务。

1. 创建一个 `AuthController`，处理 OAuth 请求的回调阶段。

2. 在 `router.ex` 文件中包含我们新的控制器和路由。

3. 把 Ueberauth 所需的配置放在我们的 `config/config.exs` 文件中。

4. 在 UI 中添加一个登录的按钮。虽然我们不会在本文中花时间来构建 UI，但我们会触及到必要的部分。

5. 在 GitHub 上设置你的应用程序。这里你还需要检索你的 `CLIENT_ID` 和 `CLIENT_SECRET`。

   > GitHub 的设置和配置超出了本文的范围。如果你不太清楚该怎么做，可以去 GitHub 的开发者文章 [Authorizing OAuth Apps](https://developer.github.com/apps/building-oauth-apps/authorizing-oauth-apps/)。

继续前进！

#### 我们新的控制器

完成第一个子任务需要我们为 Ueberauth 创建一个新的控制器，它将在成功登录时处理 GitHub 的 OAuth 回调。控制器唯一的硬性要求是我们必须包含 Ueberauth 插件。

```elixir
defmodule AdmissionsWeb.AuthController do
  use AdmissionsWeb, :controller

  plug Ueberauth
end
```

有了这个插件，我们将定义一个函数来处理我们的请求。我们选择将该函数命名为 `callback/2`。这个函数需要检索 Ueberauth 为我们方便地放入 `Plug.Conn` 分配的用户详细信息。我们关注的字段是用户的邮箱和 GitHub 昵称。

```elixir
defmodule AdmissionsWeb.AuthController do
  use AdmissionsWeb, :controller

  plug Ueberauth

  def callback(%{assigns: %{ueberauth_auth: ueberauth_auth}} = conn, _params) do
    %{info: %{email: email, nickname: nickname}} = ueberauth_auth
  end
end
```

_在这个例子中_，我们不需要担心匹配错误，因为所有成功的登录都会包含上述字段。

现在我们已经得到了我们需要的东西，我们需要将用户转发到流程的下一步：确定资格。为了确保我们已经得到了下一步需要的东西，我们选择将我们的 GitHub 数据放入会话中，然后将用户重定向到资格检查。

```elixir
defmodule AdmissionsWeb.AuthController do
  use AdmissionsWeb, :controller

  plug Ueberauth

  def callback(%{assigns: %{ueberauth_auth: ueberauth_auth}} = conn, _params) do
    %{info: %{email: email, nickname: nickname}} = ueberauth_auth

    conn
    |> put_session(:github, %{email: email, nickname: nickname, token: token})
    |> redirect(to: Routes.registrar_path(conn, :eligibility))
  end
end
```

有了这些，我们就完成了控制器的工作，可以继续下一个子任务，更新我们的 `router.ex`。我们将很快实现我们的 `eligibility` 请求处理程序。
#### 更新 Phoenix 的路由

更新 Ueberauth 的路由是一个相当简单和直接的变化。在我们的 `router.ex` 底部，我们添加了以下代码块。

```elixir
scope "/auth", AdmissionsWeb do
  pipe_through :browser

  get "/github", AuthController, :request
  get "/github/callback", AuthController, :callback
end
```

我们添加了 2 条路由，但只有 1 个请求处理程序，`callback/2` 在我们的控制器中。还记得我们控制器中的 `plug Ueberauth` 吗？我们的好朋友 Ueberauth 负责 OAuth 交换的请求阶段，省去了我们的麻烦。

在这个阶段，我们几乎完成了我们的集成。现在我们可以继续为我们的应用配置 Ueberauth 了。

#### Ueberauth 配置

Ueberauth GitHub 策略的文档为我们提供了我们所需要的一切。由于我们需要用户的电子邮件和配置文件的访问权限，我们必须根据 GitHub 的文档将我们的作用域更新为 `user:email,user:profile`。

我们的 `config.exs` 的修改如下：

```elixir
config :ueberauth, Ueberauth,
  providers: [
    github: {Ueberauth.Strategy.Github, [default_scope: "user:email,user:profile", send_redirect_uri: false]}
  ]

config :ueberauth, Ueberauth.Strategy.Github.OAuth,
  client_id: System.get_env("GITHUB_CLIENT_ID"),
  client_secret: System.get_env("GITHUB_CLIENT_SECRET")
```

通过 `System.get_env/1`，我们除了支持在运行时对这些值进行修改外，还避免了在源码控制中检查秘钥值。我们在后面的步骤中使用从 GitHub 应用设置中获取的值来填充 `GITHUB_CLIENT_ID` 和 `GITHUB_CLIENT_SECRET` 系统 ENV。

> 对编译和运行时的配置感到困惑？查看我们的博客文章[配置解密](https://elixirschool.com/blog/configuration-demystified/)以了解更多。

一个可选但强烈鼓励的配置是使用较新的 JSON 库 [Jason](https://github.com/michalmuskala/jason) 更新 `oauth2` 序列化器。

```elixir
config :oauth2,
  serializers: %{
    "application/json" => Jason
  }
```

为了做到这一点，我们在 `mix.exs` 中添加了 `jason`，就像之前使用 `ueberauth_github` 一样。

#### 登录按钮

为了启动 GitHub 登录的认证流程，我们需要用户点击我们定义的早期请求路径的链接。为此，我们在 `index.html.eex` 文件中添加了以下 HTML。

```html
<a class="button is-info is-medium" href="/auth/github">
  <span class="icon">
    <i class="fab fa-github"></i>
  </span>
  <span>Sign-in with GitHub</span>
</a>
```

现在我们的 UI 已经更新了，我们可以称我们集成 Ueberauth 代码完成了! 我们的最后一步是在 GitHub 上设置应用程序。完成后，我们从应用设置中调出 `CLIENT_ID` 和 `CLIENT_SECRET` ，并将它们添加到我们的 ENV 中。

现在用户可以用有效的 GitHub 账号登录了。我们需要处理下一步流程：资格认证。

### 验证贡献者状态

在请求的这个阶段，我们的用户已经成功地通过了 GitHub 的认证，现在我们需要确定他们是否对我们的仓库做出了贡献。为了达到这个目的，我们需要利用 GitHub 的 API。对于这部分应用，我们要做的高阶工作是这样的。

![image](https://user-images.githubusercontent.com/73386/67163600-19e51700-f32e-11e9-89c3-a14b6a9bde8d.png)

为了不重新造个轮子，我们选择了 [Tentacat](https://github.com/edgurgel/tentacat) 库。在这一点上，我们的 `mix.exs` 依赖关系是这样的：

```elixir
  defp deps do
    [
      {:gettext, "~> 0.11"},
      {:jason, "~> 1.0"},
      {:phoenix, "~> 1.4.0"},
      {:phoenix_html, "~> 2.11"},
      {:plug_cowboy, "~> 2.0"},
      {:tentacat, "~> 1.5"},
      {:ueberauth_github, "~> 0.7.0"},

      {:phoenix_live_reload, "~> 1.2", only: :dev}
    ]
  end
```

有了新的依赖关系，我们就可以获取 (`mix deps.get`)，然后开始我们的工作。保持我们的控制器的简单和专注于展示是我们一直以来的目标，所以我们决定在应用程序的 web 部分之外的一个单独的模块中实现资格代码。

我们把这个新模块称为 `Registrar`，以保持我们的学院主题，它可以在 `lib/admissions/registrar.ex` 文件中找到。

>  ### reg·is·trar
>
> 1. 在学院或大学中负责保管学生档案的官员

考虑到上面的流程，我们决定实现这一目标的最佳方式是检查组织中的仓库列表（支持多个组织），以查找与我们用户的 GitHub 昵称相匹配的贡献者。为此，我们知道我们需要存储组织的名称和它的仓库。为此，我们选择了一个以组织名称为键，以仓库列表为值的 map。为了避免任何类型转换，我们选择将所有的东西都存储为字符串，最终的结果被添加到我们的 `config.exs` 中。

```elixir
config :admissions, repositories: %{
  "elixirschool" => ["elixirschool", "admissions", "extracurricular", "homework"]
}
```

> 为了支持未来的一些计划，我们支持选择多个组织。这也允许其他组织和公司利用 Admissions 。

当实现实际的检查时，我们发现把事情分解成几个函数是最好的，以保持代码的干净和可读性。我们最终在新的 `lib/admissions/registrar.ex` 文件中使用了 4 个函数。

1. 我们唯一的公共函数 `eligibile?/1` 采用了一个昵称。
2. 一个私有函数 `org_contributor?/3`，它需要我们将创建的 GitHub API 客户端的 token，用户的昵称，最后是我们 `config :admissions, repositories` 映射中的键值对。
3. 一个检查每个仓库的贡献者的函数，为我们的用户。`contributor?/4`。我们需要 GitHub API 客户端、昵称、组织和一个仓库。
4. 最后，是一个从上面检索配置的函数。`organizations/0`。我们在加载配置值的时候，比较喜欢用函数来代替模块属性。

为了把它解决掉，处理了一个最简单的函数：`organization/0`，在这里我们只需要获取我们的配置即可。

```elixir
def organizations, do: Application.get_env(:admissions, :repositories)
```

有了我们的配置，我们就可以对组织进行迭代，并查找贡献者状态。为此，我们需要创建一个 Tentacat GitHub API 客户端。让我们来看看我们最终在 `eligible?/2` 函数中得到了什么。

```elixir
def eligible?(nickname) do
  client = Client.new()
  Enum.any?(organizations(), &org_contributor?(client, nickname, &1))
end
```

在这里，我们创建一个 `Tentacat.Client`，并使用 `Enum.any?/2` 对配置好的组织进行迭代。我们不太关心复杂的匿名函数，所以我们选择创建 `org_contributor/2`。这个函数非常简单。从我们的配置中抽取一个组织，然后遍历仓库，寻找匹配的组织。

```elixir
defp org_contributor?(client, nickname, {org, repos}) do
  Enum.any?(repos, &contributor?(client, nickname, org, &1))
end
```

最后但并非最不重要的是我们的 `contributor?/4` 函数，它做的是真正的工作。 我们必须检索一个仓库的贡献者列表，并验证我们的昵称是否在列表中。多亏了 Tentacat，使用 `Tentacat.Repositories.Contributors` 模块和 `list/3` 函数，这很容易，它返回一个元组，包括我们的贡献者列表，其他值我们可以忽略。

```elixir
defp contributor?(client, nickname, org, repo) do
  case Contributors.list(client, org, repo) do
    {_status, contributors, _response} ->
      Enum.any?(contributors, &(Map.get(&1, "login") == nickname))
    _ ->
      false
  end
end
```

贡献者列表是一个 map 的集合，包含了 GitHub 用户的所有信息，但我们最感兴趣的是 "login" 键，即用户的昵称。

现在我们终于可以回答这个问题了。他们是贡献者吗？

### 处理用户请求

现在我们知道了一个用户是否是贡献者，我们需要做一些事情。如果他们 **不是** 贡献者，他们就不能继续，我们应该告诉他们。然而，如果他们 **是** 贡献者，那么我们需要验证他们的电子邮件地址，以便我们可以通过 Slack API 向他们发送邀请。可视化，我们的流程看起来像这样。

![image](https://user-images.githubusercontent.com/73386/67163615-35502200-f32e-11e9-8204-294d73790e0c.png)

#### 处理资格

使用我们新的 `Registrar.qualified?/1` 函数，我们将实现我们前面简单讨论过的 `RegistrarController` 的 `eligibility/2` 路由处理程序。在我们的流程中，这将是我们的用户根据他们的贡献者身份而产生路径分歧的地方。我们得出的结论是，最简单的方法是根据问题的答案来决定视图模板，符合条件的用户看到的是包括电子邮件地址验证步骤的 `eligible.html`，而其他的则是 `ineligible.html`。

为了实现我们的目标，我们从我们的会话中检索用户信息，调用到我们新的 `eligible?/1` 函数，决定我们的模板，最后调用 `render/3` 与我们的连接，模板，以及用户的电子邮件和 GitHub 用户名。

```elixir
def eligibility(conn, _params) do
  %{email: email, nickname: nickname} = get_session(conn, :github)

  template = if Registrar.eligible?(nickname), do: "eligible.html", else: "ineligible.html"

  render(conn, template, %{email: email, nickname: nickname})
end
```

有了我们的新功能，我们将新的 `/eligibility` 路由添加到 `router.ex` 文件中，这次添加了 `:auth` 管道，以限制只对经过认证的用户进行访问。当我们在 router 文件中时，我们可以添加下一个我们需要的路由，一个用于提交电子邮件地址的 `POST`。

```elixir
scope "/", AdmissionsWeb do
  pipe_through [:browser, :auth]

  get "/eligibility", RegistrarController, :eligibility
  post "/register", RegistrarController, :register
end
```

在这一点上，非贡献者已经被处理，我们鼓励他们寻找机会贡献，以后再尝试。我们的贡献者还剩下最后一步：验证他们希望被邀请的电子邮件地址。

#### Slack 邀请

我们已经到了最后一步：邀请贡献者加入 Slack！要做到这一点，需要使用 Slack 官方的 API 和他们提供的 `users.admin.invite` 函数。这个请求必须是一个表单 POST，其中包含我们在上一步收集到的用户的电子邮件和我们组织的 Slack 令牌，你也可以包含一些可选的 Slack 设置。

> 你可以在官方文档中找到更多关于 Slack API 的信息：https://api.slack.com/。

一旦我们处理了我们的响应，我们就有了一个工作的 API 客户端。

```elixir
defmodule Admissions.Slack do
  @invite_url "https://elixirschool.slack.com/api/users.admin.invite"

  def invite(email) do
    email
    |> slack_invite()
    |> slack_response()
  end

  defp slack_invite(email) do
    data = [email: email, set_active: true, token: slack_token()]
    HTTPoison.post(@invite_url, {:form, data})
  end

  defp slack_response({:ok, %{body: body}}) do
    case Jason.decode(body) do
      {:ok, %{"ok" => true}} -> :ok
      {:ok, %{"error" => reason}} -> {:error, reason}
    end
  end

  defp slack_response({:error, _reason}) do
    {:error, "unexpected_error"}
  end

  defp slack_token, do: System.get_env("SLACK_TOKEN")
end
```

有了一个 API 客户端，剩下的就是实现 `/register` 路由处理程序。为此，我们概述了对新函数的期望，并开始着手构建它。

1. 我们知道一个请求主体有 `"email"` 键，模式匹配被用来获取我们关心的值：他们的 email 地址。
2. 我们新的 Slack API 客户端用来触发和邀请
3. 我们处理结果
   1. 成功后，我们向他们展示一个欢迎页面
   2. 失败时，我们会给他们看一条错误信息。Slack 文档中概述了一些错误代码，我们将匹配上并翻译成人类可读的信息。`already_in_team`, `already_invited`, `invalid_email`, 最后是我们在客户端返回的 `unexpected_error`。

当我们决定了工作后，更新 `RegistrarController` 就很直接了当了。

```elixir
def register(conn, %{"email" => email}) do
  case Slack.invite(email) do
    :ok ->
      render(conn, "welcome.html")
    {:error, reason} ->
      message = translated_message(reason)
      render(conn, "error.html", message: message)
  end
end

defp translated_message("already_in_team"), do: "Already in team"
defp translated_message("already_invited"), do: "Already invited"
defp translated_message("invalid_email"), do: "Invalid email address"
defp translated_message("unexpected_error"), do: "Unexpected error"
```

我们已经为这个功能添加了一条路由，所以我们已经完成了。就像完成了一样。我们有一个正常运行的应用程序，需要用 GitHub 登录，确认他们的贡献者状态，并在适当的时候邀请他们到 Slack。由于组织是可配置的，所以没有人可以阻止其他组织使用Admissions，真是太酷了。

你是否已经为 Elixir School 项目做出了贡献，但还没有加入 Slack？前往 [http://admissions.elixirschool.com](http://admissions.elixirschool.com/) 检查你的资格吧!

有兴趣看看代码的完整版吗？正在寻找解锁 Slack 访问的贡献机会？你可以在 GitHub 上找到这个项目：https://github.com/elixirschool/admissions。
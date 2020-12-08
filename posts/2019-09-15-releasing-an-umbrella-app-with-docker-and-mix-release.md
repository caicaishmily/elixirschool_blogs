---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
tags: ['docker', 'mix release', 'config', 'umbrella apps']
date: 2019-09-20
layout: post
title: Releasing an Umbrella App with Docker, Mix Release and Config
excerpt: >
  The release of Elixir 1.9 gave us `mix release` and the ability to support basic releases, runtime configuration and more, natively in Elixir. Learn how we were able to build a production release of an Elixir umbrella app with Docker, `mix release` and the new `Config` module.
---

# 使用 Docker, Mix Release, Config 发布一个 Umbrella App

今年早些时候发布的 Elixir 1.9 引入了一些 [强大的新工具](http://blog.plataformatec.com.br/2019/04/whats-new-in-elixir-apr-19/)。`mix release` 允许我们在没有 Distillery 的情况下构建一个版本；我们的伞形子应用的配置已经被移到了父应用中；增加的 `Config` 模块取代了 `Mix.Config` ，使我们的版本配置变得简单，并且通过增加 `System.fetch_env!` 等功能进一步简化了配置。

让我们利用 _这些_ 新特性，借助 Docker 来构建一个 Elixir umbrella 应用。

## 背景: 我们的构建 + 部署流程

首先，介绍一下相关应用的构建和部署过程的背景。在 The Flatiron School，我们维护了一个应用程序，即 Registrar，处理我们的学生入学和计费。Registrar 应用是一个 Elixir 伞形应用，使用 CircleCI 和 AWS Fargate 管理的 CI/CD 管道进行构建和部署。Registrar 由 CircleCI 构建，生成的镜像会被推送到 ECR（Elastic Container Repository）。Fargate 将镜像拉下来，并在 ECS 中运行容器化发布。

如果这样的设置让你感到困惑或陌生--没有问题！你唯一需要了解的是，在 ECS 中，你可以使用 Fargate 来发布镜像。在这篇博文中，你唯一需要理解的是，当我们构建发布版本时，我们的应用程序的环境变量是不可用的，但它们在运行时是可用的。

## 初始化 Release

在我们开始前，我们需要在我们的伞形应用根目录运行 `mix release.init`。 这将生成下面的文件：

* `rel/env.sh`
* `rel/env.bat`
* `rel/vm.args`

稍后将详细介绍这些文件。

## 使用 `Config` 模块配置伞形应用程序。

首先，我们需要做的是确保我们 Elixir umbrella app 的子程序能够通过新的 `Config` 模块进行正确的配置。

以前我们的伞形应用的帮助每个子应用的配置在该子应用的 `config/` 子目录中，现在我们直接在父应用中配置每个子应用。所以，config 目录顶层应用 `registrar_umbrella`，是所有操作发生的地方。

我们先来看看 `registrar_umbrella/config/config.exe` 文件。

如果我们有一个伞形应用 `registrar_umbrella`，有两个子应用 `registrar` 和 `registrar_web`，我们的 `config.exs` 文件可能看起来像这样。

```elixir
# registrar_umbrella/config/config.exs

import Config

config :registrar,
  stripe_api_base_url: System.get_env("STRIPE_BASE_URL"),
  stripe_api_key: System.get_env("STRIPE_SECRET_KEY"),
  accounts: Registrar.Accounts,
  billing: Registrar.Billing

config :registrar_web,
  learn_base_url: System.get_env("LEARN_OAUTH_BASE_URL"),
  learn_client_id: System.get_env("LEARN_OAUTH_CLIENT_ID"),
  learn_client_secret: System.get_env("LEARN_OAUTH_CLIENT_SECRET"),
  learn_client: RegistrarWeb.OAuth.LearnClient

# Configures the endpoint
config :registrar_web, RegistrarWeb.Endpoint,
  server: true,
  url: [host: "localhost"],
  secret_key_base: System.get_env("SECRET_KEY_BASE"),
  render_errors: [view: RegistrarWeb.ErrorView, accepts: ~w(html json)],
  pubsub: [name: RegistrarWeb.PubSub, adapter: Phoenix.PubSub.PG2]

...

import_config "#{Mix.env}.exs"
```

让我们来分析一下。

### `Config` 模块

请注意，我们在文件顶部加入了 `import Config`。Elixir 1.9 软取消了 `use Mix.Config` 的用法，原因如下。发布版有自己的配置，包括由 `config/releases.exe` 文件确定的运行时配置（稍后再谈）。然而，Mix 是一个构建工具。因此，它在你的版本中是不可用的。因此，我们不想依赖它，而是可以使用（新的！）原生 Elixir `Config` 模块来满足我们所有的配置需求。

### 特定环境的配置

我们可以继续在 `config/dev.exs`、`config/test.exs` 和 `config/prod.exs` 中设置特定环境的配置。`import_config "#{Mix.env}.exs"` 这一行将在编译时导入相应的配置文件。

### 使用 `System.get_env/1`

在我们的 `config.exs` 文件中，我们使用 `System.get_env/1`。这将返回给定环境变量的值，_如果它在编译时存在于系统中_。否则将返回 `nil`。在开发和测试环境中使用 `System.get_env/1` 可以很好地工作，但在生产环境中却无法实现。这是因为，对于我们特定的应用程序的构建和部署管道，我们是在一个环境中构建发行版，而这个环境的系统并不包含我们的应用程序所需要的环境变量，例如 `"STRIPE_SECRET_KEY"`。然而，我们的生产版本的 _运行时_ 环境会有这些变量。

现在我们已经看到了如何在 `Config` 模块和 `System.get_env/1` 的帮助下配置保护伞的子应用程序，让我们来看看我们的发布配置。

## 配置 Release

### 在 `config/mix.exs` 中定义 Release

我们先在顶层的 `mix.exs` 文件中，在 `project/0` 函数中的 `:reases` 键下配置我们的 release。

```elixir
# registrar_umbrella/mix.exs
defmodule Registrar.Umbrella.Mixfile do
  use Mix.Project

  def project do
    [
      apps_path: "apps",
      start_permanent: Mix.env() == :prod,
      deps: deps(),
      version: "0.1.0",
      elixir: "~> 1.9",
      releases: [
        registrar_umbrella: [
          applications: [
            registrar: :permanent,
            registrar_web: :permanent
          ]
        ]
      ]
    ]
  end
  ...
end
```

我们可以通过在 `:release` 下添加后续键来定义多个 release --例如，如果我们想创建一个只运行 `registrar` 应用程序的 release。现在，我们只定义了一个版本，`registrar_umbrella`。对于 umbrella 应用程序的 release 配置，我们必须指定当 release 启动时要启动哪些子应用程序。我们通过在 `:applications` 键下列出我们想要启动的子应用程序来实现。

有许多额外的发布配置选项，你可以查看[这里](https://hexdocs.pm/mix/Mix.Tasks.Release.html#module-customization)，但我们现在将保持我们的配置非常简单。
### 在 `config/releases.exs` 中配置运行时

由于我们的构建和部署管道要求我们的应用程序的环境变量在 _运行时_ 而不是构建时存在，我们需要 release 拥有运行时配置。为了使我们的发布版本具有运行时配置，我们创建一个文件 `config/releases.xs`。

```elixir
# registrar_umbrella/config/config.exs

import Config

config :registrar,
  stripe_api_base_url: System.fetch_env!("STRIPE_BASE_URL"),
  stripe_api_key: System.fetch_env!("STRIPE_SECRET_KEY")

config :registrar_web,
  learn_base_url: System.fetch_env!("LEARN_OAUTH_BASE_URL"),
  learn_client_id: System.fetch_env!("LEARN_OAUTH_CLIENT_ID"),
  learn_client_secret: System.fetch_env!("LEARN_OAUTH_CLIENT_SECRET")

# Configures the endpoint
config :registrar_web, RegistrarWeb.Endpoint,
  secret_key_base: System.fetch_env!("SECRET_KEY_BASE")
```

这里我们借助 `System.fetch_env!/1` 来配置所有的运行时应用环境变量。如果给定的环境变量在运行时不存在于系统中，这个函数将引发一个错误。我们希望这样的验证能够到位，这样我们的应用程序在缺少必要的环境变量时就无法启动--下游不会出现无声的失败。

重要的是，我们仍然利用 `config/prod.exs` 文件（这里不包括）来做一些事情，比如为生产配置 `ReigstrarWeb.Endpoint`。这个文件是专门为我们的运行时发布配置的。

在我们继续之前，还有最后一件事要指出。

比方说，我们在运行时发布的版本中设置了以下应用环境变量。

```elixir
# registrar_umbrella/config/config.exs

import Config

config :registrar,
  stripe_api_base_url: System.fetch_env!("STRIPE_BASE_URL")
```

我们有一个模块，`Registrar.StripeApiClient`，它使用模块属性来查找和存储该应用环境变量的值。

```elixir
# registrar_umbrella/apps/registrar/lib/stripe_api_client.ex

defmodule Registrar.StripeApiClient do
  @stripe_api_base_url Application.get_env(:registrar, :stripe_api_base_url)

  def get(url) do
    HTTPoison.get(@stripe_api_base_url <> url)
  end
end
```

虽然开发人员经常使用用户定义的模块属性作为常量，但重要的是要记住，_该值是在编译时读取的，而不是在运行时读取的_ 由于 `Application.get_env(:registrar, :stripe_api_base_url)` (来自系统环境变量)的值只存在于 _运行时_，所以在这里使用模块属性是行不通的！相反，我们将使用一个函数在运行时动态地查找该值。

取而代之的是，我们将使用一个函数在运行时动态地查找该值。

```elixir
# registrar_umbrella/apps/registrar/lib/stripe_api_client.ex

defmodule Registrar.StripeApiClient do
  defp stripe_api_base_url, do: Application.get_env(:registrar, :stripe_api_base_url)

  def get(url) do
    HTTPoison.get(stripe_api_base_url() <> url)
  end
end
```

现在我们已经完成了运行时的配置，我们已经准备好构建我们的发行版了。

## 用 Docker + `mix release` 构建版本

我们使用 Docker 来构建我们的版本，因为我们的应用程序将在 ECS 集群内的容器中运行。

我们的 Docker 文件是非常简单直接的。

```dockerfile
FROM bitwalker/alpine-elixir-phoenix:1.9.0 as releaser

WORKDIR /app

# Install Hex + Rebar
RUN mix do local.hex --force, local.rebar --force

COPY config/ /app/config/
COPY mix.exs /app/
COPY mix.* /app/

COPY apps/registrar/mix.exs /app/apps/registrar/
COPY apps/registrar_web/mix.exs /app/apps/registrar_web/

ENV MIX_ENV=prod
RUN mix do deps.get --only $MIX_ENV, deps.compile

COPY . /app/


WORKDIR /app/apps/registrar_web
RUN MIX_ENV=prod mix compile
RUN npm install --prefix ./assets
RUN npm run deploy --prefix ./assets
RUN mix phx.digest

WORKDIR /app
RUN MIX_ENV=prod mix release

########################################################################

FROM bitwalker/alpine-elixir-phoenix:1.9.0

EXPOSE 4000
ENV PORT=4000 \
    MIX_ENV=prod \
    SHELL=/bin/bash

WORKDIR /app
COPY --from=releaser app/_build/prod/rel/registrar_umbrella .
COPY --from=releaser app/bin/ ./bin

CMD ["./bin/start"]
```

让我们仔细看看我们真正关心的部分。

首先，我们将 `MIX_ENV` 改为 `prod`，并获取和编译我们的生产依赖。

```dockerfile
ENV MIX_ENV=prod
RUN mix do deps.get --only $MIX_ENV, deps.compile
```

稍后，我们为子应用 `registrar_web` 构建了生产环境的静态资源

```dockerfile
WORKDIR /app/apps/registrar_web
RUN MIX_ENV=prod mix compile
RUN npm install --prefix ./assets
RUN npm run deploy --prefix ./assets
RUN mix phx.digest
```

然后我们使用 `mix release` 根据 `mix.exs` 文件中 `project/0` 函数的 `:release` 键中的配置来构建我们的 release。

```dockerfile
WORKDIR /app
RUN MIX_ENV=prod mix release
```

这将构建我们的发布版本，并将其放在 `_build/prod/rel/registrar_umbrella` 中。

最后，我们将 release 复制到容器中，并指定启动脚本在 `./bin/start` 中。

现在我们来谈谈启动脚本。

## 启动脚本

启动我们的发布版本很简单，我们的 `./bin/start` 脚本看起来像这样：

```bash
#!/usr/bin/env bash

set -e

echo "Starting app..."
bin/registrar_umbrella start
```

在这一点上，你可能还记得 Distillery 提供了一个 "引导钩" 功能，允许你在应用程序启动时运行某些命令/执行一些代码。你 _可能_ 会想知道我们如何使用 `mix release` 来实现同样的目标。例如，我们如何确保我们的迁移在发布启动时运行？请继续阅读，了解更多

## 用 `rel/env.sh` 预启动脚本。

`mix release.init` 生成的 `rel/env.sh` 文件将在发布开始时运行。这就是我们将调用迁移脚本的地方。

假设我们有一个模块 `Registrar.ReleaseTasks` 和一个函数 `migrate/0`，它启动应用程序并执行 Ecto 迁移。

```elixir
defmodule Registrar.ReleaseTasks do
  @moduledoc false

  @start_apps [
    :crypto,
    :ssl,
    :postgrex,
    :ecto,
    :ecto_sql
  ]

  @repos Application.get_env(:registrar, :ecto_repos, [])

  def migrate do
    start_services()
    run_migrations()
    stop_services()
  end

  defp start_services do
    IO.puts("Starting dependencies..")
    # Start apps necessary for executing migrations
    Enum.each(@start_apps, &Application.ensure_all_started/1)

    # Start the Repo(s) for app
    IO.puts("Starting repos..")

    # Switch pool_size to 2 for ecto > 3.0
    Enum.each(@repos, & &1.start_link(pool_size: 2))
  end

  defp stop_services do
    IO.puts("Success!")
    :init.stop()
  end

  defp run_migrations do
    Enum.each(@repos, &run_migrations_for/1)
  end
end
```

我们可以使用 `eval` 在我们的发布中执行这个函数。调用 `bin/My_RELEASE eval` 将启动你的发布版本，并执行你给 `eval` 提供的任何函数。要在我们的版本中执行我们的迁移函数。

```bash
bin/registrar_umbrella eval "Registrar.ReleaseTasks.migrate()"
```

记得我们在 `./bin/start` 中用 `start` 命令启动我们的发布。

```bash
bin/registrar_umbrella start
```

这将依次执行 `rel/env.sh` 文件。该文件应包含一个执行以下操作的脚本。

* 如果发布的命令是 `start`，使用 `eval` 运行迁移。

类似这样的脚本应该可以做到。

```shell
if [ "$RELEASE_COMMAND" = "start" ]; then
 echo "Beginning migration script..."
 bin/registrar_umbrella eval "Registrar.ReleaseTasks.migrate()"
fi
```

就是这样！

## 结语

有了 Elixir 1.9，我们可以在不添加任何外部依赖关系的情况下构建一个版本-- Elixir 现在原生地提供了我们所需的一切。我们可以为我们的 umbrella 应用配置多个版本，定义在给定的版本中启动哪些子应用。我们可以配置运行时与构建时的环境变量，我们甚至可以定义自定义的启动脚本来做一些事情，比如运行我们的迁移。总而言之，`mix release` 为我们提供了一套全面而强大的工具。

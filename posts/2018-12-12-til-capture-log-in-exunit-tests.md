---
author: Alex Griffith
author_link: https://github.com/alexgriff
categories: til
date: 2018-12-12
layout: post
tags: ['testing', 'logging']
title:  TIL about ExUnit's capture_log option
excerpt: >
  Capture the output from Logger to clean up your test runs
---

# TIL 关于 ExUnit 的 capture_log 选项。

你是否曾经运行过 `mix test`，看到红色的错误信息被记录下来，而事实上，你的所有测试都通过了？当在 "sad path" 代码流中调用 `Logger.error/1` 增加测试覆盖范围时，经常会出现这种情况。

这里有一个（略显造作）的例子来说明这个问题。`GithubClient.get_user_repos/1` 函数接收一个 GitHub 用户名，向 GitHub API 发出请求，并返回一个用户的仓库列表。注意，如果没有找到 GitHub 用户，则调用 `Logger.error/1`。

```elixir
defmodule GithubClient do
  require Logger

  def get_user_repos(username) do
    case HTTPoison.get "https://api.github.com/users/" <> username <> "/repos" do
      {:ok, %HTTPoison.Response{status_code: 200, body: body}} ->
        body
        |> decode_response
        |> handle_success(username)
      {:ok, %HTTPoison.Response{status_code: 404, body: body}} ->
        body
        |> decode_response
        |> handle_not_found(username)
    end
  end

  def handle_not_found(response, username) do
    Logger.error "User " <> username <> " does not exist"
    {:error, response}
  end

  # ...
end
```

这是相应的 ExUnit 测试。 细节不是很重要，但是测试配置使用[模拟服务器](https://medium.com/flatiron-labs/rolling-your-own-mock-server-for-testing-in-elixir-2cdb5ccdd1a0).


```elixir
defmodule GithubClientTest do
  use ExUnit.Case

  describe "get_user_repos/1 with valid username" do
    test "returns the user's repositories" do
      {:ok, ["repo01", "repo02" | _tail]} = GithubClient.get_user_repos("valid_username")
    end
  end

  describe "get_user_repos/1 with invalid username" do
    test "returns an error tuple" do
      {:error, _message} = GithubClient.get_user_repos("invalid_username")
    end
  end
end
```

太棒了-测试通过了！ 不过，在输出中，您会看到红色错误记录，该错误记录是由 404 找不到案例的 _passing_ 测试导致的：

`17:32:00.090 [error] User invalid_username does not exist`

目前我们只有两个测试，但是随着测试套件的增加，这可能会真正分散您的注意力。 这不仅适用于包含错误日志记录的 "sad path" 测试，而且适用于我们可能使用 [`Logger`模块](https://hexdocs.pm/logger/Logger.html) 随附的其他功能进行的任何日志记录 `Logger.info/1` 或 `Logger.debug/1`


有几种解决方法。 要停止针对 _all tests_ 显示日志记录的输出，请在您的 ExUnit 配置中添加 `capture_log: true`。 在一个新的混合项目中，可以在 `test/test_helper.exs` 中找到它，如下所示：

```elixir
ExUnit.start(capture_log: true)
```

要捕获特定测试模块的日志，可以将[`moduletag`](https://hexdocs.pm/ex_unit/ExUnit.Case.html#module-module-and-describe-tags)添加到特定测试模块

```elixir
defmodule GithubClientTest do
  use ExUnit.Case

  @moduletag capture_log: true
  #...
end
```

就是这样。 看看所有的绿色！

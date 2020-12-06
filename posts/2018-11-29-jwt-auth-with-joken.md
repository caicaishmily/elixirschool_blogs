---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
tags: ['plug', 'authentication']
date:   2018-11-29
layout: post
title:  JWT Auth in Elixir with Joken
excerpt: >
  Use Joken and JOSE for a light-weight implementation of JWT Auth in your Elixir web application.
---

# JWT Auth in Elixir with Joken

[JSON Web Tokens](https://jwt.io/introduction/)，即 JWTs，允许我们通过将认证信息加密到一个安全、紧凑的 JSON 对象中，并进行数字签名来验证客户端和服务器之间的请求。在这篇文章中，我们将使用 [Joken](https://github.com/bryanjos/joken) 库在 Phoenix 应用中实现 JWT auth。我们将专注于使用 ECDSA 私钥/公钥对签署的 JWT，尽管你也可以使用 HMAC 算法签署 JWT。

## 起步

首先，我们需要将 Joken 包添加在我们应用程序的依赖关系中。

```elixir
def deps do
  # .. other deps
  {:joken, "~> 2.0-rc0"}
end
```

运行 `mix deps.get` 然后使用 Joken 的工作已准备就绪
## 关于加密的说明

**我们将对使用 ECDSA 私钥/公钥对生成的令牌进行解密**，这意味着我们需要访问公钥才能进行解密。这意味着我们需要访问公钥来进行解密，在哪里存储公钥由你来决定。你可以把它存储在一个 `.pem` 文件中，你的应用程序可以访问它；你可以从一端起个服务；你可以把它存储在一个环境变量中--这只是一些选择。

这篇文章假设你的代码可以访问 ECDSA 私钥/公钥对的公共部分，其形式是一个类似于这样的字符串。

```
-----BEGIN PUBLIC KEY-----
blahblahblah
yaddayaddayadda
-----END PUBLIC KEY-----
```

## 解密模块

我们定义一个 `JwtAuthToken` 模块，负责对给定令牌和公钥的 JWT 进行解密；

```elixir
defmodule MyAppWeb.JwtAuthToken do
  def decode(jwt_string, public_key) do
    # coming soon!
  end
end
```

我们模块的 公共 API 很简单。它暴露了一个函数 `decode/2`，它接收 JWT 字符串和 ECDSA 公钥字符串的参数。它将使用公钥来解密 JWT。

### Joken如何解码和验证？

为了解码和验证我们的 JWT 字符串， Joken 需要两个结构：

* 一个 `Joken.Token` 结构
* 一个 `Joken.Signer` 结构

因此，我们需要使用 token _字符串_ 生成一个 `Joken.Token`，我们需要使用 ECDSA 公钥 PEM 文件生成一个 `Joken.Signer` 结构。然后，我们将使用这两个结构作为参数调 用`Joken.verify/2`。

### 生成 `Joken.Token`

为了生成这个结构，我们将调用 `Joken.token/1`。我们传递一个 JWT 字符串作为参数。

```elixir
defmodule MyAppWeb.JwtAuthToken do
  def decode(jwt_string, public_key) do
   jwt_string
   |> Joken.token
  end
end
```

这将返回如下格式的 `Joken.Token` 结构：

```elixir
%Joken.Token{
  claims: %{},
  claims_generation: %{},
  error: nil,
  errors: [],
  header: %{},
  json_module: Poison,
  signer: nil,
  token: "blah.blah.blah",
  validations: %{}
}
```

#### 验证 token 到期时间

我们还没有完全完成我们的 token 结构。请注意，`:validations` 键指向一个空的映射。存储在 token 结构 `:validations` 键下的数据将被 `Joken.verify/2` 使用，以确定解码后的 token 声明的有效性。我们 token 的编码要求将包括一个 *失效日期* ，键为 `"exp"`。我们只希望解码后的 token 被认为是有效的，如果声明中的 `"exp"` 没有过期。因此我们将利用 `Joken.with_validation` 来编写一个验证函数，如果 token 的 claims `"exp"` 没有过期，则返回 true。

```elixir
defmodule MyAppWeb.JwtAuthToken do
  def decode(jwt_string, public_key) do
   jwt_string
   |> Joken.token
   |> Joken.with_validation("exp", &(&1 > Joken.current_time()))
  end
end
```

现在我们的 token 结构看上去像这样：

```elixir
%Joken.Token{
  claims: %{},
  claims_generation: %{},
  error: nil,
  errors: [],
  header: %{},
  json_module: Poison,
  signer: nil,
  token: "blah.blah.blah",
  validations: %{"exp" => {#Function<6.99386804/1 in :erl_eval.expr/5>, nil}}
}
```

这样，当我们稍后调用 `Joken.verify/2` 时，Joken 会执行储存在 `:validations` 结构 `"exp"` 键下的函数，其参数是储存在解密 token claims `"exp"` 下的值。

如果该函数返回 `true`，Joken 将公开解密 token 的 claims。

```elixir
%Joken.Token{
  claims: %{
    "aud" => ["user"],
    "email" => "guy@email.com.com",
    "exp" => 1540399830,
    "iat" => 1540392630,
    "nbf" => 1540392630,
    "sub" => "ea375e5a-f918-4017-a5ee-1fc8b641ef84"
  },
  claims_generation: %{},
  error: nil,
  errors: [],
  header: %{},
  json_module: Poison,
  signer: <coming soon!>,
  token: "blah.blah.blah",
  validations: %{
    "exp" => {#Function<0.91892837/1 in DeployerWeb.JwtAuthToken.decode/2>, nil}
  }
}
```

如果返回的是 `false`, Joken 将返回 token 结构，_不包含_ 解码后的 claims字段 和 _错误信息_。

```elixir
%Joken.Token{
  claims: %{},
  claims_generation: %{},
  error: "Invalid payload",
  errors: ["Invalid payload"],
  header: %{},
  json_module: Poison,
  signer: <coming soon!>,
  token: "blah.blah.blah",
  validations: %{
    "exp" => {#Function<0.91892837/1 in DeployerWeb.JwtAuthToken.decode/2>, nil}
  }
}
```

现在我们已经准备好了我们的 token 结构，我们可以生成 `Joken.Signer` 结构。
### 生成 `Joken.Signer`

In order to generate the signer struct, we need to build our ECDSA public key struct. We can doing this using `JOSE`.

#### Generating the ECDSA Signing Key with `JOSE`

[`JOSE`](https://github.com/potatosalad/erlang-jose) stands for JSON Object Signing and Encryption. Its a set of standards developed by the JOSE Working Group. The `JOSE` package is a dependency of Joken, so we don't need to install it ourselves via our application dependencies.

Joken needs our public key in the form of a map in order to use it to decrypt our token. We'll use the `JOSE.JWK` (JWK stands for JSON Web Key) module to turn our public key string into a map.

Let's define a private helper function, `signing_key` in our `MyAppWeb.JwtAuthToken` module:

```elixir
defmodule MyAppWeb.JwtAuthToken do
  ...

  defp signing_key(public_key) do
    { _, key_map } = public_key
      |> JOSE.JWK.from_pem
      |> JOSE.JWK.to_map
    key_map
  end
end
```

The first function call, `JOSE.JWK.from_pem` converts our public key PEM binary into a `JOSE.JWK`. The second function call, `JOSE.JWK.to_map` (you guessed it) converts that `JOSE.JWK` into a map. So, we end up with a tuple that looks like this:

{% raw %}
```elixir
{%{kty: :jose_jwk_kty_ec},
 %{
   "crv" => "P-256",
   "kty" => "EC",
   "x" => "xxxx",
   "y" => "xxxx"
 }}
```
{% endraw %}

Where the second element of the tuple is the ECDSA public key map. Joken will use this map as a key when generating an ECDSA signer.

#### Generating the Signer

`Joken.Signer` 是 Joken 的 JWK（JSON Web Key）和 JWS（JSON Web Signature）配置。签名器允许我们在解密过程中生成 token 签名或读取 token 签名。我们要用我们的公钥生成一个 ECDSA 签名器。然后，我们可以使用这个签名器来解密我们的 token。

我们将定义另一个私有辅助函数 `signer/1` 来实现这一目的。

```elixir
defmodule MyAppWeb.JwtAuthToken do
  ...
  defp signer(public_key_string) do
    public_key_string
    |> signing_key
    |> Joken.es256
  end

  defp signing_key(public_key_string) do
    { _, key_map } = public_key_string
      |> JOSE.JWK.from_pem
      |> JOSE.JWK.to_map
    key_map
  end
end
```

在这里，我们使用 `Joken.es256` 函数，并以我们的公钥作为参数，生成 ECDSA 令牌签名器。`es256` 函数封装了对 [`Joken.Signer.es/2`](https://hexdocs.pm/joken/Joken.Signer.html#es/2) 的调用，它接收算法类型和密钥映射，并返回签名器。

现在我们有了 ECDSA 签名器，我们准备好解码我们的 token 了。

### 使用 Token 和 Signer 解码 token

```elixir
defmodule MyApp.Web.JwtAuthToken do
  def decode(jwt_string, public_key_string) do
    jwt_string
    |> Joken.token
    |> Joken.with_validation("exp", &(&1 > Joken.current_time()))
    |> Joken.with_signer(signer(public_key_string))
    |> Joken.verify
  end

  defp signer(public_key_string) do
    public_key_string
    |> signing_key
    |> Joken.es256
  end

  defp signing_key(public_key_string) do
    { _, key_map } = public_key_string
      |> JOSE.JWK.from_pem
      |> JOSE.JWK.to_map
    key_map
  end
end
```

现在我们可以很容易的解密 JWTs，就像这样：

```elixir
JwtAuthToken.decode(jwt_string, public_key)
=> {
     :success,
     %{
       token: "blah.blah.blah",
       claims: %{sub: "1234", email: "guy@email.com"}
     }
   }
```

让我们在自定义的插件中使用我们的解码器，以防止任何没有有效 JWT 的人访问我们应用程序的端点。

## 认证插件

我们将构建一个自定义插件， `JwtAuthPlug` 我们将把它放在认证路由的管道中。

```elixir
# router.ex
...
pipeline :api do
  plug :accepts, ["json"]
  plug MyAppWeb.JwtAuthPlug
end
```

我们的插件很简单，它将：

1. 从请求的 cookie 中抓取 JWT。
2. 调用我们的 `JwtAuthToken.decode/2` 函数对其进行解码。

如果它能成功解码 JWT，它将允许请求通过。如果不能，它将返回一个 `401` 未授权状态。

让我们开始吧!

### 定义自定义插件

定义一个自定义插件非常简单。我们需要 `import Plug.Conn` 来访问一些有用的连接-交互函数。然后，我们需要一个 `init` 函数和一个 `call` 函数。

```elixir
defmodule MyAppWeb.JwtAuthPlug do
  import Plug.Conn
  alias MyAppWeb.JwtAuthToken

  def init(opts), do: opts

  def call(conn, _opts) do
    # coming soon!
  end
end
```

### 从 cookie 中获取 JWT

我们将定义一个辅助函数，`jwt_from_cookie` 将从请求 cookie 中提取 JWT 字符串。

```elixir
defmodule MyAppWeb.JwtAuthPlug do
  import Plug.Conn
  alias MyAppWeb.JwtAuthToken

  ...

  defp jwt_from_cookie(conn) do
    conn
    |> Plug.Conn.get_req_header("cookie")
    |> List.first
    |> Plug.Conn.Cookies.decode
    |> token_from_map(conn)
  end

  defp token_from_map(%{"session_jwt" => jwt}, _conn), do: jwt

  defp token_from_map(_cookie_map, conn) do
    conn
    |> forbidden
  end

  defp forbidden(conn) do
    conn
    |> put_status(:unauthorized)
    |> Phoenix.Controller.render(MyAppWeb.ErrorView, "401.html")
    |> halt
  end
end
```

这里，我们使用了一个 `Plug.Conn` 中便捷的 `Plug.Conn.get_req_header` 函数来获取 Cookie 请求头的值。 然后，我们使用另一个函数 `Plug.Conn.Cookies.decode` 将该值（由 `,` , `, `, 或 `;` 分隔的字符串）变成一个映射。最后，我们将 map 中的 JWT 进行模式匹配。

现在我们有了 JWT，让我们对它进行解码吧!

### 解码 JWTs

```elixir
defmodule MyAppWeb.JwtAuthPlug do
  import Plug.Conn
  alias MyAppWeb.JwtAuthToken

  def call(conn, _opts) do
    case JwtAuthToken.decode(jwt_from_map, public_key) do
      { :success, %{token: token, claims: claims} } ->
        conn |> success(claims)
      { :error, error } ->
        conn |> forbidden
    end
  end

  defp public_key do
    # your public key string that you read from a PEM file or stored in an env var or fetched from an endpoint
  end

  defp success(conn, token_payload) do
    assign(conn, :claims, token_payload.claims)
    |> assign(:jwt, token_payload.token)
  end
end
```

就是这样！

## 结语

Joken 让您在 Phoenix 应用中轻松解码 JWTs。通过使用 `JOSE` 生成您自己的 ECDSA 签名器，并构建一个简单的自定义插件，您可以保证您的路由安全。祝您编码愉快!

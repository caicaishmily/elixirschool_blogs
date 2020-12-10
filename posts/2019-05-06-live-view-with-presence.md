---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
date: 2019-05-21
layout: post
title:  Tracking Users in a Chat App with LiveView, PubSub Presence
excerpt: >
  Use Phoenix Presence in your LiveView to track user state with just a few lines of code.
---

# 使用 LiveView 和 PubSub Presence 跟踪聊天应用中的用户

在使用 LiveView 和 Phoenix PubSub 向一个 live view 的所有客户端广播消息之后，我想尝试结合 Phoenix Presence 来跟踪这些客户端的状态。于是在上周末，我利用 Phoenix LiveView、PubSub 和 Presence 搭建了一个聊天应用。LiveView 的代码只有90行，而我能够在短时间内让 Presence 支持的功能启动并运行起来。继续阅读，看看它是如何工作的。

## 应用

聊天应用是相当直接的，我们不会在这里讨论在我们的 Phoenix 应用中设置 LiveView 的细节。您可以查看[源代码](https://github.com/elixirschool/live-view-chat/tree/tutorial) 以及这个 [早先的文章](https://elixirschool.com/blog/phoenix-live-view/)，了解更多信息，让 LiveView 开始运行。

### 跟随我一起

如果你想跟着本教程一起学习，请克隆下 [repo](https://github.com/elixirschool/live-view-chat/tree/tutorial)，然后按照 README 说明来启动和运行。本教程分支的起始状态包括聊天域模型、路由、控制器和 LiveView 的初始状态，如下所述。你也可以[在这里](https://github.com/elixirschool/live-view-chat)查看完整代码。

### `ChatLiveView` 的初始化状态

我们通过告诉 `ChatController` 的 `show` 动作来渲染 `ChatLiveView`，将我们的实时视图挂载到 `/chats/:id`。我们从控制器中把给定的聊天和当前用户传入我们的实时视图。

```elixir
# lib/phat_web/controllers/chat_controller.ex

defmodule PhatWeb.ChatController do
  use PhatWeb, :controller
  alias Phat.Chats
  alias Phoenix.LiveView
  alias PhatWeb.ChatLiveView

  def show(conn, %{"id" => chat_id}) do
    chat = Chats.get_chat(chat_id)

    LiveView.Controller.live_render(
      conn,
      ChatLiveView,
      session: %{chat: chat, current_user: conn.assigns.current_user}
    )
  end
end
```

`ChatLiveView.mount/2` 函数设置了给定聊天的 LiveView 套接字的初始状态，一个空的消息变化集，用它来填充新消息的表单，以及当前用户。

```elixir
# lib/phat_web/live/chat_live_view.ex

defmodule PhatWeb.ChatLiveView do
  use Phoenix.LiveView
  alias Phat.Chats

  def render(assigns) do
    PhatWeb.ChatView.render("show.html", assigns)
  end

  def mount(%{chat: chat, current_user: current_user}, socket) do
    {:ok,
     assign(socket,
       chat: chat,
       message: Chats.change_message(),
       current_user: current_user
     )}
  end
end
```

挂载并设置好 socket 状态后，live view 会渲染 `ChatView` 的 `show.html` 模板：

```elixir
# lib/phat_web/templates/chat/show.html.leex

<h2><%= @chat.room_name %></h2>
<%=for message <- @chat.messages do %>
  <p>
    <%= message.user.first_name %>: <%= message.content %>
  </p>
<% end %>

<div class="form-group">
  <%= form_for @message, "#", [phx_submit: :message], fn _f -> %>
    <%= text_input :message, :content, placeholder: "write your message here..." %>
    <%= hidden_input :message, :user_id, value: @current_user.id  %>
    <%= hidden_input :message, :chat_id, value: @chat.id  %>
    <%= submit "submit" %>
  <% end %>
</div>
```

我们的模板很简单：它抓取我们分配给实时视图套接字的聊天，显示聊天室名称，并遍历消息以显示内容和发件人。它还包含一个新消息的表单，建立在我们分配给套接字的空消息更改集上。此时，我们的渲染模板看起来像这样。

![APP img](https://elixirschool.com/assets/live-view-1-d87784f1800c025627ca41ed2726b016fab2c3c947e7dd58c6b32b4e3f8d2b35.png)

### 推送消息至 LiveView 客户端

现在我们的实时视图已经开始运行起来了，让我们来看看当一个给定的用户提交表单以获取新消息时会发生什么。

我们将 `phx-submit` 事件附加到表单的提交中，并指示它发出一个类型为 `"message"` 的事件。

```elixir
# lib/phat_web/templates/chat/show.html.leex

  <%= form_for @message, "#", [phx_submit: :message], fn _f -> %>
  ...
```

现在，我们需要通过定义一个 `handle_event/3` 匹配函数教 live view 如何去处理这个事件

```elixir
# lib/phat_web/live/chat_live_view.ex

defmodule PhatWeb.ChatLiveView do
  ...
  def handle_event("message", %{"message" => message_params}, socket) do
    chat = Chats.create_message(message_params)
    {:noreply, assign(socket, chat: chat, message: Chats.change_message())}
  end
end
```

Live view 通过创建一个新的消息并且用更新的聊天记录和新的空消息变化集更新套接字来响应 `"message"` 事件。*请注意，虽然我们将 `phx_submit` 的值指定为一个原子 `:message`，但我们的 live view 进程将消息作为一个字符串 `"message"`* 来接收。

然后 live view 会重新显示页面的相关部分，在这种情况下，聊天和消息显示以及新消息的表单。

多亏了这段代码，我们客户端的消息表单将消息推送至套接字。但是我们实时视图中的其他客户端--聊天室中的其他用户呢？

## 使用 Phoenix PubSub 广播消息

为了将新消息广播给所有用户，我们需要使用 Phoenix PubSub。

首先，我们需要确保每个客户端在加载实时视图时开始订阅聊天室的 PubSub 主题。

```elixir
# lib/phat_web/live/chat_live_view.ex

defp topic(chat_id), do: "chat:#{chat_id}"

def mount(%{chat: chat, current_user: current_user}, socket) do
  PhatWeb.Endpoint.subscribe(topic(chat.id))

  {:ok,
   assign(socket,
     chat: chat,
     message: Chats.change_message(),
     current_user: current_user
   )}
end
```

然后，当我们的 live view 处理 `"message"` 事件时，我们需要让 live view 广播新的消息给订阅者。

```elixir
# lib/phat_web/live/chat_live_view.ex

def handle_event("message", %{"message" => message_params}, socket) do
  chat = Chats.create_message(message_params)
  PhatWeb.Endpoint.broadcast_from(self(), topic(chat.id), "message", %{chat: chat})
  {:noreply, assign(socket, chat: chat, message: Chats.change_message())}
end
```

`broadcast_from/4` 函数将向所有订阅的客户端广播一条类型为 `"message"` 的消息，其中包含我们新更新的聊天内容，*不包括发送消息的客户端*。

最后，我们需要教我们的实时视图如何用 `handle_info/2` 函数响应这个广播。

```elixir
# lib/phat_web/live/chat_live_view.ex

def handle_info(%{event: "message", payload: state}, socket) do
  {:noreply, assign(socket, state)}
end
```

实时视图通过使用 `%{chat: chat}` payload 更新套接字的状态来处理 `"message"` 消息，其中 chat 是我们新更新的聊天，包含用户的新消息。这就是确保所有订阅的客户端都能看到提交到聊天模板的新消息表单中的新消息的全部内容。

## 使用 Phoenix Presence 跟踪用户

现在我们的实时视图已经足够智能，可以向给定聊天室中的所有用户广播消息，我们已经准备好构建一些功能来跟踪这些用户并与之互动。假设我们想让我们的模板呈现聊天室中的用户列表，就像这样。

![](https://elixirschool.com/assets/live-view-presence-1-907958fcd1bf97792edc573c51247144afc05a41a3c478363ac2ef81456c33e6.png)

我们可以创建自己的数据结构来跟踪实时视图中的用户存在，将其存储在实时视图的 socket 中，当用户加入、离开或以其他方式改变其状态时，我们可以手工滚动自己的函数来更新该数据结构。然而，[Phoenix Presence behavior](https://hexdocs.pm/phoenix/Phoenix.Presence.html) 将这些工作从我们身上抽象出来。它为进程和 channels 提供了存在跟踪，利用 Phoenix PubSub 在幕后广播更新。它还使用了 CRDT（Conflict-free Replicated Data Type 无冲突重复数据类型）模型，这意味着它可以在分布式应用上工作。

现在我们对 Presence 是什么以及为什么要使用它有了一定的了解，接下来我们在应用程序中对它进行设置。

### 设置 Presence

为了在我们的 Phoenix 应用中使用 Presence，我们需要在我们的模块中定义：

```elixir
# lib/phat_web/presence.ex

defmodule PhatWeb.Presence do
  use Phoenix.Presence,
    otp_app: :phat,
    pubsub_server: Phat.PubSub
end
```

`PhatWeb.Presence` 模块做三件事：

* `uses` Presence 行为
* 指定它与应用程序的其他部分共享一个 PubSub 服务器。
* 指定共享应用程序的 OTP 应用，它保存着我们的应用程序配置。

现在，我们可以在整个应用程序中使用 `PhatWeb.Presence` 模块来跟踪用户在特定进程中的存在。

### 跟踪用户的存在

我们的 Presence 模块将通过将这些用户存储在 `"chat:#{chat_id}"` 给定主题下，来维护给定聊天室中的当前用户列表。

那么，我们应该在什么时候告诉 Presence 开始跟踪某个用户呢？在什么时候我们才会认为一个用户 "出现" 在聊天室中呢？当用户挂载实时视图的时候!

我们将其钩入我们的 `mount/2` 函数，将新用户添加到特定聊天室的 Presence 用户列表中。

```elixir
# lib/phat_web/live/chat_live_view.ex
©
def mount(%{chat: chat, current_user: current_user}, socket) do
  Presence.track(
    self(),
    topic(chat.id),
    current_user.id,
    %{
      first_name: current_user.first_name,
      email: current_user.email,
      user_id: current_user.id
    }
  )
  ...
end
```

在这里，我们使用 [`Presence.track/4`](https://hexdocs.pm/phoenix/Phoenix.Presence.html#c:track/4) 函数来跟踪我们的 Live View 进程作为一个存在。我们将 LiveView 进程的 PID 添加到 Presence 的数据存储中，同时添加一个描述新用户的 payload，主题为 `"chat:#{chat.id}"` 和用户 ID 的 key。

对给定主题 Presence 进程的状态将是这样的：

```elixir
%{
  "1" => %{
    metas: [
      %{
        email: "sophie@email.com",
        first_name: "Sophie",
        phx_ref: "TNV4PzRfyhw=",
        user_id: 1
      }
  }
}
```

#### Broadcasting Presence To Existing Users

When we call `Presence.track`, Presence will broadcast a `"presence_diff"` event over its PubSub backend. We told our Presence module to use the same PubSub server as the rest of the application––the very same server that backs our `PhatWeb.Endpoint`.
#### 向现有用户广播新加入用户

当我们调用 `Presence.track` 时，Presence 将在其 PubSub 后端广播一个 `"being_diff"` 事件。我们告诉我们的 Presence 模块使用与应用程序其他部分相同的 PubSub 服务器--也就是支持我们的 `PhatWeb.Endpoint` 的同一个服务器。

回想一下，我们的 LiveView 客户端是通过 `mount/2` 函数中的以下调用来订阅这个 PubSub 服务器的：`PhatWeb.Endpoint.subscribe(topic(chat.id))`。因此，这些订阅的 LiveView 进程将收到 `"presence_diff"` 事件，它看起来像这样：

```elixir
%{
  event: "presence_diff",
  payload: %{
    joins:
      %{
        "1" => %{
          metas: [
            %{
              email: "sophie@email.com",
              first_name: "Sophie",
              phx_ref: "TNV4PzRfyhw=",
              user_id: 1
            }
          }
        },
    leaves: %{},
  }
}
```

该事件的 payload 将描述当 `Presence.track/4` 被调用时加入 channel 的用户。虽然我们会对 `"presence_diff"` 事件做出响应，但目前我们不会对事件的 payload 做任何事情。然而，你可以想象用它来创建自定义的用户体验，比如欢迎新加入的用户或提醒现有用户某个新成员加入了聊天室。

为了响应该事件，我们将在实时视图中定义一个 `handle_info/2` 函数，它将与 `"existence_diff"` 事件相匹配。

```elixir
# lib/phat_web/live/chat_live_view.ex

def handle_info(%{event: "presence_diff"}, socket = %{assigns: %{chat: chat}}) do

end
```

这个函数做两件事：

* 从 "Presence" 数据存储中获取指定聊天室主题的当前用户列表。
* 更新 LiveView 套接字的状态，以反映该用户列表。

```elixir
def handle_info(%{event: "presence_diff", payload: _payload}, socket = %{assigns: %{chat: chat}}) do
  users =   
    Presence.list(topic(chat.id))
    |> Enum.map(fn {_user_id, data} ->
      data[:metas]
      |> List.first()
    end)

  {:noreply, assign(socket, users: users)}
end
```

首先，我们使用 `Presence.list/1` 函数来获取指定主题下的当前用户集合。这将返回以下数据结构：

```elixir
%{
  "1" => %{
    metas: [
      %{
        email: "sophie@email.com",
        first_name: "Sophie",
        phx_ref: "TNV4PzRfyhw="
        user_id: 1
      }
  },
  "2" => %{
    metas: [
      %{
        email: "beini@email.com",
        first_name: "Beini",
        phx_ref: "ZZ30QuoI/8s="
        user_id: 1
      }
  }
  ...
}
```

Presence 行为为我们处理了加入和离开事件的差异。因此，只要我们调用 `Presence.track/4`，Presence 进程就会更新自己的状态，这样当我们下一次调用 `Presence.list/1` 时，我们就会检索更新后的用户列表。

一旦我们获取了这个列表，我们就会对其进行迭代，收集描述每个用户的单个 `:metas` 有效载荷的列表。得到的列表将是这样的：

```elixir
[
  %{
    email: "sophie@email.com",
    first_name: "Sophie",
    phx_ref: "TNV4PzRfyhw="
    user_id: 1
  },
  "2" => %{
    metas: [
      %{
        email: "beini@email.com",
        first_name: "Beini",
        phx_ref: "ZZ30QuoI/8s="
        user_id: 1
      }
  }
]
```

我们进行这个转换，这样我们就有了一个简单的、易于使用的数据结构，当我们想列出当前的用户名时，就可以在模板中与之交互。

最后，我们通过添加一个指向用户列表值的 `:users` 键来更新 LiveView 套接字的状态。

```elixir
{:noreply, assign(socket, users: users)}
```

现在我们可以通过模板中的 `@users` 赋值来访问用户列表，列出聊天室中存在的用户名称。

```elixir
# lib/phat_web/templates/chat/show.html.leex

<h3>Members</h3>
<%= for user <- @users do %>
  <p>
    <%= user.first_name %>
  </p>
<% end %>
```

我们来总结一下。目前我们写的代码支持以下流程。

当用户访问了位于 `/chats/:id` 的聊天室，并且 LiveView 被挂载时... ...

* 将该用户添加到给定聊天室主题的 Presence 数据存储的用户列表中。
* 向订阅者广播，告诉他们从 Presnce 数据存储中抓取当前用户的最新名单。
* 用更新后的列表更新实时视图套接字的状态。
* 重新更新实时视图模板，以显示更新后的用户名单。

这使得 *已经在聊天室* 的用户可以看到任何加入聊天室的用户的更新列表。

但是加入聊天室的用户怎么办？我们如何确保当新用户访问聊天室时，他们能看到已经存在的用户列表？

#### 为新用户获取 Presence

为了将现有的聊天室成员显示给任何新加入的用户，我们需要从 Presence 中获取这些用户，并在实时视图挂载时将其分配给实时视图套接字。

让我们更新我们的 `mount/2` 函数来实现这个目的。

```elixir
# lib/phat_web/live/chat_live_view.ex

def mount(%{chat: chat, current_user: current_user}, socket) do
  ...
  users =   
    Presence.list(topic(chat.id))
    |> Enum.map(fn {_user_id, data} ->
      data[:metas]
      |> List.first()
    end)

  {:ok,
   assign(socket,
     chat: chat,
     message: Chats.change_message(),
     current_user: current_user,
     users: users
   )}
end
```

现在，我们的实时视图将能够为新用户加载页面呈现已有成员列表。

#### 广播用户离开事件

此时，你可能会想知道，当用户离开被跟踪的流程时，我们如何更新 Presence 状态并广播变化。这实际上是我们免费获得的功能，感谢 Presence。回想一下，我们是通过 `Presence.track/4` 函数来跟踪给定 LiveView 进程的存在，其中我们给 `track/4` 的第一个参数是 LiveView 进程的 PID。

当用户导航离开聊天页面时，他们的 LiveView 进程就会终止。这将导致 `Presence.untrack/3` 被调用，从而解除对该 PID 的跟踪。这反过来又会告诉 Presence 广播 `"presence_diff"` 事件，这次的 payload 是描述离开的用户，也就是我们在终止的 PID 下跟踪的用户。Presence 知道如何处理来自加入 *和* 离开事件的差异--它将适当地更新它存储在聊天室主题下的用户列表。

正在运行的 LiveView 进程如果接收到这个 `"presence_diff"` 事件，就需要为给定的主题获取这个更新的当前用户列表，更新 socket 状态并相应地重新渲染页面。这意味着我们可以在不做任何改变的情况下，为 `"presence_diff"` 事件重新使用我们原来的 `handle_info/2` 函数。

```elixir
# lib/phat_web/live/chat_live_view.ex

def handle_info(%{event: "presence_diff", payload: _payload}, socket = %{assigns: %{chat: chat}}) do
  users =   
    Presence.list(topic(chat.id))
    |> Enum.map(fn {_user_id, data} ->
      data[:metas]
      |> List.first()
    end)

  {:noreply,
   assign(socket,
     users: users
   )}
end
```

所以，我们根本不需要写任何额外的代码来处理 "离开" 事件!

#### 使用 Presence 来跟踪用户状态

到目前为止，我们已经利用 Presence 来跟踪用户加入或离开 LiveView 的情况。我们也可以使用 Presence 来跟踪特定用户在 LiveView 进程中的状态。让我们来看看它是如何工作的，构建一个功能，通过在模板中呈现的当前用户列表中给他们的名字添加一个 `"..."` 来表明一个给定用户正在输入新的聊天消息表单。

![](https://elixirschool.com/assets/live-view-presence-2-7cc9d9b1791ce35ff50d5c1dc861c31cfda4c756b8412898ffd5610427e91774.png)

首先，我们将更新 `:metas` payload，我们用数据点来描述一个给定用户的起始状态：`typing: false`。

```elixir
# lib/phat_web/live/chat_live_view.ex

def mount(%{chat: chat, current_user: current_user}, socket) do
  Presence.track(
    self(),
    topic(chat.id),
    current_user.id,
    %{
      first_name: current_user.first_name,
      email: current_user.email,
      user_id: current_user.id,
      typing: false
    }
  )
  ...
end
```

然后，我们将附加一个新的 `phx-change` 事件到我们的表单中，当用户在表单字段中键入时，该事件将启动 `typing` 的消息类型。

```elixir
# lib/phat_web/templates/chat/show.html.leex

<%= form_for @message, "#", [phx_change: :typing, phx_submit: :message], fn _f -> %>
  ...
<% end %>
```

接下来，我们将教我们的实时视图使用一个新的 `handle_event/2` 函数来处理这个事件，该函数与 `"typing"` 事件类型相匹配。为了响应这个事件，实时视图应该更新当前用户在指定聊天室主题下的 `:metas` map。

```elixir
# lib/phat_web/live/chat_live_view.ex

def handle_event("typing", _value, socket = %{assigns: %{chat: chat, current_user: user}}) do
  topic = topic(chat.id)
  key   = user.id
  payload = %{typing: true}
  metas =
      Presence.get_by_key(topic, key)[:metas]
      |> List.first()
      |> Map.merge(payload)

  Presence.update(self(), topic, key, metas)
  {:noreply, socket}
end
```

在这里，我们使用 `Presence.get_by_key/2` 函数获取当前用户的 `:metas`，存储在 `"chat:#{chat.id}"` 的 `topic` 下，用户 ID 的键下。

然后，我们为该用户创建一个 `:metas` map 的副本，将 `:typeing` 键设置为 `true`。

最后，我们更新 Presence 进程的主题和用户的元数据，指向这个新的 map。调用 `Presence.update/4` 将再次为我们广播一个 `"being_diff"` 事件。我们的 LiveView 进程已经知道如何处理这个事件，所以我们不需要编写任何额外的代码来确保运行中的 LiveView 进程获取最新的用户列表与新的元数据，并重新渲染页面。

最后，我们需要做的是更新我们的模板，在列表中任何将 `typing` 设置为 `true` 的用户的名字后面添加 `"..."` 。

```elixir
# lib/phat_web/templates/chat/show.html.leex

<h3>Members</h3>
<%= for user <- @users do %>
  <p>
    <%= user.first_name %><%= if user.typing, do: "..." end%>
  </p>
<% end %>
```

现在我们准备教我们的 LiveView 如何在用户 *停止* 打字时表现出来，确保模板将重新渲染，而不在用户的名字上附加 `"..."`。

我们将在消息内容表单字段中添加一个 `phx-blur` 事件。

```elixir
# lib/phat_web/templates/chat/show.html.leex

  <%= text_input :message, :content, value: @message.changes[:content], phx_blur: "stop_typing", placeholder: "write your message here..." %>
```

当用户从这个表单字段离开焦点时，这将向 LiveView 进程发送一个类型为 `"stop_typing"` 的事件。

我们将教我们的 LiveView 用 `handle_info/2` 来响应这个消息，用 `typing: false` 来更新当前用户的存在元数据。

```elixir
# lib/phat_web/live/chat_live_view.ex

def handle_event(
      "stop_typing",
      value,
      socket = %{assigns: %{chat: chat, current_user: user, message: message}}
    ) do
  message = Chats.change_message(message, %{content: value})

  topic = topic(chat.id)
  key   = user.id
  payload = %{typing: false}
  metas =
      Presence.get_by_key(topic, key)[:metas]
      |> List.first()
      |> Map.merge(payload)

  Presence.update(self(), topic, key, metas)

  {:noreply, assign(socket, message: message)}
end
```

*注：在这里，我们可以看到我们为处理 `"typing"` 事件而写的一些明显重复的代码。这段代码已经被重构，将 Presence 交互移到了我们的 `PhatWeb.Presence` 模块中，你可以查看[这里](https://github.com/elixirschool/live-view-chat/blob/master/lib/phat_web/presence.ex)和[这里](https://github.com/elixirschool/live-view-chat/blob/master/lib/phat_web/live/chat_live_view.ex)。为了方便阅读，在这篇文章中，我让这段代码保持明确。

在这里，我们更新消息变化集，以反映用户在表单字段中输入的内容。然后，我们从 Presence 中获取用户的元数据，并更新它以设置 `typing: false`。最后，我们更新实时视图的套接字，以反映用户在消息表单字段中输入的内容。这是一个必要的步骤，以便模板在重新渲染时显示这些内容，作为 `"presence_diff"` 事件的结果。

由于我们调用了 `Presence.update/4`，Presence 进程将广播 `"presence_diff"` 事件，LiveView 进程将通过获取更新后的用户列表与新的元数据并重新渲染模板来做出响应。这种重新渲染的效果是将给定用户名称中的 `"..."` 去掉，因为模板中对 `"user.typing"` 的调用现在将评估为 `"false"`。

## 结束语

让我们回过头来回顾一下我们所做的事情。

* 通过 "普通" 的 LiveView，我们让我们的聊天能够向发起更改的用户推送实时更新。换句话说，通过聊天表单提交新消息的用户会看到这些新消息出现在页面的聊天记录中。
* 有了 PubSub，我们可以将这些新的聊天消息广播给 *所有* 订阅了聊天室主题的 LiveView 客户端，也就是特定聊天室的所有成员。
* 通过利用 Presence，我们能够跟踪并显示 "在场" 的用户列表，以及特定用户的状态（即他们当前是否在打字）。

您可以[在这里](https://github.com/elixirschool/live-view-chat)看到最终的代码(略微重构！)。

Phoenix PubSub 的灵活性使得我们可以很容易地将所有正在运行的 LiveView 进程订阅到 pub sub 服务器上的同一个主题。此外，Presence 模块与我们的应用程序的其他部分共享 pub sub 服务器的能力允许每个 Presence 进程向 LiveView 进程广播 Presence 事件。总的来说，LiveView、PubSub 和 Presence 很好地结合在一起，使我们能够用很少的手工代码构建一套强大的功能。
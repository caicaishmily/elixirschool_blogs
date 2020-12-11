---
author: Sophie DeBenedetto
author_link: https://github.com/sophiedebenedetto
categories: general
date: 2019-06-04
layout: post
title:  Using Channels with LiveView for Better UX
excerpt: >
  By pairing a custom Phoenix Channel with our LiveView, with the help of a Registry, we can respond to LiveView events with custom JavaScript on the client-side to provide better UX.
---

# 使用 LiveView 的 Channels 实现更好的用户体验

LiveView 赋予我们几乎完全用服务器端的代码来实现灵活和响应式 UX 的能力。但是，当响应式 UI 的需求超过了 LiveView 提供的功能时，会发生什么？当某一特定功能的需求让我们不得不求助于 JavaScript 时，会发生什么？在自定义 LiveView Channel 和 Registry 的帮助下，可以将自定义 JS 纳入到 LiveView 的生命周期中。继续阅读，看看我们是如何做到的。

## 问题

在 [最近的文章](https://elixirschool.com/blog/live-view-with-presence/) 中，我们构建了一个由 LiveView、PubSub 和 Presence 支持的简单聊天应用程序。我们只用了 90 行 LiveView 代码就实现了几乎所有必要的功能（用户输入新消息时的实时更新，一个可以跟踪聊天室里的用户以及谁在打字的列表！）。

但后来我们遇到了一个拦路虎。

当新的聊天信息被添加到聊天窗口时，它们 *就* 不会出现在聊天框里。

![chat message not visible](https://elixirschool.com/assets/chat-message-not-visible-636e5b63bfbbe5bc65bcdeb962b3f447c140f498877a16aa09392d4d568c0f8a.png)

---

聊天窗口需要向下滚动以容纳并显示新消息。这很容易做到，只需一两行 JavaScript：获取聊天窗口的高度，并设置相应的 `scrollTop`。

如果你熟悉 Phoenix Channels，你可能会这样做：

```javascript
channel.on("new_message", (msg) => {
  const targetNode = document.getElementsByClassName("messages")[0]
  targetNode.scrollTop = targetNode.scrollHeight
})
```

等一下！ LiveView 客户端的库只响应服务器上运行的 LiveView 进程的 _一个_ 事件 -- 即 diff 事件。这个事件没有足够的粒度来告诉我们页面上发生了 _什么_ 变化。它只是强制页面的适当部分重新渲染。

那么，我们如何让我们的 LiveView 发出一个事件，让我们的前端能够响应这个事件，从而启动我们的 `scrollTop` 调整 JS 呢？

## 解决办法

我们需要做一些事情来让它工作。

* 用自定义 channel 扩展 LiveView  socket 。
* 教会我们的 LiveView 进程向该 channel 发送消息，以便该 channel 可以将消息推送给客户端。

这里值得注意的是，自定义 LiveView channel 的责任范围应该很窄。LiveView 可以也应该处理几乎所有对 LiveView 模板的更新。这就是 LiveView 的魅力所在! 我们不需要像我们在使用 Phoenix Channels 时习惯的那样，写一套自定义的客户端函数来根据特定事件更新页面。然而，当我们需要触发一个客户端的交互，比如我们的 `scrollTop` 调整，而 LiveView 客户端并不能处理时，我们可以求助于一个自定义的 channel。

现在我们对我们要解决的问题有了基本的了解，以及我们将使用的工具来解决这个问题，让我们开始吧!
## 流程

在我们开始写代码之前，我们先一步步走完这个功能的期望代码流程。

1. 用户访问 `/chats/:id`
2. 控制器挂载 live view 并且渲染静态模板
3. 客户端连接到 Live View socket，并在这个 socket 上加入一个自定义 channel

然后...

4. 用户提交新的聊天信息，发送事件给 live view
5. 作为响应，live view 更新状态，重新渲染页面并且广播该消息给其他订阅了聊天室主题的用户
6. 其他的 live view 接收到广播，更新自己的状态重新渲染模板
7. live view 给自己关联的 channel 发送消息（这个 channel 已经加入到 live view 的 socket）
8. 这个 channel 接收到消息以后将其推送至前端
9. 前端接收到消息以后通过触发我们的 `scrollTop` JavaScript 调整页面

有大量的代码需要编写，所以我们将方法组织成以下几个部分。

I. [建立 Socket 和 Channel](#-establishing-the-socket-and-channel)  

II. [LiveView 处理事件](#handling-events-in-the-liveview)

III. [LiveView 到 Channel 的通信](#communicationg-from-the-liveview-to-the-channel)

IV. [Channel 发送消息给前端](#sending-messages-from-the-channel-to-the-front-end)

## 起步

如果你想跟随本教程，我们建议你先阅读并完成[上一篇文章](https://elixirschool.com/blog/live-view-with-presence/)中的教程。这将使你的代码进入正确的初始阶段。你也可以克隆下 [repo](https://github.com/elixirschool/live-view-chat)来获得初始代码。另外，你可以检查[完整代码](https://github.com/elixirschool/live-view-chat/tree/live-view-channel-registry)。

## 第一部分: 建立 Socket 和 Channel

为了保证 live view 进程能够在正确的时间向正确的 channel 发送消息，我们需要让 live view 与该 channel 共享一个 socket。我们先来关注一下这部分代码流程。

1. 用户访问 `/chats/:id`
2. 控制器挂载 live view 并且渲染静态模板
3. 客户端连接到 Live View socket，并在这个 socket 上加入一个自定义 channel

下面就来详细了解一下这个过程是如何进行的：

![live view mounts and renders](https://elixirschool.com/assets/live-view-mount-render-dbd2c5a4848829b86ea213a127a2cc4a8b06edc3e5167d28593fa9ad6d0a599a.png)

---

![live view socket connects](https://elixirschool.com/assets/live-view-socket-connect-a0fb82f53bf580475c831dee0f1cb9f5cb29497feb914924b33f4fc407470cce.png)

---

![live view channel joins](https://elixirschool.com/assets/live-view-channel-join-7a01d2bb0395bfdb55975eb16f71c7a93aeafcc9dd85d9e20a85510e1ae6e0ff.png)

---

让我们深入其中写一些代码吧!

### 扩展 LiveView Socket

为了定义一个与我们的 LiveView 进程共享 socket 的自定义 channel ，我们需要扩展 LiveView 库提供给我们的 LiveView socket。LiveView 还没有提供一种方法让我们以编程方式扩展这个模块，所以我们将定义我们自己的socket，并提供它所需的一切来支持我们的 LiveView 和我们的自定义 channel。

```elixir
# lib/phat_web/channels/live_socket.ex
defmodule PhatWeb.LiveSocket do
@moduledoc """
  The LiveView socket for Phoenix Endpoints.
  """
  use Phoenix.Socket

  defstruct id: nil,
            endpoint: nil,
            parent_pid: nil,
            assigns: %{},
            changed: %{},
            fingerprints: {nil, %{}},
            private: %{},
            stopped: nil,
            connected?: false

  channel "lv:*", Phoenix.LiveView.Channel
  channel "event_bus:*", PhatWeb.ChatChannel

  @doc """
  Connects the Phoenix.Socket for a LiveView client.
  """
  @impl Phoenix.Socket
  def connect(_params, socket, _connect_info) do
    {:ok, socket}
  end

  @doc """
  Identifies the Phoenix.Socket for a LiveView client.
  """
  @impl Phoenix.Socket
  def id(_socket), do: nil
end
```

除了从 LiveView 源码中复制的内容之外，我们需要添加的唯一一行代码是 channel 定义，其中我们将主题 `"event_bus:*"` 映射到我们即将定义的自定义 channel。

```elixir
channel "event_bus:*", PhatWeb.ChatChannel
```

接下来，我们将告诉应用的 `Endpoint` 模块将挂载在 `"/live"` 端点的 socket 映射到我们刚刚定义的 socket。

```elixir
# lib/phat_web/endpoint.ex
defmodule PhatWeb.Endpoint do
  use Phoenix.Endpoint, otp_app: :phat
  # socket "/live", Phoenix.LiveView.Socket
  socket "/live", PhatWeb.LiveSocket
  ...
end
```

### 自定义 Channel

现在我们准备定义我们的 `ChatChannel`:

```elixir
# lib/phat_web/channels/chat_channel.ex

defmodule PhatWeb.ChatChannel do
  use Phoenix.Channel
  def join("event_bus:" <> _chat_id, _message, socket) do
    {:ok, socket}
  end
end
```

### 连接到 Socket 和加入 Channel

随着我们 socket 和 channel 的定义，我们可以告诉前端在其连接完 LiveView socket 之后加入 channel；

```javascript
// assets/js/app.js
import LiveSocket from "phoenix_live_view"

let chatId = window.location.pathname.split("/")[2] // just a hack to get the chatId from the route, there are definitely better ways to do this!

const liveSocket = new LiveSocket("/live")
liveSocket.connect()

let channel = liveSocket.channel("event_bus:" + chatId, {})
```

现在，页面加载以后，我们将：

* 连接并启动 LiveView 创建的运行中的 socket 进程。
* 在 _相同的_ socket 之上加入一个 channel

之后，我们可以在前端写一些代码，通过改变聊天框的滚动高度来响应特定事件。

```javascript
channel.on("new_message", (msg) => {
  targetNode = document.getElementsByClassName("messages")[0]
  targetNode.scrollTop = targetNode.scrollHeight
})
```

所以，我们如何让我们的 channel 发送 `"new_message"` 事件给到前端呢？ 让我们一探究竟！
## 第二部分: 在 LiveView 中处理事件

在本节中，我们将深入了解以下部分的流程。

1. 用户提交新的聊天消息，向 live view 发送一个事件；live view 更新其状态并重新渲染模板。
2. live view 将该事件广播给订阅该聊天室主题的其他 live view 进程，然后这些进程更新自己的状态并重新渲染模板。
3. live view *向自己* 发送一条消息，指示它们反过来向它们的 "关联" channel（即在 live view 的 socket 上加入的 channel）发送消息。这确保了 live view 在告诉 channel 向前端推送消息*之前*，会完成重新渲染。

下面来仔细看看这个流程。

![live view handles event](https://elixirschool.com/assets/live-view-handle-event-07e9afbb4dcefb73c975b71944bedfc5291cede4f09e4e237660a9653a465161.png)

---

![live view broadcasts event](https://elixirschool.com/assets/live-view-broadcasts-event-531b91250f86cd558d77699b277d586b5dd6ff0b103edb29cf20f99f35d8261c.png)

---

![live view sends message to self](https://elixirschool.com/assets/live-view-send-self-030345f207cfe93066044d68ef2b64570f67634f7f62617580d3d0ab420dd992.png)

---

### 在 LiveView 中接收事件

当用户通过聊天表单提交新消息时，它将通过 socket 发送 `"new_message"` 事件到 LiveView 进程。我们的 live view 进程已经通过以下方式对该消息做出响应。

* 更新自己的状态并重新渲染模板以显示新消息。
* 将消息广播给订阅了同一主题的其他运行中的 live view 进程，以便每个人都能收到新消息和随后的重新渲染。

要想了解如何工作，请查看我们之前的[文章](https://elixirschool.com/blog/live-view-with-presence/)。在这篇文章中，我们只简单地看一下这段代码。

```elixir
# lib/phat_web/live/chat_live_view.ex

# this function fires when we receive the "new_message" event from the front-end
def handle_event("new_message", %{"message" => message_params}, socket) do
  chat = Chats.create_message(message_params)
  PhatWeb.Endpoint.broadcast(topic(chat.id), "new_message", %{chat: chat})
  {:noreply, assign(socket, chat: chat, message: Chats.change_message())}
end

# this function fires when all of the subscribing live view processes receive the broadcast from above
def handle_info(%{event: "new_message", payload: state}, socket) do
  {:noreply, assign(socket, state)}
end
```

需要注意的是，LiveView 正在向 *所有* 订阅了聊天室主题的 LiveView 进程广播消息，包括它自己。然而，LiveView 很聪明，不会重新渲染一个没有差异的页面，所以这不是一个昂贵的操作。

### 从 LiveView 发送消息至 Channel 

我们需要确保在 channel 向前端发送消息之前，页面有机会重新渲染。否则，调整 `scrollTop` 的 JavaScript 函数可能会在新消息出现在页面上之前运行，从而无法真正对聊天窗口进行调整。

这个 `handle_info/2` 函数返回 *之后*，就是我们可以确定所有 LiveView 模板被重新渲染的时间点。

```elixir
def handle_info(%{event: "new_message", payload: state}, socket) do
  {:noreply, assign(socket, state)}
end
```

So, how can we make sure each LiveView process handling this message will only send a message to the channel _after_ this function finishes working? We can use `send/2` to have the live view send a message to itself! Since a process can only do one thing at a time, the live view process will finish the the current work in the `handle_info/2`  processing the `"new_message"` event *before* acting on the message it receives from itself.

那么，我们如何确保每个处理该消息的 LiveView 进程在该函数完成工作后才会向 channel 发送消息呢？我们可以使用 `send/2` 来让 live view 给自己发送消息！因为一个进程一次只能做一件事，所以 live view 进程将在 `handle_info/2` 处理 `"new_message"` 事件 *之前* 完成当前的工作，在对它从自己那里收到的消息采取行动。

```elixir
def handle_info(%{event: "new_message", payload: state}, socket) do
  send(self(), {:send_to_event_bus, "new_message"})
  {:noreply, assign(socket, state)}
end

def handle_info({:send_to_event_bus, msg}, socket) do
  # send a message to the channel here!
  {:noreply, socket}
end
```

现在我们已经捕捉到了从 LiveView 进程向 Channel 进程发送消息的时间点。但是等一下！我们如何向一个我们不知道其 PID 的进程发送消息？LiveView 进程在其当前形式下，并不知道与它共享一个 socket 的 channel 进程。为了解决这个问题，我们需要利用一个 Registry。

## 第三部分: 从 LiveView 到 Channel 的通信

在本节中，我们将注册我们的 channel 进程，以便 live view 可以查找并向适当的 channel PID 发送消息。然后，我们将教 live view 如何执行这个查询并发送消息到正确的 channel PID。

下面是我们的目标代码流程。

1. LiveView 从控制器上挂载，并在自己的状态下存储一个 "会话UUID" 的唯一标识符；它在模板上渲染一个隐藏的元素，该元素包含以 `Phoenix.Token` 编码的会话 UUID。
2. Channel 的 socket 与此令牌相连，socket 将其存储在状态中。
3. 加入 Channel；它从它的 socket 的状态中获取会话 UUID，并在该 UUID 的键下注册它的 PID。

![live view mounts with session uuid](https://elixirschool.com/assets/live-view-mount-session-uuid-7d42d71da5cbbb018ee7355b6e886f08f7d931bb36993ad71a1544a5612061de.png)

---

![live view channel connects](https://elixirschool.com/assets/live-view-connect-channel-00f63e49d325e2f1695e04e724c104d798715cc133b090bf28b81e6eb8752f2a.png)

---

![live view channel register](https://elixirschool.com/assets/live-view-channel-register-835188bac138761e9b6311391f9258993d665c93af43174f76c14eccf2d0d5c0.png)

---

然后...

4. 当用户提交新的聊天消息时，收到消息广播的 LiveView 进程会在注册表中查找会话 UUID 下的频道 PID。
5. 然后，每个 live view 都会将信息发送到他们所查找的 PID 上。

![live view looks up channel](https://elixirschool.com/assets/live-view-lookup-send-to-channel-ef04590f8407467e90778798ff76ee7755f595561355d9b83ec960ee12e42bd7.png)

---

### 定义 Channel 注册表

我们将使用 Elixir 的原生 [Registry](https://hexdocs.pm/elixir/Registry.html) 模块实现的进程注册表来跟踪 channel PID，以便 LiveView 能够查找其相关的频道，从而向其发送消息。

*需要注意的是，Elixir 的注册表模块对分布式并不友好--如果你在一个完全不同的服务器上查找一个在一个服务器上创建的给定 PID，就不能保证它是指同一个进程。但是！由于我们的 channel 与 LiveView 进程共享一个 socket，所以可以保证 live view 和 channel 运行在同一个服务器上。

我们将告诉 Elixir 的注册表监督器，当我们的应用启动时，开始监督一个名为 `SessionRegistry` 的命名注册表。

```elixir
# application.ex

def start(_type, _args) do
    children = [
      Phat.Repo,
      PhatWeb.Endpoint,
      PhatWeb.Presence,
      {Registry, [keys: :unique, name: Registry.SessionRegistry]}
    ]

    opts = [strategy: :one_for_one, name: Phat.Supervisor]
    Supervisor.start_link(children, opts)
  end
```

我们想在 channel 加入时注册我们的 channel PID。但是，我们需要将 PID 存储在一个唯一的密钥下，以便 live view 以后可以使用它来查找。所以，我们需要创建这样一个标识符，并找到一种方法让它对 live view 和 channel 都可用。

### 共享会话 UUID

当 LiveView 第一次通过控制器挂载时，我们将创建一个唯一的标识符--会话 UUID --存储在 LiveView 的状态中。

```elixir
# lib/phat_web/controllers/chat_controller.ex

def show(conn, %{"id" => chat_id}) do
  chat = Chats.get_chat(chat_id)
  session_uuid = Ecto.UUID.generate()
  LiveView.Controller.live_render(
    conn,
    ChatLiveView,
    session: %{
      chat: chat,
      current_user: conn.assigns.current_user,
      session_uuid: session_uuid
    }
  )
end

# lib/phat_web/live/chat_live_view.ex

def mount(%{chat: chat, current_user: current_user, session_uuid: session_uuid}, socket) do
  ...
  {:ok, assign(socket,
    chat: chat,
    message: Chats.change_message(),
    current_user: current_user,
    users: Presence.list_presences(topic(chat.id)),
    username_colors: username_colors(chat),
    session_uuid: session_uuid,
    token: Phoenix.Token.sign(PhatWeb.Endpoint, "user salt", session_uuid)
  )}
end
```

在 live view 的 `mount/2` 函数中，我们将 session UUID 存储在 socket 的状态中，这样我们就可以在以后使用它来查询 channel 的 PID。我们还将会话 UUID 编码成一个有签名的 `Phoenix.Token`，这样我们就可以把它放在页面上，当我们从客户端加入 channel 时使用它。

```elixir
# lib/phat_web/templates/chat/show.html.leex

<%= tag :meta, name: "channel_token", content: @token %>
```

让我们来看看我们将如何给我们的 channel 访问这个 token。

当我们从浏览器发送 socket 连接请求时，我们触发了扩展的 Live View socket `PhatWeb.LiveSocket` 的 `connect/3` 函数。此时，我们 _没有_ 访问 Live View 进程对 socket 的表示，但我们 _有_ 访问 channel 对socket 的表示。
 
我们需要让 channel 知道会话的 UUID。因此，我们将在 socket 连接请求中包含来自页面的签名令牌，并使用 `connect/3` 在 channel 的 socket 状态中存储会话 UUID。

我们会在前端的 socket 连接请求中包含这个 token。

```javascript
// assets/js/app.js
const channelToken = document.getElementsByTagName('meta')[3].content
const liveSocket = new LiveSocket("/live", {params: {channel_token: channelToken}})
liveSocket.connect()
```

我们会让 `PhatWeb.LiveSocket.connect/3` 函数验证 token，提取会话 UUID，并将其存储在 channel socket 的状态中。

```elixir
# lib/phat_web/channels/live_socket.ex

def connect(params, socket, _connect_info) do
  case Phoenix.Token.verify(socket, "user salt", params["channel_token"], max_age: 86400) do
    {:ok, session_uuid} ->
      socket = assign(socket, :session_uuid, session_uuid)
      {:ok, socket}

    {:error, _} ->
      :error
  end
end
```

### 注册 Channel 进程

现在，当我们加入 channel 时，我们可以在 channel socket 的状态中查找 `:session_uuid`，并用它在 `SessionRegistry` 中以这个 UUID 的键注册 channel 的 PID。

```elixir
# lib/phat_web/channels/chat_channel.ex

defmodule PhatWeb.ChatChannel do
  use Phoenix.Channel

  def join("event_bus:" <> _chat_id, _message, socket) do
    Registry.register(Registry.SessionRegistry, socket.assigns.session_uuid, self())
    {:ok, socket}
  end
end
```

现在我们的注册表已经启动并运行了，我们在一个唯一的标识符（会话 UUID）下注册一个给定的 channel  PID，与 channel 共享一个套接字连接的实时视图是知道的。

我们已经准备好让 live view 向它的 channel 发送消息了!

### 给 Channel 发消息

让我们回顾一下到目前为止的 "新聊天信息" 过程。

* 用户提交 "新消息" 表单并发送 `"new_message"` 事件到 live view。
* live view 通过更新自己的 socket 状态来响应这一事件，重新渲染 _并_ 向所有订阅该聊天室主题的 live view 进程（即代表聊天室中其他用户的进程）广播 `"new_message"` 事件。
* live view 进程收到该消息广播后，会通过更新自己的状态和重新渲染来做出回应，同时也会发送一个 "新消息" 事件给所有订阅该聊天室主题的进程。他们也会向自己 `send` 一条消息，一旦完成重新渲染，他们就会进行处理。
* live view 进程对自己发送的消息做出响应，告诉自己向与自己共享 socket 的 channel 发送消息。

现在我们的 live view 有了它们所需要的东西来查找它们的关联 channel。它们在状态中存储了与 channel 用于在 `SessionRegistry` 中注册其 PID _相同的_ session UUID。因此，我们的 live view 可以查找通道的 PID，并向该 PID 发送消息。

```elixir
# lib/phat_web/live/chat_live_view.ex

# handle the broadcast of the "new_message" event from the live view that received it from the user
def handle_info(%{event: "new_message", payload: state}, socket) do
  send(self(), {:send_to_event_bus, "new_message"})
  {:noreply, assign(socket, state)}
end

# handle the message sent above, after re-rendering the template
def handle_info({:send_to_event_bus, msg}, socket = %{assigns: %{session_uuid: session_uuid}}) do
  [{_pid, channel_pid}] = Registry.lookup(Registry.SessionRegistry, session_uuid)
  send(channel_pid, msg)
  {:noreply, socket}
end
```

每个 live view 进程与在其 socket 上加入的 channel 共享一个会话 UUID。从这个意义上说，每个 live view 都有一个 "关联" channel。通过在这个会话 UUID 下注册 channel 的 PID，给定的 live view 可以查找其关联 channel 的 PID，并向该 channel 和仅向该 channel发送消息。

接下来，我们需要教会我们的 channel 响应这个消息。

## 第四部分: 从 Channel 发送消息到前端

在本节中，我们将重点介绍以下部分流程。

1. channel 接收到实景的消息，并将其推送到前端。
2. 前端接收到消息后，通过触发我们的 `scrollTop` 调整 JavaScript 来进行响应。

下面就来仔细看看。

![live view channel push](https://elixirschool.com/assets/live-view-channel-push-ffbfdb98df546141488683d3b39582473ccfa0ec12342ee270a8f9c3180eba5b.png)

---
![live view front end update](https://elixirschool.com/assets/live-view-front-end-update-d69dab82dbaebc110265742953633bc00006bb7ed4a9c0c202546dea4002f4ee.png)

---

### 从 Channel 接收消息

我们需要在 `ChatChannel` 中定义一个 `handle_info/`，这个 `handle_info/` 知道如何响应 `new_message` 消息，把它们从 socket 推送到前端。

```elixir
# channel
def handle_info("new_message", socket) do
  push(socket, msg, %{})
  {:noreply, socket}
end
```

### 前端响应消息

在前端，我们的 channel JS 已经准备好了，就等着开火了。

```javascript
// assets/js/app.js

channel.on("new_message", function() {
  const targetNode = document.getElementsByClassName("messages")[0]
  targetNode.scrollTop = targetNode.scrollHeight
})
```

现在，页面重新渲染后，channel 将接收 `"new_message"` 消息，并将其推送给正在监听该事件的客户端。客户端通过启动我们的 `scrollTop` 调整 JS 做出反应，用户会体验到一个响应式的 UI--一个自动无缝滚动的聊天窗口，以实时容纳新消息。

## 结语

我们已经看到，通过结合现有的 Phoenix 实时工具，可以超越 LiveView 的一个看似 "极限" 的地方--在这里是 Phoenix Channels。这篇文章中的工作提出了一个问题。"LiveView _应该_ 能做什么？" 用自定义的 Phoenix Channel 扩展 LiveView 是否违反了 LiveView 的 "目的"？这样的用例是否意味着我们应该摒弃 LiveView 而选择 Channel？

我认为使用 LiveView 来支持像我们的聊天应用这样的功能还是有独特的优势的。几乎所有的聊天功能都在不到 100 行的 LiveView 代码中完成。这是与所有的 Channel 后端和前端代码相对应的，否则你就会写这些代码。所以，我希望看到 LiveView 变得 _更加_ 可扩展和可配置，使其更容易结合自定义 channel 开箱即用。
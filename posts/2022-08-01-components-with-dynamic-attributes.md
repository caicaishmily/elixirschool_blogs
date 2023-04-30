%{
author: "Tracey Onim",
author_link: "https://github.com/TraceyOnim",
tags: ["LiveView", "Components", "LiveView Helpers"],
date: ~D[2022-08-01],
title: "Components with dynamic attributes",
excerpt: """
Learn how you can support dynamic attributes when using reusable components with assigns_to_attribute/2 function.
"""
}

---

我很确定你们中的大多数人已经与 liveview 组件进行了互动。你可能已经创建了组件，以便在你的 liveview 应用程序中重复使用。然而，有时你在创建可重用的组件时可能会感到困难，尤其是当你想在组件的标记中传递不同值的共同属性时。

例如，假设我有一个组件的标记，它产生了这样的东西：

```html
<div>
  <div class="column bg-green">learn dynamic attributes</div>
</div>
```

在该组件中，有一个 HTML 的 `class` 属性，分配了 `column` 和 `bg-green` 值。但是这个组件应该是可重复使用的，当我在其他地方使用时，我可能想要一个 `"黄色"` 的背景。这意味着我应该设置一个 `bg-yellow` 类。

```html
<div class="column bg-yellow"></div>
```

我将向你展示如何在一个类似于看板的应用程序中准确地构建我们所需要的可重用的组件，它可以帮助用户计划他们的每周任务。

![board plan](/assets/board-plan.png)

从上面的图片来看，我把看板分成了三栏--"房子"、"工作" 和 "学校" 栏。与这个板子互动的用户可以在每一栏中添加他们应该做的每周任务。

你还会注意到，每一列都有一张不同颜色的卡片。我希望我们在我们的 LiveView 应用程序中使用可重用的组件来实现这些卡片。看板将作为父级 live view，而卡片则是组件。

我们希望每个部分的卡片都有不同的颜色。那么，当卡片每次都应该是不同的颜色时，我们怎样才能在每一列中利用相同的可重用组件呢？

我们可以借助动态组件属性和 `assigns_to_attributes/2` 函数来实现。

我们开始吧:

## 1.创建 Board live view

```elixir
defmodule SampleWeb.BoardLive do
  use  SampleWeb, :live_view

  def mount(_params, _session, socket) do
    work_cards = [
      %{task: "deploy to production", id: "#{1}-work"},
      %{task: "code challenge", id: "#{2}-work"},
      %{task: "plan community events", id: "#{3}-work"}
    ]

    house_cards = [
      %{task: "wash my dog", id: "#{1}-house"},
      %{task: "sweep the house", id: "#{2}-house"},
      %{task: "tidy my bedroom", id: "#{3}-house"}
    ]

    school_cards = [
      %{task: "group discussion", id: "#{1}-school"},
      %{task: "submit assignment", id: "#{2}-school"},
      %{task: "work on school project", id: "#{3}-school"}
    ]

    {:ok,
     assign(socket, work_cards: work_cards, house_cards: house_cards, school_cards: school_cards)}

  end

  def render(assigns) do
    ~H"""
    <h1>Board</h1>
    <h2>Weekly Board Task </h2>
    <div class="row">
      <div class="column">
        <h3>Work</h3>
        <%= for card <- @work_cards do %>
          <.card card={card} />
        <% end %>
       </div>
       <div class="column">
         <h3>House</h3>
         <%= for card <- @house_cards do %>
           <.card card={card} />
         <% end %>
       </div>
       <div class="column">
         <h3>School</h3>
         <%= for card <- @school_cards do %>
           <.card card={card} />
         <% end %>
        </div>
      </div>
    """
end

def card(assigns) do
    ~H"""
    <div>
      <div class="column">
        <%= @card.task %>
      </div>
    </div>
    """
  end
end

```

我创建了一个 BoardLive 页面，渲染了房子、工作和学校部分的每个卡片组件。

在这里，我在 socket assigns 中对卡片结构进行迭代。对于每个卡片结构，我在 `card/1`[功能组件](https://hexdocs.pm/phoenix_live_view/0.16.0/Phoenix.Component.html)的帮助下渲染该卡片的细节。函数组件是接受一些赋值并返回一些 HEEx 标记的函数。它们对于在我们的 LiveView 应用程序中重复使用标记非常有用。

**注意：** 我没有涉及用户应该如何添加他们每周任务的细节。在这个例子中，我对任务进行了硬编码，假设用户已经添加了他们的任务。我们实际上是在显示添加到卡片上的任务。

当我们打开浏览器时，我们的周计划任务板应该看起来与此类似：

![board plan output](/assets/board-plan-output.png)

## 2.问题：重复使用具有不同属性的组件

1. 我们希望每一列的卡片都有不同的颜色。例如，"工作" 栏里的卡片应该是蓝色的，"房子" 栏里的卡片应该是绿色的，而 "学校" 栏里的卡片应该是黄色的。

2. 我们应该能够使用我们之前定义的 `card/1` 函数组件，同时仍然确保卡片在不同的列中可以有不同的颜色。

我们可以像这样解决第一个问题：

```elixir
<div class="row">
  <div class="column">
    <h3>Work</h3>
    <%= for card <- @work_cards do %>
      <div>
      <div class="column bg-blue">
        <%= card.task %>
      </div>
      </div>
    <% end %>
    </div>
    <div class="column">
      <h3>House</h3>
      <%= for card <- @house_cards do %>
        <div>
        <div class="column bg-green">
          <%= card.task %>
        </div>
      </div>
      <% end %>
    </div>
    <div class="column">
      <h3>School</h3>
      <%= for card <- @school_cards do %>
      <div>
        <div class="column bg-yellow">
        <%= card.task %>
        </div>
      </div>
      <% end %>
    </div>
</div>

```

我们已经解决了我们的第一个问题，它起作用了，但这个代码有一些缺点。就个人而言，一遍又一遍地写相同的 `<div>` 标记是多余的，这就是我们最初选择使用函数组件的原因。然而，我们的原始实现不允许我们控制每列中卡片的颜色。

我们如何能解决这个问题呢 ？

## 3. 解决方案：使用 `assigns_to_attributes/2` 的动态组件属性

幸运的是，Phoenix LiveView v0.16.0 引入了[`assigns_to_attribute/2`]（https://hexdocs.pm/phoenix_live_view/0.16.0/Phoenix.LiveView.Helpers.html#assigns_to_attributes/2）函数。

_这个函数对于将调用者的赋值转化为动态属性，同时从结果中剥离保留键非常有用_

`assigns_to_attribute/2` 将 assigns 作为第一个参数，并将 assign 的键值列表作为可选的第二个参数排除。然后它返回一个经过过滤的关键字列表，作为 HTML 属性使用。

现在我们确信可以将传递给 `card/1` 的 assigns 转换为 HTML 属性，让我们继续前进，并在我们对 `card/1` 的调用中添加 `"class"` 的 assigns。我们将给这个赋值一个 `"bg-_"` 的值。`"bg-_"` 代表带有 CSS 添加属性的背景颜色。

```elixir
<h3>Work</h3>
<%= for card <- @work_cards do %>
  <.card card={card} class={"bg-blue"}/>
<% end %>
<!--- ... -->
<h3>House</h3>
<%= for card <- @house_cards do %>
  <.card card={card} class={"bg-green"}/>
<% end %>
<!--- ... -->
<h3>School</h3>
<%= for card <- @school_cards do %>
  <.card card={card} class={"bg-yellow"}/>
<% end %>
```

检查一下 assigns ，看看它有什么内容：

```elixir
def card(assigns) do
  IO.inspect(assigns, label: "==================card component=====")

  ~H"""
    <div>
      <div class="column">
        <%= @card.task %>
      </div>
    </div>
    """
  end

```

```elixir
 output:
==================card component=====: %{
  __changed__: nil,
  card: %{id: "1-work", task: "deploy to production"},
  class: "bg-blue"
}
```

我们可以看到 assigns 包含卡片和类分配。让我们继续调用 `card/1` 中的 `assigns_to_attribute/2` 来转换我们的类赋值，以便在 `<div>` 标签属性中使用。

```elixir
def card(assigns) do
  extra = assigns_to_attributes(assigns)
  # ...
end
```

如果我们用 assigns 调用 `assigns_to_attributes/2`，它会返回一个关键词列表，如图所示：

```elixir
[
  card: %{id: "3-school", task: "work on school project"},
  class: "column bg-yellow"
]
```

我们不想把 card assign 作为一个 HTML 属性，所以我们必须把它从列表中排除，只保留 class assign。

```elixir
def card(assigns) do
  extra = assigns_to_attributes(assigns, [:card])
  assigns = assign(assigns, :extra, extra)
  ~H"""
    <div>
      <div {@extra} >
        <%= @card.task %>
      </div>
    </div>
    """
end
```

在这里，我使用 [assign/2](https://hexdocs.pm/phoenix_live_view/0.16.0/Phoenix.LiveView.html#assign/2) 函数，用 `extra` 中包含的 HTML 属性更新了我们的 assigns。因此，我们可以使用 `@extra` 赋值在 `<div>` 标签上输出 HTML 属性。

**注意：** `class` 分配还应该包含 "column" 类，以及 `"bg-*"` 类，如下所示。

```elixir
<.card card={card} class={"column bg-green"}/>
```

当你在浏览器中检查时，这就是我们渲染出来标记的样子：

```
<div>
  <div class="column bg-green">
    wash my dog
  </div>
</div>
```

![board plan output2](/assets/board-plan-output2.png)

## 结语

到目前为止，我们已经看到了如何使用 `assign_to_attributes/2` 在组件中支持动态属性。当你想在你的应用中创建可重用的组件时，这个函数很有用，因为它可以让我们控制在组件的标记中传递的 HTML 属性。

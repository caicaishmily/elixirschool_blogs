---
author: Meryl Dakin
author_link: https://github.com/meryldakin
categories: general
date: 2019-01-15
layout: post
title:  Connecting Elixir to Kafka with Kaffe
excerpt: >
  A codealong to help connect Kafka to your Elixir project with the wrapper Kaffe.
---

# Elixir Kaffe Codealong

如果我们要在 Elixir 项目中使用流行的消息传递系统 Kafka，我们可以选择几种包装器。此博客文章介绍如何集成其中一个，[Kaffe](https://github.com/spreedly/kaffe)，该资源不多，因此很难进行故障排除。

在此代码中，我们将构建一个简单的 Elixir 应用程序，并使用 Kaffe 将其连接到本地运行的 Kafka 服务器。 稍后我们将介绍几个变体，以连接 docker 化的 Kafka 服务器或 umbrella Elixir 应用程序。

这篇文章假设你已经拥有 Elixir 的基本知识，而没有 Kafka 或 Kaffe 的知识。 这是完整项目的地址：[Elixir Kaffe Codealong](https://github.com/elixirschool/elixir_kaffe_codealong)。

## Kafka 是啥, 简单点?
Kafka 是一个消息传递系统。 它实际上完成了三件事：

1. 接收来自应用程序的消息
2. 保持这些消息的接收顺序
3. 允许其他应用程序按顺序读取这些消息

*一个使用 Kafka 的情境:*

假设我们要为用户保留活动日志。 每当用户在您的网站上触发事件（登录，搜索，单击 banner 等）时，您都希望记录该活动。 您还希望允许多个服务访问此活动日志，例如市场营销跟踪器，用户数据聚合器，当然还有您网站的前端应用程序。 我们可以将每个活动发送到 Kafka，而不是将每个活动都保存到您自己的数据库中，并允许所有这些应用程序仅从其中读取所需内容。

这是基本工作流：

![Kafka Flow 示例](https://elixirschool.com/assets/kafka-flow-example-54e132a09ce085f3d3dcfb48120bd04b1b470935b807d1cfe0fdfc35d4c9c9fa.png)

从 Kafka 读取的三个服务将仅获取它们所需的数据。 例如，第一个服务只能从 `banner_click` topic 中读取，而最后一个只能从 `search_term` 中读取。 关心活跃用户的第二项服务可能会从这两个 topic 中读取信息，以捕获所有站点活动。

## Kafka 基本术语

在开始进入代码之前，让我们先澄清一些常见的 Kafka 术语，这些术语在您学习有关此服务的更多知识时会遇到：

- **消费者：** 从 Kafka 接收邮件
- **生产者：** 向 Kafka 发送消息
- **主题：** 一种组织消息并允许消费者仅订阅他们想要接收的消息的方法
- **分区：** 允许将主题在多台计算机之间拆分并保留相同的数据，以便一次有多个消费者可以从一个主题中读取内容
- **领导者/副本：** 这些是分区的类型。 有一个领导者和多个副本。 领导者确保副本具有相同和最新的数据。 如果领导者失败，则副本将接管领导者。
- **偏移量：** 消息的唯一标识符，可在 Kafka 中保持其顺序
## 代码: 基础的 elixir 应用 & 本地运行的 Kafka 
### 设置 Kafka 服务器
请遵循 Apache Kafka 的[快速开始说明](http://kafka.apache.org/documentation/#quickstart) 的前两个步骤：

1. [下载代码](https://www.apache.org/dyn/closer.cgi?path=/kafka/2.1.0/kafka_2.11-2.1.0.tgz)

2. 启动服务器  

    Zookeeper（一项为 Kafka 处理某些协调和状态管理的服务）  

    `bin / zookeeper-server-start.sh config / zookeeper.properties`  

    Kafka  

    `bin / kafka-server-start.sh config / server.properties`

### 设置 Elixir 应用

* **1. 创建一个新项目**

  `mix new elixir_kaffe_codealong`

* **2. 配置 kaffe**

  - **2.a:** 在 `mix.exs` 中添加 `:kaffe` 至额外的应用列表中:

    ```elixir
    def application do
      [
        extra_applications: [:logger, :kaffe]
      ]
    end
    ```

  - **2.b:** 添加 kaffe 到依赖列表中:

      ```elixir
      defp deps do
        [
          {:kaffe, "~> 1.9"}
        ]
      end  
      ```

  - **2.c:** 在命令行终端运行 `mix deps.get` 以锁定新的依赖.

* **3. 配置 producer**

  在 `config/config.exs` 中添加:

    ```elixir
    config :kaffe,
      producer: [
        endpoints: [localhost: 9092],
        # endpoints references [hostname: port]. Kafka is configured to run on port 9092.
        # In this example, the hostname is localhost because we've started the Kafka server
        # straight from our machine. However, if the server is dockerized, the hostname will
        # be called whatever is specified by that container (usually "kafka")
        topics: ["our_topic", "another_topic"], # add a list of topics you plan to produce messages to
      ]
    ```

* **4. 配置 consumer**

    - **4.a:** 在 `/lib/application.ex` 中添加以下代码:

      ```elixir
      defmodule ElixirKaffeCodealong.Application do
        use Application # read more about Elixir's Application module here: https://hexdocs.pm/elixir/Application.html

        def start(_type, args) do
          import Supervisor.Spec
          children = [
            worker(Kaffe.Consumer, []) # calls to start Kaffe's Consumer module
          ]
          opts = [strategy: :one_for_one, name: ExampleConsumer.Supervisor]
          Supervisor.start_link(children, opts)
        end
      end
      ```

    - **4.b:** 回到 `mix.exs`, 在 application 函数中新增一个条目：
        
      ```elixir
      def application do
        [
          extra_applications: [:logger, :kaffe],
          mod: {ElixirKaffeCodealong.Application, []}
          # now that we're using the Application module, this is where we'll tell it to start.
          # We use the keyword `mod` with applications that start a supervision tree,
          # which we configured when adding our Kaffe.Consumer to Application above.
        ]
      end
      ```
  - **4.c:** 在 `/lib/example_consumer.ex` 添加一个 consumer 模块接收来自 Kafka 的消息

    ```elixir
    defmodule ExampleConsumer do
      # function to accept Kafka messaged MUST be named "handle_message"
      # MUST accept arguments structured as shown here
      # MUST return :ok
      # Can do anything else within the function with the incoming message

      def handle_message(%{key: key, value: value} = message) do
        IO.inspect(message)
        IO.puts("#{key}: #{value}")
        :ok
      end
    end
    ```
  - **4.d:** 在 `/config/config.exs` 中配置 consumer 模块

    ```elixir
    config :kaffe,
      consumer: [
        endpoints: [localhost: 9092],               
        topics: ["our_topic", "another_topic"],     # the topic(s) that will be consumed
        consumer_group: "example-consumer-group",   # the consumer group for tracking offsets in Kafka
        message_handler: ExampleConsumer,           # the module that will process messages
      ]
    ```

* **5. 添加生产者模块（可选，也可以从控制台调用 Kaffe）**

  我们将在自己的 ExampleProducer 方法中包装 Kaffe 提供的功能。 直接调用 Kaffe 也可以； `produce_sync` 函数最终将我们的消息发送给 Kafka。

  在 `/lib/example_producer.ex` 中增加以下代码:

    ```elixir
    defmodule ExampleProducer do
      def send_my_message({key, value}, topic) do
        Kaffe.Producer.produce_sync(topic, [{key, value}])
      end

      def send_my_message(key, value) do
        Kaffe.Producer.produce_sync(key, value)
      end

      def send_my_message(value) do
        Kaffe.Producer.produce_sync("sample_key", value)
      end
    end
    ```

* **6. 在控制台中发送和接收消息！**

  现在，我们已经完成了所有配置，可以使用我们创建的模块通过 Kafka 发送和读取消息！

  1. 我们将调用生产者将消息发送到 Kafka 服务器。
  2. Kafka 服务器收到消息。
  3. 我们配置为订阅名为 `another_topic` 的主题的使用者，将收到我们发送的消息并将其打印到控制台。

  使用 `iex -S mix` 启动一个 elixir 交互式命令行，然后调用以下命令：

  ```sh
  iex> ExampleProducer.send_my_message({"Metamorphosis", "Franz Kafka"}, "another_topic")
  ...>[debug] event#produce_list topic=another_topic
  ...>[debug] event#produce_list_to_topic topic=another_topic partition=0
  ...>:ok
  iex> %{
  ...> attributes: 0,
  ...> crc: 2125760860, # will vary
  ...> key: "Metamorphosis",
  ...> magic_byte: 1,
  ...> offset: 1, # will vary
  ...> partition: 0,
  ...> topic: "another_topic",
  ...> ts: 1546634470702, # will vary
  ...> ts_type: :create,
  ...> value: "Franz Kafka"
  ...> }
  ...> Metamorphosis: Franz Kafka
  ```

## 变体: Docker & Umbrella Apps

- 如果您是从 docker 容器（在实际应用中最常见）运行 Kafka，则将在配置文件中使用该主机名，而不是 `localhost`

- 在 umbrella app 中，您将在运行它的子应用程序中配置 Kaffe。 如果您的应用程序被环境分隔开，则可以通过将其构造为子级来启动使用者，如下所示：

```elixir
    children = case args do
      [env: :prod] -> [worker(Kaffe.Consumer, [])]
      [env: :test] -> []
      [env: :dev]  -> [worker(Kaffe.Consumer, [])]
      [_] -> []
    end
```

## 排除错误

- **没有 leader 错误**

  ```
  ** (MatchError) no match of right hand side value: {:error, :LeaderNotAvailable}
  ```

  解决方案：再试一次。 只需一分钟即可预热。

- **无效的 Topic 错误**

  ```
  ** (MatchError) no match of right hand side value: {:error, :InvalidTopicException}
  ```

  解决方案：您的 topic 中不应有空格，对吗？

## 结束语

这篇文章应该已经为您提供了基本设置，让您可以自己开始探索更多功能，但是使用 Kaffe 可以做更多的事情，因此请检查发送多条消息，使用 consumer 组等。如果您遇到其他您已解决的错误 ，请通过 [在此处创建问题](https://github.com/elixirschool/elixirschool/issues) 告诉我们。

## 资源
- [Elixir Kaffe Codealong](https://github.com/elixirschool/elixir_kaffe_codealong)
- [Kaffe on Github](https://github.com/spreedly/kaffe)
- [Kaffe on Hexdocs](https://hexdocs.pm/kaffe/Kaffe.html#content)
- [Kafka quickstart](http://kafka.apache.org/documentation/#quickstart)
- [Kafka in a Nutshell](https://sookocheff.com/post/kafka/kafka-in-a-nutshell/)
- [Application module in Elixir](https://hexdocs.pm/elixir/Application.html)

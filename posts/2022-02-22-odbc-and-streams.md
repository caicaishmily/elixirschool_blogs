%{
author: "Bobby Grayson",
author_link: "https://github.com/notactuallytreyanastasio",
tags: [],
date: ~D[2022-02-22],
title: "`:odbc` and Efficient Querying With Streams",
excerpt: """
Learn how to use Erlang's built in `:odbc` interface to query using streams effectively
"""
}

---

# `:odbc` 和使用流的高效查询

Erlang 在原生层面提供了一个[ODBC](https://docs.microsoft.com/en-us/sql/odbc/microsoft-open-database-connectivity-odbc?view=sql-server-ver15) 的接口。

这可以用来与各种不同的数据库通信。

其中特别有用的是[Snowflake](https://en.wikipedia.org/wiki/Snowflake_Inc.#cite_note-raises-5)。

这是一个伟大的通用数据仓库。

由于它是一个仓库，可以想象，查询会变得相当大。

如果你正在构建一个接口，为了保持低内存，你可能想在 `:odbc` 周围建立一个包装器，它可以以一种懒惰的方式与它对话，用来拉数据并将其移动到其他来源，比如分析，或者为另一个系统服务。

这篇文章将简单介绍如何连接以及流式处理数据的方法，如果你需要为这样的东西写一个客户端 API，这可以作为一个跳板。

## 让东西先运行起来

由于 `:odbc` 包含在 Erlang 中，我们不需要包含任何依赖。我们可以创建一个新的监督项目并立即开始工作

```shell
mix new my_etl_odbc_app --sup
cd my_etl_odbc_app
mix compile
```

现在，让我们为连接做一个配置

```shell
mkdir config
touch config/config.exs
```

然后打开这个文件：

```elixir
import Config

config :my_etl_odbc_app,
  connection: [
    server: "some.server.path",
    uid: "your_user",
    pwd: "your_password",
    role: "your_role",
    warehouse: "your_warehouse_name"
  ]
```

现在，我们可以直接把我们的代码扔到 `lib/my_etl_odbc_app.ex`.

我们将从一个简单的例子开始：让一些数据流到一个文件中。这是一个微不足道的例子，但我们可能想运行一个仓库的分析查询，然后将其持久化到 S3 或其他一些数据存储中，这并不是不可想象的。

```elixir
defmodule MyEtlOdbcApp do
  @query """
  -- obviously, fill in your own query here
  SELECT id, name, owner_id, description FROM thing_stuffs;
  """

  def run(query) do
    temp_file_stream = File.stream!("/var/tmp/#{DateTime.utc_now()}", [:utf8])
    {:ok, pid} = connect(connection_args)
    {odbc_conn_pid, count} = query_warehouse(pid, @query)

    row_stream =
      Stream.flat_map(1..count, fn _n ->
        odbc_pid
        |> :odbc.next()
        |> process_results([])
      end)

    row_stream
    |> Stream.map(fn row ->
      row_io = Jason.encode_to_iodata!(row)
      [row_io, ?\n]
    end)
    |> Stream.into(temp_file_stream)
    |> Stream.run()
  end

  defp query_warehouse(pid, query) do
    cl_query = to_char_list(query)
    {:ok, count} = :odbc.select_count(pid, cl_query)
    {pid, count}
  end

  defp connect(connection_args) do
    driver = Application.get_env(:my_etl_odbc_app, :connection)
    connection_args = [{:driver, driver} | connection_args]

    conn_str =
      connection_args
      |> Enum.reduce("", fn {key, value}, acc -> acc <> "#{key}=#{value};" end)
      |> to_charlist()

    {:ok, pid} = :odbc.connect(conn_str, [])
  end

  defp process_results(data, opts) when is_list(data) do
    Enum.map(data, &process_results(&1, opts))
  end

  defp process_results({:selected, headers, rows}, opts) do
    bin_headers =
      headers
      |> Enum.map(fn header -> header |> to_string() end)
      |> Enum.with_index()

    Enum.map(rows, fn row ->
      Enum.reduce(bin_headers, %{}, fn {col, index}, map ->
        data = elem(row, index)
        Map.put(map, col, data)
      end)
    end)
  end
  defp process_results({:updated, _} = results, _opts), do: results
end
```

## 破解代码

这里有很多移动的部分，对于 `:odbc` 来说是很特别的，而且它依赖于 charlists，而不是像大多数高级的 Elixir API 那样依赖于二进制文件。

让我们逐个来看一下。

```elixir
  def run(query) do
    temp_file_stream = File.stream!("/var/tmp/#{DateTime.utc_now()}", [:utf8])
    {:ok, pid} = connect(connection_args)
    # ...
```

在这里，我们通过创建一个临时文件流开始工作，并获得一个连接来工作。

这里是 `connect/1` 代码:

```elixir
  defp connect(connection_args) do
    driver = Application.get_env(:my_etl_odbc_app, :connection)
    connection_args = [{:driver, driver} | connection_args]

    conn_str =
      connection_args
      |> Enum.reduce("", fn {key, value}, acc -> acc <> "#{key}=#{value};" end)
      |> to_charlist()

    {:ok, pid} = :odbc.connect(conn_str, [])
  end
```

这里我们通过使用先前的配置，以驱动程序的首选格式建立连接，并得到一个我们可以开始工作的连接 pid。

回到 `run` 函数:

```elixir
    row_stream =
      Stream.flat_map(1..count, fn _n ->
        odbc_pid
        |> :odbc.next()
        |> process_results([])
      end)
```

`:odbc.next/1` 是对结果进行迭代的最简单方法。

你也可以调用 `:odbc.select/2` 并处理页面的跳转。然而，如果你想保持最小的内存使用，这个实现是相当有效的。在我们的生产系统中，在查询和处理 1.8M 行 25 列的数据时，它的内存用量激增了约 400MB。在内存中进行这些操作时，占用了大约 25GB 的内存。分页实际上比 `next/1` 占用更多的内存！所以我们坚持采用这种方式。

`process_results/2` 是我们现在工作的主要内容，以将数据处理成更有用的东西。

让我们来看看这个：

```elixir
  defp process_results(data, opts) when is_list(data) do
    Enum.map(data, &process_results(&1, opts))
  end

  defp process_results({:selected, headers, rows}, opts) do
    bin_headers =
      headers
      |> Enum.map(fn header -> header |> to_string() end)
      |> Enum.with_index()

    Enum.map(rows, fn row ->
      Enum.reduce(bin_headers, %{}, fn {col, index}, map ->
        data = elem(row, index)
        Map.put(map, col, data)
      end)
    end)
  end
  defp process_results({:updated, _} = results, _opts), do: results
```

在这里，我们以 `next/1` 返回的格式进行迭代，这是一个 `:selected` 的元组，你的头文件，以及该行的数据。在将头文件变成二进制文件而不是 charlists 之后，我们将其还原为 map。

一旦这个完成，我们用一个漂亮的 map 将所有数据返回。

在这种情况下，让我们把它写成 JSON-LD 格式。

最后一个子句抓住了查询的结束，并允许它完成。

我们在 `mix.exs` 中加入 `jason` .

```elixir
  defp deps do
    [
      {:jason, "~> 1.3.0"},
    ]
  end
```

现在，我们可以继续进行我们的运行函数的其余部分：

```elixir
    row_stream
    |> Stream.map(fn row ->
      row_io = Jason.encode_to_iodata!(row)
      [row_io, ?\n]
    end)
    |> Stream.into(temp_file_stream)
    |> Stream.run()
```

现在，我们使我们的行流和我们的查询流一起工作，写到文件中去!

一旦完成，你可以在我们硬编码的路径上找到它。

## 结语

在一个真实的系统中，由于 `:odbc` 为每个连接创建了一个进程，你会希望使用像[Poolboy]（https://hex.pm/packages/poolboy）这样的工具。
查看[关于 Poolboy 的帖子](https://elixirschool.com/en/lessons/misc/poolboy)，看看你如何将它整合到这个查询接口中，以避免你可能与之对话的任何数据库过载。

如果你对实现这个分页感兴趣，我建议你研究一下 `Stream.resource/3`。`Stream.resource/3` 将允许你使用一个 0 累加器来建立你的偏移量，以便在你的查询接口中进行插值。

我希望这篇文章能帮助那些必须深入研究 `:odbc` 模块以有效地处理数据的人。

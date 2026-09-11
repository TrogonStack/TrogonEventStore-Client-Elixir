# TrogonEventStore Elixir client

![CI](https://github.com/TrogonStack/TrogonEventStore-Client-Elixir/actions/workflows/ci.yml/badge.svg)
[![Integration](https://github.com/TrogonStack/TrogonEventStore-Client-Elixir/actions/workflows/integration.yml/badge.svg)](https://github.com/TrogonStack/TrogonEventStore-Client-Elixir/actions/workflows/integration.yml)
[![hex.pm version](https://img.shields.io/hexpm/v/spear.svg)](https://hex.pm/packages/spear)
[![hex.pm license](https://img.shields.io/hexpm/l/spear.svg)](https://github.com/TrogonStack/TrogonEventStore-Client-Elixir/blob/main/LICENSE)
[![Last Updated](https://img.shields.io/github/last-commit/TrogonStack/TrogonEventStore-Client-Elixir.svg)](https://github.com/TrogonStack/TrogonEventStore-Client-Elixir/commits/main)

An Elixir gRPC client for TrogonEventStore backed by Mint.

The client currently preserves the Spear API and package identity while
compatibility is established against TrogonEventStore.

This project contains modifications by Straw Hat, LLC to software received
under the Apache License 2.0. See [LICENSE](LICENSE) for the license terms.

## FAQ

**What's EventStoreDB?**

[EventStoreDB](https://www.eventstore.com/) is a database designed for
[Event Sourcing](https://microservices.io/patterns/data/event-sourcing.html).
Instead of tables with rows and columns, EventStoreDB stores information in
immutable _events_ which are appended to _streams_.

**Why the name "spear"?**

1. best gum flavor
1. obligatory programmer reference to ancient greek, roman, or egyptian history
1. sounds cool :sunglasses:

**Backed by... Mint?**

[`elixir-mint/mint`](https://github.com/elixir-mint/mint) is a functional
HTTP client which supports HTTP2.

gRPC is pretty thin protocol built on top of HTTP/2. Practically speaking,
gRPC just adds some well-known headers and a message format that allows
messages to not be aligned with HTTP2 DATA frames.  It's relatively trivial
to implement gRPC with a nice HTTP2 library like mint :slightly_smiling_face:.

**Why not [`elixir-grpc/grpc`](https://github.com/elixir-grpc/grpc)?**

That project historically depended on
[`:gun`](https://github.com/ninenines/gun), while this client only needs a
Mint-backed client implementation.

> [!NOTE]
> This is not true anymore: `elixir-grpc/grpc` has supported a Mint adapter since v0.6.0
> and the client and server are not split into separate packages.

**Does TLS work?**

Yep! As of v0.1.3, custom and public CAs may be used for encrypted connections.

**Does this work with EventStore <20?**

Sadly no. This library only provides a gRPC client which showed up in
EventStoreDB 20+. If you're looking for a similarly fashioned TCP client,
[`exponentially/extreme`](https://github.com/exponentially/extreme) extensively
provides an older transport option. Spear and Extreme have compatible
dependencies and similar styles of making connections.

**How many dependencies are we talking here?**

Spear's reliance on Mint and `:gpb` give it a somewhat small dependency tree:

```
$ mix deps.tree --only prod
spear
├── connection ~> 1.0 (Hex package)
├── event_store_db_gpb_protobufs ~> 2.0 (Hex package)
│   └── gpb ~> 4.0 (Hex package)
├── gpb ~> 4.0 (Hex package)
├── jason >= 0.0.0 (Hex package)
└── mint ~> 1.0 (Hex package)
```

(And `jason` is optional!)

**How close is this to being able to be used?**

Compatibility with TrogonEventStore is still being established. The manual
integration workflow tracks that work independently from server-free CI.

## Installation

Add `:spear` to your mix dependencies in `mix.exs`

```elixir
def deps do
  [
    {:spear, "~> 1.0"},
    # If you want to encode events as JSON, :jason is a great library for
    # encoding and decoding and works out-of-the-box with spear.
    # Any JSON (de)serializer should work though, so you don't *need* to add
    # :jason to your dependencies.
    {:jason, "~> 1.0"},
    # If you're connecting to an EventStoreDB with a TLS certificate signed
    # by a public Certificate Authority (CA), include :castore
    {:castore, ">= 0.0.0"}
  ]
end
```

## Usage

<details><summary>Making a connection...</summary>
<br>

Familiar with [`Ecto.Repo`](https://hexdocs.pm/ecto/Ecto.Repo.html)? It lets
you write a database connection like a module

```elixir
# note this is for illustration purposes and NOT directly related to Spear
# lib/my_app/repo.ex
defmodule MyApp.Repo do
  use Ecto.Repo,
    otp_app: :my_app,
    adapter: Ecto.Adapters.Postgres
end
```

and then configure it with application-config (`config/*.exs`)

```elixir
# note this is for illustration purposes and NOT directly related to Spear
# config/config.exs
config :my_app, MyApp.Repo,
  url: "ecto://postgres:postgres@localhost/my_database"
```

Spear lets you do the same with a connection to the EventStoreDB:

```elixir
# lib/my_app/event_store_db_client.ex
defmodule MyApp.EventStoreDbClient do
  use Spear.Client,
    otp_app: :my_app
end
```

and configure it,

```elixir
# config/config.exs
config :my_app, MyApp.EventStoreDbClient,
  connection_string: "esdb://localhost:2113"
```

add it to your application's supervision tree in `lib/my_app/application.ex`

```elixir
# lib/my_app/application.ex
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      MyApp.EventStoreDbClient
    ]
    
    Supervisor.start_link(children, strategy: :one_for_one, name: MyApp.Supervisor)
  end
end
```

</details>

<details><summary>Or connecting in IEx...</summary>
<br>

A `Spear.Connection` is just a regular ole' GenServer with a default of pulling
configuration from application-config. You can start a `Spear.Connection`
like any other process, even in IEx! Plus you can provide the configuration
straight to the `Spear.Connection.start_link/1` function.

Let's use the new `Mix.install/1` function from Elixir 1.12 to try out
Spear. Say that you have an EventStoreDB instance running locally with the
`--insecure` option.

```elixir
iex> Mix.install([:spear, :jason])
# a bunch of installation text here
:ok
iex> {:ok, conn} = Spear.Connection.start_link(connection_string: "esdb://localhost:2113")
{:ok, #PID<0.1518.0>}
```

And we're up and running reading and writing events!

</details>

<details open><summary>Reading and writing streams...</summary>
<br>

Now that we have a connection process (we'll call it `conn`), let's read and
write some events!

```elixir
iex> event = Spear.Event.new("IExAndSpear", %{"hello" => "world"})
%Spear.Event{
  body: %{"hello" => "world"},
  id: "9e3a8bcf-0c22-4a38-85c6-2054a0342ec8",
  metadata: %{content_type: "application/json", custom_metadata: ""},
  type: "IExAndSpear"
}
iex> [event] |> Spear.append(conn, "MySpearDemo")
:ok
iex> Spear.stream!(conn, "MySpearDemo")
#Stream<[
  enum: #Function<62.80860365/2 in Stream.unfold/2>,
  funs: [#Function<48.80860365/1 in Stream.map/2>]
]>
iex> Spear.stream!(conn, "MySpearDemo") |> Enum.to_list()
[
  %Spear.Event{
    body: %{"hello" => "world"},
    id: "9e3a8bcf-0c22-4a38-85c6-2054a0342ec8",
    metadata: %{
      commit_position: 18446744073709551615,
      content_type: "application/json",
      created: ~U[2021-04-12 20:05:17.757215Z],
      custom_metadata: "",
      prepare_position: 18446744073709551615,
      stream_name: "MySpearDemo",
      stream_revision: 0
    },
    type: "IExAndSpear"
  }
]
```

Spear uses Elixir `Stream`s to provide a flexible and efficient interface
for EventStoreDB streams.

```elixir
iex> Stream.repeatedly(fn -> Spear.Event.new("TinyEvent", %{}) end)
#Function<51.80860365/2 in Stream.repeatedly/1>
iex> Stream.repeatedly(fn -> Spear.Event.new("TinyEvent", %{}) end) |> Stream.take(10_000) |> Spear.append(conn, "LongStream")
:ok
iex> Spear.stream!(conn, "LongStream")
#Stream<[
  enum: #Function<62.80860365/2 in Stream.unfold/2>,
  funs: [#Function<48.80860365/1 in Stream.map/2>]
]>
iex> Spear.stream!(conn, "LongStream") |> Enum.count
10000
```

</details>

And that's the basics! Check out the [Spear documentation on
Hex](https://hexdocs.pm/spear/Spear.html).

---
layout: page
title: GenStage
category: specifics
order: 
lang: en
---

In this lesson we're going to take a closer look at the GenStage, what role it serves, and how we can leverage it in our applications. 

{% include toc.html %}

## Introduction

So what is GenStage?  From the official documentation, it is a "specification and computational flow for Elixir", but what does that mean to us?  

What it means is that GenStage provides a way for us to define a pipeline of work to be carried out by independent steps (or stages) in a separate processes; if you've worked with pipelines before then some of these concepts should be familiar.

We could go into the technical and theoretical implications of this, but instead lets try a pragmatic approach to really just get it to work.

First, Let's imagine we have a server that constantly emits numbers.
It starts at the state of the number we give it, then counts up in one from htere onward.
This is what we would call our producer.
Each time it emits a number, this is an event, and we want to handle it with a consumer.
A consumer simply takes what a producer emits and does something to it.
In our case, we will display the count.
There is a lot more to GenStage at a technical and applied level, but we will build up on the specifics and definitions further in later lessons, for now we just want a running example we can build up on.

## Getting Started: A Sample GenStage Project
We'll begin by generating a simple project that has a supervision tree:

```shell
$ mix new genstage_example --sup
$ cd genstage_example
```

Let's set up some basic things for the future of our application.
Since GenStage is generally used as a transformation pipeline, lets imagine we have a background worker of some sort.
This worker will need to persist whatever it changes, so we should get a database set up, but we can worry about that in a later lesson.
To start, all we need to do is add `gen_stage` to our deps.

```elixir
. . .
  defp deps do
    [
      {:gen_stage, "~> 0.7},
    ]
  end
. . .
```

Now, we should fetch our dependencies and compile before we start setup:

```shell
$ mix do deps.get, compile
```

Lets build a producer, our simple beginning building block to help us utilize this new tool!

## Building A Producer
To get started what we want to do is create a producer that emits a constant stream of events for our consumer to handle.
This is quite simple with the rudimentary example of a counter.
Let's create a namespaced directory under `lib` and then go from there, this way our module naming matches our names of the modules themselves:

```shell
$ mkdir lib/genstage_example
$ touch lib/genstage_example/producer.ex
```

Now we can add the code:

```elixir
defmodule GenstageExample.Producer do
  alias Experimental.GenStage
  use GenStage

  def start_link do
    GenStage.start_link(__MODULE__, 0, name: __MODULE__)
                                       # naming allows us to handle failure
  end

  def init(counter) do
    {:producer, counter}
  end

  def handle_demand(demand, state) do
    events = Enum.to_list(state..state + demand - 1)
    {:noreply, events, (state + demand)}
  end
end
```
[working commit](https://github.com/ybur-yug/genstage_example/commit/57418160a1e5cbb408e32184d913a12e207166f8)

Let's break this down line by line.
To begin with, we have our initial declarations:

```elixir
defmodule GenstageExample.Producer do
  alias Experimental.GenStage
  use GenStage
. . .
```

What this does is a couple simple things.
First, we declare our module, and soon after we alias `Experimental.GenStage`.
This is simply because we will be calling it more than once and makes it more convenient.
The `use GenStage` line is much akin to `use GenServer`.
This line allows us to import the default behaviour and functions to save us from a large amount of boilerplate.

If we go further, we see the first two primary functions for startup:

```elixir
. . .
  def start_link do
    GenStage.start_link(__MODULE__, :the_state_doesnt_matter)
  end

  def init(counter) do
    {:producer, counter}
  end
. . .
```

These two functions offer a very simple start.
First, we have our standard `start_link/0` function.
Inside here, we use`GenStage.start_link/` beginning with our argument `__MODULE__`, which will give it the name of our current module.
Next, we set a state, which is arbitrary in this case, and can be any value.
The `__MODULE__` argument is used for name registration like any other module.
The second argument is the arguments, which in this case are meaningless as we do not care about it.
In `init/1` we simply set the counter as our state, and label ourselves as a producer.

Finally, we have where the real meat of our code's functionality is:

```elixir
. . .
  def handle_demand(demand, state) do
    events = Enum.to_list(state..state + demand - 1)
    {:noreply, events, (state + demand)}
  end
. . .
```

`handle_demand/2` must be implemented by all producer type modules that utilize GenStage.
In this case, we are simply sending out an incrementing counter.
This might not make a ton of sense until we build our consumer, so lets move on to that now.

## Building A Consumer
The consumer will handle the events that are broadcasted out by our producer.
For now, we wont worry about things like broadcast strategies, or what the internals are truly doing.
We'll start by showing all the code and then break it down.

```elixir
defmodule GenstageExample.Consumer do
  alias Experimental.GenStage
  use GenStage

  def start_link do
    GenStage.start_link(__MODULE__, :state_doesnt_matter)
  end

  def init(state) do
    {:consumer, state, subscribe_to: [GenstageExample.Producer]}
  end

  def handle_events(events, _from, state) do
    for event <- events do
      IO.inspect {self(), event, state}
    end
    {:noreply, [], state}
  end
end
```
[working commit](https://github.com/ybur-yug/genstage_example/commit/472a136229e5926294babe10a5bab307ed7c1f54)

To start, let's look at the beginning functions just like last time:

```elixir
defmodule GenstageExample.Consumer do
  alias Experimental.GenStage
  use GenStage

  def start_link do
    GenStage.start_link(__MODULE__, :state_doesnt_matter)
  end

  def init(state) do
    {:consumer, state, subscribe_to: [GenstageExample.Producer]}
  end
. . .
```

To begin, much like in our producer, we set up our `start_link/0` and `init/1` functions.
In `start_link` we simple register the module name like last time, and set a state.
The state is arbitrary for the consumer, and can be literally whatever we please, in this case `:state_doesnt_matter`.

In `init/1` we simply take the state and set up our expected tuple.
It expected use to register our `:consumer` atom first, then the state given.
Our `subscribe_to` clause is optional.
What this does is subscribe us to our producer module.
The reason for this is if something crashes, it will simply attempt to re-subscribe and then resume receiving emitted events.

```elixir
. . .
  def handle_events(events, _from, state) do
    for event <- events do
      IO.inspect {self(), event, state}
    end
    {:noreply, [], state}
  end
. . .
```

This is the meat of our consumer, `handle_events/3`.
`handle_events/3` must be implemented by any `consumer` or `producer_consumer` type of GenStage module.
What this does for us is quite simple.
We take a list of events, and iterate through these.
From there, we inspect the `pid` of our consumer, the event (in this case the current count), and the state.
After that, we don't reply because we are a consumer and do not handle anything, and we don't emit events to the second argument is empty, then we simply pass on the state.

## Wiring It Together
To get all of this to work we only have to make one simple change.
Open up `lib/genstage_example.ex` and we can add them as workers and they will automatically start with our application:

```elixir
. . .
    children = [
      worker(GenstageExample.Producer, []),
      worker(GenstageExample.Consumer, []),
    ]
. . .
```

With this, if things are all correct, we can run IEx and we should see everything working:

```elixir
iex(1)> {#PID<0.205.0>, 0, :state_doesnt_matter}
{#PID<0.205.0>, 1, :state_doesnt_matter}
{#PID<0.205.0>, 2, :state_doesnt_matter}
{#PID<0.205.0>, 3, :state_doesnt_matter}
{#PID<0.205.0>, 4, :state_doesnt_matter}
{#PID<0.205.0>, 5, :state_doesnt_matter}
{#PID<0.205.0>, 6, :state_doesnt_matter}
{#PID<0.205.0>, 7, :state_doesnt_matter}
{#PID<0.205.0>, 8, :state_doesnt_matter}
{#PID<0.205.0>, 9, :state_doesnt_matter}
{#PID<0.205.0>, 10, :state_doesnt_matter}
{#PID<0.205.0>, 11, :state_doesnt_matter}
{#PID<0.205.0>, 12, :state_doesnt_matter}
. . .
```

## Tinkering: For Science and Understanding
From here, we have a working flow.
There is a producer emitting our counter, and our consumber is displaying all of this and continuing the flow.
Now, what if we wanted multiple consumers?
Right now, if we examine the `IO.inspect/1` output, we see that every single event is handled by a single PID.
This isn't very Elixir-y.
We have massive concurrency built-in, we should probably leverage that as much as possible.
Let's make some adjustments so that we can have multiple workers by modifying `lib/genstage_example.ex`

```elixir
. . .
    children = [
      worker(GenstageExample.Producer, []),
      worker(GenstageExample.Consumer, [], id: 1),
      worker(GenstageExample.Consumer, [], id: 2),
    ]
. . .
```
[working commit](https://github.com/ybur-yug/genstage_example/commit/eb2e3a66254c9af053150996f4e9657363624b04)

Now, let's fire up IEx again:

```elixir
$ iex -S mix
iex(1)> {#PID<0.205.0>, 0, :state_doesnt_matter}
{#PID<0.205.0>, 1, :state_doesnt_matter}
{#PID<0.205.0>, 2, :state_doesnt_matter}
{#PID<0.207.0>, 3, :state_doesnt_matter}
. . .
```

As you can see, we have multiple PIDs now, simply by adding a line of code and giving our consumers IDs.
But we can take this even further:

```elixir
. . .
    children = [
      worker(GenstageExample.Producer, []),
    ]
    consumers = for id <- 1..(System.schedulers_online * 12) do
                              # helper to get the number of cores on machine
                  worker(GenstageExample.Consumer, [], id: id)
                end

    opts = [strategy: :one_for_one, name: GenstageExample.Supervisor]
    Supervisor.start_link(children ++ consumers, opts)
. . .
```
[working commit](https://github.com/ybur-yug/genstage_example/commit/981ace6631fb7c8a83f0e482d6d2a21b01f48612)

What we are doing here is quite simple.
First, we get the number of core on the machine with `System.schedulers_online/0`, and from there we simply create a worker just like we had.
Now we have 12 workers per core. This is much more effective.

```elixir
. . .
{#PID<0.229.0>, 63697, :state_doesnt_matter}
{#PID<0.224.0>, 53190, :state_doesnt_matter}
{#PID<0.223.0>, 72687, :state_doesnt_matter}
{#PID<0.238.0>, 69688, :state_doesnt_matter}
{#PID<0.196.0>, 62696, :state_doesnt_matter}
{#PID<0.212.0>, 52713, :state_doesnt_matter}
{#PID<0.233.0>, 72175, :state_doesnt_matter}
{#PID<0.214.0>, 51712, :state_doesnt_matter}
{#PID<0.227.0>, 66190, :state_doesnt_matter}
{#PID<0.234.0>, 58694, :state_doesnt_matter}
{#PID<0.211.0>, 55694, :state_doesnt_matter}
{#PID<0.240.0>, 64698, :state_doesnt_matter}
{#PID<0.193.0>, 50692, :state_doesnt_matter}
{#PID<0.207.0>, 56683, :state_doesnt_matter}
{#PID<0.213.0>, 71684, :state_doesnt_matter}
{#PID<0.235.0>, 53712, :state_doesnt_matter}
{#PID<0.208.0>, 51197, :state_doesnt_matter}
{#PID<0.200.0>, 61689, :state_doesnt_matter}
. . .
```

Though we lack any ordering like we would have with a single core, but every increment is being hit and processed.

We can take this a step further and change our broadcasting strategy from the default in our producer:

```elixir
. . .
  def init(counter) do
    {:producer, counter, dispatcher: GenStage.BroadcastDispatcher}
  end
. . .
```
[working commit](https://github.com/ybur-yug/genstage_example/commit/87c5f96c74e8fa90cd5b5fd108cd9ba104f78a65)

What this does is it accumulates demand from all consumers before broadcasting its events to all of them.
If we fire up IEx we can see the implication:

```elixir
. . .
{#PID<0.200.0>, 1689, :state_doesnt_matter}
{#PID<0.230.0>, 1690, :state_doesnt_matter}
{#PID<0.196.0>, 1679, :state_doesnt_matter}
{#PID<0.215.0>, 1683, :state_doesnt_matter}
{#PID<0.237.0>, 1687, :state_doesnt_matter}
{#PID<0.205.0>, 1682, :state_doesnt_matter}
{#PID<0.206.0>, 1695, :state_doesnt_matter}
{#PID<0.216.0>, 1682, :state_doesnt_matter}
{#PID<0.217.0>, 1689, :state_doesnt_matter}
{#PID<0.233.0>, 1681, :state_doesnt_matter}
{#PID<0.223.0>, 1689, :state_doesnt_matter}
{#PID<0.193.0>, 1194, :state_doesnt_matter}
. . .
```

Note that some numbers are showing twice now, this is why.


## Setting Up Postgres to Extend Our Model
To go further we'll need to bring in a database to store our progress and status.
This is quite simple using [Ecto](LINKTOLESSON).
To get started let's add it and the Postgresql adapter to `mix.exs`:

```elixir
. . .
  defp deps do
    [
     {:gen_stage, "~> 0.7"},
     {:ecto, "~> 2.0"},
     {:postgrex, "~> 0.12.1"},
    ]
  end
. . .

```

Fetch the dependencies and compile:

```shell
$ mix do deps.get, compile
```

And now we can add a repo for setup in `lib/repo.ex`:

```elixir
defmodule GenstageExample.Repo do
  use Ecto.Repo,
    otp_app: :genstage_example
end
```

and with this we can set up our config next in `config/config.exs`:

```elixir
use Mix.Config

config :genstage_example, ecto_repos: [GenstageExample.Repo]

config :genstage_example, GenstageExample.Repo,
  adapter: Ecto.Adapters.Postgres,
  database: "genstage_example",
  username: "your_username",
  password: "your_password",
  hostname: "localhost",
  port: "5432"
```

And if we add a supservisor to `lib/genstage_example.ex` we can now start working with the DB:

```elixir
. . .
  def start(_type, _args) do
    import Supervisor.Spec, warn: false

    children = [
      supervisor(GenstageExample.Repo, []),
      worker(GenstageExample.Producer, []),
    ]
  end
. . .
```

Now we need to create our migration:

```shell
$ mix ecto.gen.migration setup_tasks status:text payload:binary
```

# still goin...

```elixir

```

## What's Next?
In the next GenStage lessons we will take this code and go beyond simply getting the basis set up.
We will dive into the actual technical details of GenStage, and eventually build up a simple clone of DelayedJob in around 100 lines of code.
[Here](https://github.com/ybur-yug/genstage_example/tree/87c5f96c74e8fa90cd5b5fd108cd9ba104f78a65) is a link to all code thus far.


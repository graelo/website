---
title: "Hello Tokio"
---

Let's get started by writing a very basic Tokio application.

It connects to the Mini-Redis server, sets the value `world` onto the key
`hello`, then reads back that value using the `hello` key. To ease things out,
it uses the Mini-Redis async client library.

# The code

## Generate a new crate

Let's start by generating a new Rust app:

```bash
$ cargo new my-redis
$ cd my-redis
```

## Add dependencies

Next, open `Cargo.toml` and add the following right below `[dependencies]`:

```toml
tokio = { version = "1", features = ["full"] }
mini-redis = "0.4"
```

## Write the code

Then, open `main.rs` and replace its content with:

```rust
use mini_redis::{client, Result};

# fn dox() {
#[tokio::main]
async fn main() -> Result<()> {
    // Open a connection to the mini-redis address.
    let mut client = client::connect("127.0.0.1:6379").await?;

    // Set the key "hello" with value "world"
    client.set("hello", "world".into()).await?;

    // Get key "hello"
    let result = client.get("hello").await?;

    println!("got value from the server; result={:?}", result);

    Ok(())
}
# }
```

The `my-redis` app is now ready to be compiled and run to interact with
Mini-Redis.

First make sure the Mini-Redis server is installed and running; in a separate
terminal window, run

```bash
$ cargo install mini-redis  # if not installed previously
$ mini-redis-server
```

Now, back to the original terminal, run the `my-redis` application:

```bash
$ cargo run
got value from the server; result=Some(b"world")
```

Success!

You can find the full code [here][full].

[full]: https://github.com/tokio-rs/website/blob/master/tutorial-code/hello-tokio/src/main.rs

# Breaking it down

Let's take some time to go over what we just did because, although there isn't
much code, a lot is happening behind the scenes.

```rust
# use mini_redis::client;
# async fn dox() -> mini_redis::Result<()> {
let mut client = client::connect("127.0.0.1:6379").await?;
# Ok(())
# }
```

The [`client::connect`] function is provided by the `mini-redis` crate. It
asynchronously establishes a TCP connection with the specified remote address.
Once the connection is established, a `client` connection handle is returned.

Note that, even though the operation is performed asynchronously, the code
_looks_ synchronous. This is a direct style benefit of the **await/async**
idiom; the only indication an operation is asynchronous is the `.await`
operator.

[`client::connect`]: https://docs.rs/mini-redis/0.4/mini_redis/client/fn.connect.html

## What is asynchronous programming?

You are probably used to the predominant programming paragdigm: synchronous
programming. Most computer programs are synchronous, meaning they execute
operations in the same order that they were written in code. The first line
executes, then the next, and so on.  An important consequence is that a program
encountering an operation which cannot be completed immediately will block
until the operation completes.

For example, establishing a TCP connection can take a sizeable amount of time,
as behind the scenes an exchange takes place with a peer over the network. In
synchronous programming, a thread waiting for the connection to be established
is then blocked this entire time.

Things are different with asynchronous programming because operations which
cannot complete immediately are suspended to the background: for instance IO
operations. A thread which initiated such an operation is not blocked, and
continues running other things while the low-level IO operation makes progress.
Once it completes, the task is unsuspended and resumes processing from where it
had been suspended.

Our trivial TCP connection example above only has one task, so no other thing
needs to happen while it is suspended, but it definitely could have had other
things to do. Asynchronous programs typically have many such tasks.

In general, although asynchronous programming may yield faster applications, it
historically required much more complicated programs than synchronous ones,
because the programmer has to track all necessary state to resume work
correctly once each asynchronous operation completes. Fortunately, Tokio makes
this tedious and error-prone task much easier.

## Compile-time green-threading

Rust implements asynchronous programming using a feature called [`async/await`].
Functions that perform asynchronous operations are labeled with the `async`
keyword. We call them asynchronous functions, also noted `async fn`.

In our previous code example, the `client::connect()` function is defined as

```rust
use mini_redis::Result;
use mini_redis::client::Client;
use tokio::net::ToSocketAddrs;

pub async fn connect<T: ToSocketAddrs>(addr: T) -> Result<Client> {
    // ...
# unimplemented!()
}
```

This definition clearly looks like that of a regular synchronous function, but
the `async` keyword indicates that it operates asynchronously.

[[info]]
| In the presence of an async function, the Rust _Compiler_ transforms its
| entire definition into a routine which operates asynchronously (we will see
| later what the result of the transform is and how it helps operate
| asynchronously). This transform happens at **compile time**.
|
| In particular, the compile-transform transform also makes sure that any call
| to `.await` within the body of the `async` function immediately yields
| control back to the executing thread, so that it may do other work while the
| operation processes in the background.

[[warning]]
| Although other languages implement [`async/await`] too, Rust takes a unique
| approach. Primarily, Rust's async operations are **lazy**. This results in
| different runtime semantics than in other languages.

[`async/await`]: https://en.wikipedia.org/wiki/Async/await

If this doesn't quite make sense yet, don't worry. We will explore `async/await`
more throughout the guide.

## Using `async/await`

On the surface, async functions are called like any other Rust function.
However, calling these functions does not result in their function body
executing but instead immediately returns a value which represents the
operation. This is conceptually analogous to a zero-argument closure.

In order to ensure the operation will be executed, you have to use the `.await`
operator on its return value. A mechanism we will describe later uses this for
scheduling and execution.

For example, the following program

```rust
async fn say_world() {
    println!("world");
}

#[tokio::main]
async fn main() {
    // Calling `say_world()` does not execute the body of `say_world()`.
    let op = say_world();

    // This println! comes first.
    println!("hello");

    // Calling `.await` on `op` starts executing `say_world`.
    op.await;
}
```

outputs:

```text
hello
world
```

The return value of an `async fn` is an anonymous type which implements the
[`Future`] trait. In the above example, the `op` return value implements that
Future trait, and as we saw, the `.await` operator is required to have it
scheduled and executed.

[`Future`]: https://doc.rust-lang.org/std/future/trait.Future.html

## Async `main` function

The return value of an asynchronous function is a computation which, similar to
a closure, does not run until scheduled and executed. Doing so is precisely the
role of the [runtime], which for this reason contains the asynchronous task
scheduler, and provides evented I/O, timers, etc.

In Rust, the runtime never automatically starts, so it is your job to start it
wherever you need to enter an asynchronous context and ultimately have
asynchronous function scheduled and executed.

Because the program's `main` function is the prominent place to start the
runtime, the `main` function in asynchronous programs differs from the usual
one found in synchronous crates:

1. It is an `async fn`
2. It is annotated with `#[tokio::main]`

The `#[tokio::main]` macro transforms the `async fn main()` into a synchronous
`fn main()` which initializes a runtime instance and executes the async main
function.

For example, the compiler transforms

```rust
#[tokio::main]
async fn main() {
    println!("hello");
}
```

into

```rust
fn main() {
    let mut rt = tokio::runtime::Runtime::new().unwrap();
    rt.block_on(async {
        println!("hello");
    })
}
```

The details of the Tokio runtime will be covered later.

[runtime]: https://docs.rs/tokio/1/tokio/runtime/index.html

## Cargo features

When depending on Tokio for this tutorial, the `full` feature flag is enabled:

```toml
tokio = { version = "1", features = ["full"] }
```

Tokio has a lot of functionality (TCP, UDP, Unix sockets, timers, sync
utilities, multiple scheduler types, etc). Because not all applications need
all functionality, when attempting to optimize compile time or the end
application footprint, the application can decide to opt into **only** the
features it uses.

For now, use the `full` feature when depending on Tokio.

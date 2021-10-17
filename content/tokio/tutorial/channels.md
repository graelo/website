---
title: "Channels"
---

Let's deepen the concurrency concepts we learned by applying them to the client
side.

Move the server code we wrote before into an explicit binary file:

```text
mkdir src/bin
mv src/main.rs src/bin/server.rs
```

In this chapter, we will ignore the basic `hello-redis.rs` we were using, and
write new client code in the following source file:

```text
touch src/bin/client.rs
```

This client code will connect our server (`src/bin/server.rs`), so whenever you
want to run the client, you will have to launch the server first in a separate
terminal window:

```text
cargo run --bin server
```

and then the client program in its own terminal too:

```text
cargo run --bin client
```

That being said, let's code!

Say the client needs to send two concurrent Redis commands. Spawning one task
per command would make the two commands happen concurrently.

Naively, we could initialize the client and use it as a reference from each
spawned task:

```rust,compile_fail
use mini_redis::client;

#[tokio::main]
async fn main() {
    // Establish a connection to the server
    let mut client = client::connect("127.0.0.1:6379").await.unwrap();

    // Spawn two tasks, one gets a key, the other sets a key
    let t1 = tokio::spawn(async {
        let res = client.get("hello").await;
    });

    let t2 = tokio::spawn(async {
        client.set("foo", "bar".into()).await;
    });

    t1.await.unwrap();
    t2.await.unwrap();
}
```

Both tasks now share the same `client`. However, this does not compile for two
reasons.

First, because `client` neither implements `Copy` nor has a builtin
synchronization mechanism.

Second, because both tasks use `Client::set` which needs `&mut self` in order
to operate. The Rust borrow checker of course rejects any code using multiple
mutable (exclusive) references to the same value.

Let's review various unsatisfying solutions to this often-encountered situation.

- A cheap / inelegant solution would open one connection per task.
- We cannot wrap the client in a `std::sync::Mutex` because while waiting for
  the response, `.await` would need to be called with the lock held.
- We could wrap the client in a  `tokio::sync::Mutex`, but while waiting for
  the response in one task, the lock on `client` would be held, and no other
  task could use it, resulting in a single in-flight request. Generally
  speaking, if the client implements [pipelining], an async mutex results in
  underutilizing the connection.

[pipelining]: https://redis.io/topics/pipelining

# Message passing

In the situation we described above, the best answer is to spawn a dedicated
task to manage the `client` resource, and have tasks that need the resource use
message passing with it.

In practice, the sender task passes a message to the `client` task, the
`client` then issues the request to the server on behalf of the sender task,
and finally the server response is sent back from the `client` to the sender
task.

Let's review the benefits of this strategy:

- A single connection is established.
- The managing task maintains exclusive access to the `client` in order to call
  its `get` and `set` methods.
- The channel used for message passing works as a buffer: work may be passed by
  sender tasks to the `client` task while the `client` is busy; the `client`
  task will pull the next request from the channel as soon as it finishes being
  busy with the current request.

Overall, this may result in better throughput, and can be extended further to
support connection pooling.

# Tokio's channel primitives

Tokio provides a [number of channels][channels], each serving a different
purpose.

- [mpsc]: multi-producer, single-consumer channel. Many values can be sent.
- [oneshot]: single-producer, single consumer channel. A single value can be sent.
- [broadcast]: multi-producer, multi-consumer. Many values can be sent. Each
  receiver sees every value.
- [watch]: single-producer, multi-consumer. Many values can be sent, but no
  history is kept. Receivers only see the most recent value.

In addition to the Tokio-provided channels, the [`async-channel`] crate
provides a multi-producer multi-consumer channel where only one consumer sees
each message.

It is worth mentioning channels _for use outside of asynchronous Rust_, such as
[`std::sync::mpsc`] and [`crossbeam::channel`]. These channels wait for
messages by blocking the thread, which is not allowed in asynchronous code.

In this chapter, we will use [mpsc] and [oneshot], and later chapters will use
the other types of message passing channels.

The full code from this chapter is found [here][full].

[channels]: https://docs.rs/tokio/1/tokio/sync/index.html
[mpsc]: https://docs.rs/tokio/1/tokio/sync/mpsc/index.html
[oneshot]: https://docs.rs/tokio/1/tokio/sync/oneshot/index.html
[broadcast]: https://docs.rs/tokio/1/tokio/sync/broadcast/index.html
[watch]: https://docs.rs/tokio/1/tokio/sync/watch/index.html
[`async-channel`]: https://docs.rs/async-channel/
[`std::sync::mpsc`]: https://doc.rust-lang.org/stable/std/sync/mpsc/index.html
[`crossbeam::channel`]: https://docs.rs/crossbeam/latest/crossbeam/channel/index.html

# Define the message type

Message passing is flexible; it allows each message passed to the resource task
to correspond to a different command. In our case, the `client` resource task
will receive and respond to `GET` and `SET` commands.

To model this, we first define a `Command` enum and include a variant
for each command type.

```rust
use bytes::Bytes;

#[derive(Debug)]
enum Command {
    Get {
        key: String,
    },
    Set {
        key: String,
        val: Bytes,
    }
}
```

# Create the channel

The `main` function creates a `tokio::sync::mpsc` channel for the senders to
_send_ commands to the `client` resource task (which still manages the
connection to Redis). The multi-producer capability allows messages to be sent
from many tasks.

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    // Create a new channel with a capacity of at most 32.
    let (tx, mut rx) = mpsc::channel(32);
# tx.send(()).await.unwrap();

    // ... Rest comes here
}
```

Creating the channel returns two values, a sender and a receiver. The two
handles are used separately, and may each be moved to different tasks.

The channel is created with a capacity of 32. If messages are sent faster than
they are received, the channel will store them. Once the 32 messages are stored
in the channel, any `send(...).await` call will go to sleep until a message has
been removed from the channel by the receiver.

Sending from multiple tasks is done by **cloning** the `Sender` handle. Here is
a simple example in which each message is a `String`:

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(32);
    let tx2 = tx.clone();

    tokio::spawn(async move {
        tx.send("sending from first handle").await;
    });

    tokio::spawn(async move {
        tx2.send("sending from second handle").await;
    });

    while let Some(message) = rx.recv().await {
        println!("GOT = {}", message);
    }
}
```

Let's describe the mechanics.

Both messages are sent to the `Receiver` handle. Because mpsc channel can only
have a single receiver, it is not possible to clone the `Receiver` handle of an
`mpsc` channel.

After all `Sender` handles have gone out of scope (or have otherwise been
dropped), it is no longer possible to send more messages via the channel. At
this point, the `recv` call on the `Receiver` can only ever return `None`. A
receiver's `recv` call returning `None` indicates that all senders are gone and
the channel is closed and no longer usable.

In our case of a task that manages the Redis connection, it knows that it
can close the Redis connection once the channel is closed, as the connection
will not be used again.

# Spawn manager task

Next, spawn a task that processes messages from the channel. First, a client
connection is established to Redis. Then, received commands are issued via the
Redis connection.

```rust
use mini_redis::client;
# enum Command {
#    Get { key: String },
#    Set { key: String, val: bytes::Bytes }
# }
# async fn dox() {
# let (_, mut rx) = tokio::sync::mpsc::channel(10);
// The `move` keyword is used to **move** ownership of `rx` into the task.
let manager = tokio::spawn(async move {
    // Establish a connection to the server
    let mut client = client::connect("127.0.0.1:6379").await.unwrap();

    // Start receiving messages
    while let Some(cmd) = rx.recv().await {
        use Command::*;

        match cmd {
            Get { key } => {
                client.get(&key).await;
            }
            Set { key, val } => {
                client.set(&key, val).await;
            }
        }
    }
});
# }
```

Now, update the two tasks to send commands over the channel instead of issuing
them directly on the Redis connection.

```rust
# #[derive(Debug)]
# enum Command {
#    Get { key: String },
#    Set { key: String, val: bytes::Bytes }
# }
# async fn dox() {
# let (mut tx, _) = tokio::sync::mpsc::channel(10);
// The `Sender` handles are moved into the tasks. As there are two
// tasks, we need a second `Sender`.
let tx2 = tx.clone();

// Spawn two tasks, one gets a key, the other sets a key
let t1 = tokio::spawn(async move {
    let cmd = Command::Get {
        key: "hello".to_string(),
    };

    tx.send(cmd).await.unwrap();
});

let t2 = tokio::spawn(async move {
    let cmd = Command::Set {
        key: "foo".to_string(),
        val: "bar".into(),
    };

    tx2.send(cmd).await.unwrap();
});
# }
````

At the bottom of the `main` function, we `.await` the join handles to ensure the
commands fully complete before the process exits.

```rust
# type Jh = tokio::task::JoinHandle<()>;
# async fn dox(t1: Jh, t2: Jh, manager: Jh) {
t1.await.unwrap();
t2.await.unwrap();
manager.await.unwrap();
# }
```

# Receive responses

The final step is to receive the response back from the manager task. The `GET`
command needs to get the value and the `SET` command needs to know if the
operation completed successfully.

To pass the response, a `oneshot` channel is used. The `oneshot` channel is a
single-producer, single-consumer channel optimized for sending a single value.
In our case, the single value is the response.

Similar to `mpsc`, `oneshot::channel()` returns a sender and receiver handle.

```rust
use tokio::sync::oneshot;

# async fn dox() {
let (tx, rx) = oneshot::channel();
# tx.send(()).unwrap();
# }
```

Unlike `mpsc`, no capacity is specified as the capacity is always one.
Additionally, neither handle can be cloned.

To receive responses from the manager task, before sending a command, a `oneshot`
channel is created. The `Sender` half of the channel is included in the command
to the manager task. The receive half is used to receive the response.

First, update `Command` to include the `Sender`. For convenience, a type alias
is used to reference the `Sender`.

```rust
use tokio::sync::oneshot;
use bytes::Bytes;

/// Multiple different commands are multiplexed over a single channel.
#[derive(Debug)]
enum Command {
    Get {
        key: String,
        resp: Responder<Option<Bytes>>,
    },
    Set {
        key: String,
        val: Bytes,
        resp: Responder<()>,
    },
}

/// Provided by the requester and used by the manager task to send
/// the command response back to the requester.
type Responder<T> = oneshot::Sender<mini_redis::Result<T>>;
```

Now, update the tasks issuing the commands to include the `oneshot::Sender`.

```rust
# use tokio::sync::{oneshot, mpsc};
# use bytes::Bytes;
# #[derive(Debug)]
# enum Command {
#     Get { key: String, resp: Responder<Option<bytes::Bytes>> },
#     Set { key: String, val: Bytes, resp: Responder<()> },
# }
# type Responder<T> = oneshot::Sender<mini_redis::Result<T>>;
# fn dox() {
# let (mut tx, mut rx) = mpsc::channel(10);
# let mut tx2 = tx.clone();
let t1 = tokio::spawn(async move {
    let (resp_tx, resp_rx) = oneshot::channel();
    let cmd = Command::Get {
        key: "hello".to_string(),
        resp: resp_tx,
    };

    // Send the GET request
    tx.send(cmd).await.unwrap();

    // Await the response
    let res = resp_rx.await;
    println!("GOT = {:?}", res);
});

let t2 = tokio::spawn(async move {
    let (resp_tx, resp_rx) = oneshot::channel();
    let cmd = Command::Set {
        key: "foo".to_string(),
        val: "bar".into(),
        resp: resp_tx,
    };

    // Send the SET request
    tx2.send(cmd).await.unwrap();

    // Await the response
    let res = resp_rx.await;
    println!("GOT = {:?}", res);
});
# }
```

Finally, update the manager task to send the response over the `oneshot` channel.

```rust
# use tokio::sync::{oneshot, mpsc};
# use bytes::Bytes;
# #[derive(Debug)]
# enum Command {
#     Get { key: String, resp: Responder<Option<bytes::Bytes>> },
#     Set { key: String, val: Bytes, resp: Responder<()> },
# }
# type Responder<T> = oneshot::Sender<mini_redis::Result<T>>;
# async fn dox(mut client: mini_redis::client::Client) {
# let (_, mut rx) = mpsc::channel::<Command>(10);
while let Some(cmd) = rx.recv().await {
    match cmd {
        Command::Get { key, resp } => {
            let res = client.get(&key).await;
            // Ignore errors
            let _ = resp.send(res);
        }
        Command::Set { key, val, resp } => {
            let res = client.set(&key, val).await;
            // Ignore errors
            let _ = resp.send(res);
        }
    }
}
# }
```

Calling `send` on `oneshot::Sender` completes immediately and does **not**
require an `.await`. This is because `send` on a `oneshot` channel will always
fail or succeed immediately without any form of waiting.

Sending a value on a oneshot channel returns `Err` when the receiver half has
dropped. This indicates the receiver is no longer interested in the response. In
our scenario, the receiver cancelling interest is an acceptable event. The `Err`
returned by `resp.send(...)` does not need to be handled.

You can find the entire code [here][full].

# Backpressure and bounded channels

Whenever concurrency or queuing is introduced, it is important to ensure that the
queueing is bounded and the system will gracefully handle the load. Unbounded queues
will eventually fill up all available memory and cause the system to fail in
unpredictable ways.

Tokio takes care to avoid implicit queuing. A big part of this is the fact that
async operations are lazy. Consider the following:

```rust
# fn async_op() {}
# fn dox() {
loop {
    async_op();
}
# }
# fn main() {}
```

If the asynchronous operation runs eagerly, the loop will repeatedly queue a new
`async_op` to run without ensuring the previous operation completed. This
results in implicit unbounded queuing. Callback based systems and **eager**
future based systems are particularly susceptible to this.

However, with Tokio and asynchronous Rust, the above snippet will **not** result
in `async_op` running at all. This is because `.await` is never called. If the
snippet is updated to use `.await`, then the loop waits for the operation to
complete before starting over.

```rust
# async fn async_op() {}
# async fn dox() {
loop {
    // Will not repeat until `async_op` completes
    async_op().await;
}
# }
# fn main() {}
```

Concurrency and queuing must be explicitly introduced. Ways to do this include:

* `tokio::spawn`
* `select!`
* `join!`
* `mpsc::channel`

When doing so, take care to ensure the total amount of concurrency is bounded. For
example, when writing a TCP accept loop, ensure that the total number of open
sockets is bounded. When using `mpsc::channel`, pick a manageable channel
capacity. Specific bound values will be application specific.

Taking care and picking good bounds is a big part of writing reliable Tokio applications.

[full]: https://github.com/tokio-rs/website/blob/master/tutorial-code/channels/src/main.rs

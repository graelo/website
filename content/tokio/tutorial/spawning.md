---
title: "Spawning"
---

Let's shift gears and start working on our basic Redis server.

First, move the client `SET`/`GET` code from the previous chapter to an example
file `hello-redis.rs`. We will use it whenever we need to prime the server with
the key `hello` and value `world`.

```bash
$ mkdir -p examples
$ mv src/main.rs examples/hello-redis.rs
```

Then create a new, empty `src/main.rs` and continue.

# Accepting sockets

The first thing our Redis server needs to do is listen and accept inbound TCP
sockets. This is done with [`tokio::net::TcpListener`][tcpl].

[[info]]
| Many of Tokio's types are named the same as their synchronous equivalent in
| the Rust standard library. When it makes sense, Tokio exposes the same APIs
| as `std` but using `async fn`.

In the following code, a `TcpListener` is bound to port `6379`, then sockets
are accepted in a loop. Each accepted socket is processed then closed before
the code loops and awaits for the next socket to be accepted. This sequential
execution is obviously not ideal, but it provides a good starting point.

Each socket is processed by reading the received message (a Redis Command),
printing it to stdout and responding with an error.

Write the code for this basic server in `src/main.rs`:

```rust
use tokio::net::{TcpListener, TcpStream};
use mini_redis::{Connection, Frame};

# fn dox() {
#[tokio::main]
async fn main() {
    // Bind the listener to the address
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        // The second item contains the IP and port of the new connection.
        let (socket, _) = listener.accept().await.unwrap();
        process(socket).await;
    }
}
# }

async fn process(socket: TcpStream) {
    // The `Connection` lets us read/write redis **frames** instead of
    // byte streams. The `Connection` type is defined by mini-redis.
    let mut connection = Connection::new(socket);

    if let Some(frame) = connection.read_frame().await.unwrap() {
        println!("GOT: {:?}", frame);

        // Respond with an error
        let response = Frame::Error("unimplemented".to_string());
        connection.write_frame(&response).await.unwrap();
    }
}
```

Now, run the server:

```bash
$ cargo run
```

In a separate terminal window, run the `hello-redis` example (as a reminder, it
connects to the server, then sets a value onto the `hello` key, and gets it
back):

```bash
$ cargo run --example hello-redis
```

In that terminal, the output should be what the server currently replies to any
command:

```text
Error: "unimplemented"
```

In the server terminal, the output is the `println!()` for any received
command:

```text
GOT: Array([Bulk(b"set"), Bulk(b"hello"), Bulk(b"world")])
```

[tcpl]: https://docs.rs/tokio/1/tokio/net/struct.TcpListener.html

# Concurrency

Besides only responding with errors, our server has a serious problem: it
processes inbound requests one at a time. Concretely, in the main loop, after a
connection is accepted and the corresponding socket is obtained, the server
thread blocks until the response is fully written to the socket and that socket
is closed.

As we want our Redis server to process **many** concurrent requests, we need to
add some concurrency.

[[info]]
| Concurrency and parallelism are not the same thing. If you alternate between
| two tasks, then you are working on both tasks concurrently, but not in
| parallel. For it to qualify as parallel, you need two workers, each
| dedicated to its task.
|
| One of the advantages of using Tokio is that asynchronous code allows you to
| work on many tasks _concurrently_, that is, without having to work on them in
| parallel using ordinary OS threads. In fact, Tokio can run many tasks
| concurrently on a single thread!

Back to our basic server, we need a new task to be spawned for each inbound
connection for it to be processed concurrently with other connections. Whether
the spawned task runs in a different OS thread is an orthogononal matter which
the runtime's scheduler takes care of.

With this in mind, the accept loop becomes:

```rust
use tokio::net::TcpListener;

# fn dox() {
#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        let (socket, _) = listener.accept().await.unwrap();

        // A new task is spawned for each inbound socket. The socket is
        // moved to the new task and processed there.
        tokio::spawn(async move {
            process(socket).await;
        });
    }
}
# }
# async fn process(_: tokio::net::TcpStream) {}
```

## Tasks

Tokio tasks are asynchronous green threads. Each is created by passing an
`async` block to `tokio::spawn`. The `tokio::spawn` function returns a
`JoinHandle`, which the caller may use to interact with the spawned task.

In particular, the `async` block may have a return value, which the caller may
obtain by using the `.await` operator on the `JoinHandle`. This works as we saw
earlier with `async fn`: the `.await` operator is needed for scheduling and
execution by the runtime.

Here is a simple example of spawning a task and awaiting its return value:

```rust
#[tokio::main]
async fn main() {
    let handle = tokio::spawn(async {
        // Do some async work
        "return value"
    });

    // Do some other work

    let out = handle.await.unwrap();
    println!("GOT {}", out);
}
```

Interestingly, awaiting on `JoinHandle` always returns a `Result`, so that,
when a task succesfully completes, the return value is wrapped inside an
`Ok()`. On the contrary, if a task encounters an error during execution, the
`JoinHandle` will return an `Err`. This happens when the task either panics, or
if the task is forcefully cancelled by the runtime shutting down.

Obviously, if the task itself returns a `Result`, that result is wrapped in the
`Result` of the `JoinHandle`.

Tasks are units of execution managed by the scheduler. Spawning a task submits
it to the Tokio scheduler, which then ensures that the task executes when it
has work to do. As we alluded to earlier, the spawned task may execute on
the same thread as where it was spawned, or it may execute on a different OS
thread managed by the runtime. Note that the task can also be moved between
threads after being spawned. This is all managed by the runtime.

Tasks in Tokio are very lightweight. Under the hood, they require only a single
allocation and 64 bytes of memory. Applications should feel free to spawn
thousands, if not millions of tasks.

## `'static` bound

When you spawn a task on the Tokio runtime, its type must be `'static`. **This
means that the spawned task must not contain any reference to data owned
outside the task**.

[[info]]
| It is a common misconception that `'static` always means "lives forever",
| but this is not the case. Just because a value is `'static` does not mean
| that you have a memory leak. You can read more in [Common Rust Lifetime
| Misconceptions][common-lifetime].

[common-lifetime]: https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#2-if-t-static-then-t-must-be-valid-for-the-entire-program

For example, the following cannot compile because the `println` call _borrows_
`v`. The spawned task thus contains a reference to data owned outside the task,
and it's lifetime is not `'static`:

```rust,compile_fail
use tokio::task;

#[tokio::main]
async fn main() {
    let v = vec![1, 2, 3];

    task::spawn(async {
        println!("Here's a vec: {:?}", v);
    });
}
```

Attempting to compile this results in a helpful error message:

```text
error[E0373]: async block may outlive the current function, but it borrows `v`, which is owned by the current function
 --> src/bin/toto.rs:7:23
  |
7 |       task::spawn(async {
  |  _______________________^
8 | |         println!("Here's a vec: {:?}", v);
  | |                                        - `v` is borrowed here
9 | |     });
  | |_____^ may outlive borrowed value `v`
  |
  = note: async blocks are not executed immediately and must either take a reference or ownership of outside variables they use
help: to force the async block to take ownership of `v` (and any other referenced variables), use the `move` keyword
  |
7 |     task::spawn(async move {
8 |         println!("Here's a vec: {:?}", v);
9 |     });
  |

error: aborting due to previous error

For more information about this error, try `rustc --explain E0373`.
error: could not compile `nothing`
```

This happens because, by default, variables are not _moved_ into async blocks.
Due to `println!` simply _borrowing_ its arguments, the vector `v` remains
owned by the `main` function. The rust compiler helpfully explains this to us
and even suggests the fix!

Changing line 7 to `task::spawn(async move {` will instruct the compiler to
**move** `v` into the spawned task. Now, the task owns all of its data, making
it `'static`.

Moving a value is not the only option; sharing is possible too. However, if a
piece of data must be accessible from multiple tasks concurrently, then it must
be shared using synchronization primitives such as `Arc`.

[[info]]
| This paragraph are not needed anymore with the updated compiler message.
|
| Note that the error message mentions "[...] requires argument type to outlive
| the `'static` lifetime". This terminology can be rather confusing because, as
| we know, the `'static` lifetime lasts until the end of the program, so it looks like requiring
| something to outlive the program means requiring it to leak memory. This is
| of course not the case, and re-reading the fine print resolves the confusion:
| it is the _type_, not the _value_, that must outlive the `'static` lifetime.
| The value may be destroyed before its type is no longer valid.
|
| When we say that a value is `'static`, it simply means that it would not be
| incorrect to keep that value around forever. This is important because the
| compiler is unable to reason about how long a newly spawned task actually
| lives, so the only way it can be sure that the task lives long enough is to
| make sure nothing prevents it from living forever.
|
| The article that the info-box earlier links to uses the terminology "bounded
| by `'static`" rather than "its type outlives `'static`" or "the value is
| `'static`" to refer to `T: 'static`. These all mean the same thing, but are
| different from "annotated with `'static`" as in `&'static T`.

## `Send` trait bound

Tasks spawned by `tokio::spawn` **must** implement `Send`. This allows the
Tokio runtime to move the tasks between threads while they are suspended at an
`.await`. Tasks are `Send` when **all** data that is held **across** `.await` calls is
`Send`.

The reason for this is when `.await` is called, the task yields back to the
scheduler. The next time the task is executed, it resumes from the point it
last yielded. To make this work, all state that is used **after** `.await` must
be saved by the task. If the state is `Send`, then it can be moved across
threads, and so does the task. Conversely, if the state is not `Send`, then
neither is the task.

The two following examples illustrate these two cases. They both use `Rc`,
which is notoriously _not_ `Send`.

This first example works because the `Rc` is dropped before the task's state is saved
the control yield point:

```rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        // The scope forces `rc` to drop before `.await`.
        {
            let rc = Rc::new("hello");
            println!("{}", rc);
        }

        // `rc` is no longer used. It is **not** persisted when
        // the task yields to the scheduler
        yield_now().await;
    });
}
```

This second example fails to compile:

```rust,compile_fail
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
    tokio::spawn(async {
        let rc = Rc::new("hello");

        // `rc` is used after `.await`. It must be persisted to
        // the task's state.
        yield_now().await;

        println!("{}", rc);
    });
}
```

with the compiler returning the following error message:

```text
error: future cannot be sent between threads safely
   --> src/main.rs:6:5
    |
6   |     tokio::spawn(async {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    |
   ::: [..]spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in
    |                          `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait
    |       `std::marker::Send` is not  implemented for
    |       `std::rc::Rc<&str>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:10:9
    |
7   |         let rc = Rc::new("hello");
    |             -- has type `std::rc::Rc<&str>` which is not `Send`
...
10  |         yield_now().await;
    |         ^^^^^^^^^^^^^^^^^ await occurs here, with `rc` maybe
    |                           used later
11  |         println!("{}", rc);
12  |     });
    |     - `rc` is later dropped here
```

We will discuss in depth a special case of this error [in the next
chapter][mutex-guard].

[mutex-guard]: shared-state#holding-a-mutexguard-across-an-await

# Storing and reading values

Now we know more about the requirements for spawning Tokio tasks, we can
implement the `process` function to handle incoming commands.

The code uses a `HashMap` to store keys and values. `SET` commands insert into
the `HashMap` and `GET` commands load them. Additionally, the code uses a loop
to accept more than one command per connection.

In the `src/main.rs` code we started earlier, update the `process` function
(and the imports):

```rust
use tokio::net::TcpStream;
use mini_redis::{Connection, Frame};

async fn process(socket: TcpStream) {
    use mini_redis::Command::{self, Get, Set};
    use std::collections::HashMap;

    // A hashmap is used to store data
    // Keys are Strings, values are Vec<u8>
    let mut db = HashMap::new();

    // Connection, provided by `mini-redis`, handles parsing frames from
    // the socket
    let mut connection = Connection::new(socket);

    // Use `read_frame` to receive a command from the connection.
    while let Some(frame) = connection.read_frame().await.unwrap() {
        let response = match Command::from_frame(frame).unwrap() {
            Set(cmd) => {
                // The value is stored as `Vec<u8>`
                db.insert(cmd.key().to_string(), cmd.value().to_vec());
                Frame::Simple("OK".to_string())
            }
            Get(cmd) => {
                if let Some(value) = db.get(cmd.key()) {
                    // `Frame::Bulk` expects data to be of type `Bytes`. This
                    // type will be covered later in the tutorial. For now,
                    // `&Vec<u8>` is converted to `Bytes` using `into()`.
                    Frame::Bulk(value.clone().into())
                } else {
                    Frame::Null
                }
            }
            cmd => panic!("unimplemented {:?}", cmd),
        };

        // Write the response to the client
        connection.write_frame(&response).await.unwrap();
    }
}
```

Now, start this server:

```bash
$ cargo run
```

As we did previously, in a separate terminal window, run the `hello-redis`
example in order to send a `SET` with `hello` and `world` and send a `GET` to
retrieve the stored value. This tests if the server works properly:

```bash
$ cargo run --example hello-redis
```

If the server works properly, you see the following output:

```text
got value from the server; result=Some(b"world")
```

This confirms the server and client communicate, and we can now get and set
values. You can find the full code [here][full].

[full]: https://github.com/tokio-rs/website/blob/master/tutorial-code/spawning/src/main.rs

However, an important remains: the values are not shared between connections.
If another socket connects and tries to `GET` the `hello` key, it will not find
anything.

In the next chapter, we will implement persisting data for all sockets.

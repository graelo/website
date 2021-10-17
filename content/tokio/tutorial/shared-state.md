---
title: "Shared state"
---

So far, we have a key-value server working. However, there is a major flaw:
state is not shared across connections. We will fix that in this chapter.

# Strategies

There are a couple of different ways to share state in Tokio.

1. Guard the shared state with a Mutex.
2. Spawn a task to manage the state and use message passing to operate on it.

When the state is simple data, the first strategy is generally the most
appropriate. When the shared state involves I/O primitives, then the second
strategy is the better suited.

In this chapter we use first strategy because the shared state is a `HashMap`,
and the associated operations on that state (`insert` and `get`) are
synchronous. A `Mutex` is sufficient here.

The next chapter uses the second strategy.

# Add `bytes` dependency

From now on, we will often use `Bytes` (from [`bytes`] crate) instead of using
`Vec<u8>`.

The first reason for doing this is `bytes` is idiomatic in asynchronous code
because it provides a robust byte array structure for
network programming. The biggest feature it adds over `Vec<u8>` is shallow
cloning. In other words, calling `clone()` on a `Bytes` instance does not copy
the underlying data. Instead, a `Bytes` instance is a reference-counted handle
to some underlying data. The `Bytes` type is roughly an `Arc<Vec<u8>>` but with
some added capabilities.

The second reason is the Mini-Redis crate uses `Bytes` as part of its API.

To add the dependency on `bytes`, add the following to your `Cargo.toml` in the
`[dependencies]` section:

```toml
bytes = "1"
```

[`bytes`]: https://docs.rs/bytes/1/bytes/struct.Bytes.html

# Initialize the `HashMap`

Because the server will only have one `HashMap` and share it across many tasks
and potentially many threads, the code wraps the `HashMap` in a `Arc<Mutex<_>>`
synchronization mechanism. Using `Arc` allows the `HashMap` to be referenced
concurrently from many tasks, potentially running on many threads.


Let's first add a conveniience type alias and its supporting `use` statements:

```rust
use bytes::Bytes;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

type Db = Arc<Mutex<HashMap<String, Bytes>>>;
```

Because there will be only `HashMap` in the program, the `main` function
initializes it once, and wraps it in a `Mutex` and an `Arc`. This
initialization call returns a _handle_ which is then passed to the `process`
function.

[[info]]
| Throughout Tokio, the term **handle** is used to reference a value that
| provides access to some shared state.

The resulting code is:

```rust
use tokio::net::TcpListener;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

# fn dox() {
#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    println!("Listening");

    let db = Arc::new(Mutex::new(HashMap::new()));

    loop {
        let (socket, _) = listener.accept().await.unwrap();
        // Clone the handle to the hash map.
        let db = db.clone();

        println!("Accepted");
        tokio::spawn(async move {
            process(socket, db).await;
        });
    }
}
# }
# type Db = Arc<Mutex<HashMap<(), ()>>>;
# async fn process(_: tokio::net::TcpStream, _: Db) {}
```

## On using `std::sync::Mutex`

Note that the `HashMap` is guarded with `std::sync::Mutex` and **not**
`tokio::sync::Mutex`.  It is a common antipattern to unconditionally use
`tokio::sync::Mutex` from within async code. Let's explain the difference
because making the wrong choice impacts performance.

A synchronous mutex, such as `std::sync::Mutex` or its faster alternative
[`parking_lot::Mutex`], will block the current thread when waiting to acquire
the lock. Of course, while the thread is blocked waiting to acquire the lock,
it cannot process other tasks. However, when protecting simple data, this is
often the best guard strategy and you should use it in asynchronous functions
as long as the contention remains low and the lock is not held across calls to
`.await` (a synchronous mutex is not `Send`; we cover this detail later in this
chapter).

[parking_lot]: https://docs.rs/parking_lot/0.10.2/parking_lot/type.Mutex.html

An asynchronous mutex such as `tokio::sync::Mutex`, is a mutex for which the
lock can be held across calls to `.await` (it is `Send`) and which does not
block when waiting to acquire the lock. However this mutex is not magic: it
uses a synchronous mutex internally, and if used to guard access to simple
data, it is slower than a `std::sync::Mutex` or `parking_lot::Mutex`.

The primary use case of the async mutex is to provide shared mutable access to
I/O resources such as a database connection, and even in these cases, spawning
a task to manage the I/O resource is often a better strategy (see next
chapter).

# Update `process()`

In our previous code, the `HashMap` initialization code moved from the
`process` function to the `main` function.  The `process` function now takes
instead the shared handle to the `HashMap` as an argument. It also needs to
lock the `HashMap` before using it.

Here is the resulting code:

```rust
use tokio::net::TcpStream;
use mini_redis::{Connection, Frame};
# use std::collections::HashMap;
# use std::sync::{Arc, Mutex};
# type Db = Arc<Mutex<HashMap<String, bytes::Bytes>>>;

async fn process(socket: TcpStream, db: Db) {
    use mini_redis::Command::{self, Get, Set};

    // Connection, provided by `mini-redis`, handles parsing frames from
    // the socket
    let mut connection = Connection::new(socket);

    while let Some(frame) = connection.read_frame().await.unwrap() {
        let response = match Command::from_frame(frame).unwrap() {
            Set(cmd) => {
                let mut db = db.lock().unwrap();
                db.insert(cmd.key().to_string(), cmd.value().clone());
                Frame::Simple("OK".to_string())
            }
            Get(cmd) => {
                let db = db.lock().unwrap();
                if let Some(value) = db.get(cmd.key()) {
                    Frame::Bulk(value.clone())
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

# Tasks, threads, and contention

We saw earlier that using a synchronous mutex to guard short critical sections
is an acceptable strategy when contention is minimal. When contention increases
however, the thread executing the current task must block and wait on the mutex
to acquire the lock. This not only blocks the _current task_ but it also blocks
_all other tasks_ scheduled on the current thread.

The contention level is a function of your code design, the runtime you choose
for your program, and of course the workload.

By default, the Tokio runtime uses a multi-threaded scheduler. Tasks are
scheduled on any number of threads managed by the runtime. If a large number of
tasks are scheduled to execute and they all require access to the mutex, then
there will be contention. On the other hand, if the
[`current_thread`][current_thread] runtime flavor is used, then the mutex will
never be contended.

[[info]]
| The [`current_thread` runtime flavor][basic-rt] is a lightweight,
| single-threaded runtime. It is a good choice when only spawning
| a few tasks and opening a handful of sockets. For example, this
| option works well when providing a synchronous API bridge on top
| of an asynchronous client library.

[basic-rt]: https://docs.rs/tokio/1/tokio/runtime/struct.Builder.html#method.new_current_thread

If contention on a synchronous mutex becomes a problem, the best fix is rarely
to switch to the Tokio mutex. Instead, options to consider are:

- Switch to a dedicated task to manage state and use message passing (next
  chapter).
- Reduce contention by sharding the shared state and the mutex.
- Restructure the code to avoid the mutex.

## Sharding storage and associated mutex

In our case, as each *key* of the `HashMap` is independent, storage and mutex
sharding will work well. To apply this strategy, replace the single
`Mutex<HashMap<_, _>>` instance with `N` distinct instances:

```rust
# use std::collections::HashMap;
# use std::sync::{Arc, Mutex};
type ShardedDb = Arc<Vec<Mutex<HashMap<String, Vec<u8>>>>>;
```

With the `ShardedDb`, finding the cell for any given key becomes a two step
process. First, the key is used to identify which shard it is part of. Then,
the key is looked up in the `HashMap` of the corresponding shard.

```rust,compile_fail
let shard = db[hash(key) % db.len()].lock().unwrap();
shard.insert(key, value);
```

The [dashmap] crate provides an implementation of a sharded hash map.

[current_thread]: https://docs.rs/tokio/1/tokio/runtime/index.html#current-thread-scheduler
[dashmap]: https://docs.rs/dashmap

## Restructure your code to not hold the lock across an `.await`

Before restructuring the code, let go back to our discussion on holding a
synchronous mutex lock across `.await`calls.

### What happens when a `MutexGuard` is held across an `.await`

You might find yourself writing erroneous code that looks like this:

```rust
use std::sync::{Mutex, MutexGuard};

async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1;

    do_something_async().await;
} // lock goes out of scope here
# async fn do_something_async() {}
```

When you try to spawn a task that calls this function, you encounter the
following error message:

```text
error: future cannot be sent between threads safely
   --> src/lib.rs:13:5
    |
13  |     tokio::spawn(async move {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    |
   ::: /playground/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-0.2.21/src/task/spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait `std::marker::Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
   --> src/lib.rs:7:5
    |
4   |     let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    |         -------- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
...
7   |     do_something_async().await;
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ await occurs here, with `mut lock` maybe used later
8   | }
    | - `mut lock` is later dropped here
```

This happens because the lock is still in scope when `.await` is called.
Because you can't send a mutex lock to another thread (due to
`std::sync::MutexGuard` **not** being `Send`), so the task cannot save what is in
scope into its state, hence the compilation error.

We already saw in the [Send bound section from the spawning
chapter][send-bound] that this `Send` requirement is necessary because the
Tokio runtime can move a task between threads at any `.await` call.

[send-bound]: spawning#send-bound

### Technique 1 - scope the mutex

If you find yourself in the above situation, you should restructure your code
such that the mutex lock exits the scope before the `.await` call. For
instance, the following code is correct and will compile.

```rust
# use std::sync::{Mutex, MutexGuard};
// This works!
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    {
        let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
        *lock += 1;
    } // lock goes out of scope here

    do_something_async().await;
}
# async fn do_something_async() {}
```

However, you might think it suffices to drop the lock before the call to
`.await`. You will be disappointed because it still insufficient for the
compiler. For instance, the following code does not compile:

```rust
use std::sync::{Mutex, MutexGuard};

// This fails too.
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock: MutexGuard<i32> = mutex.lock().unwrap();
    *lock += 1;
    drop(lock);

    do_something_async().await;
}
# async fn do_something_async() {}
```

The reason is the compiler currently calculates whether a future is `Send`
based on scope information only. It will hopefully be updated to support
explicitly dropping it in the future, but for now, you must explicitly use a
scope.

In any case, you should not try to circumvent this issue by spawning the task
in a way that does not require it to be `Send`. You would be trying to get away
with a blocking lock in the wild. This is risky because if your task acquires
the lock and Tokio suspends it in this state at an `.await` call, then some
other task which may be scheduled to run on the same thread may also try to
lock that mutex and be blocked too.  This results in a deadlock due to the task
waiting to lock the mutex preventing the task holding the lock from resuming
and releasing it.

### Technique 2 - Wrap the mutex inside sync methods of a struct

Another robust way to restructure your code is to wrap the mutex in a struct,
and only ever lock the mutex inside non-async methods on that struct. This
guarantees the mutex is not in scope on any `.await` call.

The following code illustrates this strategy:

```rust
use std::sync::Mutex;

struct CanIncrement {
    mutex: Mutex<i32>,
}
impl CanIncrement {
    // This function is not marked async.
    fn increment(&self) {
        let mut lock = self.mutex.lock().unwrap();
        *lock += 1;
    }
}

async fn increment_and_do_stuff(can_incr: &CanIncrement) {
    can_incr.increment();
    do_something_async().await;
}
# async fn do_something_async() {}
```

## Use Tokio's asynchronous mutex

The [`tokio::sync::Mutex`] type provided by Tokio can also be used. The primary
feature of the Tokio mutex is that it can be held across an `.await` without any
issues. That said, an asynchronous mutex is more expensive than an ordinary
mutex, and it is typically better to use one of the two other strategies.

```rust
use tokio::sync::Mutex; // note! This uses the Tokio mutex

// This compiles!
// (but restructuring the code would be better in this case)
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock = mutex.lock().await;
    *lock += 1;

    do_something_async().await;
} // lock goes out of scope here
# async fn do_something_async() {}
```

[`tokio::sync::Mutex`]: https://docs.rs/tokio/1/tokio/sync/struct.Mutex.html

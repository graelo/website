---
title: "Setup"
---

This tutorial guides you through the process of building a [Redis] client and
server. It starts with the basics of asynchronous programming with Rust and
builds up from there. Following this tutorial, you will implement only a subset
of Redis commands, but doing so provides a comprehensive tour of Tokio.

# Mini-Redis

The final project built in this tutorial is available as [Mini-Redis on
GitHub][mini-redis]. Mini-Redis is designed with the primary goal of learning
Tokio, and is therefore very well commented. However, this also means that
Mini-Redis is missing some features you would expect from a real Redis library;
you can find such production-ready libraries on [crates.io](https://crates.io/).

For simplicity, this tutorial starts by interacting with the existing fully
functional implementation of Mini-Redis, but you will reimplement some parts
later in the process.

# Getting Help

At any point, if you get stuck, you can always get help on [Discord] or [GitHub
discussions][disc]. Don't worry about asking "beginner" questions. We all start
somewhere and are happy to help.

[discord]: https://discord.gg/tokio
[disc]: https://github.com/tokio-rs/tokio/discussions

# Prerequisites

You should already be familiar with [Rust]. The [Rust book][book] is an
excellent resource to get started with.

While not required, some experience with writing networking code using the
[Rust standard library][std] or another language can be helpful.

No pre-existing knowledge of Redis is required.

[rust]: https://rust-lang.org
[book]: https://doc.rust-lang.org/book/
[std]: https://doc.rust-lang.org/std/

## Rust

Before getting started, you should make sure that you have the
[Rust][install-rust] toolchain installed and ready to go. If you don't have it,
the easiest way to install it is using [rustup].

This tutorial requires a minimum of Rust version `1.45.0`, but the most
recent stable version of Rust is always recommended.

To check that Rust is installed on your computer, run the following:

```bash
$ rustc --version
```

You should see some output like `rustc 1.46.0 (04488afe3 2020-08-24)`.

## Mini-Redis server

Next, install the Mini-Redis server. As explained, this will be used to test
our client as we build it.

```bash
$ cargo install mini-redis
```

Check your installation by starting the server:

```bash
$ mini-redis-server
```

Then, in a separate terminal window, try to get the value for the key `foo`
using `mini-redis-cli` tool

```bash
$ mini-redis-cli get foo
```

The output should be `(nil)`.

# Ready to go

With everything installed and tested, you are ready to go. Navigate to the next
page in order to write your first asynchronous Rust application.

[redis]: https://redis.io
[mini-redis]: https://github.com/tokio-rs/mini-redis
[install-rust]: https://www.rust-lang.org/tools/install
[rustup]: https://rustup.rs/

title: "tibrv-rs - Migrating to Tokio 0.1"
published_date: "2018-06-05 09:00:00 +0000"
layout: default.liquid
is_draft: false
---
In the three months since the first 'real' release of tibrv-rs (v0.2.0)
the Rust community have been hard at work revamping the Tokio and Futures
projects, aiming to make asynchronous programming in Rust easier to understand.
Unfortunately, this means a few changes are required to libraries which
are already using Tokio components.

While the changes to futures are important in the wider Rust context,
of significance to tibrv-rs is the introduction of the `tokio` facade crate,
and the new 'Tokio Runtime': a 'batteries included' reactor and
work-stealing thread pool.

These new defaults place a couple of restrictions on the Futures the executor
can handle, mainly that thread-pooling requires all the members of the
Future to have a `'static` lifetime.

More information on the refactor can be found here:

* [Futures 0.2 RFC](https://github.com/rust-lang-nursery/futures-rfcs/blob/master/futures-02.md)
* [Tokio Reform RFC](https://github.com/tokio-rs/tokio-rfcs/blob/master/text/0001-tokio-reform.md)
* <https://tokio.rs/blog/2018-02-tokio-reform-shipped/>
* <https://tokio.rs/blog/2018-03-tokio-runtime/>

## Implications for tibrv-rs

Two changes have been made to accomodate the new requirements for Futures:

* `RvCtx` handles are now consumed rather than borrowed.
* `Subscription` and `AsyncSub` now wrap their own `Queue` or `AsyncQueue` rather than borrowing
an existing queue.

The first is really just a change in syntax: you will need to `.clone()` the context rather than provide it
as a borrow.

```rust
// tibrv-rs 0.2.0
let ctx = RvCtx::new().unwrap();
let tp = TransportBuilder::new(&ctx).create().unwrap();

// tibrv-rs 0.3.0
let ctx = RvCtx::new().unwrap();
let tp = TransportBuilder::new(ctx.clone()).create().unwrap();
```

The second change may have more of an impact, depending on your use case.

Queues are now owned and managed by subscriptions, and it's no longer possible to
subscribe to multiple Rendezvous subjects on a single event queue. This could potentially
have performance implications for users subscribing to very large numbers of subjects, but
hopefully the flexibility gained from the new Tokio runtime will offset this potential loss.

`Subscription` and `AsyncSub` are now obtained directly from the transport:

```rust
// tibrv-rs 0.2.0
let queue = AsyncQueue::new(&ctx).unwrap();
let sub1 = queue.subscribe(&core.handle, &transport, "TEST").unwrap();
let sub2 = queue.subscribe(&core.handle, &transport, "TEST2").unwrap();


// tibrv-rs 0.3.0
let transport = TransportBuilder::new(ctx.clone()).create().unwrap();

let sub1 = transport.async_sub(&handle, "TEST").unwrap();
let sub2 = transport.async_sub(&handle, "TEST2").unwrap();
```

### tokio::run() and tokio::spawn()

Finally, the new methods for bootstrapping the runtime and spawning a task
onto it have slightly stronger bounds on the types of the Future they accept.

To be run via `run()` or `spawn()`, the Future must be of type `Future<Item = (), Error = ()> + Send + 'static`

Previous versions of the tibrv-rs examples show running the `Stream` or `Sink` to completion
by passing it directly to `core.run()`:

```rust
let events = incoming.and_then(|mut msg| {
    msg.set_send_subject("REPLY").unwrap();
    Ok(msg)
}).forward(transport);

core.run(events).unwrap();
```

In tibrv-rs 0.3.0, you need to map the Sink or Stream to a Future producing `()`:

```rust
let events = incoming.and_then(|mut msg| {
    msg.set_send_subject("REPLY").unwrap();
    Ok(msg)
}).forward(transport).then(|_| Ok(()));

tokio::run(events);
```

title: tibrv-rs - Migrating to Tokio 0.1
layout: default.liquid
published_date: "2018-03-13 00:26:51 +0000"
is_draft: false
---
In the three months since the first 'real' release of tibrv-rs (v0.2.0)
the Rust community have been hard at work revamping the Tokio and Futures
projects, aiming to make asynchronous programming in Rust a bit easier to
grok, especially for people new to the language. Unfortunately, this means
a few changes are required to libraries which already use Tokio components.

While the changes to futures are important in the wider Rust context,
of significance to tibrv-rs is the introduction of the `tokio` facade crate,
and the new 'Tokio Runtime': a bundled 'batteries included'
reactor and work-stealing thread pool.

These new defaults place a couple of restrictions on the Futures the executor
can handle, mainly that the thread-pooling requires that all the members of the
Future have the `'static` lifetime.

More information on the refactor can be found here:

* [Futures 0.2 RFC](https://github.com/rust-lang-nursery/futures-rfcs/blob/master/futures-02.md)
* [Tokio Reform RFC](https://github.com/tokio-rs/tokio-rfcs/blob/master/text/0001-tokio-reform.md)
* <https://tokio.rs/blog/2018-02-tokio-reform-shipped/>
* <https://tokio.rs/blog/2018-03-tokio-runtime/>

## Implications for tibrv-rs

Two main changes have been made to accomodate the new requirements for Futures:

* `RvCtx` handles are now consumed rather than borrowed.
* `Subscription` and `AsyncSub` now consume their underlying `Queue`/`AsyncQueue` rather than borrow it.

The first is really just a change in syntax: you will need to `.clone()` the context rather than provide it
as a borrow.

```rust
// tibrv-rs 0.2.0
let ctx = RvCtx::new().unwrap();
let tp = TransportBuilder::new(&ctx).create().unwrap();
let queue = AsyncQueue::new(&ctx).unwrap();

// tibrv-rs 0.3.0
let ctx = RvCtx::new().unwrap();
let tp = TransportBuilder::new(ctx.clone()).create().unwrap();
let queue = AsyncQueue::new(ctx.clone()).unwrap();
```

The second change may have more of an impact, depending on your use case.

As queues are now consumed by subscriptions, it's no longer possible to subscribe to
multiple Rendezvous subjects on a single queue. This could have performance implications
for users subscribing to a large number of different subjects, but hopefully the
flexibility gained from the new Tokio Runtime should offset this potential loss.

```rust
// tibrv-rs 0.2.0
let queue = AsyncQueue::new(&ctx).unwrap();
let sub1 = queue.subscribe(&core.handle, &transport, "TEST").unwrap();
let sub2 = queue.subscribe(&core.handle, &transport, "TEST2").unwrap();


// tibrv-rs 0.3.0
let queue1 = AsyncQueue::new(ctx.clone()).unwrap();
let queue2 = AsyncQueue::new(ctx.clone()).unwrap();

let sub1 = queue1.subscribe(&handle, &transport, "TEST").unwrap();
let sub2 = queue2.subscribe(&handle, &transport, "TEST2").unwrap();
```

It's likely that in future the concept of `Queue` and `AsyncQueue` will be
hidden behind the `Subscription` and `AsyncSub` types, as without the ability
to subscribe to multiple subjects the distinction is more confusing than helpful.

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

title: Beautiful Pipelines in Rust
published_date: "2018-07-10 08:00:00 +0100"
layout: default.liquid
---
Often in data processing we need to perform a series of separate but
sequentially dependent steps on a data set. This 'pipeline' can be
thought of as a linear state machine, often with `Error` and `Complete`
states to indicate that the pipeline is finished.
A naïve implementation might be to use a single `run()` function to
represent the entire pipeline, but with the help of Rust's type system,
we can do a lot better.

This post, and the patterns implemented, were heavily inspired by Samuel
Zeller's [Pretty State Machine Patterns in Rust](https://hoverbear.org/2016/10/12/rust-state-machine-pattern/)

## Representing State

Two types of state exist in a pipeline, global state which may be used
by every stage, and local state which might only be accessed by the
current stage and its immediate successors.

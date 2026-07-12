---
title: "Come Async You Are"
date: 2026-07-09
summary: "ruxe's async write path: getting events into a synchronous store from any task, without tying the library to an async runtime."
tags: ["rust", "async", "library-design", "redux", "learning"]
draft: false
---

*Bring your own async runtime to ruxe.*

No, this post isn't about hearing Nirvana on shuffle. It's about async events reaching a synchronous store on their own schedule.

[ruxe](https://github.com/corentin-core/ruxe) is my Redux-flavored Rust learning library: state lives in one store, and it changes only when you dispatch an event through a reducer. [Last time](/posts/ruxe-type-level-disjointness/) I got the compiler to guarantee that each state slice has exactly one reducer, which is what makes running them in parallel safe, without a lock. That post was about how the store turns an event into new state. This one is about the step before that: how events reach the store.

The store is synchronous. On its own it has no way to take an event from work that finishes *later*, and real systems are mostly made of that kind of work: an MQTT message coming in, a system signal firing, an SMS arriving over mmcli, or a periodic job ticking away. In the energy-management systems I have in mind, a service polls the meters and inverters on a period and feeds each reading into the store as an event.

{{< mermaid >}}
flowchart LR
    subgraph add["Async work — what this post adds"]
        direction TB
        S["socket frame"]
        T["timer tick"]
        J["finished job"]
    end
    S -.-> Store
    T -.->|dispatch, later| Store
    J -.-> Store
    App([App code]) -->|dispatch event| Store
    Store -->|state + event| Reducer
    Reducer -->|new state| Store
    Store -->|read| App
{{< /mermaid >}}

Adding async isn't hard. The constraint I put on it matters more: it shouldn't make ruxe depend on an async runtime. Someone on tokio, on async-std, on smol, or on their own executor should be able to use it. Keeping that constraint is less obvious than it sounds.

## Why not just ship the synchronous store?

A fair question before the design: why build an async layer into ruxe? It could stop at the synchronous store and let you wire it into your own runtime.

The reason not to is that driving a single-writer store from many async tasks has an obvious do-it-yourself answer that's wrong. Better to provide the one correct driver than have every user rebuild it, a little differently each time. And it costs nothing if you don't use it: the synchronous store is unchanged, and async stays opt-in. What ruxe ships is one primitive, not a framework.

## What "runtime-agnostic" actually costs

Rust's async is runtime-agnostic at the language level: `async`/`await` and the `Future` trait live in the standard library. The standard library then stops. There's no executor and no `spawn`, so running a task means naming a runtime.

For a library, that's a real constraint: spawn a task and you've picked a runtime for everyone who depends on you. The [async book](https://rust-lang.github.io/async-book/08_ecosystem/00_chapter.html) puts it plainly:

> Libraries exposing async APIs should not depend on a specific executor or reactor, unless they need to spawn tasks or define their own async I/O or timer futures. Ideally, only binaries should be responsible for scheduling and running tasks.

Most libraries skip it, because staying agnostic costs conditional compilation and shims, and so the ecosystem [standardized on tokio](https://corrode.dev/blog/async/), now a dependency of tens of thousands of crates. There's even a wg-async story, ["Barbara writes a runtime-agnostic library"](https://rust-lang.github.io/wg-async/vision/submitted_stories/status_quo/barbara_writes_a_runtime_agnostic_lib.html), on how awkward the path is.

For an application, committing to tokio is fine. A library might run somewhere tokio can't. Tokio needs an operating system and the standard library, and plenty of embedded targets have neither; they run async on [embassy](https://embassy.dev) instead. My own hardware runs a full Linux, so tokio would work for me. But I can't decide the runtime for the people who use ruxe.

ruxe does no I/O. It never reads a byte; the socket, the MQTT client, the timer live in the user's tasks, and only events cross into the store. (I/O is the sharper trap: tokio's `AsyncRead` isn't the one async-std and smol use, so touching a socket picks a camp.) This is the same split [sans-IO](https://www.firezone.dev/blog/sans-io) protocol libraries make: the logic is a pure state machine, the I/O and the runtime are the caller's. ruxe just gets it for free, since a store has no bytes to move. The other half is spawning, and it's the rule the rest of the design builds on: **ruxe never spawns.**

## What "never spawn" forces

That line does more than it looks. If the library can't spawn, it can't run a background loop of its own, which means it can't own the store inside a task nobody handed it. Someone else has to drive it.

Start from ownership. `dispatch` takes `&mut self`:

```rust
pub fn dispatch(&mut self, event: E) -> Result<(), DispatchError>
```

One writer at a time, enforced by the compiler. Async ingestion means many tasks, on many threads, all wanting to dispatch. They can't all hold `&mut Store`.

`Arc<Mutex<Store>>` would compile. I didn't want it. It adds contention, and it gives up the property that makes the store worth having: dispatch as a single ordered sequence of state transitions. A mutex serializes too, but in lock-wakeup order, and every producer blocks on every dispatch.

The alternative is to stop sharing the store. One owner, in a loop the user drives: it owns the store outright, pulls events off a channel, and dispatches them one at a time. Everyone else gets a cheap, clonable handle to the sending end of the channel. That's [an actor](https://ryhl.io/blog/actors-with-tokio/). I didn't reach for the pattern by name; it's just what a single owner of `&mut Store`, fed by many tasks and forbidden from spawning, turns into.

{{< mermaid >}}
flowchart LR
    subgraph rt["User's runtime — ruxe never spawns"]
        T1["task: meter poller"]
        T2["task: MQTT ingest"]
    end
    T1 --> H["DispatchHandle<br/>Clone + Send + 'static"]
    T2 --> H
    H --> Q(["bounded mpsc"]) --> L
    subgraph core["Actor loop = sole owner of the Store"]
        L["run(): dispatch events serially"]
    end
{{< /mermaid >}}

The public surface is one constructor:

```rust
let (handle, actor_loop) = init_actor_loop(store, 32); // 32 = channel buffer size

// Move the handle into your producer task (clone it for more than one).
tokio::spawn(async move { produce(handle).await });

let store = actor_loop.run().await?; // returns once every handle has dropped
```

Two details matter. The channel is `futures::channel::mpsc`, not tokio's, so no runtime dependency slips in. And it's bounded: a full channel applies back-pressure to the producers, and on shutdown `run()` drains what's buffered before handing the `Store` back.

## Where the effects go

Anyone coming from Redux will ask about [redux-thunk](https://github.com/reduxjs/redux-thunk): a middleware that fires an async job and dispatches the result. In ruxe a middleware can't hold a dispatch handle, for ownership reasons that are their own post. The rule that replaces it is simpler: everything that touches the store lives outside the loop and holds a handle. Producers, timers, effects, and subscribers are the same shape, and the loop is the one place every dispatch is serialized.

{{< mermaid >}}
flowchart LR
    subgraph out["Outside the loop — each holds a handle"]
        P["producers, timers"]
        E["effects, subscribers"]
    end
    L["the loop<br/>every dispatch serialized here"]
    P -->|dispatch| L
    E -->|dispatch| L
{{< /mermaid >}}

## What's next

This post shipped the primitive: the actor loop and the handle. The rest of ruxe's async story builds on it. Reacting to state and to the event stream become external subscribers, and spawning the loop on a multi-threaded runtime becomes a feature-gated adapter. All still ahead, all outside the loop.

{{< mermaid >}}
flowchart TB
    P["Actor loop + DispatchHandle<br/>shipped in this post"]
    P -.-> A["state subscribers<br/>planned — #35"]
    P -.-> B["event-stream subscribers<br/>planned — #36"]
    P -.-> C["tokio adapter, feature-gated<br/>planned — #37"]
{{< /mermaid >}}

---

*Source code: [ruxe on GitHub](https://github.com/corentin-core/ruxe). Pull request that landed this: [#38](https://github.com/corentin-core/ruxe/pull/38). Design discussion: [issue #23](https://github.com/corentin-core/ruxe/issues/23).*

*Drafted with Claude (Anthropic), then reworked over many passes. The design, the decisions, and every claim I kept are mine.*

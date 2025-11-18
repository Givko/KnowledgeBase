# Part 3 - Async/Await Patterns: How Non-Blocking I/O Works

In the previous parts, we built an epoll-based chat server and then migrated it to
Tokio. But what's actually happening under the hood when we write `async fn` and `.await`?

This article explores the universal patterns that make async/await possible across languages like Rust, Go, Python, JavaScript, and C#. While implementations differ, the core concepts—**reactor**, **executor**, and **futures**—remain remarkably similar.

## What You'll Learn

- The three components of async systems: reactor, executor, and futures
- How async functions compile to state machines
- The difference between cooperative (async) and preemptive (threads) scheduling
- Why async is efficient for I/O-bound work but not CPU-bound work
- How these patterns appear across different programming languages

## The Three Components: Reactor, Executor, and Futures

Every async system is built from three core components working together. Let's start with the most visible one: futures.

### Futures: Representing Work Not Yet Complete

A **future** (or **promise** in JavaScript, **Task** in C#) represents a unit of work that will eventually complete—but not right now.

#### How Futures Work

**State machine representation**:

- In most languages, including Rust, the compiler transforms `async fn` into a state machine
- Each `.await` point becomes a state transition
- The future remembers where it paused and what data it needs to resume

**Two states in Rust**:

- `Poll::Pending`: Work is not yet complete, check back later
- `Poll::Ready(value)`: Work is done, here's the result

**Key rule**: Once a future returns `Poll::Ready`, it must never be polled again.

**Why?** Because `Poll::Ready(value)` moves the result out of the future, consuming it. Polling again would be asking for a value that's already been taken—violating the `Future` trait contract and potentially causing panics or undefined behavior.

Think of it like a vending machine: once it dispenses your item, you can't press the button again expecting another one. The executor tracks completed futures and removes them from the schedule.

#### Important Differences Across Languages

**Rust (lazy execution)**:

- Futures do **nothing** until `.await`-ed
- You must explicitly drive them by calling `.await` or spawning them on an executor
- Example: `let fut = async_operation();` does nothing until you `.await fut`

**C# and JavaScript (eager execution)**:

- Tasks/Promises start executing immediately when created
- `.await` (C#) or `.then()` (JS) just waits for the already-running work
- Example: `var task = DoWorkAsync();` starts work immediately

**Why this matters**: Rust gives you precise control over when work starts; C#/JS optimize for simplicity.

#### The Non-Blocking Rule

**Critical invariant**: Futures must never use blocking operations.

Why? Because futures run on a shared executor (coming next). If one future blocks, it prevents other futures from making progress.

**Example**:

- ❌ Bad: `std::net::TcpStream` (blocking)
- ✅ Good: `tokio::net::TcpStream` (returns `Poll::Pending`, not blocking)

When a non-blocking operation would block, it returns `WouldBlock` (or `Poll::Pending` in Rust), allowing the executor to switch to another future.

### Executor: Driving Futures to Completion

An **executor** (also called a **scheduler** or **runtime** in some languages) is responsible for running futures and driving them to completion. It repeatedly polls pending futures until they're ready.

Think of it as a conductor managing an orchestra: it asks each musician (future) "are you ready?" and coordinates when each plays their part.

#### How Executors Work (Simplified)

The explanation below is simplified for learning—real implementations are more sophisticated, but the core idea remains the same.

**The polling loop**:

1. Executor maintains a queue of futures that need to run
2. It loops through the queue, calling `poll()` on each future
3. If a future returns `Poll::Pending`, it stays in the queue
4. If a future returns `Poll::Ready(value)`, the result is returned to the `.await` site and the future is removed

**Why futures must never block**:

If one future blocks (e.g., waits synchronously for I/O), it freezes the entire executor loop. No other futures can make progress until the blocking operation completes. This defeats the purpose of async—one slow task shouldn't prevent thousands of others from running.

This is why we use `tokio::net::TcpStream` (non-blocking) instead of `std::net::TcpStream` (blocking).

#### How Tokio Does It (Simplified)

**Multi-threaded work-stealing executor**:

- `#[tokio::main]` creates an executor with a **thread pool** (default: one thread per CPU core)
- Each worker thread has its own queue of futures
- Workers **steal** tasks from each other when idle (work-stealing scheduler)
- This provides parallelism across CPU cores while maintaining efficiency

**Concurrency vs. Parallelism**:

- **`.await`**: Runs the future on the current task, cooperatively with other futures on the same thread (concurrent, not parallel)
- **`tokio::spawn`**: Creates a new task that can run on any thread in the pool (enables parallelism)

**Example**:

```rust
// Concurrent: runs on current task, yields at .await
let result = fetch_data().await;

// Parallel: spawns separate task, can run on different thread
let handle = tokio::spawn(async {
  fetch_data().await
});

```

**Key insight**: Tokio's multi-threaded executor provides both cooperative concurrency (many futures per thread) and parallelism (multiple threads), giving you the best of both worlds for I/O-bound workloads.

### Reactor: The I/O Watcher

The **reactor** is the component that monitors I/O readiness using OS primitives (epoll on Linux, kqueue on BSD/macOS, IOCP on Windows). When I/O becomes ready, it notifies the executor that a specific future can make progress.

Think of it as a notification system: futures register interest in I/O events ("wake me when this socket is readable"), and the reactor alerts them when it happens.

#### How It Works: The Full Picture

Let's connect all the pieces with a more detailed look at how Tokio coordinates futures, the executor, and the reactor.

**Two queues per worker thread**:

In reality, each Tokio worker thread maintains two queues:

- **Run queue**: Futures ready to make progress (executor polls these)
- **Waiting list**: Futures waiting for I/O (registered with reactor)

**The cycle**:

1. Executor polls a future from the run queue
2. Future tries to read from a socket → socket not ready yet
3. Future returns `Poll::Pending` and registers with reactor
4. Executor moves future to waiting list, polls next future
5. Reactor monitors all registered file descriptors using epoll
6. When FD becomes ready, reactor moves corresponding future back to run queue
7. Executor polls that future again → socket now ready → future makes progress

#### The Waker Mechanism

**How does the reactor know which future to wake?**

When a future returns `Poll::Pending`, it must ensure it will be polled again when progress is possible. It does this through the **waker** pattern:

1. Future registers with the reactor: "Wake me when FD 42 (this socket) is readable"
2. Reactor stores the mapping: `FD 42 → Waker (handle to this future)`
3. Reactor calls `epoll_wait()` to monitor all registered FDs
4. When `epoll_wait()` returns "FD 42 is ready", reactor calls the corresponding `Waker`
5. Waker moves the future back to the executor's run queue

**This is the bridge** between the OS (epoll) and Rust's async system (futures).

#### Connection to Your epoll Implementation

Remember your epoll chat server from Part 1? You manually:

- Called `epoll_wait()` to check which sockets were ready
- Looked up the connection from the token/FD
- Performed the read/write operation

Tokio's reactor automates this:

- Futures register themselves
- Reactor manages the epoll loop (you don't call `epoll_wait` manually)  
- Wakers notify the executor (you don't manually dispatch to handlers)

**The pattern is the same—Tokio just provides the abstraction.**

#### Implementation Details

The exact mechanism varies by runtime:

- **Tokio**: Uses `mio` library (cross-platform I/O abstraction over epoll/kqueue/IOCP)
- **async-std**: Similar architecture with own reactor
- **smol**: Lightweight reactor with simpler implementation

But the pattern—futures register interest, reactor monitors FDs, waker notifies executor—is universal across Rust async runtimes.

```
┌─────────────────────────────────┐          ┌──────────────────────────────┐
│          EXECUTOR               │          │         REACTOR              │
│                                 │          │                              │
│  State:                         │          │  State:                      │
│  • Run Queue: [Future A, B]     │          │  • Watching FDs via epoll:   │
│  • Wait List: [Future C, D]     │          │    - FD 42 → Future C        │
│                                 │          │    - FD 13 → Future D        │
│  polls futures from run queue   │          │                              │
│                                 │          │  epoll_wait() blocks until   │
│                                 │          │  any FD becomes ready        │
│                                 │          │                              │
└─────────────┬───────────────────┘          └──────────────┬───────────────┘
              │                                             │
              │                                             │
              │         ┌───────────────────┐               │
              │         │      WAKER        │               │
              │         │   (notification)  │               │
              │         └───────────────────┘               │
              │                   ▲                         │
              │                   │                         │
              │  3. Register:     │      5. Wake:           │
              │  "Call this       │      "FD 42 ready!      │
              │   waker when      │       Call waker"       │
              │   FD 42 ready"    │                         │
              └───────────────────┴─────────────────────────┘
                                  │
                                  │
                                  ▼
                      ┌───────────────────────┐
                      │      FUTURE C         │
                      │                       │
                      │  async fn fetch() {   │
                      │    socket.read()      │
                      │      .await           │
                      │  }                    │
                      │                       │
                      │  Returns:             │
                      │  • Poll::Pending      │
                      │    (when not ready)   │
                      │  • Poll::Ready(data)  │
                      │    (when complete)    │
                      └───────────────────────┘
```

## Cooperative vs. Preemptive Scheduling

Understanding the difference between cooperative and preemptive scheduling is crucial for grasping why async systems work the way they do—and their limitations.

### Preemptive Scheduling: How OS Threads Work

**Preemptive scheduling** means the scheduler forcibly interrupts running tasks, whether they want to yield or not.

**How the CPU scheduler works**:

- The kernel runs each thread for a time slice (typically 1-10ms)
- When the time slice expires, a **timer interrupt** fires
- The kernel forcibly pauses the thread (saves its state)
- Kernel switches to another thread (context switch)
- Threads have **no control** over when they're paused

**Why this works**:

- Guarantees fairness: no thread can hog the CPU indefinitely
- Handles misbehaving code: even infinite loops get interrupted
- Provides isolation: one bad thread can't freeze the system

**The cost**:

- Context switches are expensive (save/restore CPU state, TLB flush)
- Each thread needs its own stack (typically 2MB on Linux)
- Kernel overhead: scheduling decisions, interrupt handling

### Cooperative Scheduling: How Async/Await Works

**Cooperative scheduling** means tasks voluntarily yield control—the scheduler cannot force them to stop.

**How async executors work**:

- Executor polls a future
- Future runs until it hits `.await` or returns `Poll::Ready`
- Future **voluntarily yields** by returning `Poll::Pending`
- Executor switches to the next future in the queue
- No forced interruption—futures must be "good citizens"

**Why this works**:

- Extremely cheap task switching (just a function call, no kernel involvement)
- Thousands of tasks can share a single OS thread
- Minimal memory overhead: futures are just state machines on the heap/stack

**The limitations**:

- One blocking future freezes the entire thread
- No protection against misbehaving code
- CPU-bound work must manually yield or use `tokio::spawn_blocking`

### Key Differences

| Aspect | Preemptive (OS Threads) | Cooperative (Async Tasks) |
|--------|------------------------|---------------------------|
| **Who decides when to switch?** | Kernel (timer interrupt) | Task (`.await` points) |
| **Can force interruption?** | Yes | No |
| **Context switch cost** | High (~1-10µs) | Very low (~10-100ns) |
| **Memory per task** | ~2MB (stack) | Bytes to KBs (state machine) |
| **Protection from blocking** | Yes | No |
| **Best for** | CPU-bound work, isolation | I/O-bound work, high concurrency |
| **Scalability** | ~10,000 threads max | Millions of tasks |

### Why Async Uses Cooperative Scheduling

**For I/O-bound workloads**, cooperative scheduling is optimal:

- Most time is spent **waiting** (network, disk, database)
- Tasks naturally yield at I/O boundaries (`.await`)
- Cheap switching enables handling 100,000+ concurrent connections
- Example: Your chat server handles many clients on just a few threads

**For CPU-bound workloads**, cooperative scheduling breaks down:

- Task computes for seconds without yielding → blocks executor
- Other tasks starve (can't make progress)
- Solution: Use `tokio::spawn_blocking` to run on dedicated thread pool

### Real-World Analogy

**Preemptive (OS threads)**: A teacher with a timer who strictly enforces "each student gets 2 minutes to speak." Fair but overhead-heavy (constantly managing the timer, interrupting students mid-sentence).

**Cooperative (async tasks)**: Students voluntarily pass the microphone when they're done speaking. Efficient when everyone cooperates, but one long-winded student can monopolize the discussion.

### Connection to Your Learning

In your epoll server (Part 1), you manually implemented cooperative scheduling:

- Your event loop polled each connection
- Each connection's read/write was non-blocking
- You explicitly moved to the next connection

Tokio abstracts this pattern, but the principle is identical: **voluntary yielding enables efficient I/O multiplexing on a single thread.**

## How These Patterns Appear Across Languages

While the core concepts remain the same, each language implements async differently. Here's how the reactor-executor-future trinity maps across popular languages.

### Rust (Tokio)

| Component | Implementation | Notes |
|-----------|---------------|-------|
| **Future** | `Future` trait | Poll-based, lazy (doesn't start until `.await`-ed) |
| **Executor** | Tokio runtime with work-stealing scheduler | Multi-threaded by default |
| **Reactor** | `mio` (wraps epoll/kqueue/IOCP) | Explicit, pluggable |

**Key characteristic**: **Explicit and zero-cost**. You choose your runtime (Tokio, async-std, smol) and have fine-grained control.

---

### Go

| Component | Implementation | Notes |
|-----------|---------------|-------|
| **Future** | Channels (no explicit futures) | Goroutines communicate via channels, not futures |
| **Executor** | Go runtime scheduler (M:N model) | Built into language, M goroutines on N OS threads |
| **Reactor** | Netpoller (part of runtime) | Uses epoll/kqueue internally, fully hidden |

**Key characteristic**: **Implicit and simple**. Everything is built-in; you just write `go func()` and the runtime handles everything. No explicit async/await syntax—blocking operations automatically yield.

**Important difference**: Go doesn't expose the future abstraction. Goroutines + channels replace explicit async/await.

---

### JavaScript (Node.js)

| Component | Implementation | Notes |
|-----------|---------------|-------|
| **Future** | `Promise` | Callback-based internally, eager (starts immediately) |
| **Executor** | Event loop | Single-threaded (but can spawn worker threads) |
| **Reactor** | `libuv` | Cross-platform I/O library (wraps epoll/kqueue/IOCP) |

**Key characteristic**: **Single-threaded event loop**. All async operations run on one thread; parallelism requires worker threads.

**Example**:

**Important difference**: Promises are eager (start immediately when created), unlike Rust futures (lazy).

---

### C# (.NET)

| Component | Implementation | Notes |
|-----------|---------------|-------|
| **Future** | `Task<T>` | Eager (starts immediately when created) |
| **Executor** | Task Scheduler (ThreadPool-based) | Multi-threaded, uses .NET ThreadPool |
| **Reactor** | IOCP on Windows, epoll/kqueue on Linux/macOS | Hidden behind `async`/`await` |

**Key characteristic**: **Enterprise-focused with excellent tooling**. Tasks start immediately and the runtime handles everything.

**Important differences**:

- Tasks are eager like JavaScript (not lazy like Rust)
- `async`/`await` can run on different threads (unlike JS single-threaded model)

## Resources and Further Reading

Want to dive deeper into async patterns and implementations? Here are carefully selected resources organized by topic.

### Video Tutorials

**Jon Gjengset's Async Rust Series** (Highly Recommended)

- [The What and How of Futures and async/await in Rust](https://www.youtube.com/watch?v=9_3krAQtD2k) - Explains Future trait, poll, and executor internals
- [Crust of Rust: async/await](https://www.youtube.com/watch?v=ThjvMReOXYM) - Live coding session building async primitives from scratch
- [Decrusting Tokio](https://youtu.be/o2ob8zkeq2s?si=Lqvepna56SO3BwHP)

---

### Books

**Operating Systems Concepts** (The foundational knowledge)

- [Operating Systems: Three Easy Pieces](https://pages.cs.wisc.edu/~remzi/OSTEP/) - Free online book
  - Chapter 26: Concurrency - Threads vs. Processes
  - Chapter 28: Locks - Synchronization primitives
  - Chapter 30: Condition Variables - Coordination patterns
  - Chapter 33: Event-based Concurrency - The reactor pattern foundation

Understanding OS-level scheduling and I/O is crucial for grasping why async works the way it does.

---

### Official Documentation

**Rust Async**

- [Async Book](https://rust-lang.github.io/async-book/) - Official guide to async Rust
- [Tokio Tutorial](https://tokio.rs/tokio/tutorial) - Hands-on Tokio learning
- [Tokio API Docs](https://docs.rs/tokio/latest/tokio/) - Complete API reference

**Go Concurrency**

- [Effective Go: Concurrency](https://go.dev/doc/effective_go#concurrency) - Official concurrency guide

**JavaScript Async**

- [MDN: async/await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function) - Complete async/await reference
- [MDN: Promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise) - Understanding Promise internals
- [Node.js Event Loop Guide](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/) - How Node.js handles async I/O

**C# Async**

- [Async programming with async and await](https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/) - Official Microsoft guide
- [Task-based Asynchronous Pattern](https://learn.microsoft.com/en-us/dotnet/standard/asynchronous-programming-patterns/task-based-asynchronous-pattern-tap) - Deep dive into Task model
- [ConfigureAwait FAQ](https://devblogs.microsoft.com/dotnet/configureawait-faq/) - Understanding context switching in C#

---

- [The C10K Problem](http://www.kegel.com/c10k.html) - The historical problem async solves
- [epoll man page](https://man7.org/linux/man-pages/man7/epoll.7.html) - Linux's efficient I/O monitoring

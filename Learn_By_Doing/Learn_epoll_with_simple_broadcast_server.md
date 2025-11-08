# Learn epoll by Building a Simple Broadcast Server

Most network applications can't afford one thread per connection. With 10,000
concurrent clients, they'd need 10,000 threads. Instead, they use event-driven
IO - a single thread that handles thousands of connections by asking the kernel:
"who's ready to read? who's ready to write?"

This is how nginx, Redis, Node.js, and Tokio work. Under the hood: epoll
(Linux's event notification system).

## What We'll Build

A TCP server that accepts multiple connections and broadcasts incoming messages
to all clients. The protocol is deliberately simple (newline-delimited text) so
we can focus on epoll mechanics, not application logic.

## Learning Approach

**This is a hands-on tutorial, not epoll theory.** We'll build working code in
progressive milestones and explain concepts just-in-time as we need them. For
deeper theoretical understanding (epoll syscall internals, kernel implementation,
performance characteristics), see the Resources section at the end.

## What You'll Learn

- How epoll monitors thousands of sockets on one thread
- Non-blocking sockets and handling WouldBlock
- TCP stream fragmentation and message parsing
- Write buffer management for partial sends
- Dynamic Interest registration (edge-triggered events)

## Prerequisites

- Basic Rust (know how to use `TcpListener` and `TcpStream`)
- Understand TCP is a byte stream, not message-oriented
- No prior epoll/async experience needed

## Testing

You'll verify it works at each step using `netcat` - no special tools needed.

Let's build it.

## Milestone 1: Building a Simple Event Loop That Accepts Connections

### Setting Up the Event Loop

First we'll create a basic event loop that polls a TCP listener for READ events.
We'll be using [mio](https://docs.rs/mio/latest/mio/index.html) (Rust's epoll wrapper).

#### **Step 1: Create the project**

```bash
cargo new rat
cd rat
cargo add mio --features os-poll,net
```

We're calling it "rat" (short for Rust Chat - or just a fun name).

#### **Step 2: Add imports**

In `src/main.rs`:

```rust
use mio::{Events, Interest, Poll, Token};
use mio::net::TcpListener;

fn main() {
// Code goes here
}
```

#### **Step 3: Create core components**

```rust
let mut poll = Poll::new().unwrap();
let mut events = Events::with_capacity(1024);
let mut listener = TcpListener::bind("127.0.0.1:8080".parse().unwrap())
.expect("Unable to bind");
let listener_token = Token(0);
```

**What each does:**

- `Poll` - our epoll instance (talks to kernel)
- `Events` - buffer for event notifications (holds up to 1024 events per poll)
- `TcpListener` - listening socket bound to port 8080
- `Token(0)` - unique ID for the listener (we'll use Token(1+) for clients)

**Note:** We're using `unwrap()` and `expect()` to keep focus on epoll mechanics,
not error handling.

#### **Step 4: Register the listener**

```rust
poll.registry()
.register(&mut listener, listener_token, Interest::READABLE)
.expect("Failed to register listener");
```

**What this does:**

- Tells epoll: "Monitor this socket for READABLE events"
- READABLE for a listener means "new connection available to accept"
- When a client connects, epoll fires an event with Token(0)

**The Token is how we know which socket is ready.** Later when we have multiple
clients (Token(1), Token(2), etc.), we'll use the token to look up the
correct connection.

#### **Step 5: Create a simple event loop with polling**

```rust
    loop {
        poll.poll(&mut events, Some(Duration::from_millis(100)))
            .expect("failed to poll for events");
    }
```

**What this does:**

- `poll.poll()` blocks until events occur OR timeout expires
- `events` buffer gets populated with ready sockets
- Returns control to our loop (either with events or after 100ms)

**Why the timeout?**  
Without timeout (`None`), poll blocks forever if no events occur. With timeout,
we regain control every 100ms - useful for:

- Graceful shutdown (check exit flag)
- Periodic housekeeping (timeouts, cleanup)
- Not critical for this tutorial, but good practice

**Note:** 100ms is arbitrary - production servers often use longer (1-10 seconds).

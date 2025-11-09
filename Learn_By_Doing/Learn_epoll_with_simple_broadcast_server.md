# Learn *epoll* by Building a Simple Broadcast Server - Part 1: Event Loop and Accepting Connections

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

### What is epoll?

epoll is Linux's event notification system for monitoring multiple file descriptors
(sockets, files, pipes) simultaneously. Instead of checking each socket individually,
you ask the kernel once: "which sockets are ready?"

**One syscall instead of N syscalls** - this is why nginx handles 100,000+
connections on one thread.

**Alternatives:** kqueue (BSD/macOS), IOCP (Windows). mio abstracts these differences.

### Edge-Triggered vs Level-Triggered

**Level-Triggered:** "Notify me **while** condition is true"  

- Socket has 200 bytes to read
- You read 100 bytes  
- epoll fires again immediately: "Still readable"
- Forgiving but generates repeated notifications

**Edge-Triggered (mio default):** "Notify me **when** condition **becomes** true"  

- Socket has 200 bytes to read
- You read 100 bytes
- epoll does NOT fire again (no state change - already readable)
- More efficient but you must read until WouldBlock or lose data

**Throughout this tutorial, we'll handle edge-triggered behavior by reading
until WouldBlock.**

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

#### Step 6: Accept connections

Now we'll actually accept incoming connections.

**First, iterate over events:**

```rust
for event in events.iter() {

}
```

**Second, match on the token to identify which socket is ready:**

Right now we only have Token(0) registered (the listener).

```rust
for event in events.iter() {
    match event.token() {
          Token(0) => {
              // Just in case check. Here we only want read events
              if !event.is_readable() {
                  continue;
              }

              loop {
                  match listener.accept() {
                      Ok((mut stream, addr)) => {
                          println!("Accepted connection");
                      }
                      Err(e) if e.kind() == std::io::ErrorKind::WouldBlock => {
                          println!("no more connection to accept");
                          break;
                      }
                      Err(e) => {
                          eprintln!("failed to accept connection with error {e}");
                          break;
                      }
                  }
              }
          }
          other_token => {
            continue;
          }
    }
}
```

**Let's break down what's happening:**

1. **Match the token:** We check if `event.token()` is Token(0) (our listener).

2. **Check readability:** Verify the event is actually readable (defensive programming).

3. **Accept loop:** This is the critical part for edge-triggered mode. We loop until
   WouldBlock because:
   - Multiple clients might connect between poll calls
   - Edge-triggered only notifies on state change (not-ready → ready)
   - If we only accept one connection, we won't get another notification until
     the next NEW connection arrives
   - Must drain the accept queue completely

4. **Handle WouldBlock:** When `listener.accept()` returns WouldBlock, it means
   the accept queue is empty. This is normal and expected - we break the loop
   and wait for the next epoll event.

5. **Why the loop is necessary:**
   - Scenario: 5 clients connect before our next poll
   - Without loop: Accept 1 client, return to poll, miss 4 clients
   - With loop: Accept all 5 clients in one go

**Note:** The default `std::net::TcpListener` is blocking but can be set to
non-blocking (this is what mio does under the hood). Non-blocking means
operations return immediately with WouldBlock instead of waiting.

### Testing Milestone 1

**Full code so far:**

```rust
use mio::{Events, Interest, Poll, Token};
use mio::net::TcpListener;
use std::time::Duration;

fn main() {
    let mut poll = Poll::new().unwrap();
    let mut events = Events::with_capacity(1024);
    let mut listener = TcpListener::bind("127.0.0.1:8080".parse().unwrap())
    .expect("Unable to bind");
    let listener_token = Token(0);

    poll.registry()
        .register(&mut listener, listener_token, Interest::READABLE)
        .expect("Failed to register listener");

    loop {
        poll.poll(&mut events, Some(Duration::from_millis(100)))
            .expect("failed to poll for events");

        for event in events.iter() {
            match event.token() {
                Token(0) => {
                    if !event.is_readable() {
                        continue;
                    }

                    loop {
                        match listener.accept() {
                            Ok((stream, addr)) => {
                                println!("Accepted connection from {}", addr);
                            }
                            Err(e) if e.kind() == std::io::ErrorKind::WouldBlock => {
                                break;
                            }
                            Err(e) => {
                                eprintln!("Failed to accept: {}", e);
                                break;
                            }
                        }
                    }
                }
                _ => continue,
            }
        }
    }

}
```

**Run the server:**

```bash
cargo run
```

**In another terminal, connect:**

```bash
nc localhost 8080
```

**Expected output:**

```
Accepted connection from 127.0.0.1:xxxxx
```

**Try connecting multiple times quickly** (open 3 terminals, connect all at once):

```
Terminal 1: nc localhost 8080
Terminal 2: nc localhost 8080
Terminal 3: nc localhost 8080
```

**Expected output:**

```
Accepted connection from 127.0.0.1:xxxxx
Accepted connection from 127.0.0.1:xxxxx
Accepted connection from 127.0.0.1:xxxxx
```

**What's happening:** The accept loop processes all pending connections in one
epoll event, thanks to the `loop` + `WouldBlock` pattern.

**Current limitation:** We accept connections but don't store them. They close
immediately. Next milestone: store and register client connections.

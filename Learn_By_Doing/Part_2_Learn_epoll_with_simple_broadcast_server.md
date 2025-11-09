# Learn epoll by Building a Simple Broadcast Server - Part 2: Reading, Parsing, and Broadcasting

In [Part 1](https://github.com/Givko/KnowledgeBase/blob/master/Learn_By_Doing/Part_1_Learn_epoll_with_simple_broadcast_server.md), we built an event loop
that accepts TCP connections. We can detect when clients connect and call
`accept()`, but the connections close immediately because we don't store them.

Today we'll complete the broadcast server. When a client sends "Hello\n", all
other connected clients will receive it.

## What We'll Build

**Milestone 2:** Store accepted connections in a HashMap and register them for
READABLE events

**Milestone 3:** Read data from client sockets (handling edge-triggered mode
with WouldBlock)

**Milestone 4:** Parse complete messages from TCP byte streams (handle message
fragmentation with read buffers)

**Milestone 5:** Broadcast messages to all connected clients

**Milestone 6:** Manage write buffers for partial sends (handle WRITABLE events
and backpressure)

By the end, you'll have a working multi-client broadcast server that demonstrates
the core patterns behind nginx, Redis, and async runtimes.

## Prerequisites

- Complete [Part 1](https://github.com/Givko/KnowledgeBase/blob/master/Learn_By_Doing/Part_1_Learn_epoll_with_simple_broadcast_server.md) first
- Starting code provided below (final code from Part 1)

## What You'll Learn

- Connection lifecycle management (store, register, cleanup)
- Reading until WouldBlock (edge-triggered requirement)
- TCP stream fragmentation and message parsing with read buffers
- Broadcasting pattern (iterate connections, queue messages)
- Write buffer management (track partial sends, handle backpressure)
- Dynamic Interest registration (only request WRITABLE when needed)

## Starting Code

Here's where we left off in Part 1:

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

    text
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
                                // TODO: Store and register the stream
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

**Problem:** Accepted streams are dropped immediately. We need to store them.

Let's fix that.

## Milestone 2: Store and Register Client Connections

First challenge: How do we store multiple client connections and look them up
when epoll fires events?

**Answer:** HashMap with Token as key.

### Step 1: Define ClientConnection struct

We need to store three things per client:

- The TCP stream (for reading/writing)
- A read buffer (accumulate bytes until we have a complete message)
- A write buffer (queue outgoing bytes when socket isn't ready)

```rust
use std::collections::{HashMap, VecDeque};
use mio::net::TcpStream;

struct ClientConnection {
    stream: TcpStream,
    read_buf: VecDeque<u8>, // Accumulate incoming bytes
    write_buf: VecDeque<u8>, // Queue outgoing bytes
}
```

**Why VecDeque?**

- Efficient front removal: `pop_front()` is O(1)
- We'll append bytes to the back, remove from the front
- Alternative: `Vec` requires shifting elements (O(n))

### Step 2: Create storage in main

Add these to your `main()` function (before the event loop):

```rust
// Store connections by token
let mut connections: HashMap<Token, ClientConnection> = HashMap::new();

// Generate unique tokens for each client
let mut next_token_id = 1; // Token(0) is reserved for listener
```

**How tokens work:**

- Token(0) = listener socket (already registered)
- Token(1), Token(2), ... = client sockets
- `next_token_id` increments each time we accept a connection

### Step 3: Store accepted connections

**Replace the TODO in your accept loop with:**

```rust
Ok((mut stream, addr)) => {
    println!("Accepted connection from {}", addr);

    // Generate unique token for this client
    let stream_token = Token(next_token_id);
    next_token_id += 1;

    // Register client socket for READABLE events
    poll.registry()
        .register(&mut stream, stream_token, Interest::READABLE)
        .expect("Failed to register stream");

    // Store connection in HashMap
    let connection = ClientConnection {
        stream,
        read_buf: VecDeque::new(),
        write_buf: VecDeque::new(),
    };
    connections.insert(stream_token, connection);

    println!("Registered client with token {:?}", stream_token);

}
```

**What this does:**

1. Generate Token(1), Token(2), etc. for each new client
2. Register the client's TCP stream with epoll (listen for READABLE)
3. Create ClientConnection with empty read/write buffers
4. Store in HashMap so we can find it later when epoll fires

**Key insight:** When epoll fires an event, we use `event.token()` to look up
the correct connection in our HashMap.

**Full code so far:**

```rust
use mio::{Events, Interest, Poll, Token};
use mio::net::{TcpListener, TcpStream};
use std::collections::{HashMap, VecDeque};
use std::time::Duration;

struct ClientConnection {
    stream: TcpStream,
    read_buf: VecDeque<u8>,
    write_buf: VecDeque<u8>,
}

fn main() {
    let mut poll = Poll::new().unwrap();
    let mut events = Events::with_capacity(1024);
    let mut listener = TcpListener::bind("127.0.0.1:8080".parse().unwrap())
    .expect("Unable to bind");
    let listener_token = Token(0);

    poll.registry()
        .register(&mut listener, listener_token, Interest::READABLE)
        .expect("Failed to register listener");

    let mut connections: HashMap<Token, ClientConnection> = HashMap::new();
    let mut next_token_id = 1;

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
                            Ok((mut stream, addr)) => {
                                println!("Accepted connection from {}", addr);

                                let stream_token = Token(next_token_id);
                                next_token_id += 1;

                                poll.registry()
                                    .register(&mut stream, stream_token, Interest::READABLE)
                                    .expect("Failed to register stream");

                                let connection = ClientConnection {
                                    stream,
                                    read_buf: VecDeque::new(),
                                    write_buf: VecDeque::new(),
                                };
                                connections.insert(stream_token, connection);

                                println!("Registered client with {:?}", stream_token);
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
                other_token => {
                    // TODO: Handle client events (Milestone 3)
                    println!("Client event for {:?}", other_token);
                }
            }
        }
    }
}
```

# Learn epoll by Building a Simple Broadcast Server - Part 2: Reading, Parsing, and Broadcasting

In [Part 1](https://github.com/Givko/KnowledgeBase/blob/master/Learn_By_Doing/Part_1_Learn_epoll_with_simple_broadcast_server.md), we built an event loop
that accepts TCP connections. We can detect when clients connect and call
`accept()`, but the connections close immediately because we don't store them.

Today we'll complete the broadcast server. When a client sends "Hello\n", all
other connected clients will receive it.

## What We'll Builda

**Milestone 2:** Store and Register Client Connections

**Milestone 3:** Read Data and Broadcast Messages
- Handle edge-triggered reads
- Parse message boundaries
- Queue messages to other clients

**Milestone 4:** Write Buffered Data to Clients
- Handle partial writes
- Dynamic interest registration

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

### Testing Milestone 2

**Run the server:**

```bash
cargo run
```

**Connect 3 clients:**

```bash
#Terminal 1
nc localhost 8080

#Terminal 2
nc localhost 8080

#Terminal 3
nc localhost 8080
```

**Expected output:**

```
Accepted connection from 127.0.0.1:xxxxx
Registered client with Token(1)
Accepted connection from 127.0.0.1:xxxxx
Registered client with Token(2)
Accepted connection from 127.0.0.1:xxxxx
Registered client with Token(3)
Client event for Token(1)
Client event for Token(2)
Client event for Token(3)
```

**What's happening:**

- Each client gets unique Token (1, 2, 3)
- Clients are registered for READABLE
- When you type in netcat, epoll fires "Client event" messages
- We're not reading the data yet (Milestone 3)

**Current state:** Connections stay open and are tracked. Ready to read data!

## Milestone 3: Reading Data from Clients

### Challenge: Edge-Triggered Reading

When clients send data, epoll fires a READABLE event. But remember: edge-triggered mode only notifies when state CHANGES (not-readable → readable).

**Problem:** If we read once and return to poll(), we won't get another notification until NEW data arrives. Any unread data is lost.

**Solution:** Read in a loop until WouldBlock.

### Step 1: Handle client READABLE events

Replace the `other_token` match arm with:

```rust
other_token => {
    if event.is_readable() {
        let connection = connections
        .get_mut(&other_token)
        .expect("Connection not found");

        // Read loop - drain all available data
        loop {
            let mut buf = [0u8; 4096];
            match connection.stream.read(&mut buf) {
                Ok(0) => {
                    // Connection closed by client
                    println!("Client {:?} disconnected", other_token);
                    // TODO: Remove connection (Milestone 3.5)
                    break;
                }
                Ok(n) => {
                    // Read n bytes into buf
                    println!("Read {} bytes from {:?}", n, other_token);
                    // TODO: Store in read_buf (next step)
                }
                Err(e) if e.kind() == std::io::ErrorKind::WouldBlock => {
                    // No more data available - all done
                    break;
                }
                Err(e) => {
                    eprintln!("Read error: {}", e);
                    break;
                }
            }
        }
    }
}
```

**What's happening:**

1. **Check if readable:** Only process READABLE events (ignore others)
2. **Get connection:** Look up by token in HashMap
3. **Read loop:** Keep reading until WouldBlock (edge-triggered requirement)
4. **Handle results:**
   - `Ok(0)` = Client closed connection (EOF)
   - `Ok(n)` = Read n bytes successfully
   - `WouldBlock` = All data consumed, break loop
   - Other errors = Log and break

**Why the loop?**

- Client might send 10KB of data
- Each `read()` gets max 4096 bytes
- Without loop: Read 4096, return to poll, lose 6KB
- With loop: Read 4096, read 4096, read 1808, WouldBlock

### Step 2: Handle disconnections (0 bytes read)

When `read()` returns `Ok(0)`, the client closed the connection. We need cleanup.

```rust
Ok(0) => {
    println!("Client {:?} disconnected", other_token);

    // Deregister from epoll (stop monitoring)
    poll.registry()
        .deregister(&mut connection.stream)
        .expect("Failed to deregister");

    // Remove from HashMap (drops the connection)
    connections.remove(&other_token);

    break;  // Exit read loop
}
```

**Why each step:**

- `deregister()` - Tells epoll to stop monitoring this socket
- `remove()` - Drops connection from HashMap, closes socket automatically
- **No `shutdown()` needed** - Client already closed, socket closes when dropped

**Note:** Calling `shutdown()` on already-closed socket returns error. Skip it.

### Step 3: Parse messages and broadcast

Now we need to extract complete messages (ending with `\n`) and broadcast them to all other clients.

#### Understanding TCP Stream Fragmentation

TCP doesn't preserve message boundaries. "hello\nworld\n" might arrive as:

- Read 1: "hel"
- Read 2: "lo\nwor"
- Read 3: "ld\n"

**Our strategy:**

- Accumulate bytes in `read_buf`
- Find the last `\n` (complete message boundary)
- Extract all complete messages
- Leave partial message in buffer for next read

#### The Code

```rust
Ok(n) => {
    // Append newly read bytes
    connection.read_buf.extend(&buf[..n]);

    // Find last newline
    let last_newline =
        connection.read_buf.iter().rposition(|&b| b == b'\n');
    match last_newline {
        Some(pos) => {
            // Extract all complete messages
            let message: Vec<u8> =
                connection.read_buf.drain(..(pos + 1)).collect();
            messages.extend(message);

            // Broadcast to all other clients
            println!(
                "Broadcast {} bytes from {:?}",
                messages.len(),
                other_token,
            );
        }
        None => {
            println!(
                "Buffering partial message from {:?}",
                other_token
            );
        }
    }
}
```

#### After the read loop we need to broadcast the messages

```rust
if messages.is_empty() {
    continue;
}

for (token, conn) in connections.iter_mut() {
    if *token == other_token {
        continue;
    }

    conn.write_buf.extend(&messages);

    poll.registry()
        .reregister(
            &mut conn.stream,
            *token,
            Interest::READABLE | Interest::WRITABLE,
        )
        .expect("Failed to reregister");
}
```

#### What's Happening

**1. Extend read_buf:**

```rust
connection.read_buf.extend(&buf[..n]);
```

- Append the n bytes we just read
- Buffer accumulates across multiple reads

**2. Find last newline with `rposition`:**

```rust
let last_newline = connection.read_buf.iter().rposition(|&b| b == b'\n');
```

- `rposition` scans from the END, finding last `\n`
- More efficient than repeatedly searching from start
- Example: "hello\nworld\npartial" → finds position of second `\n`

**3. Drain complete messages:**

```rust
let messages: Vec<u8> = connection.read_buf.drain(..(pos + 1)).collect();
```

- Extracts bytes from position 0 to last newline (inclusive)
- Leaves partial message in buffer
- Example: buffer "hello\nworld\npart" becomes "part", messages = "hello\nworld\n"

**4. Broadcast to other clients:**

```rust
for (token, conn) in connections.iter_mut() {
    if *token == other_token {
      continue; // Skip sender
    }
    conn.write_buf.extend(&messages);
    // ... reregister ...
}

```

- Iterate all connections except sender
- Append messages to each client's write_buf
- We'll actually write these bytes in next milestone

**5. Reregister with WRITABLE:**

```rust
Interest::READABLE | Interest::WRITABLE
```

- Initially registered with READABLE only
- Now we have pending writes → need WRITABLE notifications
- Epoll will fire WRITABLE event when socket ready for writing
- This is **dynamic interest registration** - only request events we need

**Full code so far:**

```rust
use mio::net::{TcpListener, TcpStream};
use mio::{Events, Interest, Poll, Token};
use std::collections::{HashMap, VecDeque};
use std::io::Read;
use std::time::Duration;

struct ClientConnection {
    stream: TcpStream,
    read_buf: VecDeque<u8>,
    write_buf: VecDeque<u8>,
}

fn main() {
    let mut poll = Poll::new().unwrap();
    let mut events = Events::with_capacity(1024);
    let mut listener =
        TcpListener::bind("127.0.0.1:8080".parse().unwrap()).expect("Unable to bind");
    let listener_token = Token(0);

    poll.registry()
        .register(&mut listener, listener_token, Interest::READABLE)
        .expect("Failed to register listener");

    let mut connections: HashMap<Token, ClientConnection> = HashMap::new();
    let mut next_token_id = 1;
    loop {
        poll.poll(&mut events, Some(Duration::from_millis(100)))
            .expect("Failed to poll");

        for event in events.iter() {
            match event.token() {
                Token(0) => {
                    // Listener - accept connections
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
                    // Client event
                    if event.is_readable() {
                        let connection = connections
                            .get_mut(&other_token)
                            .expect("Connection not found");

                        // Read loop (edge-triggered)
                        let mut messages: Vec<u8> = Vec::new();
                        loop {
                            let mut buf = [0u8; 4096];

                            match connection.stream.read(&mut buf) {
                                Ok(0) => {
                                    // Connection closed
                                    println!("Client {:?} disconnected", other_token);

                                    poll.registry()
                                        .deregister(&mut connection.stream)
                                        .expect("Failed to deregister");

                                    connections.remove(&other_token);
                                    break;
                                }
                                Ok(n) => {
                                    // Append newly read bytes
                                    connection.read_buf.extend(&buf[..n]);

                                    // Find last newline
                                    let last_newline =
                                        connection.read_buf.iter().rposition(|&b| b == b'\n');
                                    match last_newline {
                                        Some(pos) => {
                                            // Extract all complete messages
                                            let message: Vec<u8> =
                                                connection.read_buf.drain(..(pos + 1)).collect();
                                            messages.extend(message);

                                            // Broadcast to all other clients
                                            println!(
                                                "Broadcast {} bytes from {:?}",
                                                messages.len(),
                                                other_token,
                                            );
                                        }
                                        None => {
                                            println!(
                                                "Buffering partial message from {:?}",
                                                other_token
                                            );
                                        }
                                    }
                                }
                                Err(e) if e.kind() == std::io::ErrorKind::WouldBlock => {
                                    break;
                                }
                                Err(e) => {
                                    eprintln!("Read error: {}", e);
                                    break;
                                }
                            }
                        }

                        if messages.is_empty() {
                            continue;
                        }

                        for (token, conn) in connections.iter_mut() {
                            if *token == other_token {
                                continue;
                            }

                            conn.write_buf.extend(&messages);

                            poll.registry()
                                .reregister(
                                    &mut conn.stream,
                                    *token,
                                    Interest::READABLE | Interest::WRITABLE,
                                )
                                .expect("Failed to reregister");
                        }
                    }
                }
            }
        }
    }
}
```

## Milestone 4: Write Buffered Data to Clients

Now we handle WRITABLE events to actually send queued messages.

### The Challenge: Partial Writes

`write()` might not send all bytes:

- `write_buf` has 10KB
- Socket buffer only has 4KB space
- `write()` returns `Ok(4096)` (partial write)
- Must track what was sent, retry rest later

**Solution:** Write in loop until WouldBlock, track bytes sent.

### The Code

Add after the `is_readable()` block:

```rust
if event.is_writable() {
    let connection = connections
        .get_mut(&other_token)
        .expect("Connection not found");

    // Write loop (edge-triggered - drain until WouldBlock)
    loop {
        if connection.write_buf.is_empty() {
            // Nothing left to write - reregister for READABLE only
            poll.registry()
                .reregister(
                    &mut connection.stream,
                    other_token,
                    Interest::READABLE,
                )
                .expect("Failed to reregister");
            break;
        }

        // Get contiguous slice of write buffer
        let buf = connection.write_buf.make_contiguous();

        match connection.stream.write(buf) {
            Ok(0) => {
                // Socket closed
                println!("Client {:?} closed during write", other_token);

                poll.registry()
                    .deregister(&mut connection.stream)
                    .expect("Failed to deregister");

                connections.remove(&other_token);
                break;
            }
            Ok(n) => {
                // Wrote n bytes - remove from buffer
                println!("Wrote {} bytes to {:?}", n, other_token);

                connection.write_buf.drain(..n);
                // Continue loop - might have more to write
            }
            Err(e) if e.kind() == std::io::ErrorKind::WouldBlock => {
                // Socket not ready - will get another WRITABLE event
                println!("Write would block for {:?}", other_token);
                break;
            }
            Err(e) => {
                eprintln!("Write error for {:?}: {}", other_token, e);
                break;
            }
        }
    }
}
```

### What's Happening

**1. Check if buffer empty:**

```rust
if connection.write_buf.is_empty() {
// Reregister for READABLE only
}
```

- No pending writes → don't need WRITABLE events
- Dynamic interest: only request what we need

**2. Get contiguous buffer:**

```rust
let buf = connection.write_buf.make_contiguous();
```

- VecDeque may store data non-contiguously (ring buffer)
- `make_contiguous()` returns `&[u8]` slice for `write()`
- Rearranges internal storage if needed (rarely)

**3. Write and handle results:**

**`Ok(n)`** - Wrote n bytes successfully:

```rust
connection.write_buf.drain(..n);
```

- Remove sent bytes from front of buffer
- Continue loop (might have more data)

**`Ok(0)`** - Socket closed (rare during write):

- Deregister and remove connection
- Client closed while we were writing

**`WouldBlock`** - Socket buffer full:

- Stop writing (don't remove WRITABLE interest)
- Epoll will fire again when socket ready
- Remaining data stays in write_buf

**4. Loop until done:**

- Write as much as possible
- Stop on WouldBlock or empty buffer
- Edge-triggered requirement

### Why the Write Loop?

**Without loop (your code):**

```
write_buf: 10KB
write() returns: 4KB written
write_buf after: 6KB remaining
Result: 6KB never sent (no more WRITABLE events)
```

**With loop:**

```
Iteration 1: write 4KB, 6KB remaining
Iteration 2: write 4KB, 2KB remaining
Iteration 3: write 2KB, 0KB remaining
Reregister for READABLE only
```

### Example Walkthrough

**Scenario:** Broadcast "hello\n" (6 bytes) to 3 clients

**Step 1: READABLE event**

- Client A sends "hello\n"
- Parse message, queue in all other write_bufs
- Reregister all with READABLE | WRITABLE

**Step 2: WRITABLE events (edge-triggered)**

**Client B's WRITABLE event:**

```
write_buf: "hello\n" (6 bytes)
write() returns: Ok(6)
Remove 6 bytes from buffer
write_buf now empty
Reregister for READABLE only
```

**Client C's WRITABLE event:**

```
write_buf: "hello\n" (6 bytes)
write() returns: Ok(6)
Remove 6 bytes from buffer
write_buf now empty
Reregister for READABLE only
```

**Result:** All clients received message, back to READABLE-only mode.

**Final Code**

```rust
use mio::net::{TcpListener, TcpStream};
use mio::{Events, Interest, Poll, Token};
use std::collections::{HashMap, VecDeque};
use std::io::{Read, Write};
use std::time::Duration;

struct ClientConnection {
    stream: TcpStream,
    read_buf: VecDeque<u8>,
    write_buf: VecDeque<u8>,
}

fn main() {
    let mut poll = Poll::new().unwrap();
    let mut events = Events::with_capacity(1024);
    let mut listener =
        TcpListener::bind("127.0.0.1:8080".parse().unwrap()).expect("Unable to bind");
    let listener_token = Token(0);

    poll.registry()
        .register(&mut listener, listener_token, Interest::READABLE)
        .expect("Failed to register listener");

    let mut connections: HashMap<Token, ClientConnection> = HashMap::new();
    let mut next_token_id = 1;
    loop {
        poll.poll(&mut events, Some(Duration::from_millis(100)))
            .expect("Failed to poll");

        for event in events.iter() {
            match event.token() {
                Token(0) => {
                    // Listener - accept connections
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
                    // Client event
                    if event.is_readable() {
                        let connection = connections
                            .get_mut(&other_token)
                            .expect("Connection not found");

                        // Read loop (edge-triggered)
                        let mut messages: Vec<u8> = Vec::new();
                        loop {
                            let mut buf = [0u8; 4096];

                            match connection.stream.read(&mut buf) {
                                Ok(0) => {
                                    // Connection closed
                                    println!("Client {:?} disconnected", other_token);

                                    poll.registry()
                                        .deregister(&mut connection.stream)
                                        .expect("Failed to deregister");

                                    connections.remove(&other_token);
                                    break;
                                }
                                Ok(n) => {
                                    // Append newly read bytes
                                    connection.read_buf.extend(&buf[..n]);

                                    // Find last newline
                                    let last_newline =
                                        connection.read_buf.iter().rposition(|&b| b == b'\n');
                                    match last_newline {
                                        Some(pos) => {
                                            // Extract all complete messages
                                            let message: Vec<u8> =
                                                connection.read_buf.drain(..(pos + 1)).collect();
                                            messages.extend(message);

                                            // Broadcast to all other clients
                                            println!(
                                                "Broadcast {} bytes from {:?}",
                                                messages.len(),
                                                other_token,
                                            );
                                        }
                                        None => {
                                            println!(
                                                "Buffering partial message from {:?}",
                                                other_token
                                            );
                                        }
                                    }
                                }
                                Err(e) if e.kind() == std::io::ErrorKind::WouldBlock => {
                                    break;
                                }
                                Err(e) => {
                                    eprintln!("Read error: {}", e);
                                    break;
                                }
                            }
                        }

                        if messages.is_empty() {
                            continue;
                        }

                        for (token, conn) in connections.iter_mut() {
                            if *token == other_token {
                                continue;
                            }

                            conn.write_buf.extend(&messages);

                            poll.registry()
                                .reregister(
                                    &mut conn.stream,
                                    *token,
                                    Interest::READABLE | Interest::WRITABLE,
                                )
                                .expect("Failed to reregister");
                        }
                    }
                    if event.is_writable() {
                        let connection = connections
                            .get_mut(&other_token)
                            .expect("Connection not found");

                        // Write loop (edge-triggered - drain until WouldBlock)
                        loop {
                            if connection.write_buf.is_empty() {
                                // Nothing left to write - reregister for READABLE only
                                poll.registry()
                                    .reregister(
                                        &mut connection.stream,
                                        other_token,
                                        Interest::READABLE,
                                    )
                                    .expect("Failed to reregister");
                                break;
                            }

                            // Get contiguous slice of write buffer
                            let buf = connection.write_buf.make_contiguous();

                            match connection.stream.write(buf) {
                                Ok(0) => {
                                    // Socket closed
                                    println!("Client {:?} closed during write", other_token);

                                    poll.registry()
                                        .deregister(&mut connection.stream)
                                        .expect("Failed to deregister");

                                    connections.remove(&other_token);
                                    break;
                                }
                                Ok(n) => {
                                    // Wrote n bytes - remove from buffer
                                    println!("Wrote {} bytes to {:?}", n, other_token);

                                    connection.write_buf.drain(..n);
                                    // Continue loop - might have more to write
                                }
                                Err(e) if e.kind() == std::io::ErrorKind::WouldBlock => {
                                    // Socket not ready - will get another WRITABLE event
                                    println!("Write would block for {:?}", other_token);
                                    break;
                                }
                                Err(e) => {
                                    eprintln!("Write error for {:?}: {}", other_token, e);
                                    break;
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}
```

## Testing the Complete Broadcast Server

**Run the server:**
```bash
cargo run
```

**Open 3 client terminals and connect:**
```bash
# Terminal 1 (Client A)
nc localhost 8080

# Terminal 2 (Client B)  
nc localhost 8080

# Terminal 3 (Client C)
nc localhost 8080
```

**In Client A, type:**
```
Hello from A
```

**Expected result:**
- Client B sees: `Hello from A`
- Client C sees: `Hello from A`
- Client A does NOT see their own message (sender excluded)

**In Client B, type:**
```
Hello from B
```

**Expected result:**
- Client A sees: `Hello from B`
- Client C sees: `Hello from B`
- Client B does NOT see their own message

**Server output should show:**
```
Accepted connection from 127.0.0.1:xxxxx
Registered client with Token(1)
Accepted connection from 127.0.0.1:xxxxx
Registered client with Token(2)
Accepted connection from 127.0.0.1:xxxxx
Registered client with Token(3)
Broadcast 14 bytes from Token(1)
Wrote 14 bytes to Token(2)
Wrote 14 bytes to Token(3)
Broadcast 14 bytes from Token(2)
Wrote 14 bytes to Token(1)
Wrote 14 bytes to Token(3)
```

**Test disconnection:**

In Client A, press `Ctrl+C` to disconnect.

**Expected:**
- Server prints: `Client Token(1) disconnected`
- Clients B and C continue working
- Messages from B and C still broadcast to each other

**Success!** You've built a working epoll-based broadcast server. Every pattern you've learned here (edge-triggered loops, message parsing, write buffering) is used in production servers like nginx and Redis.

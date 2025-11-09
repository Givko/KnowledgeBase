# KnowledgeBase

I kept reading about how nginx handles 100k connections, how Kafka guarantees ordering, how TCP prevents duplicates. The explanations were always either too theoretical or too hand-wavy.

So I built them myself. Turns out they all use the same patterns.

## What's Here

**Two tutorial series:**

1. **Building an epoll broadcast server** - The thing that makes nginx, Redis, and async runtimes fast. You'll write a working TCP server that handles multiple clients on one thread.

2. **Systems patterns from TCP** - State machines, sequence numbers, idempotency. Once you see these patterns in TCP, you'll spot them everywhere (Kafka, Raft, circuit breakers).

## Start Here

### If you want to understand async/epoll internals

Read the epoll tutorials in order. You'll build a chat server from scratch:

- [Part 1](Learn_By_Doing/Part_1_Learn_epoll_with_simple_broadcast_server.md) - Event loops and accepting connections
- [Part 2](Learn_By_Doing/Part_2_Reading_Parsing_Broadcasting.md) - Reading, parsing, broadcasting

Takes about 90 minutes. You'll test it with netcat.

#### What You'll Build

By the end of Part 2, you'll have a working broadcast server:

```bash
# Terminal 1
cargo run

# Terminal 2, 3, 4
nc localhost 8080

# Type in any terminal, see it in all others
```

It handles edge-triggered events, TCP fragmentation, partial writes, and dynamic interest registration. The patterns you learn here are the same ones in nginx and Redis.

### If you want to spot patterns faster

Read the patterns series. Each article is ~20 minutes:

- [State Machines](Systems_Patterns/Part_1_State_Machine.md) - How TCP and Raft prevent invalid states
- [Sequence Numbers](Systems_Patterns/Part_2_Sequence_Numbers.md) - Why position beats timestamps
- [Idempotency](Systems_Patterns/Part_3_Idempotency.md) - Detecting duplicates automatically

You'll start recognizing these patterns in every distributed system you touch.

## Who This Is For

You should know basic TCP (it's a byte stream, not message-oriented). You should know basic Rust.

You don't need epoll or async experience. That's what we're learning.

This helps if you:

- Debug production issues with TCP, Kafka, or distributed systems
- Want to understand how Tokio/async actually works
- Learn better by building than by reading theory

Skip this if you just want to use libraries without understanding internals.

## Why I Wrote This

Most tutorials skip the hard parts. They show you `poll.poll()` but don't explain why you need a loop until `WouldBlock`. They mention sequence numbers but don't show how TCP and Kafka use them the same way.

I wanted explanations that showed working code AND explained the reasoning. So I wrote them.

## Contributing

Found something unclear? Open an issue.

## License

MIT. Learn freely, share widely.

---

If this helped you understand systems programming, star the repo so others can find it.


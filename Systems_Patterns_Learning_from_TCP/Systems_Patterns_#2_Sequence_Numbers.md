# Sequence Numbers: Ordering Without Synchronized Clocks

**Series**: Systems Patterns: Learning from TCP | Part 2 of 3

## The Pattern

Both TCP and Kafka solve the same problem: **How do you order events in a stream when you can't rely on physical clocks?**

**Answer**: Assign each event a position in a sequence.

- TCP: Sequence numbers track byte positions
- Kafka: Offsets track message positions
- Both: Enable ordering, gap detection, and crash recovery

**Key insight**: You don't need synchronized clocks when you have a single ordered log. The position IS the timestamp.

**Terminology note**: These are "sequence numbers", not "monotonic counters". True monotonic counters (Lamport timestamps, vector clocks) handle ordering across multiple streams. We will not cover this here.

## Why Not Use Wall-Clock Time?

**Naive approach**: Timestamp each event with system time

**Why this fails**:

1. **Clock skew**: Different machines have different times
   - Server A: 12:00:05
   - Server B: 12:00:00 (5 seconds behind)
   - Event on B happens after event on A, but timestamp says opposite

2. **Clock drift**: System clocks drift apart over time

3. **NTP corrections**: Time can jump backwards during clock sync

**The insight**: If you control a single stream (one TCP connection, one Kafka partition), you don't need timestamps at all. Just assign positions in order.

**TCP's solution**:

- Ignore wall-clock time completely
- Number the bytes: byte 0, byte 1, byte 2...
- No clock synchronization needed

**Kafka's solution**:

- Broker assigns offsets as messages arrive
- Order = arrival order at the broker
- Producer timestamps are ignored for ordering

**The lesson**: Position-based ordering is simpler and more reliable than time-based ordering when you have a single ordered stream.

## Monotonic Counters in TCP (Sequence Numbers)

TCP uses Sequence Numbers to keep track of what bytes it has sent and what bytes have been received(acknowledged).

- Each packet has a Sequence number which represents the FIRST byte in the packet
- Acknowledgment Number indicates the NEXT expected byte (cumulative acknowledgment)
- Using this mechanism TCP server/client can keep track if
  - There are any gaps(not received bytes)
  - In-order delivery
  - Guarantee exactly once delivery
  - Idempotency

### Example

```
Client sends:
  Packet 1: SEQ=100,Payload="Hello" (5 bytes) -> bytes 100-104

Server view:
  Receives packet 1: SEQ=100 -> ACK=105 (expecting byte 105 next)
ACK=105
```

**Why bytes, not packets**: Partial retransmission. If a 1000-byte packet is lost, TCP can resend just the missing 500 bytes instead of the entire packet.

#### Gap Detection

**Scenario**: Client sends 3 packets, middle one is lost

```
Client sends:
  Packet 1: SEQ=100, "Hello" (5 bytes) → bytes 100-104
  Packet 2: SEQ=105, "World" (5 bytes) → LOST ❌
  Packet 3: SEQ=110, "Test!" (5 bytes) → bytes 110-114

Server's view:
  Receives packet 1: SEQ=100 → ACK=105 (expecting byte 105 next)
  Receives packet 3: SEQ=110 
    → Expected 105, got 110
    → Gap detected! Bytes 105-109 missing
    → Send duplicate ACK=105 ("I'm still waiting for byte 105")

Client's view:
  Sees duplicate ACK=105
    → "Server hasn't received bytes 105-109"
    → Retransmit packet 2
```

**The key insight**: The sequence number gap (105 → 110) immediately reveals packet loss. No timeout needed for detection—the gap IS the signal.

## Monotonic Counters in Kafka (Offsets)

Kafka uses offsets to order messages within a partition, similar to how TCP orders bytes within a connection.

**How it works**:

Each partition is an append-only log with monotonic offsets assigned by the broker:

```
Partition 0 (independent stream, like TCP connection):

Offset 0: {"user": "alice", "action": "login"}
Offset 1: {"user": "bob", "action": "purchase"}
Offset 2: {"user": "alice", "action": "logout"}
Offset 3: {"user": "charlie", "action": "login"}
```

**Consumer operations**:

1. **Poll**: Consumer reads message from Kafka
2. **Process**: Consumer applies business logic
3. **Commit**: Consumer tells Kafka "I've processed up to offset X"
4. **Kafka stores**: Which offset each consumer group has committed

### Example

Producer sends message "Hello":
→ Broker assigns offset=100 to Partition 0

Consumer polls:
→ Receives message with offset=100

Consumer processes message successfully:
→ Commits offset=101 (next message to process)

Producer sends message "World":
→ Broker assigns offset=101 to Partition 0

#### Gap Example

```
Consumer commits offset=100
Next poll receives offset=105
→ Messages 101-104 are missing (producer skipped them or partition reassigned)
→ Consumer can either: fail, or continue (depends on requirements)
```

In both cases, **non-consecutive sequence numbers reveal data loss**.

**Compare to TCP**:

| TCP | Kafka | Pattern |
|-----|-------|---------|
| Client sends SEQ=100 | Producer writes, Kafka assigns offset=100 | Position tracking |
| Server sends ACK=105 | Consumer commits offset=101 | Acknowledge progress |
| TCP remembers ACK state | Kafka stores committed offset per consumer group | Both remember progress |

**Key point**: Kafka stores each consumer group's committed offset, so after crash, consumer knows where to resume. Just like TCP remembers which bytes were acknowledged.

## Why Monotonic Counters Work

Three benefits apply to both TCP and Kafka:

1. **Total ordering within a stream**: Unambiguous sequence - no "did A or B happen first?" within a TCP connection or Kafka partition. Counter values definitively establish order.

2. **Gap detection**: Missing counter value signals data loss. TCP detects missing bytes (SEQ jump), Kafka can detect missing messages (offset jump). Immediate problem visibility.

3. **Resumability**: Counter acts as checkpoint. TCP receiver tracks ACK position, Kafka consumer commits offset. After crash, both resume from last acknowledged position without reprocessing.

## Resources

### TCP Sequence Numbers (20 min)

1. [RFC 9293 - TCP Specification, Section 3.4](https://datatracker.ietf.org/doc/html/rfc9293#section-3.4) - Official sequence number semantics (15 min)

### Kafka Offsets (30 min)

2. [Confluent - Kafka Consumer Offsets](https://docs.confluent.io/platform/current/clients/consumer.html#offset-management) - Official offset management guide (20 min)
3. [Kafka Architecture - Consumer Position](https://kafka.apache.org/documentation/#design_consumerposition) - How Kafka tracks position (10 min)

### Distributed Ordering (Advanced) (40 min)

4. [Lamport (1978) - "Time, Clocks, and the Ordering of Events"](https://lamport.azurewebsites.net/pubs/time-clocks.pdf) - Extending to distributed systems (30 min, Sections 1-4)
5. [Designing Data-Intensive Applications - Chapter 9](https://dataintensive.net/) - Consistency and consensus (10 min overview)

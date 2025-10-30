# Idempotency: Detecting Duplicates with Sequence Numbers

**Series**: Systems Patterns: Learning from TCP | Part 3 of 3

In Article 2, we learned how sequence numbers establish ordering. Today we'll see they also prevent duplicates - the same sequence number mechanism detects if we've already processed data.

## The Pattern

**Problem**: Networks are unreliable. Packets get lost, responses timeout, systems retry.

- **What will happen if I retrieve the same packet/message again and how do I avoid reprocessing it?**
**Scenario**:

```
Client sends: "Transfer $100"
Network drops response (ACK lost)
Client doesn't know if server received it
Client retries: "Transfer $100" (AGAIN)
→ Without duplicate detection: $200 transferred ❌
→ With duplicate detection: $100 transferred once ✅
```

**Solution**: **Idempotency** - processing a message multiple times has the same effect as processing once.

**Mechanism**: Use sequence numbers as unique identifiers

- TCP: SEQ number = byte position (prevents duplicate bytes)
- Kafka: Producer sequence = message identifier (prevents duplicate messages)

**Key insight**: Sequence numbers from Article 2 serve dual purpose: ordering AND deduplication.

## TCP

- For this TCP uses the RCV.NXT from its Receive Sequence Space and the packets sequence number to track if it has received a duplicate package.
  - RCV.NXT stores which SEQ number the TCP server expects next
  - You can find more information for the Receive Sequence Space in the RFC of TCP linked in the Resources Section

### Example: Network Retransmission

**Initial state**:

- RCV.NXT = 100 (expecting byte 100 next)

**Packet arrives normally**:

```
Packet: SEQ=100, "Hello" (5 bytes) → bytes 100-104

Server checks:
SEG.SEQ (100) >= RCV.NXT (100)? YES ✅
→ Acceptable - process data
→ Update: RCV.NXT = 105
→ Send: ACK=105
```

**ACK lost in network, client retransmits**:

```
Same packet arrives AGAIN: SEQ=100, "Hello"

Server checks:
SEG.SEQ (100) >= RCV.NXT (105)? NO ❌
→ SEQ (100) < RCV.NXT (105)
→ DUPLICATE detected
→ Discard packet (don't process "Hello" twice)
→ Send: ACK=105 anyway (idempotent response)
```

**Result**: "Hello" processed exactly once, despite retry ✅

## Kafka Idempotent Producer

Kafka uses similar pattern but with TWO numbering systems:

1. **Offsets** (broker-assigned) → order messages in partition (Article 2)
2. **Sequence numbers** (producer-assigned) → detect duplicates (Article 3)

**Why separate systems?**

- Offset = position in log (like page number)
- Sequence = unique identifier for deduplication (like ISBN)

### How It Works

**Enable idempotency**:

```
producer = KafkaProducer(
enable_idempotence=True, # Prevent duplicates
acks='all' # Ensure durability)
```

1. **Producer startup**:
   - Broker assigns unique Producer ID (PID=7005)
   - Producer initializes sequence number = 0

2. **First message sent**:

Producer sends:

```
message="User login"
PID=7005, sequence=0
```

Broker receives:

```
Checks: (PID=7005, partition=0, last_seq=None)
→ New producer, accept
→ Assigns: offset=100 (position in partition)
→ Stores: (PID=7005, last_seq=0)
→ Sends: ACK
```

3. **Network failure - ACK lost**:

Producer doesn't receive ACK
Retries same message:

```
message="User login"
PID=7005, sequence=0 ← SAME sequence
```

Broker receives:

```
Checks: (PID=7005, partition=0, last_seq=0)
→ incoming_seq (0) <= last_seq (0)
→ DUPLICATE detected
→ Does NOT write (no new offset assigned)
→ Sends: ACK anyway (idempotent response)
```

4. **Next message**:

Producer sends:

```
message="User purchase"
PID=7005, sequence=1 ← Incremented
```

Broker receives:

```
→ incoming_seq (1) == last_seq (0) + 1
→ NEW message, accept
→ Assigns: offset=101
```

### Compare to TCP

| TCP | Kafka |
|-----|-------|
| SEQ=100 (byte position) | (PID=7005, SEQ=0) |
| RCV.NXT tracks last received | Broker tracks (PID, last_seq) |
| if SEQ < RCV.NXT → duplicate | if SEQ <= last_seq → duplicate |
| Both ensure: process once, even with retries ✓ |

## Why Idempotency Works

Three benefits apply to both TCP and Kafka:

1. **Retries are safe**: Network failure? Just retry - duplicate detection is automatic. No risk of double-processing.

2. **Exactly-once semantics**: TCP delivers each byte once. Kafka idempotent producer delivers each message once. Even with retries.

- Note: End-to-end exactly-once (producer → broker → consumer) requires additional settings beyond idempotence (transactional.id, isolation level)

3. **Builds on ordering**: Sequence numbers (Article 2) serve dual purpose - ordering AND deduplication. Same mechanism, two benefits.

**The insight**: "Idempotency transforms unreliable networks into reliable systems. Retry without fear."

## Resources

- [Confluent Idempotent Writer](https://developer.confluent.io/patterns/event-processing/idempotent-writer/)
- [TCP RFC Section Sequence Numbers](https://datatracker.ietf.org/doc/html/rfc9293#name-sequence-numbers)
- [Red Hat: Kafka Producer configuration tuning](https://docs.redhat.com/en/documentation/red_hat_streams_for_apache_kafka/2.2/html/kafka_configuration_tuning/con-producer-config-properties-str#ordered_delivery)
- [AutoMQ: Understanding Kafka Producer Part 2](https://www.automq.com/blog/understanding-kafka-producer-part-2)

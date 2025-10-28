# State Machine Pattern

**Series**: Systems Patterns: Learning from TCP | Part 1 of 3

After implementing TCP in Rust, I kept seeing the same pattern everywhere: Raft consensus, circuit breakers, HTTP servers. They all use state machines to prevent invalid states. This article shows how to recognize the pattern across systems.

## What Is a State Machine?

A finite state machine (FSM) has three parts:

1. **States**: Distinct conditions (e.g., CLOSED, ESTABLISHED)
2. **Transitions**: Rules for moving between states
3. **Events**: Inputs that trigger transitions (packet arrival, timeout)

**Key property**: Illegal states are impossible by design. A TCP connection can be CLOSED or ESTABLISHED, but never both simultaneously.

**Why this matters**: Without a state machine, you might use boolean flags like `is_connected` and `is_closed`. These can both be true at once—what does that mean? A state machine (using an enum) prevents this at compile time.

## TCP's State Machine

**TCP states** (RFC 9293):

- `LISTEN` - Waiting for connection request
- `SYN-SENT` - Sent SYN, waiting for SYN-ACK
- `SYN-RECEIVED` - Received SYN, waiting for ACK
- `ESTABLISHED` - Connection open, data transfer active
- `FIN-WAIT-1` - Sent FIN, waiting for ACK or remote FIN
- `FIN-WAIT-2` - FIN acknowledged, waiting for remote FIN
- `CLOSE-WAIT` - Received remote FIN, waiting for local close
- `CLOSING` - Sent FIN, waiting for final ACK
- `LAST-ACK` - Sent final FIN, waiting for ACK
- `TIME-WAIT` - Waiting for delayed packets to clear
- `CLOSED` - No connection

**Events triggering transitions**:
User calls (OPEN, SEND, CLOSE), incoming segments (SYN, ACK, FIN, RST), timeouts

**What the state machine enforces**:

- Cannot send data in `CLOSED` state (no connection exists)
- Cannot skip `SYN_SENT` state (must complete handshake)
- Cannot reopen connection from `FIN_WAIT` (must return to CLOSED first)

**Example transition**:

- **State**: `SYN_SENT`
- **Event**: Receive SYN-ACK packet
- **Action**: Send ACK packet
- **New state**: `ESTABLISHED`

## Raft's State Machine

Raft uses the same FSM pattern for leader election. Three states:

- `Follower` - Default state, listens for leader heartbeats
- `Candidate` - Requests votes after election timeout
- `Leader` - Won election, sends heartbeats to followers

**Compare to TCP handshake**:

| TCP | Raft | Pattern |
|-----|------|---------|
| CLOSED → SYN_SENT | Follower → Candidate | SYN packet sent/Timeout triggers transition |
| SYN_SENT → ESTABLISHED | Candidate → Leader | Confirmation needed (ACK vs votes) |
| Can't skip SYN_SENT | Can't skip Candidate | Middle state enforces safety |

**Raft transitions**:

1. **Follower → Candidate**
   - Event: Election timeout (no heartbeat for 150-300ms)
   - Action: Increment term, vote for self, request votes

2. **Candidate → Leader**  
   - Event: Receives majority votes (e.g., 3 out of 5 nodes)
   - Action: Send heartbeats to all followers

3. **Candidate → Follower** (two ways)
   - Event A: Another server becomes leader (receive their heartbeat)
   - Event B: Election timeout expires, no majority → start new election

4. **Leader → Follower**
   - Event: Receives message with higher term number
   - Action: Step down, reset election timer

**Why FSM prevents bugs**:

- Can't have two leaders in same term (Candidate state enforces voting)
- Can't become leader without majority (like TCP can't establish without ACK)
- Higher term always wins (prevents split-brain)

## Why State Machines Work

Four benefits apply to both TCP and Raft:

1. **Type safety**: Enum prevents impossible states (can't be Closed AND Established, can't be Follower AND Leader)

2. **Explicit transitions**: All valid paths documented. Invalid transitions cause compile error or panic.

3. **Testability**: Can verify "never skip Candidate state" or "never skip SYN_SENT state"

4. **Debugging**: State history acts as breadcrumb trail showing what happened before failure

**Key insight**: "State machines make illegal states unrepresentable" - if your code compiles, invalid states are impossible.

## Resources

### FSM Theory (60 min)

1. [RFC 9293 - TCP State Machine](https://datatracker.ietf.org/doc/html/rfc9293#section-3.3.2) - Official TCP spec (10 min)
2. [Hopcroft et al. - Automata Theory Ch 2](https://www-2.dc.uba.ar/staff/becher/Hopcroft-Motwani-Ullman-2001.pdf) - Formal FSM definition (20 min)
3. [Refactoring.Guru - State Pattern](https://refactoring.guru/design-patterns/state) - OOP implementation (10 min)
4. [Game Programming Patterns - State](https://gameprogrammingpatterns.com/state.html) - Practical examples (15 min)

### Raft Examples (30 min)

5. [The Secret Lives of Data - Raft](https://thesecretlivesofdata.com/raft/) - Interactive visualization (5 min)
6. [Raft Paper - Section 5.2](https://raft.github.io) - Leader election algorithm (20 min)

# TCP Deep Dive Report
## A Comprehensive Guide to Transmission Control Protocol

---

**Table of Contents**

1. [Introduction](#introduction)
2. [Three-Way Handshake](#three-way-handshake)
3. [TCP Segment Structure](#tcp-segment-structure)
4. [TCP Reliability Mechanisms](#tcp-reliability-mechanisms)
5. [Connection Termination and Failure Detection](#connection-termination-and-failure-detection)
6. [Sequence Numbers and ACKs](#sequence-numbers-and-acks)
7. [Flow Control](#flow-control)
8. [Congestion Control](#congestion-control)
9. [Retransmission Mechanisms](#retransmission-mechanisms)
10. [Acknowledgment Mechanisms](#acknowledgment-mechanisms)
11. [TCP State Machine](#tcp-state-machine)
12. [Complete Example: Data Transfer](#complete-example-data-transfer)
13. [Summary and Key Takeaways](#summary-and-key-takeaways)

---

## Introduction

TCP (Transmission Control Protocol) is a connection-oriented, reliable transport protocol that provides:
- **Ordered delivery** of data
- **Error detection** via checksums
- **Flow control** to prevent overwhelming receivers
- **Congestion control** to prevent network collapse
- **Reliability** through retransmissions

---

## Three-Way Handshake

### Purpose
Establishes a connection between client and server, synchronizing sequence numbers.

### The Process

```
Client                              Server
   │                                   │
   │────── SYN, seq=x ───────────────→│  "I want to connect, my ISN is x"
   │                                   │
   │←───── SYN-ACK, seq=y, ack=x+1 ───│  "I got it, my ISN is y, expecting x+1"
   │                                   │
   │────── ACK, ack=y+1 ─────────────→│  "Got it, expecting y+1"
   │                                   │
   │◄───────── CONNECTION ESTABLISHED ─────────►│
```

### Detailed Breakdown

| Step | Direction | Flags | seq | ack | Meaning |
|------|-----------|-------|-----|-----|---------|
| 1 | C → S | SYN | x | - | "I want to connect" |
| 2 | S → C | SYN+ACK | y | x+1 | "I agree, expecting x+1" |
| 3 | C → S | ACK | x+1 | y+1 | "Got it, expecting y+1" |

### Key Points

- **ISN (Initial Sequence Number)**: Random starting point for security
- **Half-open connections**: SYN flood attacks exploit this
- **Simultaneous open**: Both sides send SYN at same time (rare)

### Example

```
Client ISN = 1000
Server ISN = 5000

SYN:      seq=1000, ack=0,  flags=SYN
SYN-ACK:  seq=5000, ack=1001, flags=SYN,ACK
ACK:      seq=1001, ack=5001, flags=ACK

Result:
  Client sends data starting at seq=1001
  Server sends data starting at seq=5001
```

---

## TCP Segment Structure

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
├───────────────────────────────────────────────────────────────┤
│          Source Port          │       Destination Port        │
├───────────────────────────────────────────────────────────────┤
│                        Sequence Number                        │
├───────────────────────────────────────────────────────────────┤
│                    Acknowledgment Number                      │
├───────────────────────────────────────────────────────────────┤
│  Data │           │U│A│P│R│S│F│                               │
│ Offset│ Reserved  │R│C│S│S│Y│I│            Window             │
│       │           │G│K│H│T│N│N│                               │
├───────────────────────────────────────────────────────────────┤
│           Checksum            │         Urgent Pointer        │
├───────────────────────────────────────────────────────────────┤
│                    Options (optional)                         │
├───────────────────────────────────────────────────────────────┤
│                             data                              │
└───────────────────────────────────────────────────────────────┘

Header Size: 20-60 bytes (without options: 20 bytes)
```

### Key Fields Explained

| Field | Size | Purpose |
|-------|------|---------|
| **Sequence Number** | 32 bits | Identifies first byte of data |
| **Acknowledgment Number** | 32 bits | Next expected byte |
| **Window** | 16 bits | Available buffer space (rwnd) |
| **Flags** | 6 bits | SYN, ACK, FIN, RST, PSH, URG |
| **Checksum** | 16 bits | Error detection |

---

## TCP Reliability Mechanisms

TCP achieves reliability through multiple layered mechanisms that work together to ensure data is delivered completely, correctly, and in order.

### Overview of Reliability Stack

```
┌─────────────────────────────────────────────────────────────────┐
│                    TCP RELIABILITY STACK                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  1. CHECKSUM                                               │  │
│  │     Detects corrupted data                                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  2. SEQUENCE NUMBERS                                       │  │
│  │     Identifies every byte uniquely                         │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  3. ACKNOWLEDGMENTS                                        │  │
│  │     Confirms receipt of data                               │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  4. RETRANSMISSION                                         │  │
│  │     Resends lost/corrupted data                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  5. DUPLICATE DETECTION                                    │  │
│  │     Ignores duplicate packets                              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  6. IN-ORDER DELIVERY                                      │  │
│  │     Reorders out-of-sequence packets                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1. Checksum - Error Detection

**Purpose:** Detect if data was corrupted during transmission.

**How it works:**
```
Sender:
  1. Calculate checksum over TCP header + data
  2. Put checksum value in TCP header
  3. Send packet

Receiver:
  1. Calculate checksum over received data
  2. Compare with header checksum
  3. If different → Packet corrupted! Discard silently.
  4. If same → Process packet

Note: TCP doesn't retransmit on checksum failure!
      It relies on sender's timeout to detect loss.
```

**Example:**
```
Original Data: "Hello" → Checksum = 0xABCD
Corrupted:     "Hxllo" → Checksum = 0x1234 (different!)

Receiver detects mismatch → Discards packet
Sender eventually times out → Retransmits
```

### 2. Sequence Numbers - Unique Identification

**Purpose:** Every byte of data gets a unique identifier.

**Key Properties:**
- 32-bit number (wraps around after 4GB)
- ISN (Initial Sequence Number) is random for security
- Increments by bytes sent, not packets

**Example:**
```
Connection starts with ISN = 1000

Data sent: "Hello World" (11 bytes)

Byte:     H    e    l    l    o    _    W    o    r    l    d
Seq:    1000 1001 1002 1003 1004 1005 1006 1007 1008 1009 1010

Segment 1: seq=1000, len=5 ("Hello")
Segment 2: seq=1005, len=6 (" World")

ACK=1011 means: "I received everything up to byte 1010"
```

### 3. Acknowledgments - Receipt Confirmation

**Purpose:** Confirm which data was received successfully.

**Types:**
- **Cumulative ACKs:** "Got everything up to N"
- **Duplicate ACKs:** "Still waiting for N" (signals potential loss)
- **Selective ACKs (SACK):** "Got these specific ranges"

**Example:**
```
Sender sends:  [seq=100, len=100] [seq=200, len=100] [seq=300, len=100]

Receiver ACKs:
  ACK=200 → "Got bytes 100-199"
  ACK=300 → "Got bytes 100-299"
  ACK=400 → "Got bytes 100-399"

If [seq=200] is lost:
  Receiver gets [100] and [300]
  Sends: ACK=200, ACK=200 (dup!), ACK=200 (dup!)
  
  3rd dup ACK → Fast Retransmit!
```

### 4. Retransmission - Recovery from Loss

**Purpose:** Ensure lost/corrupted data is resent.

**Two Mechanisms:**

#### A. Timeout-Based Retransmission
```
Sender sends packet
       ↓
    Starts timer (RTO)
       ↓
    Timer expires (no ACK received)
       ↓
    "Packet lost!"
       ↓
    Retransmit same packet
    Double RTO (exponential backoff)
    Reduce cwnd (congestion signal!)
```

#### B. Fast Retransmit
```
Sender gets 3 duplicate ACKs
       ↓
    "Later packets arrived, but gap exists"
       ↓
    "Gap packet must be lost!"
       ↓
    Retransmit immediately (no waiting!)
       ↓
    Reduce cwnd moderately (not as severe as timeout)
```

**Comparison:**
| Mechanism | Trigger | Delay | Severity |
|-----------|---------|-------|----------|
| Timeout | Timer expires | 200ms-1s+ | High (cwnd→1) |
| Fast Retransmit | 3 dup ACKs | Immediate | Medium (cwnd/=2) |

### 5. Duplicate Detection - Idempotent Delivery

**Purpose:** Handle retransmitted packets that eventually arrive.

**The Problem:**
```
Sender sends [seq=100]
       ↓
    Packet delayed (not lost!)
       ↓
    Sender times out → Retransmits [seq=100]
       ↓
    Original arrives! → ACK sent
       ↓
    Retransmit also arrives!
       ↓
    Receiver: "I already have this!"
```

**Solution:**
```
Receiver tracks received sequence numbers:
  receivedMap = {100: true, 200: true, 300: true}

On receiving packet with seq=N:
  if receivedMap[N] == true:
    discard()  // Already have it!
  else:
    process()
    receivedMap[N] = true
    sendACK(N + len)
```

### 6. In-Order Delivery - Reordering

**Purpose:** Deliver data to application in correct sequence.

**The Problem:**
```
Sender sends: [1] [2] [3] [4]
Network routes differently:
  [1] → Path A (fast)
  [2] → Path B (slow, lost!)
  [3] → Path A (fast)
  [4] → Path A (fast)

Receiver gets: [1], [3], [4] (out of order!)
```

**Solution - Reordering Buffer:**
```
Receiver buffer:
┌─────────────────────────────────────────┐
│  Delivered  │  Gap  │  Buffered         │
│  to App     │       │  (out-of-order)   │
│  [1]        │  [2]? │  [3] [4] [5]      │
└─────────────────────────────────────────┘
       ↑
   NextExpected = 2

When [2] arrives:
  → Deliver [2], [3], [4], [5] all at once!
  → Update NextExpected = 6
```

**Algorithm:**
```
receivePacket(seq, data):
  if seq < NextExpected:
    discard()  // Old data, already delivered
  
  else if seq == NextExpected:
    deliverToApp(data)
    NextExpected += len(data)
    
    // Check buffered packets
    while buffer.contains(NextExpected):
      deliverToApp(buffer[NextExpected])
      NextExpected += len(buffer[NextExpected])
  
  else:  // seq > NextExpected (out of order)
    buffer[seq] = data  // Save for later
    sendDupACK(NextExpected)  // "I'm still waiting!"
```

### 7. Complete Reliability Example

```
Scenario: Send "Hello World" (11 bytes)

Step 1: Segmentation
  Segment 1: "Hello" (seq=1000, len=5)
  Segment 2: " World" (seq=1005, len=6)

Step 2: Sender transmits
  → Send [seq=1000, "Hello", checksum=0xABCD]
  → Send [seq=1005, " World", checksum=0x1234]

Step 3: Network issues
  [seq=1000] → Arrives! ✓
  [seq=1005] → Corrupted! Checksum mismatch ✗

Step 4: Receiver processing
  Receives [seq=1000]:
    - Checksum OK ✓
    - seq=1000 == NextExpected ✓
    - Deliver "Hello" to app
    - Send ACK=1005
    - NextExpected = 1005
  
  Receives [seq=1005] (corrupted):
    - Checksum FAIL ✗
    - Discard silently
    - No ACK sent

Step 5: Sender timeout
  - No ACK for [seq=1005] after RTO
  - "Packet lost!"
  - Retransmit [seq=1005, " World"]

Step 6: Successful delivery
  Receives [seq=1005] (retransmit):
    - Checksum OK ✓
    - seq=1005 == NextExpected ✓
    - Deliver " World" to app
    - Send ACK=1011
    - NextExpected = 1011

Step 7: Duplicate arrives!
  Original [seq=1005] finally arrives (delayed, not lost):
    - Checksum OK ✓
    - seq=1005 < NextExpected (1011)
    - "Already delivered!"
    - Discard (duplicate detection)

Result: "Hello World" delivered correctly! ✓
```

### Summary of Reliability Mechanisms

| Mechanism | Solves | How |
|-----------|--------|-----|
| **Checksum** | Data corruption | Mathematical verification |
| **Sequence Numbers** | Identification | Unique number per byte |
| **ACKs** | Confirmation | Receipt notification |
| **Retransmission** | Packet loss | Resend on timeout/dup ACK |
| **Duplicate Detection** | Double delivery | Track received seq nums |
| **Reordering Buffer** | Out-of-order | Buffer until gap filled |

---

## Connection Termination and Failure Detection

After a connection is established, what happens if one side goes down? TCP handles both graceful shutdowns and abrupt failures.

### Scenario 1: Graceful Shutdown (4-Way Handshake)

When both sides agree to close the connection cleanly:

```
Client                              Server
   │────── FIN, seq=X ───────────────→│  "I'm done sending"
   │                                  │
   │←───── ACK, ack=X+1 ─────────────│  "Got your FIN"
   │                                  │
   │←───── FIN, seq=Y ───────────────│  "I'm done too"
   │                                  │
   │────── ACK, ack=Y+1 ─────────────→│  "Got your FIN"
   │                                  │
   │◄───── BOTH CLOSED ─────────────►│
```

**Why 4-way?** Each side must independently close its half of the connection. The FIN says "I have no more data to send," and the ACK confirms receipt.

**TCP States During Close:**

```
Client Side:                    Server Side:
ESTABLISHED ──FIN──→            ESTABLISHED
     │                              │
     ↓                              ↓
FIN_WAIT_1 ←──────ACK──────→    CLOSE_WAIT
     │                              │
     ↓                              ↓
FIN_WAIT_2 ←────FIN,ACK────→    LAST_ACK
     │                              │
     ↓                              ↓
TIME_WAIT  ─────ACK───────→     CLOSED
     │
     ↓ (wait 2*MSL, typically 2-4 minutes)
   CLOSED
```

**TIME_WAIT Purpose:**
- Ensures the final ACK reaches the other side
- Prevents stale packets from old connections interfering with new ones
- Waits 2 × Maximum Segment Lifetime (MSL)

### Scenario 2: Abrupt Failure (Crash/Power Loss)

What if one side crashes without sending FIN?

```
Client                              Server
   │◄───── ESTABLISHED ─────────────►│
   │                                  │
   💥 CLIENT CRASHES!                 │
   │                                  │
   │  ?                               │
   │  ?  Server still thinks          │
   │  ?  connection is alive!         │
   │                                  │
   │←───── Server sends data ─────────│
   │      (no response)               │
   │                                  │
   │←───── Retransmit... ─────────────│
   │      (no response)               │
   │                                  │
   │←───── Timeout! ──────────────────│
   │      "Connection dead!"          │
```

**The Problem:** The surviving side has no idea the other side crashed!

### How TCP Detects Dead Connections

#### Method 1: Retransmission Timeout (RTO)

```
Server sends data to dead client:
  T=0:    Send [seq=100]
  T=1s:   No ACK, retransmit
  T=3s:   No ACK, retransmit (exponential backoff)
  T=7s:   No ACK, retransmit
  T=15s:  No ACK, retransmit
  T=31s:  No ACK...
  
  After ~9 minutes (typical Linux default):
  "Connection timed out!"
  → Close connection
  → Notify application (connection reset by peer)
```

**Exponential Backoff:** Timeouts double each retry: 1s, 2s, 4s, 8s, 16s, 32s...

#### Method 2: TCP Keepalive (Optional)

```
Server enables keepalive (SO_KEEPALIVE socket option):
  Every 2 hours (default, configurable):
    Send keepalive probe (empty ACK segment)
    
  If no response:
    Send 9 more probes every 75 seconds
    
  If still no response after all probes:
    "Connection dead!"
    → Close connection
```

**Keepalive Configuration (Linux):**
```bash
# Time before first probe (seconds)
sysctl net.ipv4.tcp_keepalive_time    # Default: 7200 (2 hours)

# Interval between probes (seconds)
sysctl net.ipv4.tcp_keepalive_intvl   # Default: 75

# Number of probes before giving up
sysctl net.ipv4.tcp_keepalive_probes  # Default: 9
```

#### Method 3: Application-Level Heartbeat

```
Application implements its own health check:
  Every 30 seconds:
    Client: "PING" message
    Server: "PONG" response
    
  If no PONG after 3 attempts (90 seconds):
    "Other side is dead!"
    → Close connection
    → Reconnect if needed
```

### Comparison: Detection Methods

| Method | Trigger | Detection Time | Overhead | Use Case |
|--------|---------|----------------|----------|----------|
| **RTO** | Data sent, no ACK | Slow (minutes) | None | Always active |
| **Keepalive** | Periodic probes | Medium (hours) | Low | Idle connections |
| **App Heartbeat** | Custom messages | Fast (seconds) | Higher | Critical systems |

### Scenario 3: Half-Open Connections

A "half-open" connection occurs when one side closes but the other doesn't know (e.g., FIN was lost):

```
Client closes, FIN lost in network:
  Client: "I'm done!" → FIN ──X──→ (lost!)
  Client: CLOSED
  
  Server: (never got FIN)
  Server: "Connection still open!"
  Server: Sends more data...
  
  Client: "I don't have that connection!"
  Client: → RST (Reset packet)
  
  Server: "Oh, connection was already dead!"
  Server: Discards data, closes connection
```

**RST (Reset) Packet:**
- Sent when receiving data for non-existent connection
- Immediately terminates the connection
- No graceful close - just abort!

### Summary: Connection Death Scenarios

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  SCENARIO 1: Graceful Close                                      │
│  Both sides agree → 4-way handshake → Clean termination         │
│  Result: Both sides know connection closed                      │
│                                                                  │
│  SCENARIO 2: One Side Crashes                                    │
│  Other side detects via timeouts or keepalive probes            │
│  Result: Eventually times out, connection closed                │
│                                                                  │
│  SCENARIO 3: Network Partition                                   │
│  Both sides think other is dead → Both eventually timeout       │
│  Result: Both close independently                               │
│                                                                  │
│  SCENARIO 4: Half-Open Connection                                │
│  One side closed, other sends data → RST received               │
│  Result: Abrupt termination                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Takeaways

| Question | Answer |
|----------|--------|
| **How does TCP detect dead peer?** | Through timeouts, keepalive probes, or lack of ACKs |
| **How long does it take?** | Minutes to hours depending on method |
| **What happens to unsent data?** | Lost if not ACKed before crash |
| **Can data loss be prevented?** | Application-level acknowledgments needed |
| **What is TIME_WAIT for?** | Ensures clean termination, prevents stale packets |

**Important:** TCP does not instantly know when a peer dies - it relies on timeouts and lack of responses to infer failure!

---

## Sequence Numbers and ACKs

### Core Concept

Every byte of data has a unique sequence number.

```
Data: "Hello World" (11 bytes)

Byte:  H  e  l  l  o     W  o  r  l  d
       │  │  │  │  │  │  │  │  │  │  │
Seq:  100 101 102 103 104 105 106 107 108 109 110

Segment 1: seq=100, len=5 ("Hello")
Segment 2: seq=105, len=6 (" World")

ACKs are cumulative:
  ACK=105 means "I got everything up to byte 104"
  ACK=111 means "I got everything up to byte 110"
```

### Cumulative ACK Example

```
Sender sends:
  [seq=100, len=100] → "Packet 1"
  [seq=200, len=100] → "Packet 2"
  [seq=300, len=100] → "Packet 3"

Receiver ACKs:
  ACK=200 → "Got bytes 100-199"
  ACK=300 → "Got bytes 100-299"
  ACK=400 → "Got bytes 100-399"
```

### Visual Representation

```
Sender's View:
┌─────────────────────────────────────────────────────────────┐
│  0-99   │  100-199  │  200-299  │  300-399  │  400+        │
│  ACKed  │   ACKed   │   SENT    │  NOT SENT │  NOT SENT    │
│         │           │  (in-flight)│           │              │
└─────────────────────────────────────────────────────────────┘
         ↑
    LastByteAcked = 200
    LastByteSent = 300
    LastByteWritten = 400
```

---

## Flow Control

### The Problem

```
Fast Server ──────────────────────→ Slow Mobile Device
    1 Gbps                             10 Mbps
    
Without flow control:
  Server sends 100MB instantly
  Mobile buffer (1MB) overflows
  Packets dropped!
  Retransmissions needed!
  Wasted bandwidth!
```

### The Solution: Sliding Window

```
Sender's Window (what can be sent):
┌──────────────────────────────────────────────────────────────┐
│ ACKed │  Sent, not ACKed  │ Can send now │  Cannot send yet │
│       │   (in-flight)     │              │                  │
└──────────────────────────────────────────────────────────────┘
        ↑                   ↑              ↑
    LastByteAcked      LastByteSent    LastByteWritten
                       + rwnd limit
```

### Receiver Window (rwnd)

The receiver advertises available buffer space in every ACK:

```
Receiver Buffer:
┌────────────────────────────────────────┐
│  Received  │  Empty Space  │          │
│   & ACKed  │   (rwnd)      │          │
└────────────────────────────────────────┘
              ↑
         rwnd = BufferSize - BufferUsed
```

### Effective Window Formula

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   Effective Window = min(rwnd, cwnd) - InFlight            │
│                                                             │
│   Where:                                                    │
│   • rwnd = Receiver's advertised window                    │
│   • cwnd = Congestion window (network capacity)            │
│   • InFlight = Data sent but not ACKed                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Example Scenario

```
Initial State:
  rwnd = 8000 bytes (receiver can take 8KB)
  cwnd = 4000 bytes (network can handle 4KB)
  InFlight = 0

  Effective Window = min(8000, 4000) - 0 = 4000 bytes

Sender sends 4000 bytes:
  InFlight = 4000
  Effective Window = min(8000, 4000) - 4000 = 0
  → BLOCKED! Must wait for ACK.

ACK arrives:
  rwnd = 8000 (receiver processed data)
  InFlight = 0
  Effective Window = min(8000, 4000) - 0 = 4000
  → Can send 4000 more bytes!
```

### Zero Window Scenario

```
Receiver buffer fills up:
  BufferSize = 10KB
  BufferUsed = 10KB
  rwnd = 0

Sender:
  Effective Window = min(0, cwnd) - InFlight = 0
  → COMPLETELY BLOCKED!

Sender must wait until receiver opens window:
  Receiver app reads data
  BufferUsed = 9KB
  rwnd = 1KB
  → Sender can send 1KB!
```

---

## Congestion Control

### The Problem

```
Many hosts sending at full speed:

  Host A ──┐
  Host B ──┼──→ Router ──→ Internet
  Host C ──┘      ↑
                  │
              Buffer fills up!
              Packets dropped!
              
Without control: CONGESTION COLLAPSE
  More loss → More retransmits → More traffic → More loss!
```

### The Solution: AIMD

**A**dditive **I**ncrease **M**ultiplicative **D**ecrease

```
  cwnd
    │
    │      ╭──╮
    │     ╱    ╲
    │    ╱      ╲___
    │   ╱            ╲___
    │  ╱                  ╲___
    │ ╱                        ╲
    │╱                          ╲___
    └────────────────────────────────
      Slow  Congestion  Packet   Slow
      Start Avoidance   Loss     Start
```

### Three Phases

#### 1. Slow Start (Exponential Growth)

```
Goal: Quickly find available bandwidth

Algorithm:
  Start: cwnd = 1 MSS
  Each ACK: cwnd += 1 MSS
  Effect: cwnd doubles every RTT

Example:
  RTT 0: cwnd = 1   (can send 1 MSS)
  RTT 1: cwnd = 2   (can send 2 MSS)
  RTT 2: cwnd = 4   (can send 4 MSS)
  RTT 3: cwnd = 8   (can send 8 MSS)
  RTT 4: cwnd = 16  (can send 16 MSS)
  
  Until: cwnd >= ssthresh (slow start threshold)
```

#### 2. Congestion Avoidance (Linear Growth)

```
Goal: Gently probe for more bandwidth

Algorithm:
  Each RTT: cwnd += 1 MSS
  
Example:
  RTT 0: cwnd = 16
  RTT 1: cwnd = 17
  RTT 2: cwnd = 18
  RTT 3: cwnd = 19
  
  (Much slower growth than Slow Start)
```

#### 3. Fast Recovery (After Loss)

```
When 3 duplicate ACKs received:
  ssthresh = cwnd / 2
  cwnd = ssthresh + 3
  Enter Congestion Avoidance (skip Slow Start!)

Why skip Slow Start?
  - Duplicate ACKs mean network still works
  - Packets are flowing (ACKs getting through)
  - Less severe than timeout
```

### Congestion Detection

| Signal | Meaning | Action |
|--------|---------|--------|
| **Timeout** | Severe congestion | cwnd = 1, ssthresh = cwnd/2 |
| **3 Dup ACKs** | Mild congestion | cwnd = cwnd/2, ssthresh = cwnd/2 |
| **ECN** | Router warning | cwnd = cwnd/2 |

### Why Reduce cwnd?

```
Packet lost → Router buffer was full
            → Network is congested
            → Need to send LESS
            → Give network time to recover

Without reduction:
  More packets → Full buffers → More drops
  More retransmits → Even MORE traffic
  💥 CONGESTION COLLAPSE 💥

With reduction:
  Fewer packets → Buffers drain → Recovery
  → Gradually increase again
```

---

## Retransmission Mechanisms

### 1. Timeout-Based Retransmission

```
Sender sends packet [seq=100]
       ↓
    Starts timer (RTO = Retransmission Timeout)
       ↓
    [Timer expires!]
       ↓
    "Packet lost!"
       ↓
    Retransmit [seq=100]
       ↓
    cwnd = 1 (severe congestion!)
```

**RTO Calculation:**
```
RTO = Estimated RTT + 4 × RTT Variance

Typical values:
  Initial RTO = 1 second
  Minimum RTO = 200ms
  Maximum RTO = 120 seconds
```

### 2. Fast Retransmit

```
Scenario: Packets 1, 2, 3 sent, but 2 is lost

Sender:    [1] → [2] → [3]
              ↓     ✗     ↓
Receiver:  Got 1   Lost  Got 3
              ↓           ↓
           ACK=2      ACK=2 (duplicate!)
              ↓           ↓
           ACK=2 (dup #2)
              ↓
           ACK=2 (dup #3!) ← FAST RETRANSMIT!
              ↓
Sender: "3 dup ACKs! Retransmit [2] immediately!"
              ↓
        Retransmit [2] (no waiting for timeout!)
```

**Why 3 duplicate ACKs?**
```
1 dup ACK:  "Maybe delayed ACK, ignore"
2 dup ACKs: "Possible loss, but wait..."
3 dup ACKs: "High probability of loss - ACT NOW!"
```

### Comparison

| Mechanism | Trigger | Delay | Severity |
|-----------|---------|-------|----------|
| Timeout | Timer expires | 200ms - 1s+ | High |
| Fast Retransmit | 3 dup ACKs | Immediate | Medium |

---

## Acknowledgment Mechanisms

### 1. Cumulative ACKs

```
One ACK acknowledges ALL bytes up to that point.

Sender:  [100-199] [200-299] [300-399]
            ↓         ↓         ↓
Receiver:  Got       Got       Got
            ↓         ↓         ↓
          ACK=200   ACK=300   ACK=400

ACK=400 means: "I got everything up to byte 399"
```

### 2. Duplicate ACKs

```
Same ACK number sent multiple times = "I'm still waiting!"

Sender:    [1] [2] [3] [4] [5]
Receiver:  Got 1, 3, 4, 5 (missing 2!)

ACKs sent:
  ACK=2 (after receiving 1)
  ACK=2 (after receiving 3) ← duplicate!
  ACK=2 (after receiving 4) ← duplicate!
  ACK=2 (after receiving 5) ← duplicate! (3rd!)

3rd duplicate ACK → Fast Retransmit!
```

### 3. Delayed ACKs

```
Instead of ACKing every packet, wait up to 500ms:

Option 1: ACK every other packet
  [1] → wait
  [2] → ACK=3 (acks both 1 and 2!)

Option 2: Timer expires
  [1] → wait 200ms
  timer expires → ACK=2

Why? Reduce ACK traffic by 50%!
```

### 4. Selective ACK (SACK) - Optional

```
Without SACK:
  Sender: [1] [2] [3] [4] [5]
  Lost:       [2]
  Receiver: ACK=2, ACK=2, ACK=2, ACK=2
  Sender: "Something lost, retransmit from 2!"
  → Retransmits [2], [3], [4], [5]!

With SACK:
  Receiver: ACK=2, SACK=3-5
  Sender: "Got it, only [2] is lost!"
  → Retransmits only [2]!
  
Much more efficient!
```

---

## TCP State Machine

```
                              +--------+
                    active    | CLOSED |
                    open      +--------+
                    or        /    ↑
                    passive  /     | close
                            /      |
                           ↓       |
                    +----------+   |
              +---->|  LISTEN  |   |
              |     +----------+   |
              |     |     ↑        |
              |     |     |        |
              |     |     |        |
    connect() |     |     | passive
              |     |     | open
              |     |     |
              |     ↓     |
              |   +-----------+    +-----------+
              +---| SYN_SENT  |--->|ESTABLISHED|<----+
                  +-----------+    +-----------+     |
                                   |   ↑   |          |
                                   |   |   |          |
                                   |   |   | close()  |
                                   |   |   ↓          |
                                   |   |  +--------+  |
                                   |   +--|CLOSE_  |  |
                                   |      | WAIT  |  |
                                   |      +-------+  |
                                   |                 |
                                   +-----------------+
```

### Key States

| State | Description |
|-------|-------------|
| **CLOSED** | No connection |
| **LISTEN** | Waiting for connection |
| **SYN_SENT** | Sent SYN, waiting for SYN-ACK |
| **SYN_RECEIVED** | Received SYN, sent SYN-ACK |
| **ESTABLISHED** | Connection active, data transfer |
| **FIN_WAIT** | Sent FIN, waiting for ACK |
| **CLOSE_WAIT** | Received FIN, waiting for app close |
| **TIME_WAIT** | Waiting for delayed packets |
| **CLOSED** | Connection terminated |

---

## Complete Example: Data Transfer

### Setup
```
Client: ISN = 1000
Server: ISN = 5000
MSS = 1000 bytes
rwnd = 5000 bytes
cwnd = 1 (Slow Start)
```

### Phase 1: Connection Establishment

```
Client                              Server
   │────── SYN, seq=1000 ────────────→│
   │                                  │
   │←───── SYN-ACK, seq=5000, ack=1001│
   │                                  │
   │────── ACK, ack=5001 ────────────→│
   │                                  │
   │◄───── ESTABLISHED ─────────────►│
```

### Phase 2: Slow Start

```
Round 1 (cwnd=1):
  Client → [seq=1001, len=1000]
  Server ← ACK=2001, rwnd=4000
  cwnd = 2

Round 2 (cwnd=2):
  Client → [seq=2001, len=1000]
  Client → [seq=3001, len=1000]
  Server ← ACK=4001, rwnd=2000
  cwnd = 4

Round 3 (cwnd=4):
  Client → [seq=4001, len=1000]
  Client → [seq=5001, len=1000]
  Client → [seq=6001, len=1000]
  Client → [seq=7001, len=1000]
  Server ← ACK=8001, rwnd=0  ← BUFFER FULL!
  cwnd = 8 (but rwnd=0!)
```

### Phase 3: Flow Control Blocks

```
Client: Effective Window = min(rwnd=0, cwnd=8) - InFlight=0 = 0
Client: ⛔ BLOCKED! Waiting for rwnd > 0

Server app reads 2000 bytes:
  BufferUsed = 3000
  rwnd = 2000
  Server ← ACK=8001, rwnd=2000

Client: Effective Window = min(2000, 8) - 0 = 2000
Client: ✓ Can send 2 more packets!
```

### Phase 4: Packet Loss Detected

```
Client sends: [seq=9001], [seq=10001]
  [seq=10001] LOST!

Server: Got 9001, expecting 10001
Server ← ACK=10001

Client: No ACK for 10001...
3 dup ACKs arrive!
Client: "Packet lost! Fast Retransmit!"

Action:
  ssthresh = cwnd / 2 = 4
  cwnd = ssthresh + 3 = 7
  Retransmit [seq=10001]
  
Enter Congestion Avoidance (skip Slow Start!)
```

### Phase 5: Congestion Avoidance

```
Round 1 (cwnd=7):
  Client sends 7 packets
  All ACKed
  cwnd = 7 + 1 = 8

Round 2 (cwnd=8):
  Client sends 8 packets
  All ACKed
  cwnd = 8 + 1 = 9

... (continues linear growth)
```

### Phase 6: Connection Termination

```
Client                              Server
   │────── FIN, seq=X ───────────────→│  "I'm done sending"
   │                                  │
   │←───── ACK, ack=X+1 ─────────────│  "Got your FIN"
   │                                  │
   │←───── FIN, seq=Y ───────────────│  "I'm done too"
   │                                  │
   │────── ACK, ack=Y+1 ─────────────→│  "Got your FIN"
   │                                  │
   │◄───── CLOSED ──────────────────►│
```

---

## Summary and Key Takeaways

### The Big Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                     TCP RELIABILITY STACK                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  APPLICATION DATA                                          │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  SEGMENTATION                                              │  │
│  │  Split data into MSS-sized chunks                          │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  SEQUENCE NUMBERS                                          │  │
│  │  Every byte gets a unique number                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  RELIABLE TRANSMISSION                                     │  │
│  │  • ACKs confirm receipt                                    │  │
│  │  • Timeouts detect loss                                    │  │
│  │  • Fast retransmit for quick recovery                      │  │
│  │  • Retransmission until ACKed                              │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  FLOW CONTROL                                              │  │
│  │  • Receiver advertises rwnd                                │  │
│  │  • Sender respects receiver's buffer limit                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  CONGESTION CONTROL                                        │  │
│  │  • cwnd limits in-flight data                              │  │
│  │  • Slow Start probes bandwidth                             │  │
│  │  • AIMD maintains stability                                │  │
│  │  • Reduces on packet loss                                  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                       │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  NETWORK                                                   │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Formulas

```
1. Effective Window:
   Effective = min(rwnd, cwnd) - InFlight

2. Slow Start:
   cwnd = cwnd × 2 (per RTT)

3. Congestion Avoidance:
   cwnd = cwnd + 1 (per RTT)

4. On Timeout:
   ssthresh = cwnd / 2
   cwnd = 1

5. On 3 Dup ACKs:
   ssthresh = cwnd / 2
   cwnd = ssthresh + 3
```

### Key Concepts

| Concept | Purpose |
|---------|---------|
| **Three-Way Handshake** | Establish connection, sync sequence numbers |
| **Sequence Numbers** | Identify every byte uniquely |
| **Cumulative ACKs** | Confirm receipt of all bytes up to N |
| **Sliding Window** | Enable pipelining, track in-flight data |
| **rwnd** | Flow control - protect receiver |
| **cwnd** | Congestion control - protect network |
| **Slow Start** | Quickly find available bandwidth |
| **AIMD** | Maintain stability, share bandwidth fairly |
| **Fast Retransmit** | Quick recovery without timeout |

### The TCP Philosophy

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  "Be polite, but efficient"                                      │
│                                                                  │
│  • Start slow (Slow Start)                                      │
│  • Speed up gradually (Congestion Avoidance)                    │
│  • Back off quickly when there's trouble (Multiplicative Dec)   │
│  • Don't overwhelm the receiver (Flow Control)                  │
│  • Don't collapse the network (Congestion Control)              │
│  • Never give up (Retransmission)                               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## References

- [RFC 793](https://tools.ietf.org/html/rfc793) - Transmission Control Protocol
- [RFC 5681](https://tools.ietf.org/html/rfc5681) - TCP Congestion Control
- [RFC 6582](https://tools.ietf.org/html/rfc6582) - The NewReno Modification
- [RFC 2018](https://tools.ietf.org/html/rfc2018) - TCP Selective Acknowledgment

---

*TCP Deep Dive Report - Educational Resource*

# Distributed Systems Project: Replicated Threaded Chat

University of Stuttgart, Distributed Systems (Prof. Marco Aiello).

Course material in this folder:
- `lectures/01 Introduction.pdf`: project requirements (slides 12–13)
- `DS_One_Page_Proposal.pdf`, `DS_Project_Report_Form.pdf`: the forms we must fill

## Idea
A CLI group chat with threads, in Python.
- 3–5 replicated chat servers find each other automatically and elect a leader.
- They keep identical chat history through reliable causally ordered multicast.
- Clients connect to any server and fail over automatically when it crashes.
- Core guarantee: a reply never appears before the message it answers, on any replica.

## Decisions made so far (and why)
- **Tier 2 in every graded requirement:** crash faults (not byzantine), causal ordering (not total).
- **Threaded chat over a social feed.**
  - Replies and reactions come from a *different* sender than the original message, so FIFO is not enough and causal ordering is clearly justified.
  - The feed's "unfriend then post" example is same-sender, so FIFO would already handle it.
  - Chat also has almost no conflict cases, because concurrent messages don't interfere.
- **Why not total order:** concurrent unrelated chat messages don't need the same order everywhere, so total order adds cost without benefit. Lecture 10 says cheaper orderings should be used when they suffice. Total order with the leader as sequencer was considered and rejected.
- **Justification quote for the report** (lecture 10, bulletin-board example): causal ordering "makes replies come after original message".
- **Python is fine:** the work is I/O-bound, so the GIL doesn't matter. Use stdlib (`socket`, `threading`, `uuid`, `json`) + `prompt_toolkit` for the CLI.
- **LCR over Bully:**
  - LCR is ring-based and its assumption of FIFO channels is met by TCP.
  - Lecture 12 says Bully is only safe "if no new process starts during an election", and server joins are one of our triggers.
- **UUIDs as node IDs:** lecture 12 shows there is no election in anonymous rings, IDs must be totally ordered, and several nodes run per host.

## Chat features
- **Must:** username, join/leave channel, post, reply in thread (`/reply <id>`), react, `/history`, `/who`, `/quit`
- **Should:** edit/delete own message (author only, so FIFO keeps the order)
- **Display:** threads indented; short message IDs; wall-clock time is shown but never used for ordering.
- **Consistency model:** causal consistency. Concurrent messages may be ordered differently on different replicas; that is by design.

## Project requirements: why each one matters and how we meet it
`01 Overview and logistics`: every requirement must have a purpose in the application; if one has none, the application is the wrong choice. So each requirement below needs two answers:
- **Why it's relevant:** asked by the preliminary and updated project forms.
- **How:** "how we plan to meet it" in the project forms, "how we met it" (what we implemented) in the report form.

Grading tiers are from `01 Introduction` slide 12. The report form also asks detail questions (TCP/UDP use, threading, discovery, voting, group view, node IDs, heartbeats, recovery, ordering, reliability), so every design choice needs a one-line justification.

### Architectural model (binary)
Form section: architecture type (client-server / peer-to-peer / hybrid), what TCP, UDP and other protocols are used for, and what multithreading or multiprocessing is used for. This is *not* the diagram; that is the System Architecture Design section.
- **Why it's relevant:** with a single server, one crash loses the whole chat history and every user. Several servers must hold the same data as equal peers, while clients stay simple and only talk to one server.
- **How:**
  - Hybrid. Servers form a peer group, each with the full log and no master for data. Clients use client-server with one server.
  - TCP/UDP split as in [Architecture](#architecture).
  - Multithreading: one thread per listener socket and per client, plus a heartbeat timer. No multiprocessing, because the work is I/O-bound.

### System architecture design (separate form section, diagram only)
Not a requirement of its own, but both forms and the report need it: one clear diagram of the key components and the connections between them.
- **Components:**
  - Each server has discovery, group view, election, failure detector, reliable causal multicast, chat log and client handler; mark the leader.
  - Each client has the CLI and session/failover logic.
- **Connections, each labelled with its protocol:**
  - Discovery broadcast (UDP)
  - Server-to-server multicast, heartbeats and NACKs (UDP)
  - Election ring (TCP)
  - State transfer (TCP)
  - Client↔server (TCP)

### Dynamic discovery of hosts (binary)
- **Why it's relevant:**
  - Servers start, crash and rejoin at runtime on different machines with no configured addresses.
  - A new server must find the group mid-chat to restore the replication level.
  - A client must find a live server at startup and again after its server crashes.
- **How:**
  - UDP broadcast: a server broadcasts `HELLO`, a client broadcasts `DISCOVER`, and the leader answers.
  - This covers both "client discovers server" and "server discovers servers".
  - Runs when the system starts and whenever a new component comes in.

### Leader election (correctness/robustness)
- **Why it's relevant:**
  - Group-view changes must be decided by one node. If every server changed the view itself, simultaneous joins or crashes could give replicas different views: they would multicast to different sets of servers and the ring would break.
  - The leader serializes view changes (admit, remove) and does the state transfer to new servers, so one server sends the log instead of all of them.
  - The leader assigns clients to servers.
  - When the leader crashes, these jobs must move to another server automatically.
  - State in the forms that the leader is deliberately *not* a sequencer for chat messages, so chat keeps working during an election. This explains why the leader holds no unique data without it looking purposeless.
- **How:**
  - LeLann-Chang-Roberts on a ring ordered by UUID.
  - Runs at startup, on server join and on leader failure.
  - The group view is used for the ring, multicast destinations, heartbeat targets and client assignment.
  - Nodes are identified by UUIDs.

### Fault tolerance (fail-stop → crash → byzantine)
- **Why it's relevant:**
  - Fault tolerance is the reason the servers are replicated: users must keep chatting without losing a message when a server, the leader or a laptop crashes.
  - UDP datagrams get lost on Wi-Fi (omission).
  - A crashed client must not stay in `/who` forever.
- **How:**
  - Tolerates crash faults of the leader, a regular server and a client, plus omission faults.
  - Heartbeats as in Failure detection; recovery as in the Recovery table below.

### Reliable ordered multicast (FIFO → causal → total)
- **Why it's relevant:**
  - Each server accepts posts from its own clients and must get them to every replica, so every user sees every message: that needs reliability.
  - Bob's reply (via server B) must never appear before Alice's question (via server A) on any replica. These are different senders, so FIFO is not enough; total order is not needed (see Decisions).
- **How:**
  - Causal ordering with vector clocks and a hold-back queue.
  - Reliability with per-sender sequence numbers, NACK retransmission, and acks piggybacked on heartbeats.

## What needs to be built
**Discovery**
- A new server broadcasts `HELLO`.
- If a leader exists, the leader admits the server, sends it the chat log + vector clock over TCP, and multicasts the new group view.
- If no leader exists, the servers start an election.
- A client broadcasts `DISCOVER`; the leader replies with a server to connect to and the server list as backup.

**Group view**
- The leader is the only one that changes the view, which is a list of servers sorted by UUID with a version number.
- The view is used for: the election ring, multicast destinations, heartbeat targets, and client assignment.

**Election (LCR)**
- Implement it as a state machine with states Idle / Participant / Leader (lecture 03).
- Tag election messages with the view version and drop stale ones.
- If the ring successor dies, skip to the next one; restart the election on timeout.
- Chat keeps working during an election, because multicast doesn't depend on the leader.

**Failure detection**
- Every server sends UDP heartbeats to every other server every 1 s; a server counts as failed after 3 misses.
- Clients ping their server over TCP every 2 s.
- Timeouts can be wrong (FLP), so call this an unreliable failure detector in the report.

**Recovery**
| Failure | What happens |
|---|---|
| Leader crash | New election + new view |
| Regular server crash | The leader removes it and the ring is repaired |
| Client crash | Its session is cleaned up |
| Client's server crash | The client reconnects to another server and resends un-acked requests with the same request ID (deduplicated) |

**Causal multicast, built in three layers (lecture 10)**
1. **Basic multicast:** UDP datagrams sent to every server in the view.
2. **Reliable multicast:**
   - Per-sender sequence number (the sender's vector-clock entry).
   - NACK for gaps.
   - Any server can retransmit, because every server keeps the full log.
   - Heartbeats carry each server's delivered vector clock (piggybacked acks), so gaps are noticed even when a sender goes quiet.
3. **Causal ordering (Birman):**
   - Vector clock as a dict keyed by server UUID.
   - Deliver a message from server j only when `m.vc[j] == VC[j] + 1` and `m.vc[k] <= VC[k]` for every other k; otherwise put it in the hold-back queue.

**Session causality**
- The client keeps the last vector clock it saw and sends it with every request.
- A server holds a request until it has caught up to that clock.
- This matters after failover. Source: the gossip architecture's front-end `prev` timestamp, lecture 13.

**Fault-injection switches** (needed for the demo)
- Drop a percentage of UDP datagrams
- Delay datagrams to one peer
- Hard crash
- Debug commands to print the view, vector clock and hold-back queue

## Demo
The demo must show each slide-12 requirement working live. Layout: one terminal per server (logs visible) and one per client.

| Requirement | What the demo shows |
|---|---|
| Architectural model | Explain the hybrid model: peer group of servers + clients on one server each; what TCP, UDP and threads are used for |
| System architecture design | Walk through the diagram: components and their connections, each labelled TCP or UDP |
| Dynamic discovery | Start servers one by one with no addresses configured; logs show them finding each other and the view growing. Clients start without addresses and get assigned to a server |
| Election | LCR messages travel around the ring; the highest UUID wins and all servers agree. Repeat when the leader is killed and when a new server joins |
| Causal multicast | Delay one server's link on purpose; Alice posts, Bob replies; the delayed server holds the reply back and shows question → reply in order |
| Reliability | Drop ~30% of UDP datagrams; NACK logs show retransmissions and every replica still ends up with the same history |
| Crash: leader | Kill the leader; the others detect it within ~3 s, elect a new leader; its clients fail over; no message lost or duplicated |
| Crash: regular server | Kill a non-leader; the view shrinks, the ring is repaired, its clients fail over, no election needed |
| Crash: client | Kill a client; it disappears from `/who`, nobody else is affected |
| Join | Start a new server mid-chat; it receives the history, a new election runs, it starts serving clients |

## Architecture
| Traffic | Protocol |
|---|---|
| Discovery (broadcast), heartbeats, server-to-server multicast, NACKs | UDP |
| Client↔server, election ring, state transfer to new servers | TCP |

- Messages are JSON. Keep UDP datagrams under ~1200 bytes; use framed JSON over TCP.
- Multithreading: one thread per socket listener and per client, plus a heartbeat timer. Ideally all protocol logic runs through a single event queue to avoid race conditions. No multiprocessing.
- The leader's jobs: group view, admitting new servers, state transfer, assigning clients to servers. The leader holds no unique data, because every server has the full log.

## Pitfalls already identified
- Set `SO_BROADCAST`. Set `SO_REUSEADDR`/`SO_REUSEPORT` so several servers on one laptop can share the discovery port.
- On macOS, prefer the subnet broadcast address (e.g. `192.168.x.255`) over `255.255.255.255`.
- University Wi-Fi may block broadcast. Demo on a phone hotspot or our own router, and test on several laptops early.
- `input()` breaks when messages arrive while the user is typing. Use `prompt_toolkit` with `patch_stdout()`.
- Protect shared state with locks: vector clock, hold-back queue, view.
- Known limitation for the report: network partitions are not handled (both sides could elect a leader).
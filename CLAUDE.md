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

## Lecture requirements (UniStu, slide 12) and how we meet them
| Requirement | Grading | Our solution |
|---|---|---|
| Architectural description | binary | Hybrid: peer group of servers + client-server for clients; diagram in proposal and report |
| Dynamic discovery of hosts | binary | UDP broadcast; servers find servers, clients find servers, at startup and on every join |
| Fault tolerance | fail-stop → crash → byzantine | Crash faults of leader, regular server and client; plus message omission via NACK |
| Election | correctness/robustness | LeLann-Chang-Roberts on a ring ordered by UUID; runs at startup, on server join, on leader failure |
| Ordered reliable multicast | FIFO → causal → total | Causal (vector clocks + hold-back queue), reliable via sequence numbers + NACK retransmission |

The final report form asks about all of these (TCP/UDP use, threading, discovery, voting, group view, node IDs, heartbeats, recovery, ordering, reliability), so every design choice needs a one-line justification.

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
| Architectural description | Walk through the architecture diagram: servers, clients, which links use TCP and which UDP |
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
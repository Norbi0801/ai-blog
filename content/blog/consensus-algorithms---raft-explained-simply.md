+++
title = "Consensus algorithms - Raft explained simply"
date = 2025-03-18
description = "How distributed systems agree on a single source of truth using the Raft consensus algorithm - leader election, log replication, and what happens when the network splits."

[taxonomies]
tags = ["distributed-systems", "algorithms", "rust", "architecture"]
+++

Five servers. Three data centers. One database. A client writes `balance = 500`. The write hits server A. Servers B through E haven't seen it yet. Server A crashes. What's the balance?

This is the consensus problem. Not in the abstract academic sense, but in the "your customers just lost money" sense. Every replicated database, every distributed lock service, every configuration store that spans multiple machines has to solve this. And for decades, the standard answer was Paxos - an algorithm so notoriously difficult to understand that Lamport himself had to publish a simplified version of his own paper, and people still couldn't implement it correctly.

Then in 2014, Diego Ongaro and John Ousterhout at Stanford published ["In Search of an Understandable Consensus Algorithm"](https://raft.github.io/raft.pdf). It won Best Paper at USENIX ATC. The algorithm was called Raft, and its primary design goal wasn't performance or features - it was understandability.

<!-- more -->

## What consensus actually means

Consensus is getting a group of machines to agree on a sequence of values, even when some machines crash or the network between them drops packets. The key word is "sequence" - it's not enough for everyone to eventually agree on one value. You need an ordered log of decisions that every surviving machine applies in the same order.

This is what makes it hard. Any two machines might see events in different orders. A network partition might split your cluster in half. A leader might accept a write, crash before replicating it, and come back with stale data. The algorithm has to handle all of this and guarantee that no two machines ever disagree about what was committed.

Formally, a consensus algorithm must satisfy three properties:

1. **Agreement** - all non-faulty nodes decide on the same value
2. **Validity** - the decided value was actually proposed by some node
3. **Termination** - all non-faulty nodes eventually decide

FLP impossibility (Fischer, Lynch, Paterson 1985) proved that no deterministic algorithm can guarantee all three in an asynchronous system with even one crash failure. Every practical algorithm, including Raft, works around this by using timeouts - technically breaking the "purely asynchronous" assumption.

## Why not Paxos?

Paxos, published by Leslie Lamport in 1989 (and famously written as a parable about a Greek parliament), solves consensus correctly. The problem is that almost nobody can implement it correctly from the paper alone. The single-decree version handles agreeing on one value. To build a real system you need Multi-Paxos, which Lamport never fully specified - every production implementation (Google's Chubby, Apache ZooKeeper's ZAB) ended up being a custom variant that diverged significantly from the original.

Ongaro ran a [user study](https://raft.github.io/raft.pdf) with 43 students across two universities. After learning both algorithms, 33 of 43 answered Raft questions more accurately than Paxos questions. The median Raft quiz score was substantially higher.

The core insight: Paxos is a monolithic algorithm with interleaved concerns. Raft decomposes consensus into three relatively independent subproblems - leader election, log replication, and safety - and addresses each one with mechanisms that are easy to reason about individually.

Heidi Howard and Richard Mortier later argued in ["Paxos vs Raft: Have we reached consensus on distributed consensus?"](https://arxiv.org/abs/2004.05074) that the algorithms are more similar than different, and that much of Raft's advantage comes from the clarity of the paper rather than fundamental algorithmic differences. Fair point - but clarity is exactly what matters when you're debugging a production outage at 3 AM.

## The three states

Every node in a Raft cluster is in exactly one state at any time:

```rust
#[derive(Debug, Clone, PartialEq)]
enum NodeState {
    Follower,
    Candidate,
    Leader,
}
```

**Followers** are passive. They respond to RPCs from the leader and candidates but don't initiate anything. **Candidates** are followers that got tired of waiting for a heartbeat and decided to run for leader. **Leaders** handle all client requests, replicate log entries to followers, and send periodic heartbeats.

All nodes start as followers. If a follower doesn't hear from a leader within its election timeout (randomized, typically 150-300ms), it becomes a candidate and starts an election.

## Terms: Raft's logical clock

Time in Raft is divided into **terms** - monotonically increasing integers. Each term starts with an election. If a candidate wins, it serves as leader for the rest of that term. If no one wins (split vote), the term ends with no leader and a new election begins.

```rust
#[derive(Debug, Clone)]
struct RaftNode {
    id: u64,
    state: NodeState,
    current_term: u64,
    voted_for: Option<u64>,
    log: Vec<LogEntry>,
    commit_index: u64,
    last_applied: u64,
}

#[derive(Debug, Clone)]
struct LogEntry {
    term: u64,
    index: u64,
    command: Vec<u8>,
}
```

Terms act as a logical clock for detecting stale information. Every RPC includes the sender's term. If a node receives a message with a higher term than its own, it immediately updates its term and reverts to follower. If it receives a message with a lower term, it rejects it. This is how old leaders discover they've been replaced.

## Leader election

When a follower's election timeout fires, here's what happens step by step:

1. The follower increments its `current_term`
2. It transitions to Candidate state
3. It votes for itself
4. It resets its election timer
5. It sends `RequestVote` RPCs to all other nodes in parallel

```rust
#[derive(Debug, Clone)]
struct RequestVote {
    term: u64,
    candidate_id: u64,
    last_log_index: u64,
    last_log_term: u64,
}

#[derive(Debug, Clone)]
struct RequestVoteResponse {
    term: u64,
    vote_granted: bool,
}
```

A node grants its vote if **all** of the following are true:
- The candidate's term is at least as large as the voter's current term
- The voter hasn't already voted for someone else in this term
- The candidate's log is at least as up-to-date as the voter's (compared by last entry's term first, then log length)

That last condition is critical. It's called the **election restriction** and it guarantees that any elected leader already has all committed entries in its log. Without this check, a newly elected leader could overwrite committed data.

A candidate wins when it gets votes from a **majority** of the cluster (itself included). In a 5-node cluster, that's 3 votes. The majority requirement means at most one leader can be elected per term - two majorities in the same cluster always overlap by at least one node, and each node votes at most once per term.

### Split votes

What if two candidates start elections at the same time? They might split the vote - neither gets a majority. When this happens, both candidates' election timers expire, they increment their terms again, and try once more. The randomized election timeout makes this unlikely to repeat - one candidate will almost always time out before the other.

This is beautifully simple compared to Paxos, where the proposer/acceptor/learner roles and the prepare/accept phases can create much more complex contention patterns.

## Log replication

Once a leader is elected, it handles all client requests. When a client sends a command:

1. The leader appends the command to its own log
2. It sends `AppendEntries` RPCs to all followers in parallel
3. Once a majority of nodes have the entry, the leader marks it as **committed**
4. The leader applies the committed entry to its state machine and responds to the client

```rust
#[derive(Debug, Clone)]
struct AppendEntries {
    term: u64,
    leader_id: u64,
    prev_log_index: u64,
    prev_log_term: u64,
    entries: Vec<LogEntry>,
    leader_commit: u64,
}

#[derive(Debug, Clone)]
struct AppendEntriesResponse {
    term: u64,
    success: bool,
}
```

The `prev_log_index` and `prev_log_term` fields implement a consistency check. A follower only accepts new entries if it already has the entry at `prev_log_index` with a matching `prev_log_term`. This guarantees the **Log Matching Property**: if two logs contain an entry with the same index and term, all preceding entries are also identical.

If a follower rejects (because it's missing entries or has a conflict), the leader decrements its `next_index` for that follower and retries. This backtracking continues until the leader finds the point where the logs agree, then the follower's log is brought up to date. Conflicting entries on the follower are overwritten - only the leader's log is authoritative.

### Heartbeats

When the leader has nothing to replicate, it still sends `AppendEntries` RPCs with empty `entries` vectors. These are heartbeats. They reset followers' election timers and carry the leader's `leader_commit` index so followers know which entries are safe to apply.

The heartbeat interval is typically much shorter than the election timeout - something like 50ms heartbeat vs 150-300ms election timeout. This keeps the cluster stable during normal operation. Raft spends almost all its time in this steady state: leader sends heartbeats, followers acknowledge, everyone agrees.

## Commit index and safety

The `commit_index` tracks the highest log entry known to be replicated on a majority. The leader maintains it; followers learn about it through `leader_commit` in AppendEntries RPCs.

There's a subtle but important rule: a leader only counts replicas to commit entries from its **own term**. It never directly commits entries from previous terms by counting replicas. Those older entries get committed indirectly - when a new entry from the current term is committed, all preceding entries (including ones from older terms) are implicitly committed too.

Why? Without this restriction, a scenario exists where a leader could count replicas for an old-term entry, commit it, then crash - and a new leader could legitimately overwrite that entry because it won an election without having it. The Raft paper (Figure 8) walks through this scenario in detail. It's one of the trickiest parts of the protocol and a common source of bugs in implementations.

Raft guarantees five safety properties:

1. **Election Safety** - at most one leader per term
2. **Leader Append-Only** - leaders never overwrite or delete their own log entries
3. **Log Matching** - same index + same term = identical logs up to that point
4. **Leader Completeness** - if an entry is committed in term T, every leader in terms > T has that entry
5. **State Machine Safety** - if a server applies entry at index I, no other server applies a different entry at index I

Together, these guarantee that every node's state machine sees the exact same sequence of commands. That's the whole point.

## Network partitions

This is where consensus algorithms earn their keep. Consider a 5-node cluster: nodes A, B, C, D, E. Node A is the leader. A network partition splits the cluster into {A, B} and {C, D, E}.

**In the majority partition {C, D, E}:**
- C, D, E stop receiving heartbeats from A
- After election timeout, one of them (say C) starts an election with a higher term
- C gets votes from D and E - that's 3 out of 5, a majority
- C becomes the new leader
- Client writes to {C, D, E} succeed normally - majority acknowledgment is possible

**In the minority partition {A, B}:**
- A still thinks it's the leader
- A accepts client writes and appends them to its log
- A tries to replicate to followers, but can only reach B
- 2 out of 5 is not a majority - these entries **never get committed**
- A cannot respond to clients with success (assuming the client waits for commit)

**When the partition heals:**
- A receives a message from C (or any node in the majority partition) with a higher term
- A immediately steps down to follower
- A's uncommitted entries get overwritten by C's log during the next AppendEntries consistency check
- The cluster converges to a single consistent state

No split brain. No data loss (committed data was never lost - only uncommitted writes from the minority partition were discarded). This is Raft being a **CP system** under the [CAP theorem](https://en.wikipedia.org/wiki/CAP_theorem) - it sacrifices availability in the minority partition to maintain consistency.

This is also why Raft clusters should always have an **odd number of nodes**. A 4-node cluster and a 3-node cluster both tolerate exactly 1 failure (both need 3 nodes for a majority of 4, vs 2 nodes for a majority of 3). The 4th node costs money and network bandwidth without improving fault tolerance. Use 3, 5, or 7 nodes.

## Who uses Raft in production

Raft isn't academic anymore. It runs some of the most critical infrastructure on the internet.

**[etcd](https://github.com/etcd-io/etcd)** (51k+ GitHub stars) is the configuration store behind Kubernetes. Every Kubernetes cluster runs etcd, and etcd uses Raft for replication. The [etcd Raft library](https://github.com/etcd-io/raft) has been extracted as a standalone Go package and is self-described as "the most widely used Raft library in production, serving tens of thousands of clusters each day."

**[CockroachDB](https://github.com/cockroachdb/cockroach)** (32k+ GitHub stars) takes it further with **Multi-Raft**. Data is divided into ranges, and each range runs its own Raft consensus group. A single CockroachDB node might participate in hundreds of thousands of Raft groups simultaneously. They use etcd's Raft library under the hood but coalesce heartbeats across groups to avoid drowning in network traffic. Their [blog post on scaling Raft](https://www.cockroachlabs.com/blog/scaling-raft/) is excellent reading.

**[HashiCorp Consul](https://github.com/hashicorp/raft)** (9k+ stars for the Raft library alone) uses Raft for service discovery and configuration. Their Go implementation is higher-level than etcd's - it includes built-in snapshotting, dynamic membership changes, and pluggable storage backends.

**[TiKV](https://github.com/tikv/tikv)** - the distributed key-value store behind TiDB - uses Raft in Rust via [raft-rs](https://github.com/tikv/raft-rs) (3.3k+ stars). Like CockroachDB, TiKV shards data into regions, each with its own Raft group.

## Raft in Rust

If you're working in Rust, there are two mature Raft libraries worth knowing.

**[raft-rs](https://github.com/tikv/raft-rs)** by TiKV is a port of etcd's Raft implementation. It's minimal - you implement storage and transport, the library handles consensus logic. This mirrors etcd's design philosophy of separating the core algorithm from I/O:

```rust
// raft-rs gives you the consensus state machine.
// You provide storage and networking.
//
// Cargo.toml:
// [dependencies]
// raft = "0.7"

use raft::prelude::*;

// You implement the Storage trait for your persistence layer
pub trait Storage {
    fn initial_state(&self) -> Result<RaftState>;
    fn entries(
        &self,
        low: u64,
        high: u64,
        max_size: impl Into<Option<u64>>,
    ) -> Result<Vec<Entry>>;
    fn term(&self, idx: u64) -> Result<u64>;
    fn first_index(&self) -> Result<u64>;
    fn last_index(&self) -> Result<u64>;
    fn snapshot(&self, request_index: u64, to: u64) -> Result<Snapshot>;
}
```

The library is event-driven - you call `RawNode::tick()` on a timer, `RawNode::propose()` when you get a client request, and `RawNode::step()` when you receive a message from another node. Then you call `RawNode::ready()` to get a batch of things that need to happen (messages to send, entries to persist, entries to apply). You do all the I/O. The library never touches the network or disk directly.

**[openraft](https://github.com/databendlabs/openraft)** (1.9k+ stars) is the async-native alternative, built on Tokio. It's higher-level than raft-rs - you implement application-level traits and it handles the tick loop internally:

```rust
// openraft takes a more opinionated, async-first approach.
//
// Cargo.toml:
// [dependencies]
// openraft = "0.9"

use openraft::Config;

// Your state machine implements the RaftStateMachine trait.
// Your log store implements the RaftLogStorage trait.
// Your network layer implements the RaftNetworkFactory trait.
//
// openraft drives the consensus loop for you.

let config = Config {
    heartbeat_interval: 500,       // ms
    election_timeout_min: 1500,    // ms
    election_timeout_max: 3000,    // ms
    ..Default::default()
};
```

openraft originated as a fork of async-raft (now archived) with bug fixes and additional features. It powers [Databend](https://github.com/datafuselabs/databend)'s meta-service cluster in production.

Which one should you pick? raft-rs if you want maximum control and battle-tested production pedigree (TiKV runs on it). openraft if you want an async-first API and less boilerplate. Both are correct implementations.

## Why you probably shouldn't implement it yourself

Raft is understandable. That's the whole selling point. You can read the paper in an afternoon and feel like you get it. And you probably do get it - the core algorithm is genuinely elegant.

The trap is thinking that understanding means it's easy to implement correctly.

Production Raft has to handle: log compaction and snapshotting (what happens when the log grows to millions of entries?), dynamic membership changes (adding and removing nodes without downtime), pre-vote protocol (preventing disruption when a partitioned node rejoins with a high term), read-only queries without log replication (for performance), leadership transfer, batching and pipelining for throughput, and dozens of edge cases around crash recovery timing.

The Raft paper is 18 pages. Diego Ongaro's [PhD dissertation](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf), which covers all the production concerns, is 257 pages. That's the gap between understanding Raft and shipping Raft.

Use etcd (or Consul, or the Raft library that fits your language) for configuration and coordination. Use CockroachDB or TiDB for distributed SQL. Use raft-rs or openraft if you're building something in Rust that genuinely needs embedded consensus - but really think about whether you do. Most applications that "need distributed consensus" actually need a distributed database or a lock service, and someone else has already built a good one.

## See it in action

The best way to internalize Raft is to watch it run. [The Secret Lives of Data](https://thesecretlivesofdata.com/raft/) has an interactive visualization by Ben Johnson (creator of BoltDB) that walks through leader election and log replication step by step. Spend ten minutes with it. It makes the entire protocol click in a way that reading never quite does.

The [official Raft website](https://raft.github.io/) also has an interactive visualization where you can trigger partitions, kill nodes, and watch the cluster recover. This one is more of a sandbox - useful once you already understand the basics.

And read [the paper](https://raft.github.io/raft.pdf). Seriously. It's one of the most well-written systems papers out there, and it's only 18 pages. If you work with distributed systems at all, understanding how consensus works at this level will change how you think about replication, consistency, and failure modes.

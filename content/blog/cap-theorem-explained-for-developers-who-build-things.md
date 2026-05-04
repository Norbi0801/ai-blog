+++
title = "CAP theorem explained for developers who build things"
date = 2025-07-13
description = "What the CAP theorem actually means for your architecture - CP vs AP systems, why CA doesn't exist, PACELC, eventual consistency, and how to pick the right tradeoff."

[taxonomies]
tags = ["distributed-systems", "architecture", "databases"]
+++

Two database replicas. A network cable gets unplugged between them. A client sends a write to replica A. What do you do?

Option one: accept the write on A, serve stale data from B, reconcile later. Option two: reject the write (or stall) until A and B can talk again, so they never disagree. There is no option three. This is the CAP theorem, and once you internalize it, half of the "why did they design it that way?" questions about distributed databases answer themselves.

<!-- more -->

## The theorem, precisely

Eric Brewer presented the CAP conjecture at the ACM Symposium on Principles of Distributed Computing in 2000. Seth Gilbert and Nancy Lynch [proved it formally](https://users.ece.cmu.edu/~adrian/731-sp04/readings/GL-cap.pdf) at MIT in 2002. The theorem states that a distributed data store can provide at most two of these three guarantees simultaneously:

**Consistency (C)** - every read receives the most recent write or an error. Not "eventually" - immediately. If client X writes `balance = 500` and the system acknowledges it, client Y reading a millisecond later must see 500. This is linearizability, the strongest form of consistency. Not to be confused with the C in ACID, which means something different (transaction constraints).

**Availability (A)** - every request to a non-failing node receives a response, without guarantee that it contains the most recent write. The key word is "every." If node B is up and reachable, it must respond. It can't say "hold on, let me check with node A first" if A is unreachable.

**Partition tolerance (P)** - the system continues to operate despite arbitrary message loss or delay between nodes. Network partitions are not theoretical - they're Tuesday. A 2011 study by Bailis and Kingsbury found that [partitions happen in every real-world distributed system they examined](https://arxiv.org/abs/1509.05393), including within single data centers.

## "Pick two" is the wrong mental model

The popular framing is "pick two out of three." This is technically accurate but practically misleading, because it implies you have three viable combinations: CP, AP, and CA. You don't. You have two.

Here's why. In any system that spans more than one network node, partitions will happen. Switches fail. Cables get cut. Cloud regions have outages. GCP had a [73-minute global networking incident in June 2022](https://status.cloud.google.com/incidents/fmEL9i2fArADKawkZAa0) that affected cross-region traffic. AWS us-east-1 goes down with enough regularity that it's become a running joke on Hacker News.

If you don't tolerate partitions, your system stops existing as a distributed system when the network splits. So P is not optional - it's a prerequisite. The actual choice is:

**During a partition, do you sacrifice consistency or availability?**

That's it. CP or AP. When the network is healthy and all nodes can communicate, you can have both C and A just fine. The tradeoff only materializes when nodes can't reach each other.

## CA systems: the option that doesn't exist

A CA system would provide consistency and availability but not tolerate partitions. This is a single-node database. PostgreSQL on one server. SQLite. Your local HashMap. The moment you add a second node with replicated data and the network between them can fail, you're in CAP territory and CA falls off the table.

Some people argue that "CA" applies to traditional single-node RDBMS systems. Technically true, but calling a single-server database "CA" is like calling a bicycle "partition tolerant within its frame" - correct but not useful. CAP is about distributed systems. If your data lives on one machine, CAP doesn't apply to you, and you have different problems (that machine is a single point of failure).

## CP systems: correctness over uptime

CP systems guarantee that every read returns the latest write, even if some nodes become unavailable during a partition. When the network splits, the minority partition stops accepting writes (or reads, depending on configuration) rather than risk serving stale data.

**etcd** is the textbook CP system. It uses [Raft consensus](/blog/consensus-algorithms---raft-explained-simply/) to replicate a key-value store across a cluster. Every write must be acknowledged by a majority of nodes before it's committed. During a partition, the minority side can't form a quorum, so it rejects writes. The majority side continues operating normally.

This is exactly the right tradeoff for etcd's use case: storing Kubernetes cluster state. If two nodes disagree about which pods are scheduled where, you get split-brain scheduling and containers running where they shouldn't. Stale config data is worse than temporarily unavailable config data.

**MongoDB** with default write concern (`w: "majority"`) and read concern (`readConcern: "majority"`) behaves as a CP system. Writes require acknowledgment from a majority of replica set members. During a partition, if the primary ends up in the minority partition, it steps down. The majority partition [elects a new primary](https://www.mongodb.com/docs/manual/replication/#automatic-failover). Clients connected to the minority partition get errors until the partition heals.

```javascript
// MongoDB: CP behavior with majority write concern.
// If the primary can't reach a majority, this write will fail
// rather than accept data that might diverge.
db.accounts.updateOne(
  { _id: "user-123" },
  { $set: { balance: 500 } },
  { writeConcern: { w: "majority", wtimeout: 5000 } }
)
```

The `wtimeout: 5000` is important. Without it, the write blocks indefinitely waiting for a majority. With it, you get an error after 5 seconds - the system is choosing consistency over availability.

**CockroachDB** and **TiKV** are also CP. Both use Raft (I covered Raft's partition behavior [in detail here](/blog/consensus-algorithms---raft-explained-simply/)). CockroachDB runs a separate Raft group per data range, so a partition might make some ranges unavailable while others keep working - a form of partial availability that pure CAP classification doesn't capture well.

## AP systems: always respond, sort it out later

AP systems guarantee that every non-failing node responds to requests, even during a partition. The cost is that different nodes might temporarily return different values for the same key.

**Apache Cassandra** is the classic AP system. Data is replicated across multiple nodes using consistent hashing. When you write, the coordinator sends the write to all replicas responsible for that key. The write succeeds as soon as the configured consistency level is met:

```
# Cassandra consistency levels for a replication factor of 3:
#
# ONE      - 1 replica acknowledges (fastest, least consistent)
# TWO      - 2 replicas acknowledge
# QUORUM   - floor(3/2) + 1 = 2 replicas acknowledge
# ALL      - all 3 replicas acknowledge (slowest, most consistent)
#
# With QUORUM writes + QUORUM reads:
#   W + R > RF  ->  2 + 2 > 3  ->  guaranteed overlap
#   At least one node participated in both the write and the read,
#   so the read is guaranteed to see the latest write.
```

Here's what makes Cassandra interesting: with `QUORUM` reads and writes, you get something that looks a lot like consistency. The formula `W + R > RF` (write replicas + read replicas > replication factor) guarantees that at least one node in the read set also participated in the write. But during a partition, if you can't reach a quorum, you face the AP tradeoff - Cassandra will still serve reads from available replicas at lower consistency levels, returning potentially stale data rather than erroring.

**Amazon DynamoDB** defaults to eventually consistent reads. When you write an item, DynamoDB replicates it across multiple storage nodes in an AWS region. A read immediately after might hit a node that hasn't received the update yet:

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('accounts')

# Write
table.put_item(Item={'user_id': 'user-123', 'balance': 500})

# Eventually consistent read (default) - might return old value
response = table.get_item(Key={'user_id': 'user-123'})
# response['Item']['balance'] could be 500... or could be the old value

# Strongly consistent read - guaranteed to return 500,
# but costs 2x the read capacity units and has higher latency
response = table.get_item(
    Key={'user_id': 'user-123'},
    ConsistentRead=True
)
```

DynamoDB added [Multi-Region Strong Consistency (MRSC)](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/multi-region-strong-consistency.html) for global tables in January 2025, which provides strongly consistent reads across AWS regions. Under the hood this requires cross-region coordination, trading latency for consistency. The default is still eventually consistent because for most read patterns, it's fast enough and cheap enough.

## Eventual consistency: what "eventually" actually means

"Eventually consistent" sounds vague. How eventual is eventual? In practice, for most AP systems, "eventually" means milliseconds to low single-digit seconds under normal operation. Cassandra anti-entropy repair, DynamoDB replication, Riak's read repair - they're all fast when the network is healthy.

The problem isn't the normal case. It's the edge cases. A node goes down for 30 minutes, comes back up, and now has 30 minutes of stale data. A network partition lasts 10 seconds and during that window, two clients write conflicting values to the same key on different sides of the partition. What happens?

This is where conflict resolution strategies matter:

**Last-write-wins (LWW)** - the write with the highest timestamp wins. Simple but dangerous. Clocks drift. Two writes at "the same time" from different nodes have different timestamps. One silently disappears. DynamoDB uses LWW by default.

**Vector clocks** - each node maintains a logical clock that tracks causal ordering. When two writes are concurrent (neither caused the other), the system detects the conflict and either keeps both versions or lets the application resolve it. Riak used this approach. Amazon's original [Dynamo paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf) describes it in detail.

**CRDTs (Conflict-free Replicated Data Types)** - data structures that are mathematically guaranteed to converge. A G-Counter (grow-only counter) assigns each node its own counter. To increment, a node bumps its own value. To read the total, sum all nodes' values. Merging two divergent states is always just taking the max of each node's counter. No conflicts possible.

```rust
use std::collections::HashMap;

/// A grow-only counter using CRDT semantics.
/// Each node tracks its own count. The global value
/// is the sum of all nodes' counts. Merge is always
/// safe - just take the max of each node's counter.
#[derive(Debug, Clone)]
struct GCounter {
    counts: HashMap<String, u64>,
}

impl GCounter {
    fn new() -> Self {
        GCounter {
            counts: HashMap::new(),
        }
    }

    fn increment(&mut self, node_id: &str) {
        *self.counts.entry(node_id.to_string()).or_insert(0) += 1;
    }

    fn value(&self) -> u64 {
        self.counts.values().sum()
    }

    /// Merge two counters. For each node, take the max.
    /// This is associative, commutative, and idempotent -
    /// the three properties that make CRDTs work.
    fn merge(&mut self, other: &GCounter) {
        for (node_id, &count) in &other.counts {
            let entry = self.counts.entry(node_id.clone()).or_insert(0);
            *entry = (*entry).max(count);
        }
    }
}

fn main() {
    let mut counter_a = GCounter::new();
    let mut counter_b = GCounter::new();

    // Node A increments 3 times
    counter_a.increment("node-a");
    counter_a.increment("node-a");
    counter_a.increment("node-a");

    // Node B increments 2 times (independently, during a partition)
    counter_b.increment("node-b");
    counter_b.increment("node-b");

    // Partition heals - merge both directions
    counter_a.merge(&counter_b);
    counter_b.merge(&counter_a);

    // Both nodes agree: total is 5
    assert_eq!(counter_a.value(), 5);
    assert_eq!(counter_b.value(), 5);
}
```

CRDTs trade expressiveness for safety. A grow-only counter is easy. A "set with arbitrary add and remove" is harder (OR-Set). A "collaboratively edited text document" pushes CRDTs to their limits (RGA, Yjs). The mathematical guarantee of convergence comes at the cost of restricting what operations you can express.

## PACELC: what CAP leaves out

CAP only talks about what happens during a partition. But most of the time, there's no partition. Your nodes are happily communicating. Do you still face tradeoffs?

Yes. Daniel Abadi proposed [PACELC](https://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf) in 2012 to capture this: if there's a **P**artition, choose between **A**vailability and **C**onsistency. **E**lse (normal operation), choose between **L**atency and **C**onsistency.

The "else" clause is the important part. Even when the network is fine, guaranteeing strong consistency requires coordination between nodes. That coordination adds latency. You need round trips, quorum acknowledgments, possibly two-phase commits. A system can skip that coordination and respond faster, at the cost of potentially returning slightly stale data.

Real systems under PACELC:

| System | During partition (PAC) | Normal operation (ELC) | Classification |
|---|---|---|---|
| etcd | PC (sacrifices availability) | EC (sacrifices latency for consistency) | PC/EC |
| MongoDB (w:majority) | PC | EC | PC/EC |
| Cassandra (ONE) | PA (sacrifices consistency) | EL (sacrifices consistency for latency) | PA/EL |
| DynamoDB (default) | PA | EL | PA/EL |
| DynamoDB (consistent read) | PA | EC | PA/EC |
| CockroachDB | PC | EC | PC/EC |
| Google Spanner | PC | EC | PC/EC |

Notice how DynamoDB appears twice. The same database can behave differently depending on how you configure it. Cassandra with `QUORUM` reads and writes is closer to PC/EC. MongoDB with `w: 1` is closer to PA/EL. CAP and PACELC describe tradeoff spaces, not fixed labels.

Google Spanner is a fascinating edge case. It's PC/EC but achieves global consistency using [TrueTime](https://cloud.google.com/spanner/docs/true-time-external-consistency), a clock system based on GPS receivers and atomic clocks in every data center. It trades extra hardware cost and write latency (commit wait of ~7ms to account for clock uncertainty) for the ability to do globally consistent reads without cross-region round trips. It's the closest thing to "buying your way out of CAP" with hardware, but it still pays the PACELC latency cost.

## How this shapes your architecture

When you're designing a system, the CAP/PACELC question isn't abstract. It translates to concrete decisions:

**User-facing reads where staleness is fine**: use AP/EL. Product catalog, social media feeds, recommendation lists. If a user sees a product that sold out 200ms ago, it's annoying but not catastrophic. DynamoDB eventually consistent reads, Cassandra with `ONE` consistency, or a read replica with async replication.

**Financial transactions, inventory counts, anything involving money**: use CP/EC. Double-spending is worse than brief unavailability. If the bank transfer service is down for 30 seconds during a network issue, that's better than crediting an account twice. MongoDB with majority concern, CockroachDB, or PostgreSQL with synchronous replication.

**Configuration and coordination**: use CP/EC. If two services disagree about which node is the primary, or which feature flags are enabled, bad things happen. This is etcd's domain. Consistency matters more than responding fast.

**Metrics, logs, analytics**: use AP/EL. Losing a few data points during a partition is acceptable. Dropping all incoming metrics because the aggregation cluster can't reach quorum is not. This is why time-series databases like InfluxDB and analytics systems like ClickHouse tend toward AP designs.

A single application often needs different tradeoffs for different data. Your user's shopping cart might be AP (merge conflicts are resolvable - just union the items). Their account balance must be CP (you can't merge two conflicting balances). The product recommendation cache is AP. The payment processing pipeline is CP.

This is also why [CQRS (Command Query Responsibility Segregation)](/blog/cqrs-in-practice---separating-reads-from-writes/) pairs naturally with CAP thinking. Your write model can use a CP store for correctness. Your read model can use an AP store (or even a plain cache) for speed. Events bridge the two, with eventual consistency as the explicit contract between them.

## What CAP doesn't tell you

CAP is a useful mental model but it has limits. It says nothing about:

- **Latency** - PACELC covers this, but CAP treats a response that takes 30 days as "available"
- **Degree of consistency** - there's a spectrum between linearizability and eventual consistency. Causal consistency, read-your-writes, monotonic reads - all fall in between and are often good enough
- **Partial failures** - CAP assumes a clean partition. Real failures are messier: a node that's slow but not dead, a network that drops 10% of packets, one replica that's behind by 2 seconds
- **Recovery time** - how fast does your system converge after a partition heals? CAP doesn't say

Martin Kleppmann's [critique of CAP](https://martin-kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html) is worth reading. His argument: real systems are too nuanced to fit into two letters. He's right. But CAP still gives you the vocabulary to ask the right questions when you're staring at a database comparison chart at 2 AM trying to pick the right store for your new service.

The theorem doesn't make the decision for you. It tells you what decisions exist. That's enough.

# Apache Kafka

[Apache Kafka](https://kafka.apache.org/) is a **distributed event streaming platform**: it stores streams of records durably, processes them in order where it matters, and lets many clients read at their own pace. Teams use it for **pipelines**, **integration**, **analytics**, and **microservice decoupling** at high throughput.

## What is event streaming?

**Event streaming** means modeling changes as an append-only log of facts over time (events), instead of only synchronous request–response calls.

### Example: food delivery

A single order produces many lifecycle events:

```text
Order created → placed → accepted → preparing → prepared
  → dispatched → delivering → delivered → complete
```

Each step can be published once; **downstream services subscribe** to what they care about (restaurant app, courier, support, analytics) without the order service calling them all directly.

### Why not only HTTP APIs?

If the order service must **push** every state change to many dependents, you get **N×M coupling**: more services means more outbound calls, retries, timeouts, and failure modes.

```text
User → Order service ──► Restaurant API
                    ├──► Delivery API
                    ├──► Customer support API
                    └──► Internal ops / analytics
```

Problems at scale:

| Concern | Why it hurts |
|--------|----------------|
| **Fan-out** | Each order multiplies outbound calls; spikes become synchronized load on every downstream. |
| **Retries** | Retries amplify traffic and can cause retry storms without careful backoff and idempotency. |
| **Discovery & config** | Every caller needs addresses, routing, auth config for every callee. |
| **Backpressure** | If one consumer is slow, the producer must shed load, buffer, or drop—policy belongs in the platform. |

For domains like **market data or trading**, synchronous fan-out at **millions of messages per minute** can overwhelm brittle chains of APIs. **Streaming** decouples producers and consumers in **time**: consumers read when they are ready, from a **durable log**, with clear retention and replay rules.

Kafka is one implementation of that pattern; others include **NATS JetStream**, **Redpanda**, **Pulsar**, **AWS Kinesis**, **Azure Event Hubs**, and **Google Pub/Sub**—each with different ops models and guarantees.

---

## Mental model: topics, brokers, partitions

```text
Producers ──► Topic (logical name)
                 │
                 ├── Partition 0 ──► ordered log segment on brokers
                 ├── Partition 1
                 └── Partition 2
                         │
Consumers (often in a consumer group) ◄── read with offsets
```

| Term | Meaning |
|------|---------|
| **Broker** | A Kafka server process that stores partitions and serves clients. |
| **Cluster** | Brokers working together under one **metadata** model (historically ZooKeeper; modern Kafka uses **KRaft**). |
| **Topic** | A named feed of messages; **physical** storage is split into **partitions**. |
| **Partition** | An ordered, append-only log; **parallelism** and **retention** are per partition. |
| **Offset** | Monotonic position of a consumer within a partition log. |
| **Producer** | Writes records (key, value, headers, timestamp) to a topic (optionally keyed to a partition). |
| **Consumer** | Reads from assigned partitions, commits offsets (manually or via the consumer client). |

### Why Kafka is distributed

Distribution gives **scale-out throughput**, **fault tolerance**, and **isolation**: more brokers spread partitions and I/O; **replication** survives broker loss within configured limits.

### Partitions and the partition leader

- **Partition**: The unit of **ordering** (per partition), **replication**, and **consumer parallelism**.
- **Leader**: For each partition, one broker is the **leader** for reads/writes; **followers** replicate the log (**ISR** = in-sync replicas).
- **Followers**: Catch up from the leader; if the leader fails, an in-sync follower can be elected (controller-mediated).

**Ordering rule of thumb:** Kafka guarantees **order per partition**, not globally across a whole topic. If you need strict order for one entity (e.g. one `orderId`), use a **stable key** so all events for that entity land in the **same partition**.

---

## Consumer groups

Consumers usually join a **consumer group** with a **group.id**:

- The group **coordinates partition assignment** (each partition is read by **one** consumer in the group at a time).
- **Scale consumers** up to the number of partitions for that topic (adding consumers beyond partition count leaves some idle).
- On membership change, the group **rebalances**: partitions move between consumers (affects latency briefly; tune **partition assignment strategy** and **session/heartbeat** timeouts in production).

**Consumer lag** (difference between log end offset and committed consumer offset) is a primary **SLO** signal: sustained lag means your consumers cannot keep up or are stalled.

---

## Replication, durability, and availability

| Concept | Industry use |
|--------|----------------|
| **Replication factor (RF)** | Typically **3** in production for fault tolerance (tune to risk and cost). |
| **ISR** | Replicas caught up enough to be promoted; **min.insync.replicas** interacts with producer acks for durability. |
| **Unclean leader election** | Allowing non-ISR leaders trades **availability** for **data consistency**—usually avoided in strict pipelines. |

Producer **acks** (`acks=all` with sufficient ISR) and broker/topic defaults define how strong **durability** is versus latency.

---

## Delivery semantics (what “exactly once” really means)

| Semantic | Meaning | Typical cost |
|----------|---------|----------------|
| **At-most-once** | May lose messages; no duplicate risk from retries alone. | Lowest latency; lossy. |
| **At-least-once** | May duplicate on retry; consumers must be **idempotent** or dedupe. | Common default path. |
| **Exactly-once effective (EOS)** | Kafka offers **transactions** and **idempotent producer** for **read-process-write** patterns *within Kafka*; end-to-end EOS with external systems still needs **idempotent sinks** or **deduplication** at the boundary. | Higher complexity; operator and client version alignment. |

Industry standard is to design for **at-least-once** plus **idempotent consumers** unless you have a full EOS story across all sinks.

---

## Retention, compaction, and replay

| Policy | Use case |
|--------|----------|
| **Time/size retention** | Event history for analytics, reprocessing, audit for a bounded window. |
| **Log compaction** | Keep **latest record per key** (changelog style) for **KTables** / materialized state topics. |

**Replay** (re-read from an earlier offset) is a core strength: new services can **backfill** without pressuring producers.

---

## Schema and contracts (often missed early)

Raw bytes in Kafka are flexible but brittle across teams. Production setups commonly add:

- **Schema Registry** (Confluent or compatible) with **Avro**, **Protobuf**, or **JSON Schema**
- **Subject compatibility** rules (backward / forward / full) so producers and consumers evolve safely

This is as important as picking partition counts: it prevents **silent deserialization failures** and uncontrolled breaking changes.

---

## Security (production baseline)

| Layer | Examples |
|-------|-----------|
| **Transport** | TLS between clients and brokers, and broker-to-broker. |
| **Authentication** | SASL (SCRAM, OAuth/OIDC brokers), mutual TLS for clients. |
| **Authorization** | ACLs on topics, groups, cluster operations; often integrated with IAM in managed offerings. |

Treat secrets via **vaults** or cloud secret managers; rotate credentials and certificates on a schedule.

---

## Platform surface: Connect, Streams, clients

| Component | Role |
|-----------|------|
| **Kafka Connect** | Off-the-shelf **source** and **sink** connectors (databases, S3, Elasticsearch, etc.) with fault tolerance and offset management. |
| **Kafka Streams** | Library for **stateful stream processing** on the JVM with changelog/compacted topics. |
| **ksqlDB / Flink / Spark** | Common **SQL or batch/stream** layers *above* Kafka—choose by team skills and latency needs. |
| **Client libraries** | Official Java client is the reference; others (librdkafka-based, Go, Python, .NET) vary in feature parity—verify transactions, EOS, and admin APIs you need. |

---

## Cluster layout: KRaft vs ZooKeeper

Modern Kafka runs metadata in **KRaft** (Kafka Raft) mode instead of **ZooKeeper**. New deployments should assume **KRaft**; ZooKeeper mode is legacy. Operators (Strimzi, Confluent Platform, managed cloud Kafka) abstract much of this but you still set replication, listeners, and storage.

---

## Operations and observability

Signals and practices teams standardize on:

| Signal / practice | Why |
|-------------------|-----|
| **Consumer lag** | Primary “are we keeping up?” metric. |
| **Under-replicated partitions** | Replication not catching up—risk before failure. |
| **Offline log directories / broker I/O** | Disk and filesystem health drive tail latency. |
| **GC / heap (JVM brokers)** | Long pauses affect latency-sensitive workloads. |
| **Topic partition plan** | Too few partitions → throughput ceiling; too many → metadata and file handle overhead. |
| **Quota & ACL audits** | Prevent noisy neighbors and accidental broad access. |

Load-test **producer batching**, **compression** (lz4, zstd), and **linger.ms** vs latency budgets.

---

## Tradeoffs and disadvantages (honest view)

| Challenge | Detail |
|-----------|--------|
| **Cost** | Brokers need **fast, replicated disk** and headroom for peaks; cross-AZ traffic can be expensive. |
| **Operational depth** | Upgrades, balancing, security, and failure drills require skilled owners (lower with **managed Kafka**). |
| **Topic/partition/broker sizing** | Wrong defaults cause hotspots, rebalance pain, or wasted capacity—**capacity planning** is ongoing. |
| **Operational complexity vs simpler queues** | For modest fan-out, **RabbitMQ** or cloud-native queues can be simpler—Kafka wins at **log-scale** ingestion and replay. |

---

## Quick glossary

| Term | One-line |
|------|----------|
| **Controller** | Broker role that manages partition leadership metadata (KRaft quorum in new versions). |
| **Coordinator** | Group coordinator for consumer group protocol. |
| **Sticky partitioning** | Default partitioner behavior for keyed messages. |
| **Dead-letter topic (DLQ)** | Idiomatic pattern: failed messages to a separate topic for inspection and replay—not built-in; you implement with consumers/processors. |

---

## Further reading

- [Kafka documentation](https://kafka.apache.org/documentation/) — configuration, security, operations.
- [Introduction to Kafka](https://kafka.apache.org/intro) — official overview.
- [KRaft](https://kafka.apache.org/documentation/#kraft) — metadata quorum mode.

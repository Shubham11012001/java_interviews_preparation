# Message Queues & Pub/Sub: The Architect's Interview Cheatsheet

*A Medium-style guide: simple enough for a 10-year-old, deep enough for a Technical Architect interview.*

---

## 1. The 10-Year-Old Explanation 🍕

**Message Queue (the pizza shop):**
You order a pizza. The cashier doesn't run to the kitchen and wait. They write your order on a **slip** and stick it on a **rail**. Cooks pick slips one by one. Even if 50 people order at once, nobody is lost; the slips just wait their turn.

- Cashier = **Producer**
- Slip = **Message**
- Rail = **Queue**
- Cook = **Consumer**

**Pub/Sub (the school announcement):**
The principal says on the speaker: *"Tomorrow is a holiday!"* The principal doesn't know who is listening. Students, teachers, and the bus driver all hear it. Each reacts differently.

- Principal = **Publisher**
- Speaker channel = **Topic**
- Listeners = **Subscribers**

> **One-line difference:** Queue = *one message → one worker*. Pub/Sub = *one message → many listeners*.

---

## 2. What Is It, Technically?

**Message Queue:** An asynchronous communication mechanism where a producer sends messages to a broker, which stores them until a consumer processes them. This gives **point-to-point** delivery: each message is consumed by **exactly one** consumer (among competing consumers).

**Pub/Sub:** A messaging pattern where publishers send messages to a **topic** without knowing the receivers, and **every subscriber** of that topic gets a copy. This is **one-to-many** (fan-out).

```
QUEUE (Point-to-Point)
Producer ──► [ m3 | m2 | m1 ] ──► Consumer A   (gets m1)
                    │         └──► Consumer B   (gets m2)
                    └ each message goes to ONE consumer

PUB/SUB (Fan-out)
Publisher ──► [ Topic ] ──┬──► Subscriber A (gets copy)
                          ├──► Subscriber B (gets copy)
                          └──► Subscriber C (gets copy)
```

---

## 3. Why Do We Need It? (The Core Problems It Solves)

| Problem with synchronous (REST) calls | How messaging fixes it |
|---|---|
| **Tight coupling**: A must know B's address and be up | Producer only knows the broker |
| **Cascading failure**: B is down, so A fails | Broker buffers messages until B recovers |
| **Traffic spikes** crash the downstream service | Queue absorbs the spike (**load leveling**) |
| **Slow operations** block the user | Respond instantly, process in background |
| **Scaling**: hard to scale one part alone | Add more consumers independently |
| **Adding a new feature** means changing the producer | Just add a new subscriber, no producer change |

**Three superpowers to remember: Decoupling, Buffering, Scalability.**

---

## 4. When to Use / When NOT to Use

### ✅ Use when
- Work can be done **later** (emails, reports, thumbnails, notifications)
- You need to **absorb traffic bursts** (flash sales, ticket booking)
- **One event, many reactions** (an order placed triggers inventory, billing, email, analytics)
- Services must be **independently deployable and failure-isolated**
- You need **retry** on transient failures
- Integrating **heterogeneous systems** (different languages and teams)
- **Event-driven architecture**, event sourcing, CDC, and streaming pipelines

### ❌ Avoid when
- You need an **immediate response with the result** (e.g., "is this card valid?" at checkout) → use sync REST/gRPC
- **Simple CRUD app**, where a broker adds needless complexity
- You need **strong, immediate consistency** across services
- Your team **can't operate** a broker (ops overhead is real)
- **Strict global ordering** at very high scale is required (hard and expensive)
- Low-latency request/response with tiny payloads (the broker adds hops)

> **Interview line:** "Messaging trades **latency and simplicity** for **resilience and scalability**."

---

## 5. Core Vocabulary (Memorize This Table)

| Term | Meaning |
|---|---|
| **Producer / Publisher** | Sends messages |
| **Consumer / Subscriber** | Receives messages |
| **Broker** | Server that stores and routes messages (Kafka, RabbitMQ) |
| **Queue** | Buffer holding messages for consumers |
| **Topic** | Named channel for pub/sub |
| **Message** | Payload + headers/metadata |
| **Ack (acknowledgement)** | Consumer tells the broker "done, you can delete it" |
| **Nack / Reject** | "I failed, redeliver or dead-letter it" |
| **Consumer Group** | Set of consumers sharing the work of a topic |
| **Partition** | Slice of a topic enabling parallelism and ordering (Kafka) |
| **Offset** | Position of a message in a partition (Kafka) |
| **DLQ** | Dead Letter Queue: parking lot for failed messages |
| **Exchange** | Router in RabbitMQ that decides which queue gets a message |
| **Backpressure** | Slowing producers when consumers can't keep up |
| **Retention** | How long messages are kept |
| **Idempotency** | Processing the same message twice has the same effect as once |

---

## 6. Queue vs Pub/Sub: Side-by-Side

| Aspect | Message Queue | Pub/Sub |
|---|---|---|
| Delivery | One consumer per message | Every subscriber gets a copy |
| Pattern | Point-to-point | One-to-many |
| Use for | **Task distribution**, work offloading | **Event broadcasting**, notifications |
| Consumers | Competing (share work) | Independent (each does own job) |
| Coupling | Producer often knows "a task" is needed | Publisher doesn't care who listens |
| Example | Resize image jobs | "OrderPlaced" event |
| Message after consume | Usually deleted | Retained/copied per subscription |

> **Pro tip:** Real systems combine them. **Each subscriber of a topic often has its own queue** (RabbitMQ fanout exchange → N queues; Kafka consumer groups; AWS SNS → multiple SQS queues).

---

## 7. Delivery Guarantees (Interview Favorite ⭐)

| Guarantee | Meaning | Risk | How |
|---|---|---|---|
| **At-most-once** | Delivered 0 or 1 time | **Message loss** | Fire and forget, auto-ack before processing |
| **At-least-once** | Delivered 1 or more times | **Duplicates** | Ack *after* processing; retry on failure |
| **Exactly-once** | Processed exactly one time | Expensive and complex | Idempotent consumers + transactions/dedupe |

**Architect truth:** True exactly-once delivery over a network is essentially impossible in general. What we really build is **"at-least-once delivery + idempotent processing = effectively exactly-once."** (Kafka offers exactly-once semantics within its own ecosystem using idempotent producers and transactions.)

**Idempotency techniques:**
- Unique **message ID** stored in a processed-IDs table/Redis (dedupe)
- **Upserts** instead of inserts
- Conditional updates (`WHERE version = ?`)
- Natural idempotent operations ("set status = PAID", not "add 100 to balance")

---

## 8. Ordering

- **Single queue + single consumer** → order preserved
- **Multiple competing consumers** → order **lost** (parallelism vs ordering trade-off)
- **Kafka:** order guaranteed **only within a partition**. Use a **partition key** (e.g., `orderId`/`userId`) so related messages land in the same partition.
- **SQS FIFO:** uses **MessageGroupId** for ordering per group.
- **Retries can reorder messages**, so design consumers to tolerate it.

> **Interview line:** "Global ordering doesn't scale; **per-key ordering** is the practical solution."

---

## 9. Popular Technologies & When to Choose

| Tool | Type | Best for | Notes |
|---|---|---|---|
| **Apache Kafka** | Distributed log / streaming | High throughput, event streaming, replay, analytics, event sourcing | Messages retained, consumers pull, partitions, consumer groups |
| **RabbitMQ** | Traditional broker (AMQP) | Complex routing, task queues, low-latency work queues | Exchanges (direct/topic/fanout/headers), push-based, rich features |
| **AWS SQS** | Managed queue | Simple, serverless, zero ops queues | Standard (at-least-once, best-effort order) and FIFO |
| **AWS SNS** | Managed pub/sub | Fan-out notifications | Often **SNS → SQS** fan-out |
| **Google Pub/Sub** | Managed pub/sub | GCP-native event ingestion | Global, at-least-once, ordering keys |
| **Azure Service Bus** | Managed broker | Enterprise messaging, topics and queues, sessions | Transactions, DLQ built-in |
| **ActiveMQ / Artemis** | Traditional JMS broker | Java/JMS legacy ecosystems | JMS-compliant |
| **Redis Pub/Sub / Streams** | In-memory | Lightweight, ephemeral messaging | Pub/Sub is fire-and-forget (no persistence); Streams add persistence |
| **Apache Pulsar** | Distributed messaging | Multi-tenant, geo-replication, unified queue and stream | Separates compute and storage |
| **NATS** | Lightweight messaging | Ultra-low-latency microservice messaging | JetStream adds persistence |

### Kafka vs RabbitMQ (asked constantly)

| | **Kafka** | **RabbitMQ** |
|---|---|---|
| Model | Distributed commit log | Smart broker, message routing |
| Consumption | Consumers **pull**, track offsets | Broker **pushes** to consumers |
| After consume | Messages **retained** (replay possible) | Messages **removed** after ack |
| Throughput | Very high (millions msg/sec) | High (tens of thousands/sec) |
| Ordering | Per partition | Per queue |
| Routing | Simple (topic/partition) | Rich (exchanges, routing keys) |
| Best for | Streaming, event sourcing, big data pipelines | Task queues, complex routing, request/reply |
| Scaling | Add partitions and brokers | Clustering/federation, harder to scale massively |

> **Rule of thumb:** *Need replay, huge scale, event streaming → Kafka. Need flexible routing and classic job queues → RabbitMQ. Want zero ops on AWS → SQS/SNS.*

---

## 10. Key Patterns Every Architect Should Know

### 10.1 Competing Consumers (Work Queue)
Many workers read from one queue → work is shared → easy horizontal scaling.

### 10.2 Fan-out (Pub/Sub)
One event, many independent reactions.
`OrderPlaced → [Inventory, Billing, Email, Analytics]`

### 10.3 Dead Letter Queue (DLQ)
A message fails N times → move it to the DLQ instead of blocking the queue forever (**poison message** problem). Then alert, inspect, fix, and **redrive**.

### 10.4 Retry with Exponential Backoff + Jitter
Retry after 1s, 2s, 4s, 8s... plus random jitter to avoid a **thundering herd**. Often implemented with **retry topics/queues** with delays.

### 10.5 Outbox Pattern ⭐ (Very common in interviews)
**Problem (dual-write):** You update the DB **and** publish a message. If one succeeds and the other fails, the system is inconsistent.
**Solution:** In **one DB transaction**, write business data **and** an `outbox` row. A separate relay/poller (or **CDC with Debezium**) publishes outbox rows to the broker.
Result: at-least-once publishing, no lost events.

### 10.6 Inbox / Idempotent Consumer
Store processed message IDs so duplicates are ignored.

### 10.7 Saga Pattern
Distributed transaction across services using events. Each step has a **compensating action**.
- **Choreography:** services react to each other's events (decentralized)
- **Orchestration:** a central coordinator tells services what to do

### 10.8 Event Sourcing & CQRS
Store state as a sequence of events (log). CQRS separates write and read models, often synced through events.

### 10.9 Request-Reply over Messaging
Use a **correlation ID** and a **reply-to queue** to get async responses.

### 10.10 Claim Check
Messages should be small. Store the large payload (file/image) in S3/blob storage and put only a **reference/URL** in the message.

### 10.11 Priority Queue
Urgent messages are processed first (RabbitMQ priority queues, or separate queues per priority).

### 10.12 Delayed / Scheduled Messages
Deliver a message after X time (SQS delay queues, RabbitMQ delayed plugin/TTL+DLX).

### 10.13 Competing Consumers + Partition Key (Kafka)
Same key → same partition → ordering per entity, with parallelism across entities.

---

## 11. Production Scenarios (Tell These as Stories)

### 🛒 Scenario 1: E-commerce Order Flow
User places an order → `OrderPlaced` event published → **Inventory** reserves stock, **Payment** charges, **Notification** sends email, **Analytics** logs, **Shipping** plans delivery. Adding a **Recommendation** service later? Just subscribe. **Zero change to the order service.**

### 🎟️ Scenario 2: Flash Sale / Ticket Booking
100K requests per second hit during a sale. The API **enqueues** requests and returns "processing". Workers process at a sustainable rate. The DB survives. *(Queue = shock absorber.)*

### 📧 Scenario 3: Email/SMS Notifications
The signup API shouldn't wait for the SMTP server. Publish `UserRegistered` → email worker sends in background; if the provider is down, messages wait and retry.

### 🖼️ Scenario 4: Media Processing
Upload a video → message → worker pool transcodes. Scale workers based on **queue depth**.

### 💳 Scenario 5: Payments & Financial Systems *(relevant to your credit risk/finance analytics domain)*
Transaction events → fraud detection, ledger, risk scoring, audit. **Idempotency is critical** to avoid double-charging. Ordering per account via a partition key. Audit via retained log (Kafka).

### 📊 Scenario 6: Analytics & Data Pipelines
Application events → Kafka → stream processing (Flink/Spark/Kafka Streams) → data lake/warehouse, dashboards, ML features. Ideal for **risk-scoring pipelines**: ingest financial data → enrich → score → publish results.

### 🔄 Scenario 7: Microservice Data Sync / CDC
DB change → Debezium → Kafka → search index (Elasticsearch), cache invalidation, read models.

### 📡 Scenario 8: IoT / Telemetry
Millions of devices publish readings; consumers aggregate, alert, store.

### 🧾 Scenario 9: Audit & Compliance Logging
Every action is published to an immutable log; retention gives a replayable audit trail.

---

## 12. Best Cases ✅ vs Worst Cases ❌

### Best cases
- Spiky workloads with steady downstream capacity
- Fire-and-forget background jobs
- Many independent consumers of the same event
- Services owned by different teams that should evolve independently
- Need for replay (Kafka) after bugs or new consumers
- Unreliable downstream dependencies (broker buffers)

### Worst cases / Pitfalls
| Pitfall | Why it hurts | Mitigation |
|---|---|---|
| **Poison message** | Endless retry loop blocks processing | DLQ + max retries |
| **Duplicate delivery** | Double charge, double email | Idempotent consumers |
| **Message loss** | Auto-ack then crash | Manual ack after processing; durable queues; replication |
| **Out-of-order events** | Stale update overwrites new | Partition keys, version numbers/timestamps |
| **Consumer lag grows** | Real-time becomes "yesterday-time" | Autoscale, tune batch size, add partitions, monitor lag |
| **Hot partition** | One key gets all traffic | Better key design, salting |
| **Dual-write inconsistency** | DB and broker disagree | Outbox pattern/CDC |
| **Schema breaking changes** | Consumers crash | Schema Registry (Avro/Protobuf), backward compatibility |
| **Unbounded queue growth** | Memory/disk exhausted | Retention, TTL, backpressure, alerts |
| **Rebalancing storms (Kafka)** | Consumers pause repeatedly | Tune session timeouts, static membership, cooperative rebalancing |
| **Using a queue as a database** | Wrong tool | Use a proper datastore |
| **Debugging difficulty** | Async flow is hard to trace | Correlation IDs, distributed tracing (OpenTelemetry) |
| **Eventual consistency surprises** | Users see stale data | Design UX/APIs for it |

---

## 13. Reliability & Production Checklist ✔️

**Producer side**
- Enable **acks=all** (Kafka) and **idempotent producer** (`enable.idempotence=true`)
- Retries with backoff
- Use the **outbox pattern** for DB + event consistency

**Broker side**
- **Replication factor ≥ 3**, `min.insync.replicas=2` (Kafka)
- Durable queues and persistent messages (RabbitMQ), mirrored/quorum queues
- Multi-AZ deployment
- Retention policy and disk monitoring

**Consumer side**
- **Manual ack after successful processing**
- **Idempotent** handler
- **DLQ** + bounded retries with backoff
- Handle **graceful shutdown** and **rebalancing**
- Use **prefetch/batch limits** (RabbitMQ `prefetch_count`, Kafka `max.poll.records`)

**Observability**
- Monitor **queue depth / consumer lag** (the #1 metric)
- Message age, DLQ size, error rate, throughput, processing latency
- Distributed tracing with correlation IDs
- Alerts on lag and DLQ growth

**Governance**
- **Schema Registry** and versioning rules
- Naming conventions for topics/queues
- Security: TLS, authN/authZ (ACLs), encryption at rest, no PII in plain messages

---

## 14. Scaling Strategies

- **Scale consumers horizontally** (Kafka: limited by number of partitions, so **max parallelism = partition count** per consumer group)
- **Scale on queue depth** (KEDA/autoscaling)
- **Increase partitions** (note: this changes key-to-partition mapping)
- **Batching** messages for throughput
- **Compression** (snappy/lz4/zstd)
- **Separate queues per priority/workload** to avoid noisy neighbors
- **Sharding** across clusters for extreme scale

---

## 15. Kafka Quick Deep-Dive (Architects Get Asked This)

- **Topic** → split into **partitions** → each partition is an **append-only log** with **offsets**
- **Producer** picks the partition (by key hash or round-robin)
- **Consumer Group**: each partition is consumed by **only one consumer in the group**; different groups each get **all** messages (that's pub/sub + queue in one design!)
- **Offsets** are committed by consumers; commit **after** processing for at-least-once
- **Replication:** leader and followers; **ISR** (in-sync replicas)
- **Retention:** by time/size, or **log compaction** (keep the latest value per key)
- **Rebalance:** partition reassignment when consumers join or leave
- **Zookeeper vs KRaft:** modern Kafka uses **KRaft** (no ZooKeeper)
- **Delivery semantics:** controlled via `acks`, idempotence, transactions, and offset commit strategy

> **Magic sentence:** *"A Kafka consumer group behaves like a queue within the group and like pub/sub across groups."*

---

## 16. RabbitMQ Quick Deep-Dive

**Flow:** Producer → **Exchange** → (binding/routing key) → **Queue** → Consumer

| Exchange type | Behavior |
|---|---|
| **Direct** | Routes to the queue whose binding key **exactly matches** the routing key |
| **Fanout** | Broadcasts to **all** bound queues (pub/sub) |
| **Topic** | Pattern matching (`order.*.created`, `#`) |
| **Headers** | Routes by message headers |

- **Ack/Nack**, **prefetch**, **TTL**, **DLX (dead-letter exchange)**, **priority queues**
- **Quorum queues** for replicated, safe queues (preferred over classic mirrored queues)

---

## 17. Cloud-Native Notes

- **SQS Standard:** nearly unlimited throughput, at-least-once, best-effort ordering
- **SQS FIFO:** ordering and dedupe, lower throughput
- **Visibility timeout:** the message is hidden while a consumer processes it; if not deleted in time, it reappears (a common cause of duplicates)
- **SNS → SQS fan-out:** the classic robust pub/sub pattern on AWS
- **Google Pub/Sub:** ack deadline, ordering keys, dead-letter topics
- **Azure Service Bus:** sessions (ordering), duplicate detection, scheduled messages

---

## 18. Message Design Best Practices

- Keep messages **small and self-descriptive** (use the claim-check pattern for big data)
- Include: **messageId, correlationId, timestamp, eventType, version, payload**
- **Event vs Command:**
  - **Event** = something that **happened** (`OrderPlaced`), past tense, many consumers
  - **Command** = **do this** (`SendEmail`), a single target
- Version your schemas; make changes **backward compatible** (add optional fields, don't rename or remove)
- Use **Avro/Protobuf** over raw JSON for large-scale systems

---

## 19. Security Considerations

- **TLS** in transit; **encryption at rest**
- **AuthN/AuthZ** per topic/queue (ACLs, IAM)
- **Don't put sensitive/PII data** unencrypted in messages
- **Audit** who publishes and consumes
- Protect against **message injection** and validate payloads

---

## 20. Messaging vs Alternatives

| Need | Better choice |
|---|---|
| Immediate response required | REST / gRPC |
| Simple background job in one app | In-process thread pool / scheduler |
| Durable workflow with state, timers, retries | **Workflow engines** (Temporal, Step Functions, Camunda) |
| Large-scale stream analytics | Kafka + Flink/Spark |
| Real-time push to browsers | WebSockets / SSE (often fed by pub/sub behind the scenes) |

---

## 21. Common Interview Questions + Crisp Answers 🎯

**Q1. Queue vs Topic?**
Queue: each message goes to one consumer (work distribution). Topic: each message goes to all subscribers (broadcast).

**Q2. How do you guarantee no message loss?**
Producer `acks=all` + retries, replicated and durable broker, consumer acks **after** processing, and DLQ for failures.

**Q3. How do you handle duplicates?**
Assume at-least-once delivery. Make consumers **idempotent** using message IDs, upserts, or dedupe stores.

**Q4. How do you preserve ordering?**
Use a **partition key** (Kafka) or **message group ID** (SQS FIFO) so related messages go to the same partition, handled by a single consumer sequentially.

**Q5. What if a consumer keeps failing on one message?**
Bounded retries with exponential backoff, then move it to the **DLQ**, alert, and fix or redrive.

**Q6. How do you keep DB and the broker consistent?**
**Transactional Outbox** or CDC (Debezium). Avoid dual writes.

**Q7. What is consumer lag and why care?**
The gap between the latest message and the consumer's position. It's the primary health metric; growing lag means consumers can't keep up.

**Q8. How would you scale consumers?**
Add consumers (up to the partition count in Kafka), autoscale on lag/queue depth, batch, tune concurrency.

**Q9. Kafka vs RabbitMQ?**
Kafka: durable replayable log, high throughput, streaming. RabbitMQ: smart routing, classic task queues, push-based. (See table in §9.)

**Q10. Is exactly-once possible?**
Not truly end to end in general. Achieve it practically with at-least-once plus idempotency; Kafka provides exactly-once semantics within Kafka via transactions.

**Q11. Sync vs Async: how do you choose?**
Sync when the caller needs the result now. Async when work can be deferred, spiky, or involves many downstream reactions.

**Q12. How do you handle backpressure?**
Bounded queues, consumer prefetch limits, rate limiting producers, autoscaling consumers, and dropping/shedding low-priority load if needed.

**Q13. How do you evolve message schemas safely?**
Schema Registry, backward/forward compatibility, additive changes, versioned event types.

**Q14. How do you monitor a messaging system?**
Lag, queue depth, message age, DLQ count, throughput, error rates, broker health (disk, ISR, under-replicated partitions), and tracing.

**Q15. Push vs Pull consumers?**
Push (RabbitMQ): low latency, but the broker can overwhelm consumers (use prefetch). Pull (Kafka): consumer controls its pace, better for batching and backpressure.

**Q16. How do you design a notification system?**
API → publish event → topic → separate subscribers per channel (email/SMS/push), retries and DLQ per channel, idempotency, rate limits, user preferences, and templating.

**Q17. How would you reprocess data after a bug?**
Kafka: reset the consumer group offset (replay from retention). Queues without retention: DLQ redrive or rebuild from the source of truth.

**Q18. What's the difference between a message broker and an event streaming platform?**
Broker: transient messages, deleted after consumption, smart routing. Streaming platform: durable ordered log, replayable, multiple independent readers, stream processing.

---

## 22. Architecture Decision Framework 🧭

Ask these questions in the interview before choosing:

1. **Throughput?** (msgs/sec; Kafka for huge volumes)
2. **Latency needs?** (RabbitMQ/NATS for low latency)
3. **Ordering required?** (per key vs global)
4. **Delivery guarantee?** (loss tolerable? duplicates tolerable?)
5. **Replay/retention needed?** (Kafka)
6. **Routing complexity?** (RabbitMQ)
7. **Number of consumers / fan-out?**
8. **Ops capacity?** (managed service vs self-hosted)
9. **Cloud lock-in tolerance?**
10. **Message size and schema evolution?**

---

## 23. 60-Second Elevator Pitch (Memorize) 🎤

> "A message queue decouples producers from consumers by buffering messages in a broker, enabling asynchronous processing, load leveling, and independent scaling. Pub/Sub extends this to fan-out: one event, many independent subscribers. I choose Kafka for high-throughput, replayable event streaming, RabbitMQ for flexible routing and task queues, and SQS/SNS when I want zero ops on AWS. In production I design for **at-least-once delivery with idempotent consumers**, use the **outbox pattern** to avoid dual-write inconsistencies, route failures to a **DLQ** with backoff retries, preserve ordering using **partition keys**, and monitor **consumer lag** as the key health signal."

---

## 24. Last-Minute Cheat Card 📝

```
QUEUE      = 1 msg → 1 consumer     (work sharing)
PUB/SUB    = 1 msg → N subscribers  (broadcast)

3 Superpowers : Decouple • Buffer • Scale
Delivery      : at-most-once | at-least-once | exactly-once(=ALO + idempotency)
Ordering      : per-key via partition key (not global)
Failures      : retry+backoff+jitter → DLQ
DB + Broker   : Outbox / CDC (never dual-write)
Distributed Tx: Saga (choreography / orchestration)
Big payloads  : Claim Check (store in S3, send reference)
Key metric    : Consumer Lag
Kafka         : log • partitions • offsets • consumer groups • replay
RabbitMQ      : exchanges • routing • ack • prefetch • DLX
AWS           : SQS (queue) + SNS (pub/sub) → fan-out
Don't use if  : need instant result / simple CRUD / can't run a broker
```

---

### 🧒 Final 10-Year-Old Recap

- A **queue** is a line at the pizza shop. Everyone waits fairly, and each order is made by one cook.
- **Pub/Sub** is a school announcement. Everybody who cares hears it.
- If a cook is sick, the orders **wait on the rail** instead of disappearing.
- If an order is **impossible to make**, it goes to a special **"problem pile"** (DLQ) so the line keeps moving.

Good luck with your interviews, Shubham! 🚀 You've got the architect's mindset, so lead with **trade-offs**. That's what interviewers love most.
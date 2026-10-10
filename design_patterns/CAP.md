# CAP Theorem and Basic System Design: The Complete Interview Guide

A beginner-friendly, architect-level tutorial for Java Backend Engineers and Technical Architects

System design interviews can feel difficult because they don't have one correct answer. An interviewer might ask you to design a payment service, a URL shortener, a notification system, or a globally distributed application. Your job is to explain how the system works, why you chose a particular architecture, what can go wrong, and how you would handle it.

Two concepts form the foundation of these discussions:

1. CAP Theorem — how distributed systems behave when network communication fails.

2. Basic System Design — how to build systems that are scalable, reliable, secure, maintainable, and performant.

Let's understand both from the ground up, then connect them to real production scenarios and interview questions.

# Part 1: CAP Theorem

## 1. What is CAP Theorem?

The CAP Theorem states that a distributed data system cannot simultaneously guarantee all three of the following properties during a network partition:

* C — Consistency

* A — Availability

* P — Partition Tolerance

The most important interview phrase is:

When a network partition occurs, a distributed system must choose between consistency and availability.

Let's understand why.

## 2. First, what is a distributed system?

Imagine you operate an online shopping application.

Instead of running your entire application on one computer, you run it on multiple servers in different locations.

![7. Space-Based Architecture - Software Architecture Patterns, 2nd Edition \[Book\]](https://images.openai.com/static-rsc-4/5Ny_cK65Sh8CJkTds5Q_11856ZyXzRRNKH1GND619yyMWjr7YnqGqXqr7rCqzn8E1KMNOf0AXIdqOcFznXxcluIq3c1Z3wtPR-x_ZsCY-ibEii3_jxiv9wmY3cwR3VBYBE4QiAjJJA2khXPujEeZrOXK0JkLo8E4Y85CZ3sPiqc?purpose=inline)

Add to Favorites

[oreilly](https://www.oreilly.com/library/view/software-architecture-patterns/9781098134280/ch07.html)

For example:

* Server A is in Mumbai.

* Server B is in Singapore.

* Server C is in Frankfurt.

All three servers work together to serve customers.

Why distribute the system?

* To handle more users.

* To survive individual server failures.

* To serve users closer to their location.

* To store more data than one machine can comfortably handle.

But here's the problem: the network connecting these machines can fail.

A server may be healthy while the connection between it and another server is broken.

That is where CAP becomes important.

## 3. Understand the three properties

### C — Consistency

Consistency means that every successful read sees the latest committed write, according to the system's consistency guarantee.

Imagine your bank account has ₹10,000.

You transfer ₹2,000 to another account.

Your new balance becomes ₹8,000.

If you read your balance from another node after the transfer has successfully committed, a strongly consistent system must return ₹8,000, not ₹10,000.

Strong consistency

All successful reads reflect the latest committed write, subject to the system's stated consistency model.

Eventual consistency

Replicas may temporarily return different values but converge if updates stop and replication progresses.

Important: Consistency in CAP is not the same as consistency in ACID transactions or the consistency rules of a database schema.

In CAP, consistency primarily refers to a single-copy, strongly consistent view of the data.

### A — Availability

Availability means every request received by a non-failed node gets a non-error response, even if that response may not contain the latest data.

Imagine you open a shopping application during a network outage.

* An available system continues responding.

* A system prioritizing consistency might reject or delay some requests because it cannot safely guarantee the latest data.

A crucial distinction: availability does not simply mean that the server is running. The request must receive a successful, non-error response within the formal availability model.

Returning stale data can count as available. Returning an error because the system refuses to serve potentially stale data does not.

### P — Partition Tolerance

A network partition occurs when a communication failure prevents parts of a distributed system from communicating reliably with each other.

Imagine this setup:

Mumbai

Database Node A

Reachable

Singapore

Database Node B

Reachable

Network link is broken

Node A cannot communicate with Node B

Both machines may be healthy, but they cannot exchange updates.

A real distributed system cannot simply assume the network will never fail. It must tolerate partitions by making a decision about how to respond.

That's P.

Architect's insight: Partition tolerance is not usually a feature you switch off. In a distributed system, network partitions are a reality. CAP describes the trade-off that appears when they occur.

## 4. Why can't we have C, A, and P simultaneously?

Let's use a bank account example.

Two database replicas store the same account balance: ₹10,000.

A network partition separates the replicas.

* Mumbai can communicate with its local clients.

* Singapore can communicate with its local clients.

* Mumbai and Singapore cannot communicate with each other.

Now two customers attempt to withdraw money.

* Customer A withdraws ₹8,000 in Mumbai.

* Customer B withdraws ₹8,000 in Singapore.

Both replicas initially believe the balance is ₹10,000.

If both accept the withdrawal independently, the system can approve ₹16,000 in total against a ₹10,000 balance.

That's a consistency problem.

The system has two main choices.

## Choice 1: Preserve consistency

The system refuses or blocks a withdrawal that it cannot safely validate because the other replica is unreachable.

Protects the balance from conflicting updates.

Some requests fail or wait.

## Choice 2: Preserve availability

Each replica continues accepting withdrawals locally.

Requests continue receiving responses.

Conflicting withdrawals may be accepted.

If both sides must continue serving requests independently, and they cannot communicate, they cannot always guarantee the same latest value.

That's the heart of CAP.

### The most common interview mistake

Saying:

> "We can choose any two of C, A, and P."

This is an oversimplification.

The more accurate statement is:

> "During a network partition, a distributed system must choose between strong consistency and availability. It must tolerate the partition if it is to continue operating as a distributed system under that failure."

When there is no partition, a system can often provide both strong consistency and availability. CAP does not say that every request always sacrifices one property.

## 5. CP vs AP systems

These are the two important choices during a partition.

### CP — Consistency + Partition Tolerance

A CP system prioritizes consistent results over serving every request during a partition.

If a node cannot safely determine the latest committed value, it may reject or delay the request.

Example: A distributed database that requires quorum or leader coordination before committing a write.

Use CP-style behavior when incorrect data is more dangerous than a temporary failure.

Production scenarios:

* Bank transfers.

* Inventory reservations where overselling must be prevented.

* Distributed locks and leader election.

* Unique username or account creation.

* Financial ledger updates.

Best case: Conflicting updates are prevented and data correctness is protected.

Worst case: A network failure or loss of quorum can make part of the service unavailable.

### AP — Availability + Partition Tolerance

An AP system continues serving requests from reachable parts of the system, even when it cannot immediately synchronize all replicas.

This may mean accepting updates that conflict with updates made elsewhere. The system must then resolve those conflicts or merge the updates later.

Production scenarios:

* Shopping-cart updates.

* Social-media reactions.

* Product-view counters.

* Some user preferences.

* Distributed systems that can tolerate temporary stale reads.

Best case: Users can continue using the application despite a network partition.

Worst case: Users may see stale data, conflicting changes, or temporary inconsistencies.

An AP design does not automatically mean the data will be correct without conflict resolution.

### Comparison table

| Property          | CP                                     | AP                                     |
| ----------------- | -------------------------------------- | -------------------------------------- |
| During partition  | Prioritizes consistency                | Prioritizes availability               |
| Request behavior  | May reject or block requests           | Continues responding where possible    |
| Data freshness    | Protects strong consistency guarantees | May return stale data                  |
| Conflict handling | Often prevents conflicting writes      | May need reconciliation                |
| Typical use       | Financial correctness                  | User experience and independent writes |
| Main trade-off    | Reduced availability                   | Temporary inconsistency                |

### What about CA?

CA means consistency and availability without tolerating a network partition.

A single-node database can provide consistency and availability under normal operating assumptions, but it does not remain fully available if the only node fails.

In practical distributed systems, CA cannot be guaranteed during a partition. That's why CA is not a third equivalent choice for handling network partitions.

## 6. A simple way to remember CAP

Imagine you are designing a distributed banking application.

During a network partition, ask yourself:

* C: Must every successful read show the latest committed data?

* A: Must every request receive a successful response, even when replicas cannot communicate?

* P: Can the system continue operating despite communication failures?

Your business requirements determine the behavior you prefer.

For example, if a payment cannot be verified safely, declining temporarily is often better than accepting an invalid payment.

If a user adds an item to a shopping cart, letting them continue shopping and synchronizing changes later may be acceptable.

Interview answer: CAP is not about picking a fashionable database. It is about deciding which guarantees your business can sacrifice when the network fails.

## 7. How does CAP relate to real databases?

Don't classify an entire database product as simply CP or AP without considering its configuration, operations, and consistency settings.

Different operations and deployment modes can provide different guarantees.

| Technology           | Useful interview understanding                                                                                                                                 |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| PostgreSQL           | Can provide strong transactional consistency on a primary; synchronous replication improves durability and consistency guarantees but can affect availability. |
| MySQL                | Primary-replica replication may be asynchronous; stale reads and failover behavior depend on configuration.                                                    |
| Apache Cassandra     | Often used for highly available distributed workloads; consistency levels and replication settings affect guarantees.                                          |
| MongoDB              | Replica sets use a primary and configurable read/write concerns; elections can interrupt writes.                                                               |
| Redis                | Replication and failover behavior depend on the deployment; acknowledged writes can still be lost during certain failovers.                                    |
| Google Cloud Spanner | Provides externally consistent distributed transactions, with availability and latency depending on quorum, placement, and failures.                           |
| DynamoDB             | Offers different consistency choices for supported operations; global-table replication behavior depends on the selected mode.                                 |
| etcd                 | Uses Raft consensus and requires a quorum for committed writes, favoring consistency over write availability without quorum.                                   |

The key is to understand the guarantees rather than memorize a label.

For example, a database with a primary and replicas might serve stale reads from replicas while sending writes to the primary. The behavior for a read is not necessarily the same as the behavior for a write.

## 8. What is quorum, and how does it relate to CAP?

Quorum is a minimum number of participating nodes required to approve an operation.

Suppose a system has three replicas.

A majority quorum is:

Q=⌊N2⌋+1Q=\left\lfloor\frac{N}{2}\right\rfloor+1Q=⌊2N⌋+1

For N=3N=3N=3:

Q=2Q=2Q=2

At least two nodes must participate in a majority-based decision.

### Three-node example

Node A

Available

Node B

Available

Node C

Unreachable

Two of three nodes can still form a majority quorum, so the cluster may continue committing operations.

If only one node remains reachable, it cannot form a majority quorum. A consensus-based system will generally stop accepting new committed writes rather than risk conflicting histories.

### Important quorum caveat

Not every replicated database uses the same quorum algorithm. Some systems use configurable read and write consistency levels; others use leader-based consensus.

For systems that use quorum reads and writes, a common relationship is:

R+W&gt;NR+W>NR+W&gt;N

Where:

* RRR = read quorum

* WWW = write quorum

* NNN = number of replicas

This overlap can help a read intersect with a write's replicas. However, it does not automatically guarantee linearizability. The actual guarantee also depends on the replication protocol, versions, concurrent writes, and read/write semantics.

Architect tip: A quorum is not a magic number that makes any database strongly consistent.

## 9. What is the PACELC theorem?

CAP explains the trade-off during a network partition. But what happens when the network is healthy?

PACELC extends the discussion.

* P: If there is a partition,

* A or C: choose availability or consistency.

* E: Else, when there is no partition,

* L or C: choose lower latency or stronger consistency.

In plain English:

> If the network breaks, do you favor availability or consistency? If the network is healthy, do you favor lower latency or stronger consistency?

For example, a geographically distributed database may need communication between regions to guarantee a strongly consistent read. That communication can increase latency.

A system that serves a nearby replica may respond faster but return stale data.

| Situation         | Choice             | Consequence                                     |
| ----------------- | ------------------ | ----------------------------------------------- |
| Network partition | Availability       | May serve stale or conflicting data             |
| Network partition | Consistency        | May reject or delay requests                    |
| Healthy network   | Low latency        | May use local replicas and weaker consistency   |
| Healthy network   | Strong consistency | May coordinate across nodes, increasing latency |

PACELC is especially useful when designing globally distributed applications.

# Part 2: Basic System Design

## 10. What is system design?

System design is the process of deciding how software components work together to satisfy business and technical requirements.

Think of designing a house.

Before construction, you decide:

* How many rooms are needed?

* How many people will live there?

* Where will the doors and windows go?

* How will water and electricity reach each room?

* What happens during a power outage?

* How much money can you spend?

Software system design asks similar questions.

* How many users will use the application?

* How much data will it store?

* Which services are required?

* How will services communicate?

* How will the system scale?

* What happens when a database or service fails?

* How will we secure and monitor it?

A good design is not the most complicated architecture. It is the simplest architecture that meets the requirements and can evolve as those requirements change.

## 11. Functional vs non-functional requirements

This is one of the first questions you should address in a system design interview.

### Functional requirements: What should the system do?

For a notification service:

* Accept a notification request.

* Send an email or SMS.

* Track delivery status.

* Retry failed deliveries.

* Prevent duplicate notifications where required.

### Non-functional requirements: How well should it do it?

* Support 100,000 notifications per hour.

* Keep API latency below an agreed threshold.

* Survive worker failures.

* Secure customer information.

* Recover after an outage.

* Provide useful logs and metrics.

| Functional         | Non-functional                                        |
| ------------------ | ----------------------------------------------------- |
| Create an order    | Handle 5,000 order requests per second                |
| Upload a file      | Support large files reliably                          |
| Search products    | Return results within 200 ms at the target percentile |
| Transfer money     | Prevent double spending                               |
| Send notifications | Retry failures without uncontrolled duplication       |

Numbers in an interview should come from stated requirements or clearly identified assumptions—not guesses presented as facts.

## 12. The main qualities of a good system

![How to Build High Throughput Systems](https://images.openai.com/static-rsc-4/cW-2FlSfj1YknwdtFv_nNQMMOZAsDaHMeJmJN7nz7wk4xkD1Mvw-VYiAHvWq16r4yEriqrrX33n3c_xI-VavULugUJ399eViKJXjX-Rpjsbv37BrCpD6oORz0H3mDTe0RCFa2Tj6lBK3aZmnP4gOIahmD8tR5hI17sA-YT9mbeY?purpose=inline)

Add to Favorites

Scalability

The system can handle growth in users, traffic, or data.

Example: add more application instances when request volume increases.

![Practical guide to zero-downtime deployments and high availability | by Bohdan Pastukh | Medium](https://images.openai.com/static-rsc-4/_z0rQeLi4b_pRcU_8YD8vtj4reeJupxIOS7X8U6H86nO-sud1bWU79heEyGRYDBRC-upLAU_NdOA9JyzlVVfe9Qt-V5GzOJG8jj60QHkrvqCx4WuaI5p_Tuptl2ToEeCXT1C-B9jh2O2-0DLk85_UE1DdBtYEhb4fjFfxWMgcMQ?purpose=inline)

Add to Favorites

Reliability

The system performs correctly and consistently over time.

Example: recover from a failed worker without losing accepted jobs.

![Open Source Cloud-Native Application Protection Platform](https://images.openai.com/static-rsc-4/6GpSPszLsIfFrItwzZmFnydt2_GhzMF39mtTuO6hunWXSpHW9efYPgiM1P2Q2dGuYl0CuKTsr0xsBteUtUVE7oYJtLMCjSg0qhgZmkylo0ykiZ9BSkTM1a3vuGMGMtDzhNXTr9RVTTiqITQthOi9CpZN9wtOwqWuEPrO0MO7Qhg?purpose=inline)

Add to Favorites

Security

The system protects data and restricts unauthorized access.

Example: use OAuth-based authentication, authorization policies, and encryption.

![Phone Speed Test Illustrations - Free Download in SVG, PNG](https://images.openai.com/static-rsc-4/M51PzAADmDsMwH0Ptfwyn6MCJxZqyNAK1wHn990Pqok5rip5-fGBzMrRb6Y9-6gjEaHa-0x4Dos4B0jg4pMHtDSBpeEtTQjWN4ttzUfi5lhV50JblZJRD72UNoQCWPSr669qPXQGdVjfhunrm8yh57rV4SUCB3CowWv7CsEKlaM?purpose=inline)

Add to Favorites

Performance

The system responds quickly and processes workloads efficiently.

Example: use database indexes and caching to reduce response times.

![Prometheus Alert Manager Setup and Alert Configurations (Slack) | by Krishabh | Medium](https://images.openai.com/static-rsc-4/tonDeldEPcQlNi0KXod5sjCnWv3T6mtcgv1Z4rU2ZIQTMJhYVxIJR1bedCmWlJXh6tHdqKF9lmnF6DHa9De3ZpBBMdUuXrM1sdbH1Jca3A13v2Eiu149Xh8nEkeiwtz1sJwWfwUOIrVfL1BYYINOp8wEIqKsaVm_NPW9heqKS7I?purpose=inline)

Add to Favorites

Observability

Engineers can understand the system's internal state using logs, metrics, and traces.

These qualities can conflict. For example, stronger consistency may increase latency, while extra replicas can improve availability but add operational complexity.

The architect's job is to explain and manage these trade-offs.

## 13. A standard system design architecture

Let's design a simple e-commerce application.

Client

Web browser / Mobile app

Load Balancer / API Gateway

Traffic routing, TLS, authentication policies, rate limits

Application Layer

Order Service

Inventory

Payment

Database

Orders, inventory, payments

Cache

Frequently read data

Message Broker + Background Workers

Notifications, analytics, and asynchronous tasks

This is a conceptual architecture, not a requirement to use every component.

For a small application, a single Spring Boot service and one relational database may be sufficient. You introduce additional components when scale, reliability, isolation, or business requirements justify them.

## 14. Monolithic vs microservices architecture

This is one of the most frequently asked system design questions.

### Monolithic architecture

A monolith is an application deployed as one unit. Its modules may still be well separated internally.

Example:

* User module

* Order module

* Payment module

* Notification module

All live in one Spring Boot application.

Advantages

* Simple deployment and local development.

* Straightforward in-process communication.

* Easier transactions across modules when they share a database.

* Lower infrastructure and operational overhead.

Disadvantages

* Scaling one module independently is difficult.

* A shared runtime can make failures affect multiple features.

* Large codebases can become difficult to maintain if boundaries are poor.

* Releases may require deploying the whole application.

### Microservices architecture

A microservices system splits functionality into independently deployable services with defined boundaries.

Example:

* Order Service

* Payment Service

* Inventory Service

* Notification Service

Advantages

* Independent deployment and scaling.

* Clear ownership of business capabilities.

* Different services can evolve independently.

* Fault isolation is possible when designed correctly.

Disadvantages

* Network failures and latency.

* Distributed tracing and debugging complexity.

* More difficult cross-service transactions.

* Data duplication and eventual consistency.

* Higher infrastructure and operational costs.

| Situation                                                  | Usually consider               |
| ---------------------------------------------------------- | ------------------------------ |
| Small team, straightforward product                        | Modular monolith               |
| Complex domain with clear boundaries                       | Microservices may fit          |
| One component has much higher load                         | Independently scalable service |
| Team lacks deployment and observability maturity           | Avoid premature microservices  |
| Need strict transaction boundaries across one small domain | A monolith may be simpler      |

Architect-level answer: Microservices are an organizational and deployment choice, not an automatic performance improvement. A poorly designed microservices system can be less reliable than a well-designed monolith.

## 15. Horizontal vs vertical scaling

Scaling means increasing the capacity of a system.

### Vertical scaling (scale up)

Increase the resources of one machine.

For example:

* 4 CPU cores → 16 CPU cores.

* 16 GB RAM → 64 GB RAM.

Benefits: simple, often requires fewer application changes.

Limitations: hardware limits, cost, and potentially a larger single point of failure.

### Horizontal scaling (scale out)

Add more machines or application instances.

For example:

- One Spring Boot instance → five instances behind a load balancer.

Benefits: increased capacity and potential fault tolerance.

Limitations: shared state, session management, coordination, load balancing, and database capacity become important.

### Vertical scaling

1 server

4 CPU / 16 GB

Bigger server

16 CPU / 64 GB

### Horizontal scaling

Instance 1

Instance 2

Instance 3

### What can go wrong when scaling horizontally?

Suppose every instance stores shopping-cart data in local memory.

A user adds an item while connected to Instance A. Their next request goes to Instance B.

Instance B does not have the cart.

Solutions include:

* Store shared session or cart data in a shared data store.

* Use stateless authentication where appropriate.

* Use sticky sessions only when their trade-offs are acceptable.

Remember: Adding application servers does not automatically increase database write capacity. The database can become the next bottleneck.

## 16. Load balancer vs API gateway

These are related but different components.

### Load balancer

Distributes traffic across healthy backend instances.

Common responsibilities:

* Traffic distribution.

* Health checks.

* Backend failover.

* TLS termination, depending on configuration.

### API gateway

Provides a managed entry point for APIs.

Common responsibilities:

* Authentication and authorization integration.

* Rate limiting and quotas.

* API routing and request policies.

* API versioning and other cross-cutting concerns.

A product may provide both sets of capabilities, but the concepts are different.

Example:

```
Mobile App
    |
    v
API Gateway
    |
    v
Load Balancer
    |
    +---- Spring Boot Instance 1
    +---- Spring Boot Instance 2
    +---- Spring Boot Instance 3
```

You do not necessarily need both as separate components in every architecture.

# Part 3: The Building Blocks Every Architect Should Know

## 17. SQL vs NoSQL databases

Choosing a database is a business and access-pattern decision, not a trend.

### SQL databases

Examples: PostgreSQL, MySQL, Oracle Database.

Data is commonly stored in tables with defined schemas and relationships.

Use SQL databases when you need:

* Relational data and joins.

* Transactions.

* Constraints and referential integrity.

* Strong consistency for critical business operations.

* Flexible queries over structured data.

Example: banking transactions, order management, financial records.

### NoSQL databases

NoSQL covers several different data models, including document, key-value, wide-column, and graph databases.

Examples: MongoDB, Redis, Cassandra, DynamoDB, Neo4j.

Use them when their specific data model and operational characteristics suit your workload.

| Requirement                                  | Possible fit                      |
| -------------------------------------------- | --------------------------------- |
| Financial ledger and relational transactions | SQL                               |
| Flexible JSON-like documents                 | Document database                 |
| Very fast key-based access and caching       | Key-value store                   |
| Large distributed write-heavy workloads      | Wide-column database              |
| Relationship traversal                       | Graph database                    |
| Complex analytics over large datasets        | Analytical warehouse or lakehouse |

Interview trap: NoSQL does not mean "no consistency" or "always faster." Many NoSQL databases offer transactions or configurable consistency guarantees. Actual performance depends on workload, schema, indexing, data distribution, and configuration.

## 18. Database replication

Replication means maintaining copies of data on multiple database nodes.

### Primary-replica replication

One node accepts writes and replicates changes to other nodes.

```
              Application
                   |
                   v
              Primary DB
              /         \
             v           v
        Replica A     Replica B
         (Reads)       (Reads)
```

Benefits:

* Additional read capacity.

* Data redundancy.

* Potential failover support.

Risks:

* Asynchronous replicas can lag behind the primary.

* A replica may return stale data.

* Failover can lose recently acknowledged writes depending on replication guarantees.

* Replication does not replace backups.

### Synchronous vs asynchronous replication

| Synchronous                                                        | Asynchronous                                                   |
| ------------------------------------------------------------------ | -------------------------------------------------------------- |
| Acknowledgment waits for the required replication condition        | Primary may acknowledge before replicas receive the change     |
| Can reduce the risk of losing acknowledged changes                 | Can improve write latency and availability                     |
| Can increase latency and reduce write availability during failures | Can expose replica lag and potential data loss during failover |

Actual guarantees depend on the protocol and what "acknowledged" means.

### Production scenario

A user updates their profile, then immediately opens the profile page. If the update goes to the primary but the read goes to a lagging replica, the old profile may appear.

Solutions include:

* Read from the primary after a write.

* Use session or read-your-writes consistency.

* Wait until a replication position is reached, where supported.

* Accept eventual consistency if the business can tolerate it.

## 19. Database sharding

Sharding means splitting a dataset across multiple database nodes.

Suppose one database stores 500 million customer records. Eventually, its storage, write throughput, or query capacity may become a bottleneck.

You could partition customers across multiple shards.

```
               Customer Service
                      |
              Shard Router
              /     |     \
             v      v      v
         Shard 1  Shard 2  Shard 3
         A - H     I - P     Q - Z
```

This is a simplified alphabetical example. Real systems often use hash-based or range-based shard keys.

### Common shard strategies

* Range-based: Divide data into ranges, such as dates or ID ranges.

* Hash-based: Hash a key and distribute records across shards.

* Directory-based: Maintain a mapping from keys to shards.

### Choosing a shard key

A good shard key:

* Distributes data and traffic reasonably evenly.

* Supports common query patterns.

* Avoids hot shards.

* Is stable enough for the data's lifecycle.

For example, sharding by customer ID can work well if traffic is distributed across customers. Sharding by a timestamp can create a hot shard if every new record targets the newest range.

### What makes sharding difficult?

* Cross-shard joins.

* Cross-shard transactions.

* Rebalancing data.

* Hot keys and uneven distribution.

* Queries that do not include the shard key.

* Unique constraints across shards.

Architect's rule: Optimize indexes, queries, connection pooling, and database capacity before introducing sharding unless scale requirements clearly justify it.

Replication and sharding are different:

* Replication creates additional copies.

* Sharding divides the dataset into portions.

A system can use both.

## 20. Caching: Redis, local cache, and CDN

Caching stores frequently accessed data in a faster layer so the application does not need to retrieve it from a slower source every time.

Imagine 100,000 users request the same product details.

Without caching, every request may hit the database.

With caching, many requests can be served from memory.

```
Request
   |
   v
Check Cache
   |
   +---- Hit ----> Return cached value
   |
   +---- Miss ---> Read database
                       |
                       v
                  Update cache
                       |
                       v
                  Return value
```

### Types of caching

| Cache                 | Where it lives                  | Good for                                  |
| --------------------- | ------------------------------- | ----------------------------------------- |
| Local cache           | Inside application memory       | Very fast, instance-local data            |
| Distributed cache     | Separate service, such as Redis | Shared cache across application instances |
| CDN                   | Near end users                  | Static assets and cacheable content       |
| Database buffer cache | Inside the database engine      | Frequently accessed database pages        |

### Common caching strategies

Cache-aside

The application checks the cache. On a miss, it reads from the database and populates the cache.

This is common in Spring Boot applications.

Write-through

Writes update the cache and backing store as part of the defined write path.

Write-behind

The cache accepts a write and persists it to the backing store later. This can improve write performance but increases data-loss and recovery risks unless carefully designed.

### Common caching failures

* Cache stampede: Many requests simultaneously rebuild an expired entry.

* Cache penetration: Repeated requests for missing records keep hitting the database.

* Cache avalanche: Many entries expire around the same time.

* Stale data: Cache values do not reflect recent database changes.

* Hot key: One cache entry receives disproportionate traffic.

Possible mitigations include TTL jitter, request coalescing, negative caching, local caching, and controlled refreshes.

### Should Redis be the source of truth?

For most ordinary caching use cases, no. The durable database remains authoritative.

Redis can be used as a primary data store for appropriate workloads, but persistence, replication, eviction, and recovery behavior must match the required durability guarantees.

## 21. Message queues and event-driven architecture

A message broker allows one component to communicate with another without requiring the consumer to process the request immediately.

Examples include Apache Kafka, Google Cloud Pub/Sub, RabbitMQ, and Amazon SQS.

Suppose a user places an order.

The order service should confirm the order without waiting for every downstream activity to finish.

```
Client
  |
  v
Order Service
  |
  +----> Order Database
  |
  +----> Message Broker
               |
               +----> Notification Worker
               |
               +----> Analytics Consumer
               |
               +----> Shipping Consumer
```

The consumers process the events independently.

### Queue vs event stream

A traditional work queue typically distributes tasks among competing consumers. A consumer acknowledges work according to the broker's delivery model.

An event stream retains ordered records within defined partitions and allows consumers to track offsets and replay events according to retention and permissions.

These are different abstractions, although modern platforms may support both patterns.

### Why use asynchronous processing?

* Reduce API response latency.

* Absorb traffic spikes.

* Decouple services.

* Retry temporary failures.

* Scale consumers independently.

### What can go wrong?

* Queue backlog grows faster than consumers can process it.

* Messages are delivered more than once.

* Consumers process events out of order.

* Poison messages repeatedly fail.

* A message is acknowledged before its work is durable.

* Events are lost between a database transaction and publication.

### How do you avoid losing an event?

Consider this sequence:

1. Save an order in the database.

2. Publish `OrderCreated` to the broker.

What happens if the database commit succeeds but the broker publish fails?

The order exists, but downstream services never hear about it.

A common solution is the Transactional Outbox Pattern.

Within one database transaction, save both:

* The business record.

* An outbox event record.

A separate publisher reads the outbox and publishes events to the broker. It marks events as published or otherwise tracks progress.

The publisher can send an event more than once, so consumers should be idempotent.

The outbox solves the database-and-event dual-write problem when correctly implemented, but it does not eliminate the need for retries, monitoring, retention, and deduplication.

## 22. Synchronous vs asynchronous communication

| Synchronous REST/gRPC                      | Asynchronous messaging                              |
| ------------------------------------------ | --------------------------------------------------- |
| Caller waits for a response                | Caller and consumer are decoupled in time           |
| Straightforward request-response flow      | Good for background jobs and events                 |
| Failures and latency propagate to callers  | Backlogs and delayed processing are possible        |
| Often simpler for immediate queries        | Better for independently processed workloads        |
| Requires timeouts and bounded resource use | Requires retries, idempotency, and queue monitoring |

Use synchronous communication when the caller genuinely needs the answer now.

Use asynchronous communication when the work can be completed later or when temporal decoupling is valuable.

Do not make every API asynchronous just because your system needs to scale. Asynchronous systems add operational and consistency complexity.

## 23. API design: REST, gRPC, and idempotency

### REST

REST-style APIs commonly use HTTP methods and resource-oriented URLs.

Example:

http

```
POST /orders
GET /orders/123
PUT /orders/123
DELETE /orders/123
```

REST is often suitable for public APIs, web applications, and service integrations.

### gRPC

gRPC uses Protocol Buffers by default and commonly runs over HTTP/2.

It is useful for strongly defined service contracts, efficient service-to-service communication, and streaming use cases.

### What is idempotency?

An operation is idempotent when repeating it has the same intended effect as performing it once.

For example, setting a user's status to `ACTIVE` repeatedly should leave the user active.

Payment creation is more challenging. If a client times out after the server processes a payment, the client might retry and accidentally create another payment.

An idempotency key can prevent duplicate processing.

http

```
POST /payments
Idempotency-Key: payment-request-abc123
```

The server stores the key with the request's outcome and ensures retries return the existing result instead of creating a second payment.

Production requirements include:

* Define the scope of uniqueness.

* Persist the key and result durably enough.

* Handle simultaneous requests with the same key.

* Reject reuse with incompatible request payloads.

* Choose a retention period that matches retry behavior.

* Coordinate external payment providers when necessary.

Idempotency is especially important when retries are used.

### A common interview trap

HTTP method semantics alone do not guarantee that your business operation is implemented correctly. A `POST` can be made idempotent using an application-level key, while a `PUT` handler can still contain bugs that violate idempotency.

## 24. Rate limiting and backpressure

Suppose your service can process 2,000 requests per second, but a client suddenly sends 20,000 requests per second.

Without protection, your application may exhaust its threads, memory, database connections, or downstream quotas.

### Rate limiting

Rate limiting controls how many requests a client or resource can make during a time window.

Common algorithms:

* Fixed window.

* Sliding window.

* Token bucket.

* Leaky bucket.

Token bucket is a common choice because it allows controlled bursts while enforcing a long-term rate.

### Backpressure

Backpressure prevents upstream producers from overwhelming downstream consumers.

Examples:

* Bound executor queues.

* Limit concurrent downstream requests.

* Slow down producers.

* Reject excess work.

* Pause message consumption when appropriate.

A bounded queue is often safer than an unlimited queue because it makes overload visible instead of allowing memory usage to grow without limit.

When the queue is full, define an explicit policy: reject, defer, throttle, or execute work in the caller where appropriate.

For a Spring Boot service, you should also configure timeouts and connection pools for downstream calls.

# Part 4: Reliability, Resilience, and Fault Tolerance

## 25. Reliability vs availability vs fault tolerance

These terms are related but not interchangeable.

* Reliability: Does the system perform correctly over time?

* Availability: Is the system usable when needed?

* Fault tolerance: Can it continue providing the required service despite specified component failures?

* Resilience: Can it withstand disturbances and recover from them?

For example, a payment API may be available but unreliable if it accepts requests and produces incorrect transactions.

A system can be highly reliable in ordinary conditions but have poor fault tolerance if one database failure takes down the entire application.

### High availability

High availability often uses redundancy and automated failover to reduce downtime.

```
                 Load Balancer
                  /         \
                 v           v
             App A         App B
                 \           /
                  v         v
                  Database HA
```

Redundancy helps only if failure domains are sufficiently independent. Two instances on the same failing machine or in the same affected zone may fail together.

### Disaster recovery

Disaster recovery focuses on recovering service after a major outage, such as losing an entire region or data center.

Two essential terms:

* RTO (Recovery Time Objective): How long can the service be unavailable?

* RPO (Recovery Point Objective): How much data loss, measured in time, can the business tolerate?

Example:

* RTO = 30 minutes.

* RPO = 5 minutes.

This means the recovery objective is to restore the service within 30 minutes and limit data loss to the equivalent of at most five minutes of changes. Whether this can be achieved depends on the actual architecture and recovery process.

Backups, replication, recovery testing, and documented procedures all matter.

## 26. Timeouts, retries, circuit breakers, and bulkheads

These four patterns frequently appear in senior Java backend interviews.

### Timeout

A timeout sets a limit on how long an operation may wait.

Without timeouts, a downstream service that stops responding can tie up application threads and connections.

Use timeouts for database operations, REST clients, RPC calls, and other potentially blocking operations.

### Retry

A retry repeats an operation after a failure that may be temporary.

Good retry candidates:

* Temporary network errors.

* Transient rate limits, when the server's retry instructions permit it.

* Temporary service unavailability.

Poor retry candidates:

* Invalid input.

* Authentication failures.

* Permanent business rule violations.

Use bounded retries, exponential backoff, jitter, and an overall deadline.

Retries can amplify load when a downstream service is already unhealthy. Do not retry every error blindly.

### Circuit breaker

A circuit breaker temporarily stops sending calls to a failing dependency.

Its common states are:

* Closed: Requests flow normally.

* Open: Calls fail fast or use a fallback.

* Half-open: A limited number of test calls determine whether recovery has occurred.

A circuit breaker prevents repeated slow failures from consuming resources. It does not fix the underlying service or guarantee that a fallback is correct.

### Bulkhead

Bulkheads isolate resources so a failure in one workload does not consume every resource needed by other workloads.

For example, separate concurrency limits for a slow reporting API and a payment API can prevent reporting from starving payment processing.

### How they work together

```
Request
   |
   v
Bulkhead / concurrency limit
   |
   v
Circuit Breaker
   |
   v
Timeout-protected call
   |
   +---- Success ----> Return result
   |
   +---- Transient failure
                 |
                 v
            Bounded Retry
                 |
                 +---- Success
                 |
                 +---- Exhausted
                              |
                              v
                        Error / Fallback
```

The exact order and interaction depend on the client library and use case. In particular, ensure that a retry policy does not unintentionally multiply the permitted call duration beyond the request's deadline.

### Example in Spring Boot

Common options include:

* Spring's HTTP clients with explicit connection and response timeouts.

* Resilience4j for circuit breakers, retries, rate limiters, and bulkheads.

* Micrometer metrics for observing failure rates, latency, and rejected calls.

For a payment operation, a fallback must not falsely report that payment succeeded. It is safer to return a clear pending or failure state than to invent a successful outcome.

# Part 5: How to Approach a System Design Interview

## 27. The interview framework: Requirements → Estimate → Design → Deep dive

As a Technical Architect, you should demonstrate not only that you know components, but also that you can justify their use.

Use this framework for almost every system design question.

Step 1 — Clarify requirements

Identify users, core features, scale, consistency requirements, latency targets, security, and availability expectations.

Step 2 — Estimate capacity

Estimate requests per second, storage, bandwidth, concurrency, and peak traffic.

Step 3 — Draw the high-level architecture

Identify clients, entry points, services, databases, caches, queues, and external dependencies.

Step 4 — Explain critical flows

Walk through important operations, such as placing an order or processing a payment.

Step 5 — Discuss failure modes

Explain retries, timeouts, failover, duplicate requests, data loss, and recovery.

Step 6 — Find bottlenecks and trade-offs

Explain what fails first, how you would scale it, and why your design is appropriate.

Don't spend 20 minutes drawing boxes before clarifying the requirements. A design can be technically elegant and still solve the wrong problem.

## 28. Capacity estimation: A practical example

Suppose an interviewer asks you to design a service for 10 million daily users.

You do not know the exact traffic, so state your assumptions.

Assume:

* 10 million requests per day.

* Average request size: 10 KB.

* Peak traffic is 5 times the average.

* Each request generates one database read, on average.

### Step 1: Calculate average requests per second

There are 86,400 seconds in a day.

Average RPS=10,000,00086,400\text{Average RPS}=\frac{10,000,000}{86,400}Average RPS=86,40010,000,000

≈116 requests/second\approx116\text{ requests/second}≈116 requests/second

### Step 2: Estimate peak traffic

Peak RPS=116×5\text{Peak RPS}=116\times5Peak RPS=116×5

≈580 requests/second\approx580\text{ requests/second}≈580 requests/second

### Step 3: Estimate incoming data

10,000,000×10 KB10,000,000\times10\text{ KB}10,000,000×10 KB

That is approximately 100 GB of incoming request payload per day using decimal units, if every request carries 10 KB.

This is a rough upper-level estimate, not necessarily the amount that must be stored in the database.

### Step 4: Estimate concurrency

Little's Law gives a useful relationship:

L=λWL=\lambda WL=λW

Where:

* LLL = average number of requests in the system.

* λ\lambdaλ = arrival rate in requests per second.

* WWW = average time spent in the system, in seconds.

If peak arrival rate is 580 requests/second and average response time is 0.2 seconds:

L=580×0.2=116L=580\times0.2=116L=580×0.2=116

That implies approximately 116 requests in the system on average under those assumptions.

This is not the number of servers you need. Server capacity depends on CPU time, I/O wait, memory, connection pools, workload variability, and latency targets.

### What should an architect estimate?

* Average and peak requests per second.

* Read-to-write ratio.

* Average and maximum payload sizes.

* Database storage growth.

* Network bandwidth.

* Concurrent requests.

* Queue growth during bursts.

* Retention and backup requirements.

Interview tip: Show your assumptions, calculate approximately, and explain which estimates need validation with load testing.

## 29. Latency vs throughput

These terms are often confused.

Latency is how long an individual operation takes.

Throughput is how much work a system completes per unit of time.

For example:

* Latency: 200 milliseconds per API request.

* Throughput: 2,000 requests per second.

A system can have high throughput but poor latency under load. It can also have low latency for a small number of requests but fail when traffic increases.

### What is p95 and p99 latency?

Imagine 1,000 API requests.

* p95 latency is the latency at or below which approximately 95% of requests complete.

* p99 latency is the latency at or below which approximately 99% complete.

If p99 latency is 2 seconds, roughly 1% of requests take at least around 2 seconds, subject to the measurement definition.

Average latency alone can hide slow requests.

In production, track:

* Request rate.

* Error rate.

* p50, p95, and p99 latency.

* Resource saturation.

* Queue lag.

* Database connection utilization.

## 30. Database indexing and query optimization

An index helps the database find rows without scanning the entire table in many query patterns.

Imagine a customer table with 10 million rows.

Without a suitable index, this query might require a large scan:

SQL

```
SELECT *
FROM customers
WHERE email = 'user@example.com';
```

A suitable index can make the lookup much faster:

SQL

```
CREATE INDEX idx_customers_email
ON customers(email);
```

Actual performance depends on data distribution, query plans, index selectivity, table size, and database behavior.

### Why not index every column?

Indexes:

* Consume storage.

* Add write overhead.

* Require maintenance.

* May not help queries that cannot use them efficiently.

### What should you inspect before adding infrastructure?

1. The actual SQL query.

2. The execution plan.

3. Existing indexes.

4. Table statistics and data distribution.

5. Locking and transaction behavior.

6. Connection pool usage.

7. Slow-query metrics.

A slow database query may be a better explanation for poor application performance than insufficient application servers.

## 31. Consistency models you should know

CAP uses consistency in a particular distributed-systems sense, but real system design interviews often explore several related models.

| Model                                | Meaning                                                              | Typical use                                    |
| ------------------------------------ | -------------------------------------------------------------------- | ---------------------------------------------- |
| Strong consistency / linearizability | Operations appear to occur in a single real-time-respecting order    | Critical account and coordination operations   |
| Eventual consistency                 | Replicas converge if updates stop and replication progresses         | Social feeds, replicated user preferences      |
| Read-your-writes                     | A user sees their own successful updates                             | Profile editing, shopping experiences          |
| Monotonic reads                      | Once a client sees a newer value, it does not later see an older one | User-facing replicated reads                   |
| Causal consistency                   | Causally related operations respect their ordering                   | Distributed collaboration and dependent events |

These are not interchangeable guarantees. A system may offer one without offering all the others.

### Example: social media

You post a comment.

Your local service accepts it, but another region has not replicated the comment yet.

* An eventual-consistency design may allow another user to see the comment later.

* A read-your-writes design tries to ensure that you see your own comment immediately.

* A strongly consistent design may coordinate reads and writes more strictly, potentially increasing latency.

A useful architect question is: Which user-visible inconsistencies are acceptable, and for how long?

## 32. Distributed transactions: Two-phase commit vs Saga

Imagine an order requires three operations:

1. Create the order.

2. Reserve inventory.

3. Charge the payment method.

Each operation may be owned by a different service and database.

A local SQL transaction cannot automatically make all three independent systems atomic.

### Two-phase commit (2PC)

Two-phase commit coordinates participants through two stages:

1. Prepare: Participants indicate whether they can commit.

2. Commit or abort: The coordinator decides the outcome.

Advantages:

- Can provide atomic commit across supported participants.

Disadvantages:

* Participants may hold resources while waiting.

* Coordinator or network failures complicate progress.

* Not all services or databases support it.

* It can reduce availability and increase latency.

2PC is not the same thing as consensus, and its practical failure guarantees depend on the implementation.

### Saga pattern

A Saga splits a business transaction into local transactions. If a later step fails, compensating actions undo or counteract earlier business effects where possible.

Example:

```
Create Order
     |
     v
Reserve Inventory
     |
     v
Charge Payment
     |
     +---- Success ----> Confirm Order
     |
     +---- Failure ----> Release Inventory
                          Cancel / Compensate Order
```

A compensation is not necessarily a literal rollback. For example, a completed payment may require a refund, which is another business transaction.

Sagas are useful for long-running workflows where distributed atomic transactions are impractical.

### Choreography vs orchestration

* Choreography: Services react to events and trigger the next step.

* Orchestration: A coordinator explicitly directs the workflow.

Choreography reduces central coordination but can make complex flows difficult to understand. Orchestration improves visibility into workflow state but adds a coordinator that must be designed for reliability.

For an order workflow with many steps, timeouts, and compensations, orchestration can be easier to operate.

## 33. Shifting work off the request path

A common system design improvement is separating immediate user-facing work from background processing.

Consider file uploads or bulk imports.

Instead of parsing and processing a 1 GB file inside an HTTP request:

1. Accept the upload.

2. Store the file in object storage.

3. Create a job record.

4. Publish a job or make it available to workers.

5. Return a job ID.

6. Process the file asynchronously.

7. Expose status and error details.

The client can query:

http

```
GET /jobs/job-123
```

Possible statuses:

* `QUEUED`

* `PROCESSING`

* `COMPLETED`

* `FAILED`

Important considerations:

* Idempotent job submission.

* Bounded concurrency.

* Checkpointing.

* Retry policy.

* Dead-letter handling.

* Partial failures.

* Job status durability.

* Cancellation and restart behavior.

This pattern is often better than extending HTTP timeouts for long-running work.

# Part 6: A Complete System Design Example — URL Shortener

Let's combine everything into one practical example.

Imagine you're asked:

> Design a service like Bitly that converts long URLs into short URLs.

## 34. Step 1: Clarify requirements

Functional requirements:

* Create a short URL from a long URL.

* Redirect users to the original URL.

* Optionally allow custom aliases.

* Optionally support expiration.

* Collect click analytics.

Non-functional requirements:

* Redirects should be fast.

* The system should handle high read traffic.

* URLs should not be accidentally overwritten.

* The service should be available during traffic spikes.

* Analytics should not significantly slow redirects.

For this example, assume analytics can be eventually consistent.

## 35. Step 2: Design the architecture

Client

Browser / Mobile app

Load Balancer / API Gateway

URL Shortener Service

Spring Boot instances

Redis

Cache popular URL mappings

Database

Durable URL mappings

Message Broker

Asynchronous click analytics

## 36. Step 3: Design the APIs

Create a short URL:

http

```
POST /api/v1/urls
Content-Type: application/json

{
  "longUrl": "https://example.com/products/123"
}
```

Example response:

JSON

```
{
  "shortCode": "aB12xY",
  "shortUrl": "https://sho.rt/aB12xY"
}
```

Redirect:

http

```
GET /aB12xY
```

The server looks up the code and returns an HTTP redirect, commonly `302 Found` or `301 Moved Permanently`, depending on the intended behavior.

Use `302` when you want a temporary redirect and greater flexibility in changing the destination. Permanent redirects may be cached more aggressively by browsers and intermediaries.

## 37. Step 4: Choose the database

A relational database can store:

| Column       | Purpose              |
| ------------ | -------------------- |
| `short_code` | Unique identifier    |
| `long_url`   | Original destination |
| `created_at` | Creation time        |
| `expires_at` | Optional expiration  |
| `owner_id`   | Optional owner       |

Add a unique constraint on `short_code`.

A short code can be generated using a unique ID encoded in Base62, or another collision-controlled strategy.

Base62 uses 62 characters:

`a-z`, `A-Z`, and `0-9`.

Do not assume that a random code will never collide. Enforce uniqueness and handle collisions safely.

## 38. Step 5: Optimize reads with caching

Redirects will often be more frequent than URL creation.

Use Redis to cache popular mappings:

```
GET /aB12xY
      |
      v
  Redis lookup
      |
      +---- Hit ----> Redirect
      |
      +---- Miss ---> Database lookup
                            |
                            v
                       Cache result
                            |
                            v
                         Redirect
```

Define TTL, cache invalidation, expiration handling, and behavior when Redis is unavailable.

For URL mappings that change infrequently, caching can reduce database load substantially.

## 39. Step 6: Handle analytics asynchronously

Do not make every redirect wait for a complex analytics pipeline.

The redirect service can publish a click event, which downstream consumers process asynchronously.

However, event publication must be designed carefully. If losing a click event is unacceptable, use a durable publication mechanism and monitor failures. If approximate analytics are acceptable, a simpler best-effort approach might be sufficient.

## 40. Step 7: Discuss failures

| Failure                                   | Possible solution                                                      |
| ----------------------------------------- | ---------------------------------------------------------------------- |
| Redis is unavailable                      | Fall back to the database, with protection against a sudden load spike |
| Database primary fails                    | Use database HA and a tested failover process                          |
| Two clients request the same custom alias | Enforce a unique constraint and return a conflict                      |
| Analytics consumer is down                | Retain messages and process them after recovery                        |
| A popular URL overloads one cache key     | Use request coalescing or another hot-key mitigation                   |
| A URL expires                             | Check expiration and return an appropriate response                    |
| The system receives abusive traffic       | Apply rate limits and abuse detection                                  |

### Why this design works

It separates durable data from cache, separates latency-sensitive redirects from analytics, and defines explicit failure behavior.

You don't need microservices for every component at the beginning. A single Spring Boot application with a relational database, Redis, and a background worker can be a perfectly reasonable starting point.

# Part 7: Frequently Asked Interview Questions

Use these as quick revision questions. Try answering each aloud before expanding the answer.

## 41. CAP Theorem — basic questions

1. What is CAP Theorem?

2. Why can't we have all three properties?

3. What is the difference between consistency and availability?

4. What is a network partition?

5. What is a CP system?

6. What is an AP system?

7. What is a CA system?

8. Does CAP apply only to databases?

9. Is eventual consistency the same as weak availability?

10. What is PACELC?

## 42. System design — basic questions

1. What is the first step in system design?

2. What is the difference between scalability and availability?

3. When should we use a cache?

4. When should we use a message queue?

5. Why not use microservices for every system?

6. What is the difference between replication and sharding?

7. How do you avoid duplicate payment processing?

8. What is the purpose of a load balancer?

9. What is a single point of failure?

10. What is the difference between RTO and RPO?

11. Why are retries dangerous?

12. How do you find a bottleneck?

13. What is a hot partition or hot shard?

14. Why is a distributed lock difficult?

15. What is the difference between fault tolerance and disaster recovery?

# Part 8: Scenario-Based Questions for Technical Architects

These questions test your ability to reason about failure, business requirements, and trade-offs.

## Scenario 1: Your distributed system loses connectivity between regions

Question: Do you keep accepting writes in both regions?

A strong answer:

"I would first identify which operations require strong consistency and which can tolerate temporary inconsistency. For financial balances or inventory reservations, I would prioritize preventing conflicting commits. For less critical user preferences, I might permit local writes with conflict resolution. I would define the expected behavior during partition and recovery rather than promise both strong consistency and full availability."

## Scenario 2: Your API has suddenly become slow

Question: Do you add more Spring Boot instances?

A strong answer:

"Not before identifying the bottleneck. I would inspect latency percentiles, traces, CPU, garbage collection, database query latency, connection pool utilization, downstream timeouts, and queue depth. If application CPU is saturated, scaling out may help. If the database is saturated, adding application instances could increase contention and make the situation worse."

## Scenario 3: Your payment service receives duplicate requests

Question: How do you prevent charging customers twice?

A strong answer:

"I would introduce durable idempotency keys and enforce uniqueness at the appropriate transaction boundary. I would model payment state transitions explicitly and coordinate retries with the external payment provider. If the request times out, I would check the recorded outcome instead of assuming the payment failed."

## Scenario 4: Your message queue has a growing backlog

Question: Do you add more consumers?

A strong answer:

"First, I would determine whether producers are exceeding consumer capacity, whether consumers are slow, and whether the database or an external API is the actual bottleneck. I would check lag, processing time, error rates, retry volume, and partition or concurrency limits. I would scale consumers only if downstream capacity and ordering constraints allow it."

## Scenario 5: Your application must process 10 million records

Question: Would you load all records into memory and process them in parallel?

A strong answer:

"No. I would use streaming or chunk-based processing, bounded concurrency, database batching, checkpointing, and idempotent writes. I would identify whether the bottleneck is parsing, external API calls, CPU, or database writes. I would limit concurrency to the actual capacity of downstream systems rather than create an unbounded number of tasks."

## Scenario 6: Your cache and database disagree

Question: Which value do you trust?

A strong answer:

"For a conventional cache-aside design, the database is authoritative. I would investigate the invalidation and update path, then invalidate or refresh the stale cache entry. If the business requires read-your-writes behavior, I would ensure the read path can observe the latest committed update. I would not assume cache eviction alone solves every consistency issue."

## Scenario 7: One availability zone fails

Question: Will your application survive?

A strong answer:

"Only if sufficient capacity and dependencies remain outside the failed zone. I would check the placement of application instances, load balancers, databases, message brokers, and cache nodes. I would also test whether surviving instances can handle peak traffic, whether quorum is maintained, and whether failover objectives are met."

# Part 9: Common System Design Mistakes

These mistakes can make an otherwise good architecture answer weaker.

1. Starting with technologies instead of requirements. Don't begin by declaring that you need Kafka, Redis, Kubernetes, and microservices.

2. Ignoring failure modes. Every network call can time out. Every queue can build a backlog. Every dependency can become unavailable.

3. Assuming retries guarantee success. Retries only help when failures are transient and the operation is safe to repeat.

4. Treating a cache as a database replacement. Define the source of truth and the cache recovery strategy.

5. Assuming replication means zero data loss. Replication guarantees depend on acknowledgment semantics, consistency settings, and failure handling.

6. Using an unlimited queue. It can convert overload into memory exhaustion and long delays.

7. Ignoring the database. A horizontally scalable application cannot compensate indefinitely for a saturated database.

8. Choosing strong consistency everywhere. Some data requires strict guarantees; other data can tolerate delayed convergence.

9. Choosing eventual consistency without a conflict policy. If two regions update the same record, decide how the conflict is resolved.

10. Adding complexity without evidence. Sharding, microservices, multi-region active-active deployments, and custom distributed coordination all carry costs.

11. Ignoring security. Consider authentication, authorization, encryption, secrets, rate limiting, audit trails, and data isolation.

12. Skipping observability. If the team cannot detect a failure, measure its impact, or trace a request, operating the architecture will be difficult.

# Part 10: Your One-Page Revision Cheat Sheet

## CAP Theorem

| Concept              | Remember this                                                                |
| -------------------- | ---------------------------------------------------------------------------- |
| Consistency          | Strong reads reflect the latest committed write under the stated model       |
| Availability         | Requests receive successful responses under the formal model                 |
| Partition tolerance  | Handle communication failures between nodes                                  |
| CP                   | Prefer consistency; some requests may fail or wait                           |
| AP                   | Prefer responses; stale or conflicting data may occur                        |
| CA                   | Not a guarantee through partitions in a distributed system                   |
| PACELC               | Partition: A vs C; otherwise: latency vs consistency                         |
| Quorum               | Enough participants to make a safe decision under the protocol               |
| Eventual consistency | Replicas can converge; convergence alone does not define conflict resolution |

## System design

| Requirement                     | Common building block                           |
| ------------------------------- | ----------------------------------------------- |
| More application capacity       | Horizontal scaling and load balancing           |
| Faster repeated reads           | Cache                                           |
| Durable business records        | Database                                        |
| Split large datasets            | Sharding                                        |
| More copies of data             | Replication                                     |
| Background work                 | Queue or event stream                           |
| Protect downstream dependencies | Timeouts, bounded concurrency, circuit breakers |
| Prevent duplicate operations    | Idempotency and uniqueness constraints          |
| Distributed business workflow   | Saga or a suitable transaction protocol         |
| Recover after major outage      | Backups, replication, DR plan, recovery tests   |
| Understand production behavior  | Logs, metrics, traces, alerts                   |
| Control overload                | Rate limits, backpressure, bounded queues       |

## The five questions to ask about every design

Before finalizing an architecture, ask yourself:

* Scale: What becomes the bottleneck first?

* Consistency: Can users see stale or conflicting data?

* Failure: What happens when a dependency or region goes down?

* Recovery: How do we restore service and reconcile unfinished work?

* Trade-off: Why is this design better for these requirements than the alternatives?

# Final advice for your Technical Architect interviews

The difference between a basic developer answer and a strong architect answer is usually not the number of technologies mentioned. It is the quality of the reasoning.

When asked to design a system, explain:

1. What the business needs.

2. What guarantees the business cannot compromise.

3. Which components satisfy those requirements.

4. How the components behave under load and failure.

5. What trade-offs the design introduces.

6. How you would measure and validate the result in production.

For CAP specifically, remember this sentence:

During a network partition, you cannot guarantee both strong consistency and availability for every request. The correct choice depends on what the business can tolerate.

For system design generally, remember this sentence:

Start with requirements, design for the expected workload, assume failures will happen, and choose the simplest architecture that meets the required guarantees.

That mindset will serve you much better in senior backend and Technical Architect interviews than memorizing lists of tools.

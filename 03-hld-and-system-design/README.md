<!-- markdownlint-disable MD001 -->
<!-- markdownlint-disable MD012 -->
<!-- markdownlint-disable MD024 -->
<!-- markdownlint-disable MD025 -->
<!-- markdownlint-disable MD026 -->
<!-- markdownlint-disable MD029 -->
<!-- markdownlint-disable MD033 -->
<!-- markdownlint-disable MD040 -->
<!-- markdownlint-disable MD056 -->
<!-- markdownlint-disable MD060 -->

# HLD and System Design: Topic Plan

## How to use this plan

Work the parts in order. Parts A to I build the vocabulary and building blocks.  
Parts J to M cover operating, securing and specializing systems.  
Part N is the architect layer. Part O is the problem canon. Part P is interview execution.

1. **Foundations (Parts A, B):** requirements, estimation, networking. Target: size any system on paper in under 5 minutes.
2. **Data (Parts C, D):** storage engines, replication, partitioning, caching. Target: choose and defend a datastore for any access pattern.
3. **Distributed systems (Parts E, F):** consistency, consensus, messaging, streams. Target: explain what breaks under a network partition and why.
4. **Architecture and quality attributes (Parts G, H, I):** styles, reliability, scalability. Target: state an availability number and show the design that earns it.
5. **Operations, security, specialized systems (Parts J, K, L, M).**
6. **Architect skills (Part N):** ADRs, trade-off analysis, cost, design reviews.
7. **Problem drills (Part O):** Tier 1 first. 45 to 60 minutes each, then a 20-minute self-review against the Part P rubric.
8. **Mock rounds:** two per week, alternating with section 02 machine coding mocks.

Reference material used:

- [system-design-primer](https://github.com/donnemartin/system-design-primer): topic index, interview approach, worked problems
- [awesome-scalability](https://github.com/binhnguyennus/awesome-scalability): principles, scalability, availability, stability, performance, real-world case studies
- [system-design-notebook](https://github.com/bregman-arie/system-design-notebook): components, reliability, networking, distributed systems, exercises, interview tips
- [awesome-software-architecture](https://github.com/mehdihadeli/awesome-software-architecture): architecture styles, distributed systems, messaging, DDD
- [awesome-design-patterns](https://github.com/DovAmir/awesome-design-patterns): cloud, serverless, microservices and distributed patterns
- [devops-exercises](https://github.com/bregman-arie/devops-exercises): operations and infrastructure questions (feeds section 09 more than this one)

---

## Part A: Foundations, requirements and estimation

The goal is to turn a vague prompt into numbers that drive every later decision.

### A1. Requirements engineering

- [ ] A1.1 Functional vs non-functional requirements; scoping to 3 to 5 core use cases
- [ ] A1.2 Quality attributes: availability, reliability, scalability, latency, throughput, durability, consistency, security, cost, operability
- [ ] A1.3 Read/write ratio, access patterns, data shape, data growth
- [ ] A1.4 Users, regions, peak vs average traffic, burst factor
- [ ] A1.5 Explicit non-goals and assumptions

### A2. Core trade-off vocabulary

- [ ] A2.1 Performance vs scalability
- [ ] A2.2 Latency vs throughput; percentiles (p50, p95, p99, p99.9) and tail latency
- [ ] A2.3 Availability vs consistency
- [ ] A2.4 Vertical vs horizontal scaling
- [ ] A2.5 Stateless vs stateful services
- [ ] A2.6 Push vs pull; sync vs async; batch vs stream
- [ ] A2.7 Build vs buy; managed vs self-hosted

### A3. Back-of-the-envelope estimation

- [ ] A3.1 Powers of two; bytes for common types; data size units
- [ ] A3.2 Latency numbers every programmer should know (L1, RAM, SSD, disk seek, same-DC round trip, cross-region round trip)
- [ ] A3.3 QPS from DAU: average, peak, seconds per day (about 86,400, round to 100,000)
- [ ] A3.4 Storage over 1, 3 and 5 years including replication factor and indexes
- [ ] A3.5 Bandwidth: ingress and egress
- [ ] A3.6 Memory for cache sizing (80/20 rule)
- [ ] A3.7 Server count from per-node capacity; connection count limits
- [ ] A3.8 Availability math: nines to downtime per year; components in series vs in parallel

---

## Part B: Networking and communication

### B1. Network fundamentals

- [ ] B1.1 OSI and TCP/IP models; what lives at L4 vs L7
- [ ] B1.2 TCP vs UDP; handshakes, congestion control, head-of-line blocking
- [ ] B1.3 HTTP/1.1, HTTP/2 (multiplexing), HTTP/3 and QUIC
- [ ] B1.4 TLS handshake, certificates, TLS termination, mTLS
- [ ] B1.5 IP addressing, NAT, subnets, VPC basics, anycast

### B2. Edge and traffic management

- [ ] B2.1 DNS: resolution flow, record types, TTL, GeoDNS, DNS-based failover
- [ ] B2.2 CDN: push vs pull, cache keys, invalidation, edge compute
- [ ] B2.3 Load balancers: L4 vs L7, algorithms (round robin, weighted, least connections, consistent hashing, power of two choices), health checks, sticky sessions
- [ ] B2.4 Reverse proxy vs forward proxy vs load balancer
- [ ] B2.5 API gateway: routing, auth, rate limiting, request aggregation, BFF (backend for frontend)
- [ ] B2.6 Service mesh: sidecar, data plane vs control plane (Envoy, Istio, Linkerd)
- [ ] B2.7 Global traffic management: geo-routing, latency-based routing, multi-region active-active

### B3. Communication styles

- [ ] B3.1 REST: resources, idempotency, pagination, versioning, HATEOAS awareness
- [ ] B3.2 RPC and gRPC: protobuf, streaming modes, deadlines
- [ ] B3.3 GraphQL: schema, resolvers, N+1, federation
- [ ] B3.4 Real-time: short polling, long polling, Server-Sent Events, WebSockets, WebRTC
- [ ] B3.5 Webhooks: delivery guarantees, retries, signing
- [ ] B3.6 Serialization formats: JSON, protobuf, Avro, Thrift, MessagePack; schema evolution
- [ ] B3.7 Service discovery: client-side vs server-side, registries (Consul, etcd, ZooKeeper, Eureka)

---

## Part C: Data storage

Choosing the datastore is the single highest-leverage decision in most designs.

### C1. Relational databases

- [ ] C1.1 ACID; transaction isolation levels (read uncommitted to serializable); anomalies (dirty read, non-repeatable read, phantom, write skew, lost update)
- [ ] C1.2 MVCC, locking, deadlocks, optimistic vs pessimistic concurrency
- [ ] C1.3 Indexes: B-tree, hash, composite, covering, partial; when indexes hurt writes
- [ ] C1.4 Normalization vs denormalization
- [ ] C1.5 Query planning basics, N+1, connection pooling
- [ ] C1.6 SQL tuning; read replicas; federation (functional partitioning)

### C2. NoSQL families

- [ ] C2.1 Key-value (Redis, DynamoDB, Riak)
- [ ] C2.2 Wide-column (Cassandra, HBase, Bigtable, ScyllaDB)
- [ ] C2.3 Document (MongoDB, Couchbase)
- [ ] C2.4 Graph (Neo4j, Neptune); when relationships dominate
- [ ] C2.5 Time-series (InfluxDB, TimescaleDB, Prometheus TSDB)
- [ ] C2.6 Search engines (Elasticsearch, OpenSearch, Solr): inverted index, relevance
- [ ] C2.7 Vector databases (pgvector, Milvus, Pinecone): ANN indexes (HNSW, IVF)
- [ ] C2.8 NewSQL (Spanner, CockroachDB, TiDB, YugabyteDB)
- [ ] C2.9 SQL vs NoSQL decision matrix by access pattern, consistency need, scale

### C3. Storage engine internals

- [ ] C3.1 B-tree vs LSM tree; write amplification, read amplification, space amplification
- [ ] C3.2 Write-ahead log, memtable, SSTable, compaction strategies
- [ ] C3.3 Bloom filters; row vs columnar storage
- [ ] C3.4 Object storage (S3 model): blobs, multipart upload, presigned URLs, storage classes
- [ ] C3.5 Block vs file vs object storage; distributed file systems (HDFS, GFS)

### C4. Replication

- [ ] C4.1 Single-leader, multi-leader, leaderless
- [ ] C4.2 Synchronous vs asynchronous vs semi-synchronous
- [ ] C4.3 Replication lag; read-your-writes, monotonic reads, consistent prefix
- [ ] C4.4 Conflict resolution: last-write-wins, version vectors, CRDTs
- [ ] C4.5 Quorums: N, W, R; sloppy quorum, hinted handoff, read repair, anti-entropy with Merkle trees
- [ ] C4.6 Failover: leader election, split brain, fencing

### C5. Partitioning and sharding

- [ ] C5.1 Range vs hash partitioning; directory-based partitioning
- [ ] C5.2 Consistent hashing, virtual nodes, rendezvous hashing
- [ ] C5.3 Choosing a shard key; hot partitions and celebrity problem; key salting
- [ ] C5.4 Secondary indexes: local vs global
- [ ] C5.5 Rebalancing and resharding without downtime
- [ ] C5.6 Cross-shard queries and transactions

### C6. Data modeling for scale

- [ ] C6.1 Modeling for query patterns (Cassandra/DynamoDB single-table design)
- [ ] C6.2 Fan-out on write vs fan-out on read
- [ ] C6.3 Counters, leaderboards, timelines, graphs at scale
- [ ] C6.4 Soft deletes, TTLs, archival, data retention
- [ ] C6.5 Multi-tenancy models: shared schema, schema per tenant, database per tenant

---

## Part D: Caching

### D1. Where to cache

- [ ] D1.1 Client, browser, CDN, edge, API gateway, application, distributed cache, database buffer pool
- [ ] D1.2 Local in-process cache vs distributed cache (Redis, Memcached)

### D2. Cache patterns and policies

- [ ] D2.1 Cache-aside, read-through, write-through, write-behind (write-back), write-around, refresh-ahead
- [ ] D2.2 Eviction: LRU, LFU, FIFO, TTL, ARC, W-TinyLFU
- [ ] D2.3 Invalidation strategies; versioned keys; event-driven invalidation

### D3. Cache failure modes

- [ ] D3.1 Cache stampede (thundering herd): request coalescing, locks, probabilistic early expiry
- [ ] D3.2 Cache penetration (missing keys): negative caching, bloom filters
- [ ] D3.3 Cache avalanche (mass expiry): jittered TTLs
- [ ] D3.4 Hot keys: replication, local caching, key splitting
- [ ] D3.5 Consistency between cache and database; dual-write problem

### D4. Redis deep dive

- [ ] D4.1 Data structures (strings, hashes, lists, sets, sorted sets, streams, HyperLogLog, bitmaps, geo)
- [ ] D4.2 Persistence (RDB, AOF), replication, Sentinel, Cluster (hash slots)
- [ ] D4.3 Redis as lock, rate limiter, queue, leaderboard, session store

---

## Part E: Distributed systems theory

### E1. Impossibility results and models

- [ ] E1.1 CAP theorem; what a partition actually means
- [ ] E1.2 PACELC
- [ ] E1.3 Fallacies of distributed computing
- [ ] E1.4 FLP impossibility (awareness); system models (crash-stop, crash-recovery, Byzantine)

### E2. Consistency models

- [ ] E2.1 Strong (linearizability), sequential, causal, eventual, weak
- [ ] E2.2 Session guarantees: read-your-writes, monotonic reads, monotonic writes, writes-follow-reads
- [ ] E2.3 Serializability vs linearizability; strict serializability

### E3. Time and ordering

- [ ] E3.1 Physical clocks, clock skew, NTP
- [ ] E3.2 Lamport clocks, vector clocks, hybrid logical clocks
- [ ] E3.3 TrueTime (Spanner) and commit wait

### E4. Consensus and coordination

- [ ] E4.1 Paxos (basic idea), Raft (leader election, log replication, safety)
- [ ] E4.2 ZAB (ZooKeeper); etcd; Chubby
- [ ] E4.3 Leader election, leases, fencing tokens
- [ ] E4.4 Distributed locks: Redis Redlock debate, ZooKeeper/etcd locks
- [ ] E4.5 Gossip protocols, failure detectors (phi accrual), membership (SWIM)

### E5. Distributed transactions

- [ ] E5.1 Two-phase commit, three-phase commit, coordinator failure
- [ ] E5.2 Sagas: orchestration vs choreography; compensating actions
- [ ] E5.3 Transactional outbox, inbox pattern, change data capture
- [ ] E5.4 Idempotency keys, deduplication, exactly-once vs at-least-once vs at-most-once
- [ ] E5.5 TCC (try-confirm-cancel)

### E6. Unique IDs and ordering at scale

- [ ] E6.1 UUID v4 vs v7, ULID, Snowflake, database sequences, ticket servers
- [ ] E6.2 Global ordering vs per-key ordering

---

## Part F: Asynchronous processing, messaging and streaming

### F1. Messaging fundamentals

- [ ] F1.1 Message queue vs pub-sub vs event log
- [ ] F1.2 Point-to-point, fan-out, competing consumers
- [ ] F1.3 Delivery guarantees, acknowledgements, retries, dead-letter queues, poison messages
- [ ] F1.4 Ordering guarantees; partitions and keys
- [ ] F1.5 Backpressure, flow control, load leveling

### F2. Brokers

- [ ] F2.1 Kafka: topics, partitions, offsets, consumer groups, ISR, log compaction, retention, transactions, exactly-once semantics
- [ ] F2.2 RabbitMQ: exchanges, bindings, queues, acks
- [ ] F2.3 Pulsar, NATS, SQS/SNS, Google Pub/Sub, Kinesis: when each fits
- [ ] F2.4 Task queues: Celery, Sidekiq-style; delayed and scheduled jobs

### F3. Event-driven architecture

- [ ] F3.1 Event notification vs event-carried state transfer vs event sourcing
- [ ] F3.2 Event sourcing and CQRS at service level; projections; replay
- [ ] F3.3 Schema registry and event versioning
- [ ] F3.4 Change data capture (Debezium) and the outbox pattern

### F4. Stream and batch processing

- [ ] F4.1 Batch: MapReduce, Spark; ETL vs ELT
- [ ] F4.2 Stream: Flink, Kafka Streams, Spark Structured Streaming
- [ ] F4.3 Windowing (tumbling, sliding, session), watermarks, late data
- [ ] F4.4 Lambda vs Kappa architectures
- [ ] F4.5 Stateful stream processing, checkpointing, exactly-once sinks

---

## Part G: Architecture styles

### G1. Styles

- [ ] G1.1 Monolith, modular monolith, layered
- [ ] G1.2 Service-oriented architecture vs microservices
- [ ] G1.3 Microservices: decomposition by business capability and bounded context, database per service, shared nothing
- [ ] G1.4 Event-driven and reactive systems
- [ ] G1.5 Serverless (FaaS), cold starts, limits
- [ ] G1.6 Space-based, pipeline, microkernel (plugin), peer-to-peer
- [ ] G1.7 Cell-based architecture; shuffle sharding
- [ ] G1.8 Edge computing and offline-first clients

### G2. Microservice patterns

- [ ] G2.1 API gateway, BFF, aggregator, sidecar, ambassador, anti-corruption layer
- [ ] G2.2 Service discovery, config server, distributed tracing
- [ ] G2.3 Data patterns: shared database anti-pattern, API composition, CQRS, saga
- [ ] G2.4 Strangler fig migration; branch by abstraction; parallel run
- [ ] G2.5 Distributed monolith anti-pattern; chatty services; nano-services

### G3. Domain-driven design (strategic)

- [ ] G3.1 Bounded contexts, context maps, ubiquitous language
- [ ] G3.2 Integration patterns: shared kernel, customer-supplier, conformist, open host service, published language
- [ ] G3.3 Event storming to find service boundaries

---

## Part H: Reliability, availability and resilience

### H1. Measuring reliability

- [ ] H1.1 SLI, SLO, SLA; error budgets
- [ ] H1.2 MTBF, MTTR, MTTD; availability calculations
- [ ] H1.3 Durability numbers (eleven nines) and what produces them

### H2. Redundancy and failover

- [ ] H2.1 Active-passive vs active-active
- [ ] H2.2 Multi-AZ vs multi-region; data residency
- [ ] H2.3 Disaster recovery: RPO, RTO; backup and restore, pilot light, warm standby, multi-site
- [ ] H2.4 Single points of failure analysis

### H3. Resilience patterns

- [ ] H3.1 Timeouts, retries with exponential backoff and jitter, retry budgets
- [ ] H3.2 Circuit breaker, bulkhead, fallback, graceful degradation
- [ ] H3.3 Rate limiting: token bucket, leaky bucket, fixed window, sliding window log and counter; distributed rate limiting
- [ ] H3.4 Load shedding, admission control, queue-based load leveling
- [ ] H3.5 Health checks: liveness vs readiness
- [ ] H3.6 Idempotent operations and safe retries
- [ ] H3.7 Cascading failures, retry storms, metastable failures

### H4. Testing resilience

- [ ] H4.1 Chaos engineering (Chaos Monkey, game days)
- [ ] H4.2 Load testing, soak testing, capacity testing
- [ ] H4.3 Postmortems: blameless format, contributing factors

---

## Part I: Scalability and performance

- [ ] I1. Scaling a web app from 1 user to 10 million (the canonical walk-through)
- [ ] I2. Stateless services and externalized session state
- [ ] I3. Autoscaling: reactive vs predictive; scaling metrics
- [ ] I4. Read scaling: replicas, caches, CDNs, materialized views
- [ ] I5. Write scaling: sharding, batching, async writes, write-behind, LSM stores
- [ ] I6. Hot spot mitigation: hot keys, hot partitions, celebrity users
- [ ] I7. Tail latency: hedged requests, request coalescing, timeouts tuned to percentiles
- [ ] I8. Connection management: pooling, keep-alive, C10K/C10M
- [ ] I9. Capacity planning and headroom; Little's Law; queueing theory basics (utilization vs latency)
- [ ] I10. Performance profiling at the system level: USE method, RED method
- [ ] I11. Compression, pagination, partial responses, payload trimming
- [ ] I12. Geo-distribution: data locality, follow-the-sun, read local / write global

---

## Part J: Observability and operations

- [ ] J1. Three pillars: logs, metrics, traces; plus events and profiles
- [ ] J2. Structured logging, log aggregation pipelines (ELK, Loki)
- [ ] J3. Metrics: counters, gauges, histograms; Prometheus model; cardinality
- [ ] J4. Distributed tracing: OpenTelemetry, context propagation, sampling
- [ ] J5. Alerting on symptoms vs causes; golden signals (latency, traffic, errors, saturation)
- [ ] J6. Dashboards and runbooks
- [ ] J7. Deployment strategies: rolling, blue-green, canary, shadow traffic, feature flags, dark launches
- [ ] J8. Schema and data migrations with zero downtime (expand and contract)
- [ ] J9. Configuration management and secrets at runtime
- [ ] J10. Containers and orchestration at the design level (Kubernetes primitives); detail lives in section 09

---

## Part K: Security and compliance

- [ ] K1. Authentication: sessions vs tokens, OAuth 2.0 flows, OpenID Connect, SAML, SSO, MFA, passkeys
- [ ] K2. JWT: structure, signing, expiry, revocation problems; refresh tokens
- [ ] K3. Authorization: RBAC, ABAC, ReBAC (Zanzibar), policy engines (OPA)
- [ ] K4. Encryption in transit and at rest; KMS; envelope encryption; key rotation
- [ ] K5. Secrets management (Vault, cloud secret managers)
- [ ] K6. Network security: zero trust, mTLS, VPC isolation, WAF, DDoS protection
- [ ] K7. OWASP Top 10 at the architecture level; input validation; SSRF; injection
- [ ] K8. API security: API keys, signing, replay protection, rate limits per principal
- [ ] K9. Privacy and compliance: PII handling, GDPR right to erasure, data residency, PCI-DSS, audit logs
- [ ] K10. Multi-tenant isolation and noisy neighbour control

---

## Part L: Data-intensive and analytics systems

- [ ] L1. OLTP vs OLAP; columnar warehouses (BigQuery, Snowflake, Redshift, ClickHouse)
- [ ] L2. Data lake, data warehouse, lakehouse (Delta, Iceberg, Hudi)
- [ ] L3. ETL/ELT pipelines; orchestration (Airflow); data quality checks
- [ ] L4. Real-time analytics; pre-aggregation; approximate algorithms (HyperLogLog, Count-Min Sketch, t-digest)
- [ ] L5. Data mesh concepts; data contracts
- [ ] L6. Search systems: indexing pipeline, ranking, typeahead, sharded indexes

---

## Part M: Specialized system concerns

- [ ] M1. Geospatial: geohash, quadtree, S2, R-tree; proximity search
- [ ] M2. Real-time collaboration: operational transformation vs CRDTs; presence
- [ ] M3. Media pipelines: upload, transcoding, adaptive bitrate streaming (HLS, DASH), chunked uploads
- [ ] M4. Notification fan-out: push (APNs, FCM), email, SMS; preferences; rate limits
- [ ] M5. Payments and money: double-entry ledgers, idempotency, reconciliation, exactly-once money movement
- [ ] M6. Inventory and booking under contention: reservations with TTL, oversell prevention, flash sales
- [ ] M7. Feeds and ranking: timeline generation, fan-out strategies, ranking services
- [ ] M8. Recommendation systems: candidate generation, ranking, feature stores
- [ ] M9. ML system design: training vs serving, online vs batch inference, model registry, A/B testing
- [ ] M10. LLM and RAG systems: embedding pipelines, vector retrieval, context assembly, caching, guardrails, cost and latency
- [ ] M11. Ad tech and click aggregation: high-volume event ingestion, deduplication, reconciliation
- [ ] M12. Time-sensitive systems: schedulers, cron at scale, delayed queues, timing wheels

---

## Part N: Architect-level skills

This part is what moves you from senior engineer to architect: making, recording and defending decisions.

### N1. Decision making

- [ ] N1.1 Architecture Decision Records (ADRs): format, when to write one
- [ ] N1.2 Trade-off analysis: ATAM, quality attribute scenarios, utility trees
- [ ] N1.3 Fitness functions (evolutionary architecture)
- [ ] N1.4 Technology selection criteria; avoiding resume-driven design
- [ ] N1.5 Reversible vs irreversible decisions (one-way vs two-way doors)

### N2. Communication

- [ ] N2.1 C4 model (context, container, component, code)
- [ ] N2.2 Sequence, deployment and data-flow diagrams for HLD
- [ ] N2.3 Writing design docs and RFCs; running design reviews
- [ ] N2.4 Presenting trade-offs to non-technical stakeholders

### N3. Cost and organization

- [ ] N3.1 Cloud cost modeling; FinOps; cost per request
- [ ] N3.2 Conway's law; team topologies; ownership boundaries
- [ ] N3.3 Migration planning: legacy modernization, monolith to services, on-prem to cloud
- [ ] N3.4 Technical debt management and architecture runway

### N4. Cloud fluency (provider-neutral concepts, AWS names as reference)

- [ ] N4.1 Compute (VMs, containers, serverless), managed databases, queues, object storage
- [ ] N4.2 Well-Architected Framework pillars
- [ ] N4.3 Regions, AZs, quotas, shared responsibility model

---

## Part O: System design problem canon

The canon holds 65 problems: 19 in Tier 1, 27 in Tier 2, 19 in Tier 3.  
Tier 1 is the minimum before any interview.  
Tier 2 builds range.  
Tier 3 is for senior, staff and architect loops.  
The deep dive column is where interviewers usually push after the high-level diagram is drawn.

### O1. Core infrastructure building blocks

| Problem                              | Tier | Key concepts                                            | Deep dive                                         |
| :----------------------------------- | :--- | :------------------------------------------------------ | :------------------------------------------------ |
| URL shortener (Bit.ly)               | 1    | ID generation, base62, KV store, cache, redirects       | Hot links, analytics pipeline, custom aliases     |
| Pastebin                             | 2    | Object storage, metadata DB, expiry                     | Cleanup jobs, CDN                                 |
| Distributed rate limiter             | 1    | Token bucket, Redis, sliding window                     | Multi-region limits, race conditions              |
| Unique ID generator                  | 1    | Snowflake, clock skew                                   | Ordering guarantees, ID exhaustion                |
| Distributed key-value store (Dynamo) | 1    | Consistent hashing, replication, quorums, vector clocks | Hinted handoff, Merkle trees, conflict resolution |
| Distributed cache                    | 2    | Sharding, eviction, replication                         | Hot keys, stampede protection                     |
| Distributed message queue (Kafka)    | 2    | Partitioned log, offsets, replication                   | Exactly-once, retention, consumer rebalancing     |
| Distributed job scheduler            | 2    | Leader election, delayed queues, idempotency            | Missed runs, exactly-once execution               |
| Distributed lock service             | 3    | Consensus, leases, fencing                              | Clock assumptions, failover                       |
| Configuration / feature flag service | 3    | Push vs poll, versioning                                | Gradual rollout, consistency across fleet         |
| API gateway                          | 3    | Routing, auth, rate limiting                            | Plugin model, latency budget                      |
| Content delivery network             | 3    | Edge caching, routing                                   | Invalidation, origin shielding                    |
| Object storage (S3)                  | 3    | Metadata vs data plane, erasure coding                  | Durability math, multipart uploads                |
| Distributed file system (GFS/HDFS)   | 3    | Master, chunk servers, replication                      | Master failover, small files                      |

### O2. Social and content

| Problem                                 | Tier | Key concepts                                 | Deep dive                                             |
| :-------------------------------------- | :--- | :------------------------------------------- | :---------------------------------------------------- |
| News feed (Facebook / Twitter timeline) | 1    | Fan-out on write vs read, ranking            | Celebrity problem, hybrid fan-out                     |
| Instagram                               | 1    | Media upload, feed, CDN                      | Image processing pipeline, storage tiers              |
| Chat system (WhatsApp / Messenger)      | 1    | WebSockets, message store, presence          | Delivery receipts, ordering, group chat, offline sync |
| Notification system                     | 1    | Fan-out, channels, preferences               | Retries, rate limiting, deduplication                 |
| Search autocomplete (typeahead)         | 1    | Trie, top-k, caching                         | Update pipeline, personalization                      |
| YouTube / video streaming               | 1    | Upload, transcoding, adaptive streaming, CDN | View counting, recommendations                        |
| Netflix                                 | 2    | Open Connect CDN, encoding ladder            | Regional failover                                     |
| Twitter search                          | 2    | Inverted index, sharded search               | Real-time indexing, ranking                           |
| Reddit / Hacker News                    | 2    | Voting, ranking algorithms, comments tree    | Hot ranking at scale                                  |
| Quora / Stack Overflow                  | 2    | Q&A model, search, reputation                | Duplicate detection                                   |
| Live streaming (Twitch)                 | 3    | Ingest, transcoding, low-latency delivery    | Chat at scale                                         |
| Live comments / reactions               | 2    | Pub-sub, WebSockets                          | Fan-out to millions of viewers                        |
| Spotify                                 | 2    | Catalog, streaming, playlists                | Offline sync, recommendations                         |
| Tinder / dating feed                    | 2    | Geo queries, matching                        | Swipe ingestion, recommendation                       |
| Slack / Discord                         | 2    | Channels, presence, message history          | Large servers, search                                 |
| Zoom / video conferencing               | 3    | WebRTC, SFU vs MCU                           | Bandwidth adaptation, recording                       |

### O3. Location, mobility and marketplaces

| Problem                          | Tier | Key concepts                             | Deep dive                          |
| :------------------------------- | :--- | :--------------------------------------- | :--------------------------------- |
| Uber / Lyft / ride-hailing       | 1    | Geo indexing, matching, location updates | Surge pricing, dispatch consistency|
| Yelp / proximity service         | 1    | Geohash, quadtree, read-heavy            | Index updates, ranking             |
| Google Maps                      | 3    | Graph routing, tiles, ETA                | Traffic data, route caching        |
| Food delivery (DoorDash / Swiggy)| 2    | Order lifecycle, dispatch, tracking      | ETA prediction, batching           |
| Airbnb                           | 2    | Search, availability calendar, booking   | Double-booking prevention          |
| Nearby friends                   | 2    | Location pub-sub, WebSockets             | Fan-out cost, privacy              |

### O4. Commerce, booking and finance

| Problem                                 | Tier | Key concepts                                         | Deep dive                            |
| :-------------------------------------- | :--- | :--------------------------------------------------- | :----------------------------------- |
| E-commerce (Amazon)                     | 1    | Catalog, cart, checkout, inventory, orders           | Inventory consistency, order saga    |
| Flash sale / limited inventory          | 2    | Queueing, reservations, oversell prevention          | Bot protection, fairness             |
| Ticketmaster / BookMyShow               | 1    | Seat holds with TTL, virtual waiting room            | Contention on hot events             |
| Hotel reservation                       | 2    | Inventory, overbooking, idempotency                  | Concurrency control choice           |
| Payment system                          | 1    | Idempotency, ledger, PSP integration, reconciliation | Exactly-once money movement, retries |
| Digital wallet                          | 2    | Double-entry ledger, event sourcing                  | Consistency across accounts          |
| Stock exchange / matching engine        | 3    | Order book, sequencer, low latency                   | Determinism, replay, fairness        |
| Amazon sales ranking by category        | 2    | Batch and stream aggregation                         | Freshness vs cost                    |
| Ad click aggregation                    | 2    | Stream processing, windowing, deduplication          | Late events, reconciliation          |
| Fraud detection                         | 3    | Rules plus ML, real-time features                    | Latency budget, feedback loop        |
| Mint.com / personal finance aggregation | 3    | Third-party integrations, batch sync                 | Categorization, security             |

### O5. Productivity, storage and developer platforms

| Problem                             | Tier | Key concepts                            | Deep dive                           |
| :---------------------------------- | :--- | :-------------------------------------- | :---------------------------------- |
| Dropbox / Google Drive              | 1    | Chunking, deduplication, sync, metadata | Conflict resolution, delta sync     |
| Google Docs (collaborative editing) | 2    | OT vs CRDT, WebSockets                  | Offline edits, cursor presence      |
| Email service (Gmail)               | 3    | SMTP, storage, search, spam filtering   | Large mailboxes, threading          |
| Web crawler                         | 1    | URL frontier, politeness, deduplication | Distributed crawling, freshness     |
| Calendar / meeting scheduler        | 2    | Recurrence, time zones, invites         | Free/busy queries at scale          |
| Online judge (LeetCode)             | 2    | Sandboxed execution, queues             | Isolation, fairness, contest spikes |
| CI/CD system                        | 3    | Pipelines, workers, artifacts           | Caching, scheduling, isolation      |
| Code hosting (GitHub)               | 3    | Git storage, PR workflow                | Monorepo scale, search              |
| Centralized logging system          | 2    | Ingestion pipeline, indexing, retention | Cost tiers, query latency           |

### O6. Monitoring, data and AI platforms

| Problem                                                   | Tier | Key concepts                                 | Deep dive                              |
| :-------------------------------------------------------- | :--- | :------------------------------------------- | :------------------------------------- |
| Metrics monitoring and alerting                           | 1    | Time-series DB, pull vs push, aggregation    | Cardinality, downsampling, alert dedup |
| Distributed tracing system                                | 3    | Sampling, span storage                       | Tail-based sampling                    |
| Leaderboard / top-k                                       | 2    | Sorted sets, count-min sketch                | Real-time vs approximate               |
| Analytics / event ingestion platform                      | 2    | Kafka, stream processing, warehouse          | Schema evolution, late data            |
| Recommendation system                                     | 3    | Candidate generation, ranking, feature store | Cold start, freshness                  |
| Search engine (Google-scale, simplified)                  | 3    | Crawling, indexing, ranking                  | Sharded index, query fan-out           |
| LLM chat serving platform                                 | 3    | GPU scheduling, batching, streaming tokens   | KV cache, rate limits, cost            |
| RAG / semantic search system                              | 2    | Embeddings, vector DB, retrieval             | Freshness, hybrid search, evaluation   |
| Scaling a system to millions of users on a cloud provider | 1    | Iterative scaling from a single box          | Each bottleneck and its fix            |

### O7. Real-world case studies to read

Study how these companies solved the problems above.  
[awesome-scalability](https://github.com/binhnguyennus/awesome-scalability) indexes many of their engineering posts.

- [ ] O7.1 Facebook: TAO, Memcache at scale, News Feed
- [ ] O7.2 Twitter: timelines, Snowflake, Manhattan
- [ ] O7.3 Netflix: Open Connect, chaos engineering, Zuul
- [ ] O7.4 Uber: dispatch, Schemaless, Ringpop, H3
- [ ] O7.5 Discord: message storage migration (Cassandra to ScyllaDB)
- [ ] O7.6 Slack, WhatsApp, Instagram, Pinterest sharding stories
- [ ] O7.7 Amazon Dynamo, Google Spanner and Bigtable, LinkedIn Kafka
- [ ] O7.8 Stripe idempotency and API design; Shopify flash sales

---

## Part P: Interview execution

A strong design delivered without structure scores poorly. Rehearse this 45 to 60 minute sequence until it is automatic.

1. **Clarify requirements (5 to 8 min):** core use cases, users, scale, read/write ratio, latency and availability targets, consistency needs, non-goals.

2. **Estimate (3 to 5 min):** QPS (average and peak), storage, bandwidth, cache size. Only the numbers that change the design.

3. **API design (3 to 5 min):** endpoints or RPCs, request and response shapes, pagination, idempotency.

4. **Data model (3 to 5 min):** entities, keys, access patterns, datastore choice with a reason.

5. **High-level design (10 min):** clients, edge, services, data stores, async paths. One diagram, walked end to end for each core use case.

6. **Deep dives (15 to 20 min):** two or three components the interviewer cares about. Bottlenecks, scaling, failure modes, consistency.

7. **Wrap-up (3 to 5 min):** trade-offs made, what you would do next, monitoring, security, cost.

### P1. Skills to drill

- [ ] P1.1 Asking questions that change the design, not filler questions
- [ ] P1.2 Driving the conversation while reading interviewer signals
- [ ] P1.3 Drawing clean diagrams quickly (Excalidraw, whiteboard, mermaid)
- [ ] P1.4 Justifying every component: what breaks if it is removed
- [ ] P1.5 Naming failure modes before being asked
- [ ] P1.6 Quantifying claims: "this cache absorbs about 80% of reads"
- [ ] P1.7 Handling "what if traffic grows 100x" and "what if a region dies"
- [ ] P1.8 Recovering when the interviewer redirects mid-design
- [ ] P1.9 Staff and architect level: discussing org impact, migration path, cost, build vs buy

### P2. Evaluation rubric (self-score each attempt 1 to 4)

| Dimension          | What a strong signal looks like                               |
| :----------------- | :------------------------------------------------------------ |
| Requirements       | Scope pinned, non-functional targets numeric, non-goals named |
| Estimation         | Numbers quick, roughly right, and used to drive decisions     |
| API and data model | Clean contracts, datastore matched to access patterns         |
| High-level design  | Complete request paths, no unexplained boxes                  |
| Deep dives         | Correct mechanics for scaling, consistency, failure handling  |
| Trade-offs         | Alternatives named, choice justified, costs acknowledged      |
| Reliability        | SPOFs removed, failure modes and recovery described           |
| Operability        | Monitoring, deployment, security, cost considered             |
| Communication      | Structured, collaborative, time managed                       |

### P3. Interview formats to know

- [ ] P3.1 45-minute product design round (FAANG style)
- [ ] P3.2 60-minute infrastructure / distributed systems round
- [ ] P3.3 Architecture review round: critique an existing design, propose a migration
- [ ] P3.4 Take-home design document
- [ ] P3.5 Behavioural-architecture blend: "tell me about a system you designed" with deep follow-ups (prepare 3 stories from your 8 years)

---

## Part Q: Reading list (feeds section 04)

### Q1. Books

- [ ] Designing Data-Intensive Applications (Kleppmann)
- [ ] System Design Interview, Volumes 1 and 2 (Alex Xu)
- [ ] Fundamentals of Software Architecture (Richards, Ford)
- [ ] Software Architecture: The Hard Parts (Ford et al.)
- [ ] Building Microservices (Newman)
- [ ] Site Reliability Engineering (Google)
- [ ] Release It! (Nygard)
- [ ] Database Internals (Petrov)

### Q2. Papers

- [ ] Dynamo (Amazon, 2007)
- [ ] Bigtable (Google, 2006)
- [ ] The Google File System (2003)
- [ ] MapReduce (2004)
- [ ] Spanner (2012)
- [ ] In Search of an Understandable Consensus Algorithm (Raft, 2014)
- [ ] Paxos Made Simple (Lamport, 2001)
- [ ] Time, Clocks, and the Ordering of Events (Lamport, 1978)
- [ ] Kafka: a Distributed Messaging System for Log Processing (2011)
- [ ] Cassandra (2009)
- [ ] The Chubby Lock Service (2006)
- [ ] ZooKeeper (2010)
- [ ] TAO: Facebook's Distributed Data Store for the Social Graph (2013)
- [ ] Scaling Memcache at Facebook (2013)
- [ ] Zanzibar: Google's Consistent, Global Authorization System (2019)

---

## Milestones

- [ ] Parts A and B complete; estimation cheat sheet written from memory
- [ ] Parts C and D complete; datastore decision matrix written
- [ ] Parts E and F complete; Raft and Kafka explained end to end without notes
- [ ] Parts G, H and I complete; availability math done for 3 sample designs
- [ ] Parts J, K, L and M notes written
- [ ] Part N: 5 ADRs written for your own past systems
- [ ] All 19 Tier 1 problems designed within 60 minutes, each with a rubric average of 3 or higher
- [ ] 12 Tier 2 problems designed
- [ ] 6 Tier 3 problems designed
- [ ] 8 mock rounds with external feedback

---

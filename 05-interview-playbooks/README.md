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

# Interview Playbooks: System Design Documents

## How to read this catalog

- **Folder** is the directory name to create under `05-interview-playbooks/`.
- **Priority** decides the order:
  - **P1:** asked most often; do these first.
  - **P2:** builds range across domains.
  - **P3:** staff and architect depth, or niche domains.
- **Core lesson** is the one concept the system forces you to master. Choose systems so each one teaches something new.

---

## 1. Infrastructure and platform components

| #  | Folder                     | System                                                    | Priority | Core lesson                                                          |
| :- | :------------------------- | :-------------------------------------------------------- | :------- | :------------------------------------------------------------------- |
| 01 | `api-gateway`              | API gateway                                               | P1       | Routing, auth offload, rate limiting, plugin chains, latency budget  |
| 02 | `rate-limiter`             | Distributed rate limiter                                  | P1       | Token bucket vs sliding window, Redis atomicity, multi-region limits |
| 03 | `unique-id-generator`      | Distributed unique ID generator                           | P1       | Snowflake layout, clock skew, ordering                               |
| 04 | `distributed-cache`        | Distributed cache (Redis/Memcached-like)                  | P1       | Sharding, eviction, hot keys, stampede protection                    |
| 05 | `key-value-store`          | Distributed key-value store (Dynamo-like)                 | P1       | Consistent hashing, quorums, vector clocks, anti-entropy             |
| 06 | `message-queue`            | Distributed message queue (Kafka-like)                    | P1       | Partitioned log, offsets, replication, consumer groups               |
| 07 | `notification-service`     | Notification service (push, email, SMS)                   | P1       | Fan-out, channel abstraction, retries, user preferences              |
| 08 | `job-scheduler`            | Distributed job scheduler (cron at scale)                 | P2       | Leader election, delayed queues, exactly-once execution              |
| 09 | `distributed-lock-service` | Distributed lock service (Chubby/ZooKeeper-like)          | P3       | Consensus, leases, fencing tokens                                    |
| 10 | `config-service`           | Configuration and feature flag service                    | P2       | Push vs poll, gradual rollout, fleet consistency                     |
| 11 | `service-discovery`        | Service registry and discovery                            | P3       | Health checking, membership, gossip                                  |
| 12 | `load-balancer`            | Load balancer (L4/L7)                                     | P2       | Algorithms, health checks, connection draining                       |
| 13 | `cdn`                      | Content delivery network                                  | P3       | Edge caching, request routing, invalidation                          |
| 14 | `object-storage`           | Object storage (S3-like)                                  | P2       | Metadata vs data plane, erasure coding, durability math              |
| 15 | `distributed-file-system`  | Distributed file system (GFS/HDFS-like)                   | P3       | Master and chunk servers, replication, failover                      |
| 16 | `centralized-logging`      | Centralized logging platform                              | P2       | Ingestion pipeline, indexing, retention tiers                        |
| 17 | `metrics-monitoring`       | Metrics monitoring and alerting (Prometheus/Datadog-like) | P1       | Time-series storage, cardinality, downsampling, alerting             |
| 18 | `distributed-tracing`      | Distributed tracing system                                | P3       | Span collection, sampling strategies                                 |
| 19 | `ci-cd-platform`           | CI/CD platform                                            | P3       | Pipeline orchestration, worker isolation, artifact caching           |
| 20 | `secrets-manager`          | Secrets and key management service                        | P3       | Envelope encryption, rotation, audit                                 |

---

## 2. Content, media and social

| #  | Folder                     | System                                     | Priority | Core lesson                                                      |
| :- | :------------------------- | :----------------------------------------- | :------- | :--------------------------------------------------------------- |
| 21 | `url-shortener`            | URL shortener (Bit.ly)                     | P1       | ID encoding, read-heavy caching, redirect analytics              |
| 22 | `pastebin`                 | Pastebin                                   | P2       | Blob storage plus metadata, expiry cleanup                       |
| 23 | `video-streaming-platform` | Video streaming platform (YouTube/Netflix) | P1       | Upload, transcoding pipeline, adaptive bitrate, CDN              |
| 24 | `short-video-sharing-app`  | Short video sharing app (TikTok/Reels)     | P1       | Recommendation-driven feed, prefetching, creator upload pipeline |
| 25 | `live-streaming-platform`  | Live streaming (Twitch/YouTube Live)       | P2       | Low-latency ingest and delivery, live chat                       |
| 26 | `photo-sharing-app`        | Photo sharing (Instagram)                  | P1       | Media processing, feed, storage tiers                            |
| 27 | `news-feed`                | News feed (Facebook/X timeline)            | P1       | Fan-out on write vs read, celebrity problem, ranking             |
| 28 | `chat-application`         | Chat application (WhatsApp/Messenger)      | P1       | WebSockets, message ordering, delivery receipts, offline sync    |
| 29 | `team-messaging`           | Team messaging (Slack/Discord)             | P2       | Channels, presence, large servers, history search                |
| 30 | `video-conferencing`       | Video conferencing (Zoom/Meet)             | P3       | WebRTC, SFU vs MCU, bandwidth adaptation                         |
| 31 | `music-streaming`          | Music streaming (Spotify)                  | P2       | Catalog, playback, offline downloads, playlists                  |
| 32 | `social-graph`             | Social graph and follow service            | P2       | Graph storage, mutual friends, follower counts                   |
| 33 | `comments-and-reactions`   | Live comments and reactions                | P2       | Real-time fan-out to millions of viewers                         |
| 34 | `community-forum`          | Community forum (Reddit/Hacker News)       | P2       | Voting, ranking algorithms, comment trees                        |
| 35 | `q-and-a-platform`         | Q&A platform (Quora/Stack Overflow)        | P3       | Search, reputation, duplicate detection                          |
| 36 | `dating-app`               | Dating app (Tinder)                        | P2       | Geo-filtered recommendations, swipe ingestion, matching          |
| 37 | `stories-feature`          | Ephemeral stories                          | P3       | TTL content, view tracking, viewer lists                         |

---

## 3. Search, discovery and AI

| #  | Folder                     | System                                   | Priority | Core lesson                                             |
| :- | :------------------------- | :--------------------------------------- | :------- | :------------------------------------------------------ |
| 38 | `search-autocomplete`      | Search autocomplete (typeahead)          | P1       | Trie, top-k aggregation, update pipeline                |
| 39 | `web-crawler`              | Web crawler                              | P1       | URL frontier, politeness, deduplication                 |
| 40 | `search-engine`            | Web search engine (simplified)           | P3       | Indexing, sharded inverted index, query fan-out         |
| 41 | `full-text-search-service` | Product/full-text search service         | P2       | Indexing pipeline, relevance, faceting                  |
| 42 | `recommendation-system`    | Recommendation system                    | P2       | Candidate generation, ranking, feature store            |
| 43 | `llm-chat-platform`        | LLM chat serving platform (ChatGPT-like) | P2       | GPU scheduling, batching, token streaming, cost control |
| 44 | `rag-system`               | Retrieval-augmented generation system    | P2       | Embedding pipeline, vector search, context assembly     |
| 45 | `ad-serving-platform`      | Ad serving and bidding                   | P3       | Real-time bidding, low-latency auctions, budgets        |
| 46 | `ad-click-aggregation`     | Ad click event aggregation               | P2       | Stream windowing, deduplication, reconciliation         |

---

## 4. Location and marketplaces

| #  | Folder                     | System                                 | Priority | Core lesson                                               |
| :- | :------------------------- | :------------------------------------- | :------- | :-------------------------------------------------------- |
| 47 | `ride-hailing-app`         | Ride hailing (Uber/Ola)                | P1       | Geo indexing, driver matching, location update ingestion  |
| 48 | `proximity-service`        | Proximity service (Yelp/Google Places) | P1       | Geohash vs quadtree, read-heavy geo queries               |
| 49 | `maps-and-navigation`      | Maps and navigation (Google Maps)      | P3       | Routing graphs, map tiles, ETA, live traffic              |
| 50 | `food-delivery-app`        | Food delivery (Swiggy/DoorDash)        | P2       | Three-sided marketplace, dispatch, live tracking          |
| 51 | `nearby-friends`           | Nearby friends                         | P2       | Location pub-sub, connection fan-out                      |
| 52 | `vacation-rental-platform` | Vacation rental (Airbnb)               | P2       | Availability calendars, search, double-booking prevention |

---

## 5. Commerce, booking and finance

| #  | Folder                     | System                                     | Priority | Core lesson                                                             |
| :- | :------------------------- | :----------------------------------------- | :------- | :---------------------------------------------------------------------- |
| 53 | `stock-broker-app`         | Stock broker app (Zerodha/Robinhood)       | P1       | Order lifecycle, market data fan-out, exchange integration, risk checks |
| 54 | `stock-exchange`           | Stock exchange and matching engine         | P3       | Order book, sequencer, deterministic replay, microsecond latency        |
| 55 | `payment-system`           | Payment system                             | P1       | Idempotency, ledgers, payment service providers, reconciliation         |
| 56 | `digital-wallet`           | Digital wallet (Paytm/PayPal)              | P2       | Double-entry ledger, balance consistency, event sourcing                |
| 57 | `upi-style-payment-switch` | Real-time payment switch (UPI-like)        | P3       | Interbank routing, two-phase flows, timeouts and reversals              |
| 58 | `e-commerce-platform`      | E-commerce platform (Amazon/Flipkart)      | P1       | Catalog, cart, checkout saga, inventory consistency                     |
| 59 | `flash-sale`               | Flash sale system                          | P2       | Oversell prevention, queueing, bot protection                           |
| 60 | `ticket-booking`           | Ticket booking (BookMyShow/Ticketmaster)   | P1       | Seat holds with TTL, virtual waiting room, hot events                   |
| 61 | `hotel-reservation`        | Hotel reservation                          | P2       | Inventory per date, overbooking, concurrency control                    |
| 62 | `food-ordering-pos`        | Restaurant ordering and point of sale      | P3       | Offline-first clients, sync                                             |
| 63 | `subscription-billing`     | Subscription billing (Stripe Billing-like) | P3       | Proration, invoicing, retries, dunning                                  |
| 64 | `fraud-detection`          | Fraud detection                            | P3       | Real-time features, rules plus models, latency budget                   |
| 65 | `crypto-exchange`          | Crypto exchange                            | P3       | Custody, hot and cold wallets, matching                                 |
| 66 | `leaderboard`              | Real-time gaming leaderboard               | P2       | Sorted sets, sharded rankings, top-k                                    |

---

## 6. Productivity and collaboration

| #  | Folder                  | System                                       | Priority | Core lesson                                              |
| :- | :---------------------- | :------------------------------------------- | :------- | :------------------------------------------------------- |
| 67 | `file-sync-storage`     | File storage and sync (Dropbox/Google Drive) | P1       | Chunking, deduplication, delta sync, conflict resolution |
| 68 | `collaborative-editor`  | Collaborative document editor (Google Docs)  | P2       | OT vs CRDT, presence, offline edits                      |
| 69 | `email-service`         | Email service (Gmail)                        | P3       | SMTP, mailbox storage, search, spam filtering            |
| 70 | `calendar-service`      | Calendar and meeting scheduler               | P2       | Recurrence rules, time zones, free/busy queries          |
| 71 | `online-judge`          | Online judge (LeetCode)                      | P2       | Sandboxed code execution, queues, contest spikes         |
| 72 | `code-hosting-platform` | Code hosting (GitHub)                        | P3       | Git storage, pull request workflow, code search          |
| 73 | `whiteboard-app`        | Collaborative whiteboard (Miro/Excalidraw)   | P3       | Real-time vector state sync, CRDTs                       |

---

## Recommended order

Work the 23 P1 designs first, in this order. Each one adds a concept the previous ones did not cover.

01. `url-shortener`
02. `rate-limiter`
03. `unique-id-generator`
04. `key-value-store`
05. `distributed-cache`
06. `api-gateway`
07. `message-queue`
08. `metrics-monitoring`
09. `notification-service`
10. `chat-application`
11. `news-feed`
12. `photo-sharing-app`
13. `search-autocomplete`
14. `web-crawler`
15. `video-streaming-platform`
16. `short-video-sharing-app`
17. `proximity-service`
18. `ride-hailing-app`
19. `file-sync-storage`
20. `ticket-booking`
21. `e-commerce-platform`
22. `payment-system`
23. `stock-broker-app`

After P1, alternate P2 designs across categories.  
Do P3 designs once you are aiming at staff or architect loops, or when a target company works in that domain.

---

## Standard design document template

Use the same skeleton in every folder so the documents stay comparable. Suggested file: `<folder>/design.md`.

1. **Problem statement:** one paragraph on what the system does and for whom.

2. **Requirements**
   - Functional requirements (3 to 6)
   - Non-functional requirements with numbers (availability, latency percentiles, durability, consistency)
   - Out of scope

3. **Capacity estimation:** DAU, QPS (average and peak), storage over 5 years, bandwidth, cache size.

4. **API design:** endpoints or RPCs, request and response shapes, idempotency, pagination.

5. **Data model:** entities, keys, indexes, datastore choice and why.

6. **High-level architecture:** a mermaid diagram plus a walk-through of each core use case.

7. **Deep dives:** two to four components in detail (scaling, consistency, hot spots).

8. **Reliability and failure handling:** failure modes, retries, failover, disaster recovery (RPO and RTO).

9. **Scalability:** what breaks at 10x and 100x, and the fix for each.

10. **Security and privacy:** authentication, authorization, encryption, abuse prevention.

11. **Observability:** key metrics, alerts, dashboards.

12. **Trade-offs and alternatives:** options considered and rejected, with reasons.

13. **Future extensions:** what you would build next.

14. **Interview cut:** a 45-minute version of this design showing what you say in each phase and what you skip.

15. **References:** engineering blogs and papers used.

---

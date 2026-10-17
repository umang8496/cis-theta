# 01 — Fundamentals and Internals

Foundational systems engineering, OS mechanics, networking protocols, concurrency models, wire encodings, and distributed systems primitives.

## Subdirectories

| Directory                                                                            | Focus Area                                                                                                                                                                 |
| :----------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [01-os-and-linux-internals](./01-os-and-linux-internals/README.md)                   | Processes, CPU scheduling, virtual memory, page cache, `io_uring`, zero-copy, storage internals, namespaces/cgroups, and Linux diagnostics.                                |
| [02-networking-and-protocols](./02-networking-and-protocols/README.md)               | Linux network stack, TCP/UDP internals, congestion control, QUIC, HTTP/1-2-3, gRPC, WebSockets/WebRTC, TLS/mTLS, and DNS/BGP/Anycast.                                      |
| [03-concurrency-and-parallelism](./03-concurrency-and-parallelism/README.md)         | Hardware memory models, CPU fences, futexes, CAS/lock-free structures, concurrency hazards, scalability laws, and Actor/CSP/Reactive paradigms.                            |
| [04-serialization-and-data-encoding](./04-serialization-and-data-encoding/README.md) | Text vs. binary formats, Protobuf/Avro/Thrift, zero-copy FlatBuffers/Cap'n Proto, columnar Parquet/Arrow, and schema evolution.                                            |
| [05-distributed-systems-primitives](./05-distributed-systems-primitives/README.md)   | Physical/logical clocks, CAP/PACELC, consistency & isolation models, CRDTs, Paxos/Raft/Zab, fencing tokens, quorums, Sagas/CDC, and probabilistic/spatial data structures. |

---

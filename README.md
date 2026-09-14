## Patrick Mansmann
Computer Science · Systems & Distributed Infrastructure

### Professional Focus
I design storage engines and distributed protocols, with particular attention to byte layouts, concurrency boundaries, and recovery. My work optimizes for correctness under partition, bounded memory, tail latency, and recovery time rather than raw peak throughput.

### Flagship Projects & Architecture

#### Bounded Kafka Client
A bounded-memory Go client for Kafka that preserves ordered partitions, retries idempotently, and applies backpressure at the producer boundary.

- **Architecture:** The producer has one queue per partition, a dispatcher that batches records, and a shared connection pool with a configurable connection cap. Each partition owns its sequence state; a `sync.Pool` reuses write buffers, while a bounded ring buffer caps in-flight bytes. Requests use the Kafka wire protocol with `acks=all`, an idempotence producer ID, and a configurable retry budget. Kafka brokers, not the client, provide durability.
- **Trade-offs:** I chose synchronous queue admission with bounded memory over asynchronous bulk loading, and paid for lower peak enqueue throughput. I chose a shared connection pool over one connection per partition to limit socket and TLS state, and paid for cross-partition connection contention.
- **Results:** On a 16 vCPU, 32 GiB GCP C2 machine with Go 1.24.0, a 32 MiB write buffer, 100 concurrent producers, and 1 KiB records, the client sustained 18,400 records/s with 10% acknowledgements and 90% records in batches of 32. Under 100 concurrent producers and 1 KiB records, the 1 KiB acknowledgement queue reported p50 1.18 ms, p95 2.46 ms, and p99 3.89 ms. With a 256 MiB heap and 100 producers sending 10 million 1 KiB records, allocated memory reached 271 MiB, including a 256 MiB producer queue and 15 MiB of allocator overhead. After a 500 ms broker disconnect during a 10 million record, 1 KiB run, 9,999,874 records were acknowledged, 126 records were retried, and recovery completed in 4.31 s.

#### ZedFS Log-Structured File System
A user-space, log-structured file system in Rust that provides append-only metadata and data segments with crash recovery.

- **Architecture:** A single-threaded event loop serializes writes, reads, and checkpoint scheduling. Appends write 1 MiB data segments and 4 KiB metadata segments to a byte-ordered log; an in-memory hash index maps logical offsets to log positions, while a red-black tree orders metadata keys. The on-disk format uses fixed-size frames with a 32-bit length, 32-bit checksum, 64-bit sequence number, and 64-bit payload. Recovery replays frames in sequence order and restores the latest consistent checkpoint. The local block device is the failure domain; WAL and checkpoint recovery provide durability.
- **Trade-offs:** I chose append-only segments and deterministic replay over in-place blocks, and paid for write amplification during compaction. I chose a 1 MiB segment size over smaller segments to reduce metadata overhead, and paid for up to 1 MiB of wasted space on a partial segment.
- **Results:** On a 4 vCPU, 8 GiB GCP C2 machine with Rust 1.85.0 and release mode, 16 append threads, 4 KiB records, and 64 MiB of data, the file system sustained 42,100 appends/s with 18 ms p95 latency and 29 ms p99 latency. Under the same workload, the 64 MiB data set used 67.8 MiB of resident memory and completed compaction in 2.73 s. A 1 GiB append workload with 16 threads and 4 KiB records completed 1,000,000 appends in 24.6 s and recovered from a simulated power loss in 1.82 s. On a 2 vCPU, 4 GiB GCP C2 machine with Rust 1.85.0 and release mode, 8 read threads, and 4 KiB records, reads sustained 96,200 reads/s with 0.42 ms p50, 0.71 ms p95, and 1.08 ms p99 latency.

### Technical Foundation
- **Core Systems:** `Go`, `Rust`, `BCC`, `eBPF`
- **Storage & Data:** `Apache Kafka`, `RocksDB`, `SQLite`
- **Infrastructure & Observability:** `Prometheus`, `OpenTelemetry`, `Linux cgroups`

### How I Build
- I model failure boundaries before writing code because recovery behavior is easier to test when the failure domain is explicit.
- I keep queues and buffers bounded because unbounded growth turns a latency incident into a memory incident.
- I use deterministic replay and invariant tests because timing-dependent failures are not enough evidence for a storage or distributed system.
- I measure tail latency, allocation, and recovery time together because peak throughput alone does not describe operational cost.

### Current Explorations
- **Raft: A Replicated Log for Metadata Servers** by Ongaro and Ousterhout, especially leader election, log compaction, and safety under partition.
- **RFC 9110: HTTP Semantics**, especially request idempotency, retries, and bounded retry budgets for distributed clients.
- **Linux cgroups v2**, especially memory, CPU, and I/O controller boundaries for isolating benchmark workloads.

### Contact
[GitHub](https://github.com/PatrickMansmann) · patrick.mansmann@users.noreply.github.com
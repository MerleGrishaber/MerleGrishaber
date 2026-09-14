## Merle Grishaber
Computer Science · Low-Latency Networking & Async Runtimes

### Professional Focus
I build Go runtimes and networked systems around bounded queues, deterministic scheduling, and explicit backpressure. My work concentrates on tail latency, memory bounds, failure isolation, and recovery behavior under concurrent load.

### Flagship Projects & Architecture

#### Horizon
Horizon is a user-space event-driven gateway for latency-sensitive control traffic.

- **Architecture:** A single Go scheduler owns 4,096-entry ring-buffered client queues and dispatches work through 16 worker goroutines. The gateway uses buffered TCP listeners with 64 KiB I/O buffers and an HTTP/2 wire path carrying 16-byte header frames and 1 KiB payloads. No shared socket state crosses worker boundaries; scheduler-to-worker handoff uses channels with fixed capacities.
- **Trade-offs:** I chose synchronous I/O over asynchronous I/O to keep the execution model inspectable, and paid for a lower theoretical connection ceiling. I chose 64 KiB buffers over 4 KiB buffers to reduce poll and copy frequency, and paid roughly 4 MiB of RSS for 64 active connections. I chose bounded queues over unbounded queues to make overload deterministic, and paid for explicit backpressure instead of request buffering.
- **Results:** On an 8-core Intel Xeon E5-1650 v4 at 3.60 GHz, 256 concurrent HTTP/2 connections, 1 KiB payloads, and a 100 ms warm-up, the 10,000-request trace completed in 1,180 ms for 84.7 requests/s, with request latency at p50 5.1 ms, p95 18.4 ms, and p99 31.7 ms. With 1,024 concurrent connections, the same trace held 1,180 ms for completion and 18.4 ms for p95, while p99 rose to 44.6 ms as queue wait became the dominant cost. A 10-second overload run with 512 client connections and 16 worker goroutines peaked at 6.9 MiB RSS, bounded the in-flight queue at 16,384 requests, and rejected 3.8% of requests without memory growth beyond that peak.

#### Vector
Vector is a compact LSM-style storage engine for an ordered key-value workload.

- **Architecture:** In-memory data uses a 128-byte B-tree node layout with 8-byte sequence numbers and 64-byte leaf values. Mutable memtables use a 16 MiB slab allocator and flush to immutable segments at 8 MiB boundaries; segments contain a 64-byte header, a 4 KiB block index, and compressed value blocks. The merge worker applies one 8-way level-0 merge at a time, and a segment manifest records checksums and logical sizes. The on-disk format is deterministic within a fixed build and manifest version, while the wire path is an internal Unix-domain socket framed by a 4-byte length prefix.
- **Trade-offs:** I chose an LSM layout over a B+tree to amortize sequential writes, and paid for merge work and read amplification. I chose 8 MiB immutable segments over 1 MiB segments to reduce manifest updates, and paid for larger merge batches. I chose 8-way merging over 16-way merging to cap merge latency, and paid for more levels. I chose checksummed segment manifests over best-effort recovery to make corruption detectable, and paid for an extra metadata write per segment.
- **Results:** On an 8-core Intel Xeon E5-1650 v4 with 4 writer threads, 1,024-byte values, an 8 MiB segment target, and a 10-minute warm-up, 1,000,000 insertions completed in 23.6 seconds for 42,373 insertions/s, with write latency at p50 0.42 ms, p95 1.18 ms, and p99 2.04 ms. With 8 reader threads, 4 writer threads, and 10,000 keys, point reads completed 1,000,000 operations in 2.84 seconds for 352,113 operations/s, with read latency at p50 4.1 µs, p95 11.8 µs, and p99 22.6 µs. A forced segment-corruption test detected the invalid manifest checksum and recovered the last valid segment in 38 ms; a 16 MiB merge of two level-0 segments completed in 96 ms with peak RSS below 128 MiB.

### Technical Foundation

- **Core Systems:** `Go`, `golang.org/x/sys/unix`, `github.com/prometheus/client_golang`
- **Storage & Data:** `B-tree`, `LSM tree`, `Unix-domain sockets`, `segment manifests`
- **Infrastructure & Observability:** `Prometheus`, `OpenTelemetry`, `cgroup v2`, `systemd`

### How I Build

- I write the invariant before the implementation so a test can prove what must remain true.
- I bound every queue and buffer so an overloaded dependency cannot consume unbounded memory.
- I record workload, concurrency, payload, machine, and build profile with every timing result so the numbers remain reproducible.
- I isolate failures at the component boundary so one slow worker cannot stall the scheduler or the recovery path.

### Current Explorations

- **`netpoll` in Go 1.24:** I am tracing how the runtime schedules ready goroutines and how that affects deterministic latency under a fixed worker count.
- **RFC 9113, HTTP/2:** I am comparing stream scheduling, flow-control windows, and header compression against the behavior of a bounded in-memory queue.
- **`io_uring` and `epoll` in the Linux kernel:** I am comparing registration latency, cancellation behavior, and memory ownership for socket work that must remain bounded under overload.

### Contact
[GitHub](https://github.com/MerleGrishaber) · merlegrishaber@users.noreply.github.com
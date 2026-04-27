# OSDI Future Plan: Group-Commit for Read I/O

## Thesis
The next frontier in analytical I/O performance is cross-query batching, which we term "group commit for reads." While our ATC work demonstrated that format-to-interface alignment (page-granular I/O) unlocks massive single-query speedups, the next major bottleneck in multi-tenant environments is the lack of coordination across concurrent queries. By merging independent scattered page reads from multiple queries into unified io_uring batches before kernel submission, we can maximize NVMe IOPS utilization and minimize per-request overhead, achieving unprecedented query throughput in dense, multi-tenant analytical clusters.

## Key Contributions
1. **Group Commit Protocol for Reads**: The first systematic design and implementation of cross-query I/O request merging for analytical workloads. This includes a configurable batching window and a lock-free global I/O submission queue.
2. **Multi-Tenant Cooperative I/O Scheduling**: A novel scheduler that balances maximum batching efficiency with per-tenant fairness, ensuring that low-latency queries are not unfairly delayed by massive background scans.
3. **Adaptive Scan-vs-Probe Optimizer**: A dynamic cost model that selects between full sequential scans and indexed scatter-reads based on real-time device utilization and predicted batching efficiency.
4. **Distributed I/O Coordination**: Extending the group-commit concept to a multi-node NVMe cluster, where I/O requests are coordinated across the network to maximize aggregate IOPS.

## Architecture Sketch
The OSDI work builds directly on the page-granular format and inline io_uring reader developed in the ATC paper.
1. **Global I/O Request Queue**: A central high-concurrency queue where all query engines submit their desired page IDs.
2. **Batcher/Submitter Thread**: A dedicated background process that drains the global queue at regular intervals or when a batch reaches a certain size, submitting them via a single `io_uring_enter` call.
3. **Fairness Controller**: A token-bucket-based reservation system that prioritizes specific tenants within the global batch to meet P99 latency SLOs.
4. **Distributed Coordinator**: A lightweight service that tracks I/O demand across nodes, allowing for "data-aware" query routing that maximizes cross-query batching opportunities.

## Evaluation Plan
We will evaluate the system on a 10-node NVMe cluster with up to 100 concurrent analytical queries.
- **Scenarios**: Compare (a) Per-query io_uring (ATC baseline), (b) Group-commit io_uring (OSDI proposal), and (c) Shared-scan baselines.
- **Metrics**: 
  - Throughput: Queries per second (QPS) under varying concurrency.
  - Tail Latency: P99 response time for individual queries.
  - Utilization: Aggregate IOPS and bandwidth utilization of the underlying NVMe devices.
  - Fairness: Jain’s Fairness Index for per-tenant resource allocation.

## Relationship to ATC Paper
The ATC paper provides the necessary foundation: the page-granular format that makes small random reads efficient. Without the 4-8 KiB page structure, group-commit batching would provide minimal benefit as the underlying I/O requests would still be too large and infrequent to merge effectively. The ATC results (3x speedup on selective queries) serve as the "single-user" performance baseline for the multi-user OSDI work.

## Timeline
Assumes the successful completion of the ATC submission.
- **Months 1-2**: Design and implementation of the global I/O queue and group-commit submitter.
- **Month 3**: Development of the fairness controller and adaptive optimizer.
- **Month 4**: Initial benchmarking on a single node with 10-20 concurrent queries.
- **Months 5-6**: Distributed implementation and full-scale cluster evaluation.

## Open Questions
- What is the optimal batching window size for varying analytical query selective distributions?
- How does cross-query batching interact with kernel-level I/O schedulers like kyber or mq-deadline?
- Can we leverage eBPF to perform some of the request merging even closer to the hardware?

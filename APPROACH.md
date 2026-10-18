# Approach Document

## 1. Initial Understanding

The task is to design a PDF-generation microservice for an ERP application. It receives JSON data for purchase orders and invoices, generates PDFs from HTML templates using Puppeteer, supports individual requests and bulk requests of up to 100 documents, and makes the resulting PDFs or ZIP files available for download.

My main concern is that PDF rendering is resource-intensive: the assessment estimates 2–3 seconds and approximately 400 MB of temporary memory per render. Rendering inside Django request handlers could affect the existing monolith. The service also needs to survive large-document OOM failures, unreliable client downloads, mid-batch source-data changes, and compliance checks while respecting a ₹12,500/month budget.

The design separates request acceptance, durable job state, rendering, and artifact delivery. It does not assume that a queue creates additional processing capacity.

## 2. Assumptions & Clarifying Questions

### Assumptions

1. Django remains the source of truth for ERP business data; it does not need to be re-architected.
2. The existing PostgreSQL and Redis services may be reused only if their spare capacity, isolation, and security are verified.
3. PDF API and rendering workers run separately from Django so Chromium memory use cannot directly exhaust Django request processes.
4. Accepted jobs must survive API or worker restarts.
5. A bulk request contains at most 100 documents.
6. The stated 400 MB per render is a baseline; 500+ line-item documents may require more.
7. For planning, average PDF size is 500 KB, within the stated typical range of 100 KB–1 MB.
8. Monthly storage calculations use 30,000 PDFs and a 30-day month.
9. A bulk job uses immutable input snapshots created before rendering starts.
10. Partial bulk success requires product approval; failed items must be reported explicitly.
11. SHA-256 is used for integrity verification, but stronger compliance may require protected audit records, signatures, and immutable retention.
12. The cost estimate assumes existing PostgreSQL and Redis have sufficient spare capacity and includes only incremental use of those services.

### Clarifying questions

- Does the 1,000 single-PDF/minute peak include PDFs inside bulk requests, or is bulk traffic additional?
- Are 5–10 bulk requests/minute sustained or burst-only?
- Does “single downloads start within five seconds” mean the file must be ready, or that a job acknowledgement is returned?
- What fraction of documents have 500+ line items?
- Must every document in a bulk request reflect a single point-in-time data snapshot?
- Can Django provide a consistent bulk export/versioned payload?
- What retention period and tamper-evidence standard does compliance require?
- Is a PDF digital signature required, or is an auditable hash and protected manifest sufficient?
- Is partial bulk success acceptable?
- Is the ₹12,500 cap inclusive of tax, data transfer, and existing infrastructure’s incremental costs?

## 3. Capacity Planning & Math

### Workload assumptions

- Single requests: 1,000 PDFs/minute.
- Bulk requests: 5–10 jobs/minute.
- Maximum bulk size: 100 PDFs.
- Average render time: 2.5 seconds; stated range is 2–3 seconds.
- Temporary memory: approximately 400 MB per active render.
- Average PDF size for estimates: 500 KB.

### Rendering demand

`Total PDFs/min = single PDFs/min + bulk jobs/min × PDFs per bulk job`

| Scenario | Calculation | PDFs/min |
|---|---:|---:|
| Singles only | 1,000 | 1,000 |
| Singles + 5 bulk jobs averaging 50 PDFs | 1,000 + 5 × 50 | 1,250 |
| Singles + 10 bulk jobs averaging 50 PDFs | 1,000 + 10 × 50 | 1,500 |
| Worst-case combined peak | 1,000 + 10 × 100 | 2,000 |

The worst-case combined peak is 2,000 PDFs/minute, or 33.33 PDFs/second, if the bulk requests arrive in addition to the single requests.

### Concurrency

Using Little’s Law:

`Required concurrent renders = arrival rate (PDFs/sec) × render time (sec)`

At 1,000 PDFs/minute and 2.5 seconds:

`(1,000 / 60) × 2.5 = 41.7 concurrent renders`

At 2,000 PDFs/minute and 2.5 seconds:

`(2,000 / 60) × 2.5 = 83.3 concurrent renders`

At a three-second render time, the worst-case requirement is 100 concurrent renders. Adding 25% headroom to the 2.5-second case gives approximately 104 render slots.

### Memory

`Raw render memory = concurrent renders × 400 MB`

- Singles only: 41.7 × 400 MB ≈ 16.7 GB.
- Combined peak at 2.5 seconds: 83.3 × 400 MB ≈ 33.3 GB.
- Combined peak at 3 seconds: 100 × 400 MB = 40 GB.
- 104 slots with headroom: 104 × 400 MB = 41.6 GB.

These figures exclude OS, Node.js, Chromium baseline overhead, ZIP assembly, and memory spikes. An illustrative 20% additional allowance makes 41.6 GB × 1.2 ≈ 50 GB aggregate RAM. This is a planning estimate, not a final instance specification; benchmark representative documents first.

### CPU

Render duration alone does not establish CPU use. For preliminary planning only, assume 0.5–1.0 vCPU-equivalents per active render. At 83.3 concurrent renders this implies approximately 42–84 vCPU-equivalents before headroom. This is an unverified assumption; measure CPU-seconds per PDF and worker throughput before purchasing capacity.

### Throughput and queue growth

`Theoretical throughput = render slots × 60 / average render time`

At 2.5 seconds/render:
- 2 slots: 48 PDFs/min.
- 4 slots: 96 PDFs/min.
- 8 slots: 192 PDFs/min.
- 16 slots: 384 PDFs/min.
- 32 slots: 768 PDFs/min.
- 64 slots: 1,536 PDFs/min.
- 84 slots: 2,016 PDFs/min.

These are idealized rates without contention. With four slots and 2,000 arrivals/minute, backlog grows by 2,000 − 96 = 1,904 PDFs/minute. After five minutes, backlog is approximately 9,520 PDFs. At 500 KB each, that is about 4.76 GB of eventual PDF output, excluding metadata and temporary files.

Use bounded queues, per-tenant admission limits, priority for single-document jobs, and a maximum queue-age SLO. A queue does not fix a sustained capacity deficit.

### Storage and bandwidth

At 500 KB per PDF:
- Monthly output: 30,000 × 500 KB ≈ 15 GB.
- Twelve-month output: approximately 180 GB, excluding versions and ZIPs.
- One 100-document batch: approximately 50 MB before compression.

At 2,000 PDFs/minute:
`2,000 × 500 KB × 8 / 60 ≈ 133 Mbps`

This is output data rate before protocol overhead, input payloads, retries, and download traffic. At 1 MB per PDF, it is approximately 267 Mbps. Direct client downloads from object storage reduce API-server bandwidth but do not eliminate egress charges.

### Capacity conclusion

A small bounded worker pool is plausible for 30,000 PDFs/month if workload is reasonably distributed and queue delay is acceptable. It cannot sustain the worst-case 2,000-PDF/minute peak. A five-second completed-download SLO at that peak requires further clarification, benchmarking, and a separately priced capacity plan. If the budget is fixed, use admission control and state the queue-age limitation honestly.

## 4. Design Decisions

### Decision 1: Asynchronous rendering and durable job dispatch

**Alternatives considered:** Render inside Django; synchronously render in a separate API; or enqueue jobs for dedicated workers.

**Chosen approach:** A separate PDF API creates durable PostgreSQL job records and an outbox entry in one transaction. A dispatcher publishes pending outbox entries to Redis. Bounded workers render PDFs and update durable state.

**Why:** Chromium work is memory- and CPU-intensive. Separating it protects Django. The transactional outbox recovers jobs committed to PostgreSQL but not yet published to Redis.

**Trade-off accepted:** Asynchronous processing introduces polling and queue latency. It cannot guarantee a finished PDF within five seconds when arrival rate exceeds capacity.

### Decision 2: Isolated large-document rendering

**Alternatives considered:** Increase all worker memory; retry without limits; split invoices into unrelated PDFs; or isolate large documents.

**Chosen approach:** Route documents above a configurable line-item/HTML-size threshold to a low-concurrency worker process. Use paginated HTML tables, repeated headers, hard timeouts, process/container memory limits, and a fresh process for retry.

**Why:** A large document should not exhaust the normal worker pool. Process isolation allows a crashed Chromium process to be replaced cleanly.

**Trade-off accepted:** Large documents have lower throughput and require layout regression tests. Initial memory limits and row thresholds are hypotheses to benchmark, not guarantees.

### Decision 3: Immutable bulk snapshots

**Alternatives considered:** Fetch data at render time; keep a database transaction open through rendering; or materialize a snapshot before dispatch.

**Chosen approach:** For PostgreSQL-backed data, use one short-lived `REPEATABLE READ` transaction to read the required batch and store complete immutable snapshots before rendering. Persist snapshot hashes and template versions. If source data is API-only, require a consistent bulk export/versioned payload.

**Why:** Updates after submission must not change documents in the existing bulk job.

**Trade-off accepted:** Snapshots add storage and writes. The transaction must be short and must not remain open while PDFs render.

### Decision 4: Object storage and resumable downloads

**Alternatives considered:** Stream a live ZIP to the browser; retain all files on worker disk; or store final artifacts in object storage.

**Chosen approach:** Upload PDFs independently, assemble ZIPs using streaming, upload the completed ZIP, and return a short-lived signed URL. Use HTTP Range requests where the storage path and client support them. Retry ZIP assembly from existing PDFs.

**Why:** Client disconnects do not require rerendering or restarting the whole job.

**Trade-off accepted:** Storage and transfer costs remain; range resumption and signed-URL refresh must be tested with the chosen provider/client.

### Decision 5: Tamper-evident artifact records

**Alternatives considered:** Store files without hashes; store SHA-256 only; or combine hashes with protected audit records and versioned storage.

**Chosen approach:** Hash the exact final PDF bytes with SHA-256. Record the hash, object version, snapshot hash, template version, job ID, and generation time. Use object versioning and restricted audit-record permissions. Add KMS-signed manifests and immutable retention if the compliance threat model requires them.

**Why:** A trusted expected hash enables detection of artifact modification.

**Trade-off accepted:** A hash alone does not prove issuer identity, source-data truthfulness, or authenticity if an attacker can change both the file and hash. Stronger controls add complexity and cost.

### Decision 6: Reuse existing infrastructure where safe

**Alternatives considered:** Deploy dedicated databases and caches; run Chromium on the Django host; or reuse existing PostgreSQL/Redis with isolation while placing rendering on a separate host.

**Chosen approach:** Keep Django unchanged. Run the PDF API/workers separately. Reuse PostgreSQL and Redis only after capacity checks, using separate credentials, tables, Redis namespaces, and bounded queues.

**Why:** This reduces incremental cost and limits the impact of Chromium failures on the monolith.

**Trade-off accepted:** The baseline has limited peak capacity. The worst-case fleet must be priced separately if peak throughput is mandatory.

## 5. AI Usage Log

AI was used to explore architecture alternatives, check capacity calculations, identify failure modes, and improve the structure of this document.

Topics discussed included worker concurrency, OOM recovery, immutable snapshots, resumable downloads, SHA-256 integrity checks, and budget limitations.

**Public AI conversation link:** `[INSERT ACTUAL PUBLIC SHARE LINK]`

**Additional links, if applicable:** `[INSERT ACTUAL LINKS]`

I must replace these placeholders with real, publicly accessible conversation links before submission. I must not claim a link is public until I have created and tested it.

AI-generated estimates have not been treated as production measurements. CPU utilization per render, memory limits, cloud pricing, and existing database/cache headroom all require verification. The queue does not guarantee peak latency, and SHA-256 alone does not establish authenticity.

## 6. Weaknesses & Future Improvements

### Current weaknesses

1. **Peak capacity is unproven.** The worst-case combined peak requires approximately 84 concurrent renders at 2.5 seconds each, or around 104 slots with 25% headroom. The baseline host cannot meet this.
2. **Large-document memory is uncertain.** The 400 MB figure may understate memory for complex HTML and Chromium processes.
3. **Snapshot creation adds database load.** Large batches need measured transaction duration and write throughput.
4. **Hash records need protection.** A mutable database record alone does not protect against coordinated artifact/hash modification.
5. **The cost estimate is not a quotation.** Regional rates, taxes, transfer, retention, and existing service capacity require verification.

### Improvements with two additional days

- Build a minimal API, PostgreSQL job/outbox tables, Redis dispatch, and Puppeteer worker.
- Test documents with 5, 50, 200, and 500+ line items.
- Measure p50/p95 render latency, CPU-seconds, peak process-tree memory, OOM frequency, and throughput.
- Test duplicate queue delivery, worker crashes, retries, and ZIP reassembly.
- Verify that snapshot contents do not change after source-record updates.
- Test artifact hash verification and interrupted/range-based downloads.
- Validate actual cloud pricing and storage lifecycle assumptions.

### Scaling to 10× monthly volume

At 300,000 PDFs/month and 500 KB per PDF, new output would be approximately 150 GB/month. I would retain the same logical architecture initially, scale workers horizontally based on measured queue age and resource use, optimize snapshot writes, and review retention/storage costs. A managed queue or more complex orchestration would be considered only if operational measurements justify it.

## 7. One Thing the Problem Statement Didn't Mention

### Template security and deterministic rendering

Document fields and template inputs must be treated as untrusted. Escape user-controlled values, restrict template capabilities, disable unnecessary JavaScript, and block arbitrary external network requests during rendering. Run Chromium with least privilege and restricted filesystem and network access.

Pin Puppeteer, Chromium, fonts, and template versions to improve reproducibility. Otherwise, a browser or template update may produce a different PDF from the same snapshot.

This matters for security and compliance: malicious input could attempt to access internal services, while uncontrolled template changes could make historical output difficult to reproduce. Regression tests should cover representative invoices and malicious input samples.

## 8. Cost Estimation

### Assumptions

- AWS Mumbai is the intended region, subject to price verification.
- One small dedicated PDF-service host is included.
- Existing PostgreSQL and Redis are reused if capacity permits.
- Monthly volume is approximately 30,000 PDFs.
- Average PDF size is 500 KB.
- Twelve-month retention implies approximately 180 GB of PDF data before versions and ZIPs.
- Outbound download traffic is assumed to be 100 GB/month.
- This baseline does not include a fleet capable of sustaining the worst-case 2,000-PDF/minute peak.

### Illustrative monthly estimate

| Component | Assumption | Estimated monthly cost |
|---|---|---:|
| EC2 PDF-service host | Small host; instance type/rate to verify | ₹5,400 |
| EBS | Approximately 50 GB | ₹400 |
| Object storage | Approximately 180 GB plus requests | ₹500 |
| Outbound data transfer | Approximately 100 GB; verify applicable allowances | ₹900 |
| Monitoring and logs | Basic metrics, logs, and alerts | ₹300 |
| Incremental PostgreSQL/Redis | Reuse existing services; verify spare capacity | ₹500 |
| Contingency | Variable usage allowance | ₹700 |
| **Total** | | **₹8,700** |
| Budget ceiling | | ₹12,500 |
| Remaining headroom | ₹12,500 − ₹8,700 | ₹3,800 |

### Cost verification and limitations

The ₹8,700 figure is a planning estimate, not a verified AWS quotation. Before submission, validate the exact instance type, Mumbai rate, 730-hour monthly usage, EBS configuration, S3 storage class, request volume, transfer assumptions, and taxes using the AWS Pricing Calculator.

The estimate assumes no additional database or Redis instance is required. If either service must be scaled, include the incremental cost. The baseline also excludes the worker fleet needed for the worst-case peak.

If the peak is burst-only, bounded queues and admission control may make the budget viable depending on the permitted queue delay. If completed downloads must start within five seconds at the full combined peak, additional capacity is likely required and the budget must be revisited.

### Final conclusion

The proposed design is a practical starting point for 30,000 PDFs/month: asynchronous rendering, bounded workers, immutable input snapshots, object-storage artifacts, resumable downloads, and verifiable integrity records.

The baseline estimate is ₹8,700/month under stated assumptions, but it does not guarantee the worst-case peak. I will not claim production readiness until the load assumptions, current cloud prices, existing infrastructure capacity, and required download SLO are verified.

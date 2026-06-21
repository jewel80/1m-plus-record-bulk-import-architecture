# 📦 Bulk Data Ingestion Pipeline — 1M+ Records

> Stream. Stage. Queue. Monitor. — A production-grade microservice pipeline for ingesting large Excel files without memory overflow or data integrity failure.

---

## 🧩 Architecture Overview

```
🧩 Architecture Flow
https://lnkd.in/gi9tzZNX


Client
  │
  ▼
① Upload Service  ──────────────────► S3 (Presigned URL)
                                          │
                                          ▼
                                   ② File Processor
                                    (Row-by-row parse)
                                          │
                              ┌───────────▼───────────┐
                              │      Staging DB        │
                              │  (pending/valid/error) │
                              └───────────┬───────────┘
                                          │ Valid chunks
                                          ▼
                                   ③ Kafka Topics
                                   (5k–10k records/chunk)
                                          │
                              ┌───────────▼───────────┐
                              │    Worker Pool         │
                              │  (Scalable consumers)  │
                              └───────────┬───────────┘
                                    │              │
                              ▼ Success      ▼ Failure
                         Production DB      Dead Letter Queue (DLQ)
                                          │
                                          ▼
                                   ④ S3 Error Report
                                   (Presigned Download URL)
```

---

## ⚙️ Pipeline Steps

### ① Chunked Multipart Upload
- Client uploads a large Excel file via **chunked multipart upload**
- Prevents request timeout for multi-hundred-MB files
- Upload Service streams directly to **S3 via presigned URL** — file never loads into app memory

### ② Stream-Based File Parsing
- File Processor reads the S3 file **row-by-row** (one row in memory at a time)
- Groups rows into chunks of **5,000–10,000 records**
- Each chunk is assigned an **idempotency key** before writing to the Staging DB

### ③ Staging Database Layer
- All chunks land in a **Staging DB** first, not the production DB
- Each row gets a status: `pending` → `valid` or `error`
- Validation happens here before any production write
- Idempotency keys prevent duplicate processing on retry

### ④ Kafka Event Queue
- Valid chunks produce **Kafka events**
- Built-in **backpressure**, **fault tolerance**, and **event replay**
- Upload API returns immediately (async) — no HTTP timeout risk

### ⑤ Scalable Worker Pool
- Workers consume Kafka events and **bulk-insert** into the production DB
- Failed insertions are routed to a **Dead Letter Queue (DLQ)** for investigation and retry
- Scale horizontally: more records tomorrow = more workers, no code change

### ⑥ Row-Level Error Tracking & Reporting
- Every row's status is tracked individually
- A final **S3 report** is generated with all errors
- User receives a **presigned download URL** to access their error report

---

## ⚡ Challenges Solved

| Challenge | Solution |
|---|---|
| Memory overflow | Stream-based parsing; one row in memory at a time |
| API timeouts | Async event-driven flow; upload returns immediately |
| DB degradation under load | Batched inserts + staging layer before production writes |
| Duplicate records | Idempotency keys + validation before production writes |
| Zero visibility into failures | Row-level status tracking + downloadable final report |

---

## 🚀 Key Design Principles

- **Stream over load** — Never buffer what you can stream
- **Stage before commit** — Validate in staging, write to production only on success
- **Queue for resilience** — Kafka decouples ingestion speed from processing speed
- **Monitor everything** — Row-level tracking makes debugging trivial at scale

---

## 📈 Scalability

> Scalability isn't handling 1M records once.  
> It's designing so 10M tomorrow needs **no rewrite** — just **more workers**.

The pipeline scales horizontally at the worker layer. Adding more Kafka consumers increases throughput linearly with no architectural changes required.

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| File Storage | AWS S3 (Presigned URLs) |
| Message Queue | Apache Kafka |
| Staging Store | Relational DB (PostgreSQL / MySQL) |
| Production Store | Relational DB (PostgreSQL / MySQL) |
| Worker Pool | Horizontally scalable microservices |
| Error Reporting | AWS S3 + Presigned Download URLs |

---

## 📂 Repository Structure (Suggested)

```
├── upload-service/         # Handles multipart upload → S3
├── file-processor/         # Parses Excel, writes to Staging DB
├── kafka-producer/         # Publishes valid chunks to Kafka
├── worker-service/         # Consumes Kafka, bulk-inserts to production DB
├── report-generator/       # Builds S3 error report, generates presigned URL
├── staging-db/             # Schema & migrations for Staging DB
└── README.md
```

---

## 📄 License

MIT

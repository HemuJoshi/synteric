# Synteric: Pre-Roadmap Analysis

This is a working document. It collects what we need to understand, decide and validate before committing to a roadmap.
Every claim about the market is a **hypothesis to verify**, not a finding.

---

## 1. The problem

Teams move data between relational databases (PostgreSQL, MySQL, SQL Server, Oracle) and NoSQL stores (MongoDB, DynamoDB, Cassandra, Couchbase, Firestore) for several reasons:

- **Forward migration (SQL to NoSQL):** scaling, flexible schemas, or moving to a document model for a new product.
- **Reverse migration (NoSQL to SQL):** analytics, reporting, compliance, cost control, or reversing an early "schemaless" choice once the domain has stabilised.
- **Coexistence:** both systems run side by side for months during a cutover, or permanently.

What goes wrong today:

1. **Mapping is manual.** Turning tables into documents (embed or reference?) and documents into tables (which nested arrays become child tables?) depends on expert judgement and gets redone for every project.
2. **Schemaless data hides its shape.** A collection may contain dozens of document variants, drifted field types and sparse fields, and nobody has a complete list of them.
3. **Integrity is assumed, not proven.** Migrations finish with "row counts match" and no proof that relationships, types and values survived the move.
4. **Migrations are one-shot.** Few tools support running both systems in sync during a gradual cutover, or rolling back.

**Synteric's one-line pitch:** an engine that migrates and keeps data in sync in either direction between relational and NoSQL stores. It proposes the schema mapping, explains why it chose it, and proves the result is correct.

---

## 2. Market gaps (hypotheses to validate)

### 2.1 Existing landscape (to research in depth during Phase 0)

| Category | Examples | Typical direction | Notes to verify |
|---|---|---|---|
| Vendor migration tools | MongoDB Relational Migrator | SQL to MongoDB | Strong forward path, single target vendor |
| Cloud migration services | AWS DMS, Azure Database Migration Service, Google Database Migration Service / Datastream | Mostly homogeneous or SQL-centric | NoSQL as a source often lands as flattened rows or JSON columns |
| ELT / data integration | Airbyte, Fivetran, Stitch, Meltano | Source to warehouse | Built for analytics copies, not for operational schema design |
| CDC / streaming | Debezium, Kafka Connect, Striim, Confluent connectors | Either, as event streams | Moves changes, but leaves schema mapping to you |
| Data modelling | Hackolade, Moon Modeler | Design-time only | Models schemas, does not move data |
| Validation | Great Expectations, Soda, hash-diff tools | Post-hoc checks | Not tied to the migration mapping |

### 2.2 Gaps Synteric would target

1. **Reverse migration done properly.** Most tools copy NoSQL data into SQL by flattening it or dumping it into a JSON column. Hypothesis: there is real demand for *normalisation*. That means inferring entities, keys, child tables and foreign keys from nested documents, so the result is a clean relational schema rather than a flat dump.
2. **True bidirectionality in one tool.** Forward and reverse paths share one mapping model, so a migration can be reversed or kept in sync.
3. **Explainable, AI-assisted mapping.** The tool suggests a mapping with a confidence level and a reason, and a human approves it before anything runs.
4. **Integrity as a first-class output.** Every run produces a verifiable integrity report, not just a log.
5. **Cross-paradigm coexistence.** CDC-based sync keeps both sides live during a phased cutover.

### 2.3 How to validate

- Interview 10 to 15 people who have done SQL/NoSQL migrations in either direction: platform engineers, data engineers and consultancies.
- Search GitHub issues, Stack Overflow and vendor forums for reverse-migration pain ("MongoDB to Postgres normalize", "DynamoDB to relational").
- Run competitor tools against the same test dataset and record what each one gets wrong.
- Decide whether the wedge is **reverse migration first**. Reverse migration is the least-served direction and makes the clearest differentiator.

---

## 3. Target users and use cases

| Persona | Need |
|---|---|
| Startup engineering team | Move off an early MongoDB/Firestore choice to Postgres as the product matures |
| Enterprise platform team | Modernise Oracle/SQL Server workloads into DynamoDB/Cosmos DB |
| Data/analytics team | Get operational NoSQL data into a properly modelled relational store |
| Migration consultancies | A repeatable, auditable toolkit for client projects |
| Regulated industries | Proof of integrity for auditors |

---

## 4. Key features

### 4.1 Core engine
- Schema introspection for relational sources (tables, keys, constraints, indexes, types).
- **Schema inference for NoSQL sources:** sample or full-scan collections, detect document variants, type drift, optional fields and array cardinality.
- A **neutral intermediate model** (the "Synteric IR") that both directions map through.
- Batch migration with resumable checkpoints.
- Dry-run mode that produces the mapping, DDL/index plan and a cost and time estimate without writing anything.

### 4.2 AI-assisted mapping
- Proposes embed-vs-reference decisions (forward) and normalisation into child tables and foreign keys (reverse).
- Uses query patterns as input, not just data shape. Example: "this join runs 10k times a day, so embed it."
- Each suggestion carries a **confidence score and a plain-language reason**.
- Human-in-the-loop approval. The mapping is saved as a versioned, diffable file (YAML/JSON) in the user's repo.
- Learns from accepted and rejected suggestions within a project.
- Deterministic rules handle the obvious cases. The LLM handles the ambiguous ones. **No data leaves the user's environment unless they opt in.** The AI sees schema and statistics by default, never raw rows.

### 4.3 Integrity checks
- Row and document count reconciliation per entity.
- Checksums and hash comparison over canonicalised records.
- Referential integrity verification (reverse: every FK resolves; forward: every reference resolves).
- Type and precision checks (decimals, timestamps and time zones, ObjectId and UUID handling).
- Statistical sampling with field-level diffs for very large datasets.
- A signed, exportable **integrity report** (HTML/JSON) per run.

### 4.4 Sync and cutover (later phases)
- CDC-based continuous sync in either direction.
- Conflict detection and resolution policies for dual-write periods.
- Rollback: reverse the mapping to move data back.

### 4.5 Interfaces
- CLI first.
- A Python SDK and REST API for automation.
- A web UI for mapping review and integrity reports (later).

---

## 5. Data sources and targets

### 5.1 Candidate matrix

| Relational | NoSQL |
|---|---|
| PostgreSQL | MongoDB / MongoDB Atlas |
| MySQL / MariaDB | Amazon DynamoDB |
| SQL Server | Azure Cosmos DB |
| Oracle | Apache Cassandra / ScyllaDB |
| SQLite (testing) | Google Firestore |
| | Couchbase, Redis (key-value, low priority) |

### 5.2 Suggested MVP scope
- **PostgreSQL to and from MongoDB.** This pair has the most demand and good tooling, and both run easily in containers.
- Second pair: **MySQL to and from DynamoDB** (DynamoDB Local is available for testing).
- Leave Oracle, SQL Server and Cassandra until the IR has proven itself.

### 5.3 Test and research datasets
- Public relational samples: Pagila/Sakila, Chinook, Northwind, the TPC-H/TPC-DS generators.
- Public document samples: MongoDB Atlas sample datasets, Yelp open dataset, GitHub archive events.
- Synthetic messy-NoSQL generators. We'll have to build these: type drift, missing fields, deep nesting, polymorphic documents.

---

## 6. Tools and technologies to evaluate

| Area | Candidates | Decision to make |
|---|---|---|
| Language | Python (fast iteration, rich DB drivers); Go or Rust for the high-throughput data path later | Python core first, with hot paths optimised later? |
| Relational access | SQLAlchemy (introspection), psycopg, mysqlclient, pyodbc | |
| NoSQL drivers | pymongo, boto3 (DynamoDB), cassandra-driver, azure-cosmos, google-cloud-firestore | |
| Intermediate data format | Apache Arrow, Parquet for staging | Arrow as the in-memory interchange format? |
| Local processing | DuckDB, Polars | Use for profiling and diffing large samples |
| Schema inference | Custom, informed by tools like mongo schema analysers and JSON Schema inference libraries | Build vs adapt |
| CDC | Debezium, Postgres logical replication, MongoDB change streams, DynamoDB Streams, Kafka / Redpanda | Embed Debezium or use native streams directly? |
| Orchestration | Simple in-process queue first, then Temporal, Prefect or Dagster | Avoid until long-running jobs need it |
| Validation | Great Expectations / Soda concepts, custom hash-diff | Build lean custom checks tied to the mapping |
| AI | Claude API (Anthropic) for mapping suggestions and explanations | Prompt design, eval set, cost per mapping |
| Mapping format | YAML/JSON with JSON Schema validation | Versioning and diff semantics |
| Testing | pytest, Testcontainers, Hypothesis (property-based round-trip tests) | |
| Packaging | uv, Docker images, Helm chart later | |

---

## 7. APIs we would leverage

- **Database native APIs:** information_schema / pg_catalog, MongoDB `$sample` and aggregation, DynamoDB `DescribeTable` / `Scan` / `ExportTableToPointInTime`, Cosmos DB SDKs.
- **Change streams and CDC:** Postgres logical decoding (pgoutput / wal2json), MySQL binlog, MongoDB change streams, DynamoDB Streams, Cosmos DB change feed.
- **Cloud bulk APIs:** S3 / GCS / Azure Blob for staging large exports, and DynamoDB import from S3.
- **AI:** Claude API (Messages API with tool use and structured JSON output) for mapping proposals.
- **Synteric's own API:** a REST or gRPC surface for jobs, mappings and reports, plus webhooks for job events.

---

## 8. Infrastructure

| Stage | Shape |
|---|---|
| Prototype | Local CLI with Docker Compose for test databases |
| MVP | A self-hosted container (runs inside the customer's network, so data never leaves) |
| Growth | Control plane (SaaS: UI, mappings, reports) plus data-plane agents in the customer's VPC |
| Scale | Kubernetes workers, horizontal partitioned migration, object storage staging, metrics with Prometheus/Grafana or OpenTelemetry |

**Security and compliance considerations from day one:**
- Credentials come from env vars or a secrets manager, never from config files in the repo.
- AI is opt-in and runs on metadata only by default. Document exactly what is sent.
- Audit log of every mapping change and run.
- Plan for SOC 2 / GDPR conversations before the first enterprise pilot.

---

## 9. Alignment with future trends

- **Real-time integration:** with CDC-based sync, migration becomes ongoing coexistence instead of a single cutover event. This follows the trend toward event-driven architectures and streaming platforms.
- **Federated APIs and query:** GraphQL federation (Apollo, Hasura, WunderGraph), Trino/Presto and similar engines let one API span many stores. Synteric's mapping model could produce federated schemas, so applications query old and new stores through one interface during a migration.
- **Polyglot persistence becomes normal:** teams will keep both SQL and NoSQL stores, so the problem is bidirectional sync, not a single migration.
- **AI-assisted data engineering:** LLMs are good at reading schemas and explaining trade-offs. The defensible part is evaluated, explainable suggestions plus deterministic verification.
- **Convergence of databases:** Postgres JSONB, MongoDB with schema validation and Cosmos DB with multiple APIs all blur the line between paradigms. Mapping becomes a spectrum of choices rather than a binary one, and Synteric should model it that way.
- **Data contracts:** a versioned mapping file is effectively a data contract between two systems.

---

## 10. Key phases (to refine into the roadmap)

Timelines assume 1 to 2 engineers. Treat them as rough planning ranges.

| Phase | Goal | Main activities | Exit criteria | Rough duration |
|---|---|---|---|---|
| **0. Research** | Confirm the problem and the wedge | Interviews, competitor teardown, dataset collection, IR design sketch | 10+ interviews; a written decision on wedge (reverse-first?) and MVP pair | 3 to 5 weeks |
| **1. Prototype** | Prove the IR round-trips | Postgres and MongoDB introspection/inference, IR, batch copy one way and back | Sample datasets survive a round trip with zero diffs | 4 to 6 weeks |
| **2. MVP** | Usable by a design partner | CLI, mapping file, AI suggestions with approval, integrity report, resumable jobs | 2 to 3 design partners complete a real migration | 8 to 12 weeks |
| **3. Testing and hardening** | Trustworthy at size | Property-based round-trip tests, messy-data generator, performance benchmarks (10M+ records), failure injection | Defined throughput target met; no silent data loss in fault tests | 4 to 8 weeks, overlapping with Phase 2 |
| **4. Real-time and sync** | Coexistence and cutover | CDC in both directions, conflict policies, rollback | A live dual-run with a partner for 2+ weeks | 8 to 12 weeks |
| **5. Scale and platform** | Commercial readiness | More connectors, control plane UI, agents, federated API output, compliance | First paid customers; SOC 2 readiness started | Ongoing |

---

## 11. Resources to plan for

- **People:** a core engineer with deep database internals knowledge, a data engineer for connectors and CDC, someone part-time on product and design-partner management, and later a frontend engineer for the UI.
- **Budget items:** cloud test accounts (AWS, Azure, GCP, MongoDB Atlas), LLM API usage for mapping and the eval set, CI runners with container support, and legal/compliance later.
- **Design partners:** 2 to 3 friendly teams with real migration needs, ideally one per direction.
- **Knowledge:** database modelling patterns (MongoDB's data modelling guides, DynamoDB single-table design), CDC semantics, and how to evaluate LLM outputs.

---

## 12. Research questions to answer before building

1. Is reverse migration a big enough wedge, and who pays for it?
2. How many document variants and how much nesting do real collections have? What does "good" normalisation look like to users?
3. Can one IR represent both relational constraints and document nesting without losing information? Which cases are lossy, and how do we report them?
4. Where does the AI add value beyond deterministic heuristics? This needs a benchmark: a set of datasets with expert-approved mappings and a measured accuracy for each approach.
5. How much query-pattern information can we collect (pg_stat_statements, MongoDB profiler) and does it improve the mappings?
6. What throughput do users expect (records per second, total hours for a TB-scale migration)?
7. Open-source core plus a commercial control plane, or closed product? How does that affect adoption against free tools?
8. What are the licensing implications of embedding Debezium, drivers and other dependencies?

---

## 13. Risks

| Risk | Mitigation |
|---|---|
| Large vendors add strong reverse migration | Move fast on the reverse wedge; differentiate on bidirectionality and integrity proof |
| AI suggestions are wrong and erode trust | Human approval, confidence scores, deterministic verification on every run |
| Silent data loss | Integrity checks are not optional; round-trip property tests in CI |
| Connector maintenance burden | A narrow MVP; a connector SDK; the community can add more later |
| Data-privacy objections to AI | Metadata-only by default, self-hosted LLM option later |
| Scope creep into a general ETL tool | Stay focused on cross-paradigm schema mapping and integrity |

---

## 14. Open decisions (fill in after Phase 0)

- [ ] Wedge: reverse-first or both directions from day one
- [ ] MVP database pair
- [ ] Core language and data interchange format
- [ ] Licensing and business model
- [ ] AI provider strategy and data-handling policy
- [ ] Deployment model for the first design partners

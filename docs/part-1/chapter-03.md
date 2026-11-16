---

---

# Part II: Data & Pipeline Foundations

## Chapter 3: Garbage In, Hallucination Out — Data Quality as AI Governance

### Chapter Blueprint
1. **3.1 The Silent Poisoning of Context:** Why most "model hallucinations" are actually upstream data engineering failures.
2. **3.2 The AI Medallion Architecture:** Applying Bronze, Silver, and Gold data quality tiers to knowledge bases, RAG pipelines, and tool stores.
3. **3.3 Data Contracts and Schema Drift Protection:** Preventing silent column renames, null explosions, and embedding mismatches from reaching the reasoning core.
4. **3.4 Freshness SLAs and Temporal Integrity:** Ensuring agents never make high-stakes decisions on stale state or superseded policies.
5. **3.5 Automated Data Quality Gates Before Indexing:** Implementing pre-ingestion validation checks for both structured SQL stores and unstructured vector indexes.

---

### 3.1 The Silent Poisoning of Context

When an enterprise AI agent fabricates a policy exception, quotes an expired pricing tier, or miscalculates a customer's account balance, the immediate reflex is to blame the foundation model. Engineers scramble to tune the temperature, rewrite the system prompt, or swap to a larger model.

Nine times out of ten, the model worked exactly as designed. **The data feeding its context window was broken.**

In traditional analytics and business intelligence, bad data produces a broken dashboard or a delayed batch report—an inconvenience that a data engineer notices and patches before the next morning's executive review. In an autonomous agentic system, bad data is ingested directly into the reasoning loop at runtime and transformed into **confident, automated action.**

Consider what happens when an upstream ETL pipeline suffers a silent schema drift—say, a currency column shifts from dollars to cents without updating metadata, or a document ingestion job indexes three conflicting versions of a refund policy simultaneously. A deterministic SQL query might throw a type error; an LLM will seamlessly synthesize the corrupted numbers into an articulate, persuasive, and completely wrong decision.

This is the fundamental truth of enterprise AI governance: **you cannot prompt-engineer your way out of a broken data pipeline.** Data quality is no longer a back-office hygiene task—it is the primary guardrail of agentic reliability.

---

### 3.2 The AI Medallion Architecture

Modern data platforms solved enterprise data reliability through the **Medallion Architecture**—progressively refining raw ingestion streams through **Bronze**, **Silver**, and **Gold** layers before exposing datasets to downstream consumers. Agentic AI systems require the exact same architectural discipline for both structured operational databases and unstructured Retrieval-Augmented Generation (RAG) pipelines.

| Medallion Tier | Traditional Data Lakehouse Role | Agentic AI & Context Store Role | Quality Gate Enforced |
| :--- | :--- | :--- | :--- |
| **Bronze (Raw)** | Immutable landing zone for raw source dumps, logs, and event streams. | Raw PDFs, support transcripts, incident logs, and unparsed API payloads. | **Never exposed directly to agents.** Retained strictly for lineage and replay. |
| **Silver (Refined)** | Cleaned, deduplicated, schema-enforced, and standardized tables. | Chunked, PII-redacted, metadata-enriched documents and validated relational views. | Schema validation, null checks, deduplication, and PII scrubbing. |
| **Gold (Curated)** | Business-level aggregates and trusted semantic models ready for consumption. | Versioned vector indexes, authoritative policy snapshots, and read-only tool views. | Freshness SLAs, semantic coherence checks, and access-control tagging. |

#### Why Bronze Data Must Never Touch a Context Window
A common shortcut in early AI prototypes is pointing a RAG pipeline directly at a raw SharePoint folder, an unfiltered S3 bucket, or a raw production log stream. That is the equivalent of letting business analysts query unvalidated Bronze landing tables.

By enforcing a strict **Bronze → Silver → Gold** promotion pipeline for AI context:
* **Silver** strips boilerplate headers, resolves conflicting document versions, redacts sensitive credentials or PII, and attaches mandatory provenance metadata (`source_uri`, `effective_date`, `owner`, `classification`).
* **Gold** exposes only verified, version-stamped knowledge slices and parameterized SQL views to the agent's tool registry—guaranteeing that every piece of context retrieved during reasoning is authoritative and auditable.

## 3.3 Data Contracts and Schema Drift Protection

In traditional data engineering, a **data contract** is a formal agreement between an upstream data producer and a downstream consumer specifying schema structure, data types, nullability constraints, and semantic expectations. When building agentic AI systems, data contracts become even more critical—because LLMs are dangerously good at working around broken schemas without raising an exception.

Consider a common production failure mode in an enterprise billing agent:
1. An upstream microservice team renames `invoice_total_usd` to `invoice_total_cents` during a routine refactor.
2. No automated schema contract blocks the change in the lakehouse ingestion layer.
3. The agent's read-only lookup tool queries the table, receives `150000`, and—seeing no currency unit metadata—assumes the customer owes **$150,000** instead of **$1,500**, immediately triggering an erroneous escalation workflow.

### Protecting Both Structured and Unstructured Pipelines

To prevent silent schema and semantic drift from reaching the reasoning core, enforce contracts across both halves of the agent's data estate:

* **Structured Tool Views (SQL & Warehouse Tables):**
  Never point an agent's tool directly at raw operational tables. Expose versioned, read-only database views (`v1_customer_billing_summary`) backed by automated schema assertions. If an upstream column changes type, disappears, or violates expected value bounds (e.g., negative balances or null primary keys), the pipeline quarantines the batch before the Gold view updates.
* **Unstructured Vector & RAG Pipelines:**
  Embedding models have strict structural contracts too. Changing an embedding model (e.g., migrating from `text-embedding-3-small` to a newer dense model) or altering chunk-overlap strategies without re-indexing the entire corpus creates **vector space drift**—causing similarity search to silently return irrelevant context. Version your vector collections (`policies_v2_1536d`) and enforce metadata schema validation (`doc_id`, `effective_from`, `status`, `tenant_id`) on every chunk prior to upsert.

---

## 3.4 Freshness SLAs and Temporal Integrity

An accurate answer built on yesterday's state is still a wrong answer in production.

Agents operating in dynamic enterprise environments—incident response, inventory allocation, financial reconciliation, or customer support—face two acute temporal hazards: **stale context** and **conflicting historical versions.**

### 1. Enforcing Explicit Freshness SLAs
Every Gold dataset and knowledge index exposed to an agent must carry an explicit **Time-to-Live (TTL) / Freshness SLA** enforced deterministically by the retrieval tool, not left to the LLM's judgment:

* **Real-Time Operational State (SLA: `< 1 minute`):** Service health metrics, active incident locks, and account balances must be queried directly from transactional stores or low-latency caches—never from overnight batch tables or static vector embeddings.
* **Policy & Knowledge Documents (SLA: `< 24 hours`):** Standard operating procedures, pricing tiers, and compliance rules can live in a vector store, provided every chunk carries an `indexed_at` timestamp and an explicit `is_active` lifecycle flag.

If a data pipeline stalls and a Gold table breaches its freshness SLA, the tool wrapper must return a structured staleness warning (`{"status": "STALE_DATA", "last_updated": "2026-11-14T02:00:00Z", "sla_minutes": 60}`) so the agent's policy engine can halt automated execution and escalate to a human operator.

### 2. Solving the "Superseded Policy" Trap
When an organization updates its refund policy or runbook, simply uploading `Runbook_v2.pdf` into a vector database without archiving `Runbook_v1.pdf` guarantees intermittent hallucinations. Because both documents share 85% of the same vocabulary, semantic search will routinely retrieve chunks from *both* versions into the same context window, forcing the LLM to guess which rule applies.

Temporal integrity requires **hard lifecycle state transitions** in the Silver-to-Gold pipeline: whenever a document with a matching `canonical_doc_id` is ingested with a newer `effective_date`, the ingestion pipeline must atomically mark all prior chunks as `status = 'SUPERSEDED'` and exclude

---
## 3.5 Automated Data Quality Gates Before Indexing

Just as CI/CD pipelines block broken code from reaching production, **Data Quality (DQ) Gates** must block corrupted, incomplete, or unredacted data from entering Gold AI stores.

Before any batch of records or document chunks is promoted from Silver to Gold, an automated validation gate should execute four deterministic checks:

| Quality Dimension | What the Gate Validates | Failure Action |
| :--- | :--- | :--- |
| **Completeness & Null Checks** | Mandatory fields (`tenant_id`, `effective_date`, `chunk_text`, `primary_key`) are non-null and meet minimum length thresholds. | Quarantine malformed records; alert data engineering owner. |
| **Referential & Ground-Truth Integrity** | Foreign keys align across joined tables; numeric aggregates match source system control totals within tolerance. | Block Gold table swap; preserve previous known-good snapshot. |
| **PII & Secret Redaction** | Regex and NER scanners verify no raw API keys, passwords, SSNs, or unmasked credentials survived Silver scrubbing. | Hard-fail pipeline immediately; purge raw buffer and log security event. |
| **Embedding & Retrieval Sanity** | Newly generated vectors contain no `NaN`/zero-norm anomalies and pass a canary similarity query against golden test prompts. | Hold index alias promotion until canary retrieval accuracy passes threshold. |

### Reference Implementation: A Silver-to-Gold Context Gate

Here is how a lightweight, deterministic validation gate inspects document chunks before allowing them into a production RAG index:

```python
from datetime import datetime, timezone
from pydantic import BaseModel, Field, field_validator
import re

SECRET_PATTERN = re.compile(r"(sk-[a-zA-Z0-9]{20,}|Bearer\s+[a-zA-Z0-9\-\._~\+\/]+=*)")

class GoldContextChunk(BaseModel):
    chunk_id: str
    canonical_doc_id: str
    tenant_id: str
    status: str = Field(pattern="^ACTIVE$")
    effective_date: datetime
    content: str = Field(min_length=50, max_length=4000)

    @field_validator("content")
    @classmethod
    def reject_unredacted_secrets(cls, text: str) -> str:
        if SECRET_PATTERN.search(text):
            raise ValueError("SECURITY_GATE_FAILED: Unredacted credential detected in chunk.")
        return text

    @field_validator("effective_date")
    @classmethod
    def reject_future_dated_policies(cls, dt: datetime) -> datetime:
        if dt > datetime.now(timezone.utc):
            raise ValueError("TEMPORAL_GATE_FAILED: Document effective_date is in the future.")
        return dt

def promote_to_gold(raw_chunks: list[dict]) -> tuple[list[GoldContextChunk], list[dict]]:
    approved, quarantined = [], []
    for item in raw_chunks:
        try:
            approved.append(GoldContextChunk(**item))
        except Exception as exc:
            quarantined.append({"record": item.get("chunk_id"), "reason": str(exc)})
    return approved, quarantined
By placing deterministic gates like promote_to_gold ahead of your vector database and tool views, you eliminate an entire class of production hallucinations before the LLM ever sees a single token.

!!! abstract "Chapter 3 Key Takeaways"
* Data quality is the primary AI guardrail: Most production hallucinations stem from corrupted, stale, or ambiguous upstream data rather than foundation model flaws.
* Adopt the AI Medallion Architecture: Isolate raw Bronze inputs from agents; promote data through Silver cleaning/redaction into versioned, access-controlled Gold context stores.
* Enforce contracts across SQL and vectors: Protect structured tool views from silent schema drift and version vector indexes whenever embedding models or chunking rules change.
* Guard temporal integrity: Enforce explicit freshness SLAs at the tool layer and atomically deprecate superseded documents so conflicting policy versions never share a context window.
* Gate promotion with deterministic code: Validate completeness, temporal validity, and secret redaction before indexing any record into production.


---


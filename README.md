# AI Place-Based Q&A Platform (Insurance Analyst Demo)

A portfolio-grade solution architecture + runnable system that lets analysts ask **place-based risk questions in natural language**, uses an **LLM (Amazon Bedrock)** to generate **safe SQL**, queries curated **serving views in Amazon Redshift**, and returns a grounded answer in a **web UI**.

This project is prepared for applying for [Solution Architect (Principal) - Core Data Platforms & Cloud FinOps role](https://www.linkedin.com/jobs/view/4366420855/?alternateChannel=search&eBP=CwEAAAGcCQAU2d8JJfSIZ-PkF3_k5GqHp0OcDkrfKasXZRJKPor6Se6XQcqIHo23nANZGe3GlLoVY6yDUoONQu544UpGrCwpxFN0weheo2bolvNMNZCdVVg6uEqKuGC_--7FyYncRcuFPT7JxcKpcSl5Vt8RgkapLBtFTE5kA1ahYaGzUBsPa5epLbeq2EBVDbkmyIsC5TvjC3-Be4T3ucaNiH2ww3xYapY4uxvqjP0r7hv6EpIactfLciIW4jcJL5EOPDxSVVIgtJjLMyOJF_m5gmPKg7hQgCXwbduG9qlQT39cJLQnyAyeHQPu0tnJ4VLWk2gqhnioI23-pCsYGyyuxjG4Ge3jSlxn_jZM2yjJpQkvxFczizuQDRnAWPrw3m5dJFRIQiSp_iNU9meMjoM2dlXFxm_60CONo9TdpWeg1cMR3T5FjcXMKusoiURwZv8y7gKvnEjfYSvNcdEDRG2jatFsg52McnfodnJkXCzGUHe9Iq6A8w5onss_AYj4JqmCDg8TS_XSV1Gs1R7gRELVpNQ9E-ZKM6eCUImKAvs&refId=T%2BPJUPsRocVR28PGcKiM6g%3D%3D&trackingId=%2BsYdO%2BXTxZUzNq%2BrjcE7EA%3D%3D) at Suncorp.

The project is inspired by the research paper [The semantics of place-related questions](https://josis.org/index.php/josis/article/view/161)

This project is designed to demonstrate Solution Architect skills across:
- data ingestion patterns (incremental, CDC-inspired, schema evolution)
- lakehouse + warehouse design (Databricks + Delta Lake + Redshift)
- governance + operational excellence (policy-as-code, quality gates, lineage-ready)
- secure cloud architecture on AWS (VPC, IAM, KMS, Secrets, MWAA, ECS/Fargate, ALB)
- CI/CD for infra + apps using GitHub Actions (OIDC, no long-lived keys)

---

## Table of Contents
- [Project Scope](#project-scope)
- [Solution Design](#solution-design)
- [Data Model](#data-model)
- [AI (NL→SQL) Design](#ai-nlsql-design)
- [Infrastructure Architecture](#infrastructure-architecture)
- [Security & Governance](#security--governance)
- [Operational Excellence](#operational-excellence)
- [Repository Structure](#repository-structure)
- [Roadmap](#roadmap)
- [Disclaimer](#disclaimer)

---

## Project Scope
## Expected User Scenario

### Primary user
**Insurance analytics / pricing / portfolio risk analyst** (or a Solution Architect demoing to an analytics stakeholder).

### Analyst goal
Quickly answer **place-based risk questions** using natural language—without needing to know table schemas, write SQL, or manually join multiple datasets—while still keeping results **auditable and governed**.

### Typical workflow (happy path)
1. The analyst opens the **web UI**.
2. They type a question such as:
   - “Top 10 LGAs in QLD by maximum flood exposure.”
   - “Compare Brisbane City vs Ipswich for bushfire exposure.”
   - “Which LGAs had high flood exposure in the 2011 event?”
3. The system:
   - Uses **Amazon Bedrock** to translate the question into **read-only SQL** against allowlisted **semantic views**.
   - Validates SQL (SELECT-only, table allowlist, LIMIT, etc.).
   - Queries **Amazon Redshift** and returns a result table.
4. The UI displays:
   - The generated SQL (expandable for transparency)
   - A short, grounded narrative summary based on returned rows
   - A results table (sortable/filterable)
   - A sidebar panel showing **data freshness** and pipeline status.

### Ambiguity handling (clarification loop)
If a place name is ambiguous (e.g., “Brisbane City” could mean **LGA** vs **suburb**), the assistant asks a follow-up question:
- “Do you mean Brisbane City LGA or the suburb Brisbane City?”
The analyst selects/clarifies, and the system regenerates SQL using the correct `area_id`.

### Operational scenario (data platform view)
A data engineer / platform owner uses Airflow/MWAA to:
- Confirm daily ingestion/transform jobs succeeded
- Trigger **backfills** for specific hazard events if needed
- Investigate schema change alerts or data quality gate failures
- Monitor freshness SLAs via `gold.data_freshness_v`

### What “success” looks like
- Analysts can answer common portfolio risk questions in seconds
- Queries are **safe, bounded, repeatable**, and tied to curated views
- The platform is **operationally credible**: scheduled pipelines, quality gates, and monitoring are present

### What this system does
1. **Ingests public geospatial datasets** (boundaries + hazards) into a lakehouse on **S3 + Delta Lake (Databricks)**.
2. Builds **curated “Gold” metrics** (e.g., hazard exposure per LGA/event) and publishes **serving tables/views** into **Amazon Redshift**.
3. Provides a **web UI** for analysts to ask natural-language questions.
4. Uses **Amazon Bedrock** to translate questions into **read-only, allowlisted SQL**.
5. Executes SQL against **Redshift semantic views** and returns **tables + narrative answers**.

### What’s included (MVP)
- **Geography:** Queensland-focused (e.g., Brisbane City LGA, Ipswich)
- **Datasets (public):**
  - ABS ASGS boundaries (LGA/SA2) for area geometries
  - Flood extent datasets (QLD)
  - Historical bushfire boundaries (Australia) filtered to QLD
- **Orchestration:** Airflow (local Docker for dev + **Amazon MWAA** for AWS)
- **Serving warehouse:** **Amazon Redshift Serverless**
- **UI + API:** **ECS + Fargate** behind **ALB**
- **LLM:** **Amazon Bedrock**
- **IaC:** Terraform (infra provisioning) + GitHub Actions (OIDC-based CI/CD)

### Out of scope (initially)
- Private insurance claims/PII data
- Advanced access control (Lake Formation) beyond a solid IAM baseline
- Full enterprise catalog integrations (Collibra/Purview/Alation). We design for it and can add DataHub/OpenMetadata later.

---

## Solution Design

### End-to-end workflow
**(1) Ingest → (2) Standardize → (3) Compute exposure → (4) Serve → (5) Ask questions**

1. **Ingestion (Bronze on S3)**
   - Airflow downloads public datasets and stores them in S3 as immutable raw assets.
   - Each ingest produces a **manifest** (source URL, hash, size, timestamp) to support idempotency and auditability.

2. **Lakehouse processing (Databricks + Delta Lake)**
   - Databricks reads manifests and loads raw files.
   - Silver stage standardizes data types, geometry, keys, and partitions.
   - Gold stage computes exposure metrics:
     - boundary ∩ hazard polygons → exposed area
     - exposed ratio per area/event
     - summary aggregates over time

3. **Publish to Redshift (Serving layer)**
   - Gold outputs exported to S3 in columnar format.
   - Redshift loads using `COPY` (S3 → Redshift).
   - Semantic views (`gold.*_v`) provide a stable contract to the LLM.

4. **AI Q&A (Bedrock → SQL → Redshift)**
   - Analyst asks a question in UI.
   - Bedrock generates SQL against **allowlisted semantic views only**.
   - SQL validator enforces:
     - SELECT-only
     - allowlisted schemas/tables
     - LIMIT and anti-abuse checks
   - Redshift query executes; UI shows results + narrative.

---

## Data Model

### Lakehouse zoning (Delta on S3)
- **Bronze:** raw immutable datasets + manifests
- **Silver:** standardized boundaries, hazard events, hazard polygons
- **Gold:** exposure metrics and analyst-friendly aggregates

### Core Gold tables (conceptual)
- `gold.area_hazard_exposure`  
  Exposure metrics per **area × hazard × event**
- `gold.area_hazard_summary`  
  Aggregates per **area × hazard** (avg/max exposure, event counts)
- `gold.place_alias`  
  Minimal place-name resolution for MVP (boundary-name driven)
- `gold.data_freshness`  
  Dataset freshness + pipeline state

### Redshift semantic views (LLM query contract)
Only these views are exposed to the LLM:
- `gold.place_search_v`
- `gold.area_dim_v`
- `gold.hazard_event_dim_v`
- `gold.area_hazard_exposure_v`
- `gold.area_hazard_summary_v`
- `gold.data_freshness_v`

---

## AI (NL→SQL) Design

### Guardrails (must-have)
- **Read-only SQL**: SELECT only, no DDL/DML
- **Allowlist**: only `gold.*_v` views
- **LIMIT enforcement**: default `LIMIT 100`
- **No metadata fishing**: block `information_schema`, `pg_catalog`, etc.
- **Clarification workflow**:
  - If place is ambiguous (e.g., “Brisbane City” could be suburb vs LGA), the model returns a **clarifying question**.

### Two-stage reasoning (simple & safe)
1. **Resolve place** (optional): query `gold.place_search_v`
2. **Generate main SQL** using the resolved `area_id/area_type`

### Output contract
The LLM must output JSON:
- Query:
  ```json
  {"intent":"query","sql":"SELECT ...","result_explanation":"..."}
  ```
- Clarify:
  ```json
  {"intent":"clarify","question":"Do you mean Brisbane City LGA or the suburb Brisbane City?"}
  ```

Infrastructure Architecture

AWS building blocks
	•	VPC: 2 public + 2 private subnets (multi-AZ)
	•	Public: ALB, NAT
	•	Private: MWAA, ECS tasks, Redshift
	•	S3 buckets:
	•	*-airflow-dags (MWAA DAG bucket)
	•	*-data-lake (Bronze/Silver/Gold)
	•	*-exports (Databricks exports for Redshift COPY)
	•	Redshift Serverless: serving warehouse for Q&A
	•	MWAA: production Airflow
	•	ECS + Fargate: API + UI services
	•	ALB: HTTPS entrypoint to UI/API
	•	Bedrock: NL→SQL generation + answer summarization
	•	Secrets Manager + KMS: secure secret storage + encryption
	•	CloudWatch: logs, metrics, alarms

Execution topology
	•	Dev iteration:
	•	Local Docker Airflow for fast DAG development
	•	AWS run:
	•	MWAA schedules ingestion/transform/publish
	•	ECS runs API/UI continuously
	•	Redshift serves fast queries

⸻

Security & Governance

Security principles
	•	Least privilege IAM:
	•	MWAA execution role: only what it needs (S3, logs, KMS as required)
	•	ECS task role: secretsmanager:GetSecretValue, bedrock:InvokeModel, Redshift access
	•	Redshift S3 role: read only from exports bucket
	•	Network segmentation:
	•	Redshift and ECS tasks in private subnets
	•	ALB public, targets private
	•	Encryption by default:
	•	S3 SSE-KMS
	•	Redshift encryption
	•	Secrets Manager with KMS CMK
	•	No long-lived keys in GitHub:
	•	GitHub Actions uses OIDC → AssumeRole to run Terraform and deploy apps

Governance (MVP)
	•	Manifest-based audit trail for ingestion
	•	Schema discovery + change detection in pipeline (alerts on breaking changes)
	•	Data quality gates before publishing to serving layer

Operational Excellence

Reliability patterns
	•	Immutable Bronze + recomputable Silver/Gold
	•	Partitioning by hazard_type/event_id/as_of_date
	•	Backfill DAG for reprocessing specific events/time ranges

Observability
	•	gold.data_freshness_v surfaced in UI sidebar
	•	CloudWatch logs for MWAA and ECS services
	•	Optional alarms:
	•	MWAA DAG failures
	•	Redshift query latency thresholds
	•	“freshness SLA breached” indicator
Repository Structure

place-qa-platform/
  airflow/          # DAGs for local & MWAA
  databricks/       # Notebooks/jobs for lakehouse transforms
  serving/          # Redshift DDL + semantic views
  app/
    api/            # FastAPI service (Bedrock + SQL guard + Redshift)
    ui/             # Streamlit UI
  infra/terraform/  # IaC for AWS resources

  Roadmap

MVP (run end-to-end)
	•	Terraform: VPC + S3 + KMS + Secrets + Redshift Serverless
	•	Terraform: MWAA environment + execution role
	•	Terraform: ECR + ECS + ALB for API/UI
	•	MWAA: DAG upload + requirements
	•	Databricks: Silver/Gold pipeline with exposure computation
	•	Redshift: serving tables + semantic views
	•	API: Bedrock NL→SQL + SQL validator + query execution
	•	UI: question input + SQL transparency + results + freshness panel

Disclaimer

This is a demo platform using public datasets and synthetic/derived metrics. It is not a production risk model. Any risk inference shown in the UI is for demonstration only.
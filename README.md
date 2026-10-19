
# Churn Risk Signal Pipelines (Relationship + Experience)

> Two production LLM-powered NLP pipelines that automatically detect and classify churn-risk signals from sales call transcripts — surfacing relationship risks (key personnel departures) and experience risks (product/support failures) with verbatim evidence for Customer Success intervention.

![SQL](https://img.shields.io/badge/Databricks_SQL-AI_Functions-blue)
![Claude](https://img.shields.io/badge/Claude_Sonnet-LLM_Engine-orange)
![NLP](https://img.shields.io/badge/NLP-Risk_Classification-purple)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-Incremental_MERGE-green)
![Databricks](https://img.shields.io/badge/Databricks-Azure-red)

---

## Problem Statement

Customer Success managers can’t listen to every call, but churn signals are buried in transcripts:

* A key executive mentions they’re leaving the company
* A customer describes recurring product bugs or failed integrations
* An account admits they haven’t deployed the product they purchased

These two pipelines scan **every external customer call** and surface evidence-grounded risk flags so CSMs can act before renewal.

---

## Architecture: Shared 4-Step Incremental Pattern

Both pipelines follow the same idempotent design:

```text
┌─────────────────────────────────────────────────────────────┐
│  Step 1: DATE WINDOWING                                      │
│  SQL session variables (start_date, end_date)                │
│  Default: last-processed date → today (incremental)          │
│  Parameterized via notebook widgets for backfills             │
└─────────────────────────────┬───────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 2: STAGE via ai_query()                                 │
│  CREATE OR REPLACE TABLE ...stg AS                           │
│  ai_query('claude-sonnet', transcript, responseFormat =>     │
│           JSON Schema (strict: true), max_tokens => 8192)    │
└─────────────────────────────┬───────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 3: INCREMENTAL MERGE                                    │
│  MERGE INTO ...final USING ...stg ON id = id                 │
│  WHEN NOT MATCHED THEN INSERT *                              │
│  (append-only dedup — re-runs are safe)                      │
└─────────────────────────────┬───────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  Step 4: EXPLODE TO FLAT TABLE                                │
│  LATERAL VIEW EXPLODE(parse_json(response)                   │
│    :CSM_Renewal_Detected_Risks)                              │
│  → One row per risk per call for dashboards                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Risk Categories

### Pipeline A: Relationship Risks

| Risk Type | Signal |
|---|---|
| **Economic Buyer / Decision Maker Churn** | Executive who authorized the purchase has left — successor may trigger vendor review |
| **Champion Churn** | Internal advocate who communicated value upward has departed or disengaged |
| **Technical Influencer Churn** | Day-to-day operator who runs the platform has departed — replacement has no experience |

### Pipeline B: Experience Risks

| Risk Type | Signal |
|---|---|
| **Support Issues / Poor Experience** | High ticket volume, slow resolution, repeated escalations |
| **Product Experience Issues / Bugs** | Broken integrations, inconsistent results, deprecated features |
| **Adoption Gap** | Customer hasn’t achieved expected outcomes — low deployment, shelfware risk |

---

## Tech Stack

| Layer | Tool |
|---|---|
| **AI Engine** | Databricks `ai_query()` with Claude Sonnet |
| **Output Enforcement** | JSON Schema with `strict: true`, enum-constrained risk categories |
| **Incremental Processing** | SQL session variables for date windowing, MERGE for safe re-runs |
| **JSON Parsing** | `parse_json()`, VARIANT path extraction (`:key::type`), `LATERAL VIEW EXPLODE` |
| **Storage** | Delta Lake (column mapping mode) |
| **Platform** | Databricks SQL Warehouse (Photon PRO), Unity Catalog |

---

## Prompt Engineering Highlights

* **6 Core Rules** per pipeline: ground in transcript, require verbatim quote, validate speaker authority, evaluate independently, default to empty array (not false positives), map precisely to persona/category definitions
* **Explicit disambiguation**: Relationship pipeline distinguishes budget authority (Economic Buyer) vs. advocacy (Champion) vs. daily operations (Technical Influencer)
* **Negative examples**: Each category includes "Do NOT flag" cases to suppress false positives
* **Multi-risk detection**: One transcript can flag multiple risks across categories
* **Confidence + Strength scoring**: Each detected risk rated Low/Medium/High on both dimensions
* **Partner observations**: Partner-reported risks accepted with confidence capped at Medium

---

## Output Schema (Both Pipelines)

```json
{
  "Is_CSM_Renewal_Risk_Present": true,
  "CSM_Renewal_Risk_Summary": "...",
  "CSM_Renewal_Risk_Customer_Sentiment": "Positive | Negative | Neutral",
  "CSM_Renewal_Detected_Risks": [
    {
      "Category": "Champion Churn | Adoption Gap | ...",
      "Reasoning": "...",
      "Key_Verbatim_Quote": "...",
      "Strength": "Low | Medium | High",
      "Confidence": "Low | Medium | High"
    }
  ]
}
```

---

## Project Structure

```text
churn-risk-signal-pipelines/
├── pipelines/
│   ├── relationship_risk_identifier.sql   # Pipeline A: personnel departure detection
│   └── experience_risk_identifier.sql      # Pipeline B: product/support failure detection
├── prompts/
│   ├── relationship_risk_prompt.md         # 3-persona prompt with 6 Core Rules
│   └── experience_risk_prompt.md           # 3-category prompt with POV inclusion rule
├── docs/
│   └── pipeline_architecture.md           # Shared 4-step pattern documentation
└── README.md
```

---

## What I Learned

* **Idempotent pipelines save you at 2 AM**: The MERGE-based append-only pattern means re-running after a failure never creates duplicates — this eliminated an entire class of operational incidents
* **Enum constraints > free-text classification**: Constraining risk categories via JSON Schema enums means downstream SQL can GROUP BY without fuzzy matching
* **Separate pipelines > one mega-prompt**: Relationship and experience risks have different persona definitions, evidence standards, and false-positive patterns — two focused prompts outperform one overloaded prompt
* **Verbatim quotes build trust**: CSMs won’t act on "the model thinks there’s a risk" — but they will act on a direct customer quote with context
```

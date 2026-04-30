<div align="center">

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:FF6D00,50:FF9800,100:FFB300&height=220&section=header&text=GA4%20Dataform%20Pipeline&fontSize=52&fontColor=fff&animation=twinkling&fontAlignY=38&desc=Raw%20Events%20%E2%86%92%20Trusted%20Analytics%20%E2%86%92%20Real%20Decisions&descAlignY=60&descSize=18" />

<br/>

[![Typing SVG](https://readme-typing-svg.demolab.com?font=Fira+Code&weight=600&size=22&pause=1000&color=FF6D00&center=true&vCenter=true&width=700&lines=4-Layer+Dataform+%2B+BigQuery+Lakehouse;Incremental+Models+%2B+Partition+Management;Last-Touch+Attribution+%2B+Session+Modeling;GA4+Nested+Events+%E2%86%92+Clean+Reporting+Tables)](https://github.com/yashxsainix/ga4_dataform_pipeline)

<br/>

[![BigQuery](https://img.shields.io/badge/Google_BigQuery-4285F4?style=for-the-badge&logo=googlebigquery&logoColor=white)](https://cloud.google.com/bigquery)
[![Dataform](https://img.shields.io/badge/Dataform-FF6D00?style=for-the-badge&logo=google&logoColor=white)](https://cloud.google.com/dataform)
[![SQL](https://img.shields.io/badge/SQL-336791?style=for-the-badge&logo=postgresql&logoColor=white)](https://cloud.google.com/bigquery/docs/reference/standard-sql)
[![Google Ads](https://img.shields.io/badge/Google_Ads-4285F4?style=for-the-badge&logo=googleads&logoColor=white)](https://ads.google.com)
[![MIT License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)

<br/>

<img src="https://img.shields.io/badge/Data_Layers-4-FF6D00?style=flat-square&labelColor=1a1a1a" />
<img src="https://img.shields.io/badge/Output_Tables-3-FF9800?style=flat-square&labelColor=1a1a1a" />
<img src="https://img.shields.io/badge/Processing-Incremental-FFB300?style=flat-square&labelColor=1a1a1a" />
<img src="https://img.shields.io/badge/Attribution-Last_Non--Direct-FF6D00?style=flat-square&labelColor=1a1a1a" />

</div>

---

## 🧠 The Problem

GA4 exports a massive volume of nested, semi-structured event data into BigQuery. It is powerful but almost unusable in its raw form.

Every query requires unnesting arrays. Every session must be reconstructed from individual events. Traffic source logic is incomplete — GA4 credits direct sessions even when a meaningful prior channel exists. Marketing teams end up rebuilding the same logic in every report, inconsistently.

This project solves that once, correctly, in a reproducible pipeline.

---

## 🏗️ Architecture

```mermaid
graph TD
    A["📥 Source Layer<br/>GA4 events_* tables<br/>Google Ads transfer"] --> B["🔧 Staging Layer<br/>stg_events.sqlx<br/>stg_ads_click_stats.js<br/>Column casting + unnesting"]
    B --> C["⚙️ Intermediate Layer<br/>int_event.sqlx<br/>Business logic + attribution<br/>Session keys + timestamps"]
    C --> D["📊 Output Layer<br/>event · session · user_transaction_daily"]
    
    E["🔄 Incremental Logic<br/>Partition pruning<br/>Lookback windows<br/>Delta processing"] --> B
    F["📐 Dataform Orchestration<br/>Dependency graph<br/>Schema docs<br/>Assertions"] --> C

    style A fill:#FF6D00,color:#fff,stroke:#FF6D00
    style B fill:#FF9800,color:#fff,stroke:#FF9800
    style C fill:#FFB300,color:#fff,stroke:#FFB300
    style D fill:#4285F4,color:#fff,stroke:#4285F4
    style E fill:#1a1a1a,color:#FF6D00,stroke:#FF6D00
    style F fill:#1a1a1a,color:#FF9800,stroke:#FF9800
```

---

## ⚡ What This Pipeline Produces

<details>
<summary><b>📊 event — Event-level attribution table</b></summary>

One row per GA4 event, enriched with attribution-ready fields.

- Carries forward **last non-direct traffic source** when current session is direct
- Unnested event parameters: `ga_session_id`, `page_location`, `engagement_time_msec`, `gclid`
- Full cross-channel, SA360, CM360, and DV360 campaign fields
- Partition by `event_date`, clustered by `user_pseudo_id` and `ga_session_id`

</details>

<details>
<summary><b>📊 session — One row per user session</b></summary>

Session-level model for traffic source analysis and engagement reporting.

- Groups events into sessions with start time, engagement time, and session flag
- Applies **last non-direct attribution** at the session level
- Outputs `session_start_date`, `session_engaged`, `engagement_time_seconds`
- Ready for Looker Studio, Tableau, or Power BI connection

</details>

<details>
<summary><b>📊 user_transaction_daily — Purchase tracking by user</b></summary>

Ecommerce model aggregating purchase and refund metrics by user and date.

- Disabled by default — enable when `events` table contains purchase events
- Useful for cohort revenue analysis and LTV modeling

</details>

---

## 🔑 Key Business Logic

### Last Non-Direct Attribution
```sql
-- If current session is direct or source is missing,
-- look back to the most recent meaningful channel touchpoint
-- rather than crediting (direct) / (none) by default
```
This is one of the most misunderstood parts of GA4 reporting. Direct attribution inflates when users bookmark a site or type the URL. This model corrects that — giving credit to the channel that actually influenced the visit.

### Paid Traffic Source Correction
GA4 traffic source fields are incomplete when `gclid`, `gbraid`, or `wbraid` are present but campaign labels are missing. This pipeline detects Google Ads click identifiers and optionally joins Ads transfer data to recover campaign names.

### Incremental Processing
```sqlx
config {
  type: "incremental",
  bigquery: {
    partitionBy: "event_date",
    clusterBy: ["user_pseudo_id", "ga_session_id"]
  }
}

-- Only process new records + 3-day lookback
-- No full table rebuilds on daily runs
```

---

## 🗂️ Repository Structure

```
ga4_dataform_pipeline/
│
├── workflow_settings.yaml          ← GCP project, dataset config
├── includes/
│   ├── constants.js                ← Lookback windows, time zones, toggles
│   ├── helpers.js                  ← unnestColumn macro
│   └── non_custom_events.js        ← Standard GA4 event list
│
└── definitions/
    ├── sources/                    ← Raw table declarations
    │   ├── src_events.sqlx
    │   ├── src_ads_click_stats.js
    │   └── src_ads_campaign.js
    ├── staging/                    ← Column casting + unnesting
    │   ├── stg_events.sqlx
    │   └── stg_ads_*.js
    ├── intermediate/               ← Business logic + attribution
    │   ├── int_event.sqlx
    │   └── int_ads_click_campaign.js
    └── output/                     ← Reporting-ready tables
        ├── event.sqlx
        ├── session.sqlx
        └── user_transaction_daily.sqlx
```

---

## 🚀 Setup

**Option 1 — Dataform in Google Cloud Console**
```bash
# 1. Create a Dataform repository in GCP
# 2. Add these files to the repository
# 3. Update workflow_settings.yaml with your project and dataset names
# 4. Update includes/constants.js with your GA4 source dataset
# 5. Compile → Run
```

**Option 2 — Local Development**
```bash
npm install -g @dataform/cli
dataform init ga4_pipeline
# Replace definitions/ and includes/ with files from this repo
dataform compile
dataform run
```

**Required constants to configure:**
```javascript
// includes/constants.js
const SOURCE_DATASET = "your_ga4_dataset";
const REPORTING_TIME_ZONE = "America/New_York";
const START_DATE = "2024-01-01";
const GADS_GET_DATA = false; // set true to enable Google Ads join
```

---

## 💡 Business Questions This Answers

| Question | Output Table |
|---|---|
| Which channels drive the most engaged sessions? | `session` |
| What was the last meaningful channel before a direct conversion? | `event` |
| How do paid vs organic sessions compare in engagement? | `session` |
| Which campaigns influence conversions through multi-touch paths? | `event` |
| What is daily purchase revenue by user cohort? | `user_transaction_daily` |

---

## 👤 Author

**Yashpal Saini** · [LinkedIn](https://linkedin.com/in/yash-saini-analyst) · [Portfolio](https://yashxsainix.github.io)

<img src="https://capsule-render.vercel.app/api?type=waving&color=0:FF6D00,50:FF9800,100:FFB300&height=100&section=footer" />

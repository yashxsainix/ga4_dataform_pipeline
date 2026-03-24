# GA4 Dataform Analytics Pipeline

A Dataform + BigQuery project that transforms raw Google Analytics 4 export tables into clean, analysis-ready data models for **event analysis, session reporting, attribution, and ecommerce tracking**.

This repository is designed for teams that want to move beyond raw GA4 events and create a reporting layer that is easier for analysts, marketers, and business stakeholders to use.

## Why this project matters

GA4 exports a large volume of nested event data into BigQuery. That data is powerful, but it is not easy for most teams to work with directly.

This project solves that problem by turning raw GA4 export tables into structured reporting models that answer questions such as:

- Which channels are actually driving sessions and conversions?
- What was a user's last non-direct source before a conversion?
- How do session-level and event-level performance differ?
- Which campaigns, landing pages, or traffic sources are contributing to revenue?
- How can marketing and product teams work from one reliable analytics layer instead of rebuilding logic every time?

In simple terms, this project takes messy behavioral data and turns it into something decision-makers can understand and use.

## Non-technical explanation

Imagine GA4 data like a giant warehouse full of unlabeled boxes.

The data is all there, but it is hard to find what matters quickly. This project acts like an operations team that:

1. opens the boxes,
2. sorts the contents,
3. labels everything clearly,
4. connects related records together,
5. and places the most important information into easy-to-use shelves for reporting.

The result is a cleaner system where analysts can answer business questions faster, marketers can trust attribution more, and leadership can review performance without digging through raw event logs.

## What the pipeline does

The project uses **Dataform** to orchestrate SQL-based transformations in **BigQuery**.

It reads raw GA4 export tables from BigQuery and builds a layered analytics model:

### 1) Source layer
Declares the raw data sources used by the project.

- `events_*` from the GA4 BigQuery export
- Optional Google Ads transfer tables for click and campaign enrichment

### 2) Staging layer
Standardizes the raw event records and extracts common fields from nested GA4 event parameters.

Examples include:
- session ID
- page location
- session number
- engagement time
- page title
- page referrer
- event traffic source fields

### 3) Intermediate layer
Applies business logic that improves usability and attribution quality.

This layer:
- converts timestamps into the reporting time zone
- creates user and session keys
- standardizes traffic source fields
- improves paid traffic source handling when `gclid`, `gbraid`, or `wbraid` are present
- optionally enriches GA4 events with Google Ads campaign names

### 4) Output layer
Publishes reporting-ready tables for downstream analysis.

The main outputs are:
- **event**: one row per event, with attribution-ready fields
- **session**: one row per session, with session-level traffic source and engagement metrics
- **user_transaction_daily**: purchase and refund metrics by user and date, currently disabled by default

## Key business logic included

### Last non-direct attribution
One of the most useful parts of the project is the **last non-direct traffic source** logic.

If a conversion happens during a direct session, this model looks back to find the most recent meaningful channel touchpoint rather than crediting direct traffic by default.

This creates a more practical reporting layer for:
- channel performance reviews
- campaign effectiveness analysis
- conversion path analysis
- budget allocation discussions

### Paid traffic source correction
GA4 traffic source fields are not always enough on their own, especially when click IDs exist but campaign labels are incomplete.

This project adds logic to:
- detect Google Ads click identifiers such as `gclid`
- classify paid traffic more consistently
- optionally join Google Ads transfer data to recover campaign names

### Incremental processing
The intermediate models are designed to run incrementally.

That means the pipeline does not need to rebuild all historical data every time. Instead, it processes new records and refreshes a recent lookback window, which is a more scalable approach for growing analytics datasets.

## Repository structure

```text
.
├── workflow_settings.yaml
├── includes/
│   ├── constants.js
│   ├── helpers.js
│   └── non_custom_events.js
└── definitions/
    ├── sources/
    │   ├── src_events.sqlx
    │   ├── src_ads_click_stats.js
    │   └── src_ads_campaign.js
    ├── staging/
    │   ├── stg_events.sqlx
    │   ├── stg_ads_click_stats.js
    │   └── stg_ads_campaign.js
    ├── intermediate/
    │   ├── int_event.sqlx
    │   └── int_ads_click_campaign.js
    └── output/
        ├── event.sqlx
        ├── session.sqlx
        └── user_transaction_daily.sqlx
```

## Main outputs explained

### `event`
A cleaned event-level table for detailed behavioral analysis.

Useful for:
- event trend analysis
- page-level or campaign-level investigation
- conversion event analysis
- attribution studies

Notable logic:
- carries forward the last non-direct source when current traffic source is direct or missing
- keeps event-level context for detailed drill-downs

### `session`
A session-level model with one row per user session.

Useful for:
- session reporting
- engagement analysis
- traffic source comparison
- channel mix reviews

Notable logic:
- groups events into sessions
- calculates session start time, engagement time, and session engagement flag
- keeps session traffic source information in a reporting-friendly format

### `user_transaction_daily`
A user-by-date purchase summary model, disabled by default.

Useful for:
- ecommerce analysis
- user revenue tracking
- purchase and refund summaries

## Configuration

Project settings are controlled through `workflow_settings.yaml` and `includes/constants.js`.

### `workflow_settings.yaml`
Used to define:
- default GCP project
- BigQuery location
- default dataset names
- assertion dataset
- Dataform core version

### `includes/constants.js`
Used to define:
- source dataset for GA4 exports
- reporting time zone
- staging, intermediate, and output datasets
- start date for backfill
- attribution lookback windows
- optional Google Ads data transfer settings

Important constants include:

- `SOURCE_DATASET`: raw GA4 export dataset
- `REPORTING_TIME_ZONE`: reporting time zone for event and session timestamps
- `START_DATE`: historical start point for the pipeline
- `ALL_EVENTS_LOOKBACK_WINDOW`: lookback window for non-acquisition conversion attribution
- `AQUISITION_EVENTS_LOOKBACK_WINDOW`: shorter lookback window for first-touch acquisition events
- `GADS_GET_DATA`: toggle for optional Google Ads enrichment

## How to run the project

### Option 1: Use Dataform in BigQuery
1. Create or open a Dataform repository in Google Cloud.
2. Add the files from this project.
3. Update `workflow_settings.yaml` with your project, location, and dataset names.
4. Update `includes/constants.js` with your GA4 source dataset, reporting time zone, and optional Google Ads settings.
5. Make sure the Dataform service account has access to the source and target BigQuery datasets.
6. Compile and run the project.

### Option 2: Use Dataform with local development
1. Set up a Dataform-compatible project environment.
2. Update the configuration files.
3. Validate references and dataset access.
4. Run compilation and execution against your BigQuery environment.

## Recommended setup steps

Before running the project, confirm the following:

- GA4 export is already connected to BigQuery
- your source dataset contains `events_*` tables
- your reporting time zone matches the GA4 property expectation
- the service account can read source datasets and write target datasets
- if using Google Ads enrichment, the Ads transfer tables are available and accessible

## Example business questions this project can answer

Once the output tables are built, analysts can answer questions like:

- Which traffic sources generate the most engaged sessions?
- Which non-direct channels influence conversions before users return directly?
- Which campaigns or landing pages are creating strong engagement but weak conversion?
- How much purchase revenue is associated with different user cohorts or acquisition sources?
- How do session trends change over time by source, medium, or campaign?

## Example analyst workflow

A marketing analyst could use this project in the following way:

1. Pull session trends from the `session` table.
2. Compare source and medium performance.
3. Use the `event` table to inspect conversion paths and landing page interactions.
4. Use attribution fields to understand which campaigns influenced outcomes.
5. Build a dashboard in Looker Studio, Tableau, or another BI tool for weekly stakeholder reviews.

## Technical highlights

- Built with **Dataform** for SQL orchestration and dependency management
- Uses **BigQuery** as the analytical warehouse
- Applies **incremental model logic** for scalable processing
- Extracts nested GA4 parameters into analysis-friendly columns
- Standardizes traffic source fields for reporting consistency
- Supports optional **Google Ads** campaign enrichment via transferred Ads data
- Produces session- and event-level models for both detailed analysis and executive reporting

## Why this is a strong analytics engineering / strategy project

This project is valuable because it sits at the intersection of:

- **analytics engineering**: building reliable transformation logic
- **marketing analytics**: improving channel and campaign visibility
- **business operations**: creating reporting layers decision-makers can actually use
- **growth analysis**: making attribution and performance data easier to interpret

It is not just a data cleanup exercise. It creates a reusable analytics foundation that can support:
- campaign reviews
- growth planning
- budget discussions
- product and marketing performance analysis
- executive reporting

## Limitations

A few things to keep in mind:

- The quality of the final models still depends on the quality of the original GA4 implementation.
- Attribution logic is rule-based and should be reviewed against the business's reporting standards.
- Google Ads enrichment is optional and requires transfer data availability.
- The ecommerce model is disabled by default and should be enabled only when needed.
- Additional custom event mappings may be required for specific product or business setups.

## Potential future improvements

Possible next steps for extending this project include:

- adding custom channel grouping logic
- building cohort retention models on top of session outputs
- creating reusable KPI views for dashboards
- adding dbt-style tests or more Dataform assertions for key fields
- publishing a Looker Studio dashboard on top of the output tables
- adding stakeholder-ready business review templates

## Who this project is for

This project is useful for:
- marketing analysts
- growth analysts
- analytics engineers
- product analysts
- BI developers
- teams using GA4 + BigQuery for performance reporting

It is especially relevant for organizations that want a cleaner way to analyze digital performance without manually rewriting GA4 logic in every report.

## Resume-ready project summary

Here is a concise way to describe this project in a resume or portfolio:

**Built a Dataform and BigQuery pipeline that transformed raw GA4 exports into session-, event-, and attribution-ready reporting models, improving the usability of marketing performance data for channel analysis, campaign reporting, and business decision-making.**

## Final takeaway

This repository demonstrates how to turn raw GA4 event exports into a structured analytics layer that supports both technical analysis and business reporting. It helps bridge the gap between data collection and decision-making by making attribution, session behavior, and performance trends easier to understand and act on.

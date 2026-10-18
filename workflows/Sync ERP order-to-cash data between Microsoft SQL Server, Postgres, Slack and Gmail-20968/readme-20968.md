Sync ERP order-to-cash data between Microsoft SQL Server, Postgres, Slack and Gmail

https://n8nworkflows.xyz/workflows/sync-erp-order-to-cash-data-between-microsoft-sql-server--postgres--slack-and-gmail-20968


# Sync ERP order-to-cash data between Microsoft SQL Server, Postgres, Slack and Gmail

### 1. Workflow Overview

This workflow automates the Order-to-Cash (O2C) data pipeline by extracting open sales orders from a Microsoft SQL Server ERP, normalizing and upserting them into a PostgreSQL data mart, evaluating financial risks, and alerting stakeholders via Slack and Gmail. It executes on a daily schedule and contains a robust error-handling subprocess.

The logic is grouped into the following functional blocks:
- **1.1 Schedule & Configuration:** Triggers the pipeline daily and establishes global execution variables (run IDs, lookback windows, thresholds, and alert channels).
- **1.2 Extraction & Normalization:** Pulls open sales orders from the ERP using parameterized queries, standardizes data into a canonical model, and deduplicates entries.
- **1.3 Batch Processing & Data Mart Persistence:** Iterates through records in controlled batches, upserts them idempotently into PostgreSQL, and executes risk evaluations.
- **1.4 Alerting & Auditing:** Evaluates credit risks and high-value order thresholds, notifying the finance team via Slack and logging audit events.
- **1.5 Reporting & Summary:** Compiles key performance indicators (KPIs) from the data mart, generates an HTML and Slack daily digest, emails stakeholders, and writes final execution summaries.
- **1.6 Error Handling:** Captures runtime failures, formats error diagnostics, logs them to an audit table, and pages the platform engineering team via Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration
- **Overview:** Initializes the daily execution context and defines static parameters governing lookback periods, thresholds, and notification routing destinations.
- **Nodes Involved:** `Schedule: Daily 06:30 (Berlin)`, `Configure Run`
- **Node Details:**
  - **Schedule: Daily 06:30 (Berlin)**
    - *Type & Role:* `Schedule Trigger` — Initiates the workflow daily at 06:30 Europe/Berlin time.
    - *Configuration:* Interval configured to trigger at hour 6, minute 30.
    - *Inputs:* None (Entry point)
    - *Outputs:* `Configure Run` (main index 0)
    - *Edge Cases:* Ensure n8n server timezone matches or accounts for the intended execution offset.
  - **Configure Run**
    - *Type & Role:* `Set (Edit Fields)` — Centralizes global variables for the execution run.
    - *Configuration:* Manual assignment mode mapping variables: `runId` (`$now.toFormat('yyyyLLdd-HHmm')`), `lookbackDays` (7), `highValueThreshold` (25000), `financeChannelId` (`C0123456789`), `alertEmail` (`user@example.com`), `erpCompanyCode` (`US01`).
    - *Inputs:* `Schedule: Daily 06:30 (Berlin)`
    - *Outputs:* `ERP: Fetch Open Sales Orders` (main index 0)
    - *Edge Cases:* Missing configuration fields will cause downstream expression lookups using `$('Configure Run').first().json` to fail.

#### 2.2 Extraction & Normalization
- **Overview:** Connects to the ERP database to query open orders based on the configured lookback window, normalizes the relational output into a canonical format, and removes duplicates.
- **Nodes Involved:** `ERP: Fetch Open Sales Orders`, `Normalize ERP Rows`
- **Node Details:**
  - **ERP: Fetch Open Sales Orders**
    - *Type & Role:* `Microsoft SQL` — Executes a read-only aggregation query against the ERP database replica.
    - *Configuration:* Executes a parameterized SQL query filtering orders by `OrderDate >= DATEADD(day, -{{ $json.lookbackDays }}, GETDATE())` and excluding shipped/cancelled statuses.
    - *Inputs:* `Configure Run`
    - *Outputs:* `Normalize ERP Rows` (main index 0)
    - *Edge Cases:* Must use a read-only replica connection to prevent locking production master databases. Schema column mismatches will trigger query execution errors.
  - **Normalize ERP Rows**
    - *Type & Role:* `Code (JavaScript)` — Maps raw ERP database fields to a canonical data model and deduplicates by `orderId`.
    - *Configuration:* JavaScript mapping script utilizing `Set` objects for deduplication, calculating `creditUtilisationPct`, and applying boolean `creditRisk` flags.
    - *Inputs:* `ERP: Fetch Open Sales Orders`
    - *Outputs:* `Batch Over Orders` (main index 0)
    - *Edge Cases:* Unhandled data type conversions (e.g., null numeric values) will evaluate as `0` via fallback statements.

#### 2.3 Batch Processing & Data Mart Persistence
- **Overview:** Splits normalized records into manageable batches of 100 to optimize transaction sizing and upserts records idempotently into the PostgreSQL data mart.
- **Nodes Involved:** `Batch Over Orders`, `Upsert Order into Mart`
- **Node Details:**
  - **Batch Over Orders**
    - *Type & Role:* `Split In Batches` — Iterates through input records in batches to control database load.
    - *Configuration:* Batch size set to `100`. Output index `1` loops data to the persistence node; output index `0` triggers downstream reporting once batches are exhausted.
    - *Inputs:* `Normalize ERP Rows`, `Continue Batch Loop`
    - *Outputs:* `Upsert Order into Mart` (batch loop, index 1), `Query Mart Totals` (done, index 0)
    - *Edge Cases:* Infinite loops can occur if batch pointers fail to reset properly; ensure proper connection routing from downstream loopgers.
  - **Upsert Order into Mart**
    - *Type & Role:* `Postgres` — Persists and updates order records in the data mart using idempotent upserts.
    - *Configuration:* Parameterized SQL execution running an `INSERT INTO erp.order_sync (...) VALUES (...) ON CONFLICT (order_id) DO UPDATE SET ...` statement returning affected attributes.
    - *Inputs:* `Batch Over Orders`
    - *Outputs:* `Credit Block or High Value?` (main index 0)
    - *Edge Cases:* Target PostgreSQL table requires a unique index or primary key constraint on `order_id` to support `ON CONFLICT` clauses.

#### 2.4 Alerting & Auditing
- **Overview:** Evaluates whether inserted orders violate credit risk conditions or exceed high-value thresholds, triggering Slack alerts and logging audit records.
- **Nodes Involved:** `Credit Block or High Value?`, `Slack: Alert Finance`, `Log Blocked Order Event`, `Continue Batch Loop`
- **Node Details:**
  - **Credit Block or High Value?**
    - *Type & Role:* `If` — Evaluates financial risk criteria using an OR combinator.
    - *Configuration:* Conditions check `{{ $json.creditRisk }}` equals `true` OR `{{ $json.orderValue }}` >= `{{ $('Configure Run').first().json.highValueThreshold }}`.
    - *Inputs:* `Upsert Order into Mart`
    - *Outputs:* Index 0 (True) to `Slack: Alert Finance`; Index 1 (False) to `Continue Batch Loop` (index 1).
    - *Edge Cases:* Boolean type mismatches in incoming payloads can cause rule evaluation failures.
  - **Slack: Alert Finance**
    - *Type & Role:* `Slack` — Sends a formatted warning message to the designated finance channel.
    - *Configuration:* Message template containing order metadata, customer details, value, and run ID, routed to the channel ID fetched from the configuration node.
    - *Inputs:* `Credit Block or High Value?` (True branch)
    - *Outputs:* `Log Blocked Order Event` (main index 0)
    - *Edge Cases:* Invalid Slack authentication tokens or missing channel IDs will abort execution of this branch.
  - **Log Blocked Order Event**
    - *Type & Role:* `Postgres` — Appends audit trails for risk-flagged orders.
    - *Configuration:* Runs an `INSERT INTO erp.order_events` query binding order ID, event type, and serialized JSON payload. Configured with error handling (`continueRegularOutput`).
    - *Inputs:* `Slack: Alert Finance`
    - *Outputs:* `Continue Batch Loop` (main index 0)
    - *Edge Cases:* Database constraint violations are swallowed due to error tolerance, ensuring batch processing continuity.
  - **Continue Batch Loop**
    - *Type & Role:* `Merge` — Reconciles paths from the conditional evaluation branches and loops execution back to the batch processor.
    - *Configuration:* Mode set to `Append`.
    - *Inputs:* `Log Blocked Order Event`, `Credit Block or High Value?` (False branch)
    - *Outputs:* `Batch Over Orders` (main index 0)
    - *Edge Cases:* Misconfigured input linkages will stall the batch loop prematurely.

#### 2.5 Reporting & Summary
- **Overview:** Aggregates overall performance metrics from the data mart once batch processing completes, builds distribution digests, emails stakeholders, and records the run summary.
- **Nodes Involved:** `Query Mart Totals`, `Build Digest HTML`, `Email Ops Digest`, `Slack: Post Daily Digest`, `Log Run Summary`
- **Node Details:**
  - **Query Mart Totals**
    - *Type & Role:* `Postgres` — Queries aggregate KPIs (total orders, revenue, averages, credit-risk counts, and top customers) for orders synced in the last 6 hours.
    - *Configuration:* Executes a CTE-based SQL query returning aggregated metrics and report generation timestamps.
    - *Inputs:* `Batch Over Orders` (Done branch)
    - *Outputs:* `Build Digest HTML` (main index 0)
    - *Edge Cases:* If the sync window returns zero records, CTE aggregations must fallback cleanly (handled via `COALESCE`).
  - **Build Digest HTML**
    - *Type & Role:* `Code (JavaScript)` — Transforms raw KPI rows into formatted HTML document structures and Slack markdown summaries.
    - *Configuration:* JavaScript string template builders compiling tables for top customers, revenue trackers, and operational metadata.
    - *Inputs:* `Query Mart Totals`
    - *Outputs:* `Email Ops Digest` (main index 0)
    - *Edge Cases:* Array validation checks are implemented for `top_customers` to prevent rendering crashes on empty datasets.
  - **Email Ops Digest**
    - *Type & Role:* `Gmail` — Emails the compiled HTML operational digest to the designated recipient.
    - *Configuration:* Sets recipient from `{{ $('Configure Run').first().json.alertEmail }}` using dynamic subject lines and HTML message bodies.
    - *Inputs:* `Build Digest HTML`
    - *Outputs:* `Slack: Post Daily Digest` (main index 0)
    - *Edge Cases:* OAuth token expiration for Gmail credentials will cause delivery failures.
  - **Slack: Post Daily Digest**
    - *Type & Role:* `Slack` — Publishes the markdown summary report to the finance operations channel.
    - *Configuration:* Posts message payload stored in `slackSummary` to the configured channel ID.
    - *Inputs:* `Email Ops Digest`
    - *Outputs:* `Log Run Summary` (main index 0)
    - *Edge Cases:* Rate limiting or connection timeouts with the Slack API.
  - **Log Run Summary**
    - *Type & Role:* `Postgres` — Persists the final execution summary metrics into the audit database.
    - *Configuration:* Executes an upsert query against `erp.sync_runs` matching on `run_id`. Configured with `continueRegularOutput`.
    - *Inputs:* `Slack: Post Daily Digest`
    - *Outputs:* None (Terminal node for success path)
    - *Edge Cases:* Non-fatal errors are bypassed to ensure workflow completion status is preserved.

#### 2.6 Error Handling
- **Overview:** Intercepts unhandled workflow exceptions, formats failure diagnostics, writes logs to PostgreSQL, and notifies the platform team.
- **Nodes Involved:* `On Workflow Error`, `Format Error Payload`, `Log Failure to Postgres`, `Slack: Alert Platform Team`
- **Node Details:**
  - **On Workflow Error**
    - *Type & Role:* `Error Trigger` — Global error handler activated whenever any node in the workflow fails.
    - *Configuration:* Standard error trigger listener.
    - *Inputs:* None (Trigger)
    - *Outputs:* `Format Error Payload` (main index 0)
    - *Edge Cases:* Error triggers will not catch errors explicitly swallowed by nodes configured with error continuation unless unhandled downstream.
  - **Format Error Payload**
    - *Type & Role:* `Code (JavaScript)` — Extracts and truncates error diagnostic properties into a clean failure payload.
    - *Configuration:* JavaScript parsing snippet extracting workflow name, failed node identifier, error messages, execution URL, and pre-formatted Slack alert texts.
    - *Inputs:* `On Workflow Error`
    - *Outputs:* `Log Failure to Postgres` (main index 0)
    - *Edge Cases:* Missing execution context properties handled via default fallbacks (`'unknown'`, `''`).
  - **Log Failure to Postgres**
    - *Type & Role:* `Postgres` — Records pipeline failures into an error audit log table.
    - *Configuration:* Executes `INSERT INTO erp.pipeline_errors` using parameterized values. Configured with error continuation (`continueRegularOutput`).
    - *Inputs:* `Format Error Payload`
    - *Outputs:* `Slack: Alert Platform Team` (main index 0)
    - *Edge Cases:* If the database is entirely unreachable, this step may fail, though error continuity ensures Slack alerting is still attempted.
  - **Slack: Alert Platform Team**
    - *Type & Role:* `Slack` — Pages the platform or engineering team regarding pipeline execution failure.
    - *Configuration:* Sends warning payloads to the engineering/finance Slack channel ID.
    - *Inputs:* `Log Failure to Postgres`
    - *Outputs:* None (Terminal node for error path)
    - *Edge Cases:* Network failures prevent notification delivery.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 📘 Overview | n8n-nodes-base.stickyNote | Documentation | None | None | ## ERP Order-to-Cash Sync (Order-to-Cash = O2C)<br>Pulls open sales orders from a **Microsoft SQL Server ERP** (SAP B1 / Dynamics / custom schema),<br>normalises them, upserts into a **PostgreSQL data mart** in batches, alerts Finance about<br>credit-blocked or high-value orders, and mails/posts a daily digest.<br><br>**Runs:** every day at 06:30 · **Batch size:** 100 orders<br>**Edit me first:** the `Configure Run` node (thresholds, channels, e-mail). |
| ① Extract from ERP | n8n-nodes-base.stickyNote | Documentation | None | None | **Microsoft SQL node** – read-only query against the ERP.<br>Connection name in query uses a parameters list, so no SQL injection from config. |
| ② Normalise + batch | n8n-nodes-base.stickyNote | Documentation | None | None | **Code node** maps ERP rows to a canonical order model and removes duplicates.<br>**Split In Batches** keeps the DB transaction small (100 rows per commit). |
| ③ Persist to data mart | n8n-nodes-base.stickyNote | Documentation | None | None | **Postgres nodes** do the heavy lifting:<br>`INSERT … ON CONFLICT` upsert, blocked-order event log and the daily KPI roll-up. |
| Schedule: Daily 06:30 (Berlin) | n8n-nodes-base.scheduleTrigger | Trigger daily execution | None | Configure Run | |
| Configure Run | n8n-nodes-base.set | Set global execution variables | Schedule: Daily 06:30 (Berlin) | ERP: Fetch Open Sales Orders | |
| ERP: Fetch Open Sales Orders | n8n-nodes-base.microsoftSql | Query ERP open sales orders | Configure Run | Normalize ERP Rows | |
| Normalize ERP Rows | n8n-nodes-base.code | Normalize records and deduplicate | ERP: Fetch Open Sales Orders | Batch Over Orders | |
| Batch Over Orders | n8n-nodes-base.splitInBatches | Split records into batches | Normalize ERP Rows, Continue Batch Loop | Upsert Order into Mart, Query Mart Totals | |
| Upsert Order into Mart | n8n-nodes-base.postgres | Upsert order records | Batch Over Orders | Credit Block or High Value? | |
| Credit Block or High Value? | n8n-nodes-base.if | Check credit risk or high value | Upsert Order into Mart | Slack: Alert Finance, Continue Batch Loop | |
| Slack: Alert Finance | n8n-nodes-base.slack | Send finance warning alert | Credit Block or High Value? | Log Blocked Order Event | |
| Log Blocked Order Event | n8n-nodes-base.postgres | Log audit event | Slack: Alert Finance | Continue Batch Loop | |
| Continue Batch Loop | n8n-nodes-base.merge | Join execution branches for loop | Log Blocked Order Event, Credit Block or High Value? | Batch Over Orders | |
| Query Mart Totals | n8n-nodes-base.postgres | Query KPI aggregates | Batch Over Orders | Build Digest HTML | |
| Build Digest HTML | n8n-nodes-base.code | Build HTML and Slack digests | Query Mart Totals | Email Ops Digest | |
| Email Ops Digest | n8n-nodes-base.gmail | Email operational report | Build Digest HTML | Slack: Post Daily Digest | |
| Slack: Post Daily Digest | n8n-nodes-base.slack | Post Slack report summary | Email Ops Digest | Log Run Summary | |
| Log Run Summary | n8n-nodes-base.postgres | Save run summary audit log | Slack: Post Daily Digest | None | |
| On Workflow Error | n8n-nodes-base.errorTrigger | Capture workflow exceptions | None | Format Error Payload | |
| Format Error Payload | n8n-nodes-base.code | Format execution error details | On Workflow Error | Log Failure to Postgres | |
| Log Failure to Postgres | n8n-nodes-base.postgres | Log failure to audit table | Format Error Payload | Slack: Alert Platform Team | |
| Slack: Alert Platform Team | n8n-nodes-base.slack | Alert engineers of failure | Log Failure to Postgres | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually:

1. **Create the Trigger Node:**
   - Add a `Schedule Trigger` node named `Schedule: Daily 06:30 (Berlin)`.
   - Set interval rule: Trigger at Hour `6`, Minute `30`.

2. **Create the Configuration Node:**
   - Add a `Set (Edit Fields)` node named `Configure Run`.
   - Add assignments:
     - `runId`: String = `={{ $now.toFormat('yyyyLLdd-HHmm') }}`
     - `lookbackDays`: Number = `7`
     - `highValueThreshold`: Number = `25000`
     - `financeChannelId`: String = `C0123456789`
     - `alertEmail`: String = `user@example.com`
     - `erpCompanyCode`: String = `US01`
   - Connect `Schedule: Daily 06:30 (Berlin)` to `Configure Run`.

3. **Create the ERP Extraction Node:**
   - Add a `Microsoft SQL` node named `ERP: Fetch Open Sales Orders`.
   - Configure credentials pointing to your read-only ERP database replica.
   - Set operation to `Execute Query` and provide the open sales orders SQL extraction query filtering by `DATEADD(day, -{{ $json.lookbackDays }}, GETDATE())`.
   - Connect `Configure Run` to `ERP: Fetch Open Sales Orders`.

4. **Create the Normalization Node:**
   - Add a `Code` node named `Normalize ERP Rows`.
   - Paste the JavaScript normalization snippet mapping raw fields to the canonical order model, calculating utilization percentages, and establishing deduplication sets.
   - Connect `ERP: Fetch Open Sales Orders` to `Normalize ERP Rows`.

5. **Create the Batch Processing Node:**
   - Add a `Split In Batches` node named `Batch Over Orders`.
   - Set batch size to `100`.
   - Connect `Normalize ERP Rows` to `Batch Over Orders`.

6. **Create the Data Mart Persistence Node:**
   - Add a `Postgres` node named `Upsert Order into Mart`.
   - Configure credentials pointing to your data mart database.
   - Set operation to `Execute Query` and input the parameterized `INSERT INTO erp.order_sync ... ON CONFLICT (order_id) DO UPDATE ...` query.
   - Connect output index 1 of `Batch Over Orders` to `Upsert Order into Mart`.

7. **Create the Risk Evaluation Node:**
   - Add an `If` node named `Credit Block or High Value?`.
   - Set combinator mode to `OR`.
   - Condition 1: Left value `={{ $json.creditRisk }}` equals boolean `true`.
   - Condition 2: Left value `={{ $json.orderValue }}` greater than or equal to `={{ $('Configure Run').first().json.highValueThreshold }}`.
   - Connect `Upsert Order into Mart` to `Credit Block or High Value?`.

8. **Create the Finance Alerting & Audit Nodes:**
   - Add a `Slack` node named `Slack: Alert Finance`. Set resource/select to `channel`, message template to warning text, and channel ID to `={{ $('Configure Run').first().json.financeChannelId }}`. Connect True branch (index 0) of `Credit Block or High Value?` here.
   - Add a `Postgres` node named `Log Blocked Order Event`. Set query to `INSERT INTO erp.order_events (...) VALUES (...)`. Enable `Always Output Data` and set error handling to `Continue Regular Output`. Connect `Slack: Alert Finance` here.

9. **Create the Loop Reconcile Node:**
   - Add a `Merge` node named `Continue Batch Loop`. Set mode to `Append`.
   - Connect both `Log Blocked Order Event` and the False branch (index 1) of `Credit Block or High Value?` to `Continue Batch Loop`.
   - Connect `Continue Batch Loop` back into `Batch Over Orders`.

10. **Create Reporting & Summary Nodes:**
    - Add a `Postgres` node named `Query Mart Totals`. Set query to fetch KPI rollups from `erp.order_sync`. Connect output index 0 (Done) of `Batch Over Orders` here.
    - Add a `Code` node named `Build Digest HTML`. Paste the snippet generating `subject`, `slackSummary`, and `html` content. Connect `Query Mart Totals` here.
    - Add a `Gmail` node named `Email Ops Digest`. Set `sendTo` to `={{ $('Configure Run').first().json.alertEmail }}`, message to `={{ $json.html }}`, and subject to `={{ $json.subject }}`. Connect `Build Digest HTML` here.
    - Add a `Slack` node named `Slack: Post Daily Digest`. Set message to `={{ $json.slackSummary }}` and channel ID to `={{ $('Configure Run').first().json.financeChannelId }}`. Connect `Email Ops Digest` here.
    - Add a `Postgres` node named `Log Run Summary`. Set query to upsert into `erp.sync_runs`. Enable `Always Output Data` and set error handling to `Continue Regular Output`. Connect `Slack: Post Daily Digest` here.

11. **Create Error Handling Sub-flow:**
    - Add an `Error Trigger` node named `On Workflow Error`.
    - Add a `Code` node named `Format Error Payload` to format error details. Connect `On Workflow Error` here.
    - Add a `Postgres` node named `Log Failure to Postgres` to log errors into `erp.pipeline_errors`. Enable `Always Output Data` and set error handling to `Continue Regular Output`. Connect `Format Error Payload` here.
    - Add a `Slack` node named `Slack: Alert Platform Team` to notify engineers of execution failures. Connect `Log Failure to Postgres` here.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Database Schema Requirements | Requires pre-created PostgreSQL tables (`erp.order_sync`, `erp.order_events`, `erp.sync_runs`, `erp.pipeline_errors`) with unique keys on `order_id` and `run_id` to support idempotent upserts. |
| Security Best Practice | Ensure the Microsoft SQL Server credential points strictly to a read-only ERP replica rather than a production master database to prevent locking issues. |
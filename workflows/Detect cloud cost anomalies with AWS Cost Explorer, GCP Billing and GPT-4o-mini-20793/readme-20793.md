Detect cloud cost anomalies with AWS Cost Explorer, GCP Billing and GPT-4o-mini

https://n8nworkflows.xyz/workflows/detect-cloud-cost-anomalies-with-aws-cost-explorer--gcp-billing-and-gpt-4o-mini-20793


# Detect cloud cost anomalies with AWS Cost Explorer, GCP Billing and GPT-4o-mini

### 1. Workflow Overview

This workflow functions as an automated FinOps cloud cost anomaly detection agent. It runs on a daily schedule to identify budget spikes across AWS and GCP, correlates AWS cost spikes with historical usage drivers and CloudTrail event logs, uses OpenAI to generate root-cause explanations, sends interactive actionable alerts to Slack, and maintains a stateful audit log in Google Sheets via secure, token-secured webhooks.

The workflow logic is partitioned into the following functional blocks:
- **1.1 Schedule & Configuration:** Initializes execution at a set time (9 AM IST), defines parameters, thresholds, and operational flags, and reads existing audit records from Google Sheets.
- **1.2 Cloud Cost Data Ingestion:** Conditionally queries AWS Cost Explorer and Google Cloud BigQuery billing export tables, then normalizes different schemas into a uniform internal format.
- **1.3 Anomaly Detection & Summary:** Applies financial filters (dollar floor, percentage jump, Z-score) alongside historical suppression rules, loops through identified spikes, and posts a daily run summary to Slack.
- **1.4 AI Root Cause Analysis:** Enriches valid AWS anomalies by querying regional CloudTrail write events and submitting structured context to OpenAI (`gpt-4o-mini`) for an explanation and remediation steps.
- **1.5 Slack Alert & Audit Logging:** Dispatches interactive Slack block messages containing direct "Approve" or "Ignore" webhook action links, then writes the anomaly record to Google Sheets.
- **1.6 Interactive Decision Handling:** Receives inbound webhook requests when users click Slack action buttons, validates cryptographic single-use tokens, updates Google Sheets records, renders HTML confirmation or error pages, and notifies Slack of the decision.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration
- **Overview:** Establishes the daily execution cadence, calculates target date ranges, and loads historical state from Google Sheets to prevent duplicate alerting.
- **Nodes Involved:** `Every Day at 9 AM IST`, `Set Cost Agent Config`, `Read Anomaly Log Sheet`
- **Node Details:**
  - **Every Day at 9 AM IST**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Cron trigger)
    - *Configuration Choices:* Configured with a cron expression (`0 9 * * *`) targeting 9:00 AM daily.
    - *Input/Output:* No input connections; outputs execution timestamp to `Set Cost Agent Config`.
    - *Edge Cases/Failure Types:* Clock drift or missed executions if the n8n instance is offline.
  - **Set Cost Agent Config**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript processing block)
    - *Configuration Choices:* Establishes global parameters including cloud toggles (`aws_enabled`, `gcp_enabled`), project identifiers, financial thresholds (`min_abs_usd: 20`, `min_pct: 0.30`, `z_threshold: 3`), currency definitions, and date offset variables relative to UTC/IST.
    - *Key Expressions/Variables:* Uses `$now.setZone('UTC')` to dynamically calculate relative dates (`target_date`, `prev_date`, `window_start`, `driver_start`).
    - *Input/Output:* Input from `Every Day at 9 AM IST`; outputs a single JSON item object to `Read Anomaly Log Sheet`.
  - **Read Anomaly Log Sheet**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets integration)
    - *Configuration Choices:* Executes once per run (`executeOnce: true`), using the `Anomaly Log` sheet name and a placeholder document ID.
    - *Input/Output:* Input from `Set Cost Agent Config`; outputs rows of historical anomaly records to `AWS Enabled?`.
    - *Edge Cases/Failure Types:* Authentication expiration, API rate limits, or network timeouts when reading large spreadsheets.

#### 2.2 Cloud Cost Data Ingestion
- **Overview:** Conditionally fetches spend metrics from AWS Cost Explorer (Service/Region breakdown and Usage Type drivers) and GCP BigQuery, standardizing the data structures.
- **Nodes Involved:** `AWS Enabled?`, `Get AWS Cost by Service & Region`, `Get AWS Cost by Usage Type`, `Normalize AWS Costs`, `GCP Enabled?`, `Query GCP Billing Export`, `Normalize GCP Costs`
- **Node Details:**
  - **AWS Enabled?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional evaluation)
    - *Configuration Choices:* Evaluates whether `aws_enabled` equals `true`.
    - *Input/Output:* Input from `Read Anomaly Log Sheet`; outputs true branch to `Get AWS Cost by Service & Region` and false branch to `GCP Enabled?`.
  - **Get AWS Cost by Service & Region**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (REST API invocation)
    - *Configuration Choices:* POST request to `https://ce.us-east-1.amazonaws.com/` using predefined AWS credentials, passing target time periods, daily granularity, `UnblendedCost` metrics, and dimension grouping by `SERVICE` and `REGION`. Includes pagination configuration for `NextPageToken`.
    - *Input/Output:* Input from `AWS Enabled?` (True); outputs response items to `Get AWS Cost by Usage Type`.
    - *Edge Cases/Failure Types:* AWS IAM permission errors (`ce:GetCostAndUsage`), throttling limits on Cost Explorer API calls.
  - **Get AWS Cost by Usage Type**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (REST API invocation)
    - *Configuration Choices:* POST request to Cost Explorer API grouping by `SERVICE` and `USAGE_TYPE` for the driver start period.
    - *Input/Output:* Input from `Get AWS Cost by Service & Region`; outputs response data to `Normalize AWS Costs`.
  - **Normalize AWS Costs**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript data transformation)
    - *Configuration Choices:* Maps AWS usage type prefixes (e.g., `APS3-NatGateway-Hours`) to normalized regional strings using a lookup dictionary, dropping nominal costs below $0.005.
    - *Input/Output:* Input from `Get AWS Cost by Usage Type`; outputs consolidated `{ series, drivers }` JSON to `GCP Enabled?`.
  - **GCP Enabled?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional evaluation)
    - *Configuration Choices:* Evaluates whether `gcp_enabled` equals `true`.
    - *Input/Output:* Input from `Normalize AWS Costs` or `AWS Enabled?` (False); outputs true branch to `Query GCP Billing Export` and false branch to `Detect Day-over-Day Anomalies`.
  - **Query GCP Billing Export**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (REST API invocation)
    - *Configuration Choices:* POST request to BigQuery REST API executing standard SQL against billing export tables using Google OAuth2 credentials. Sets a 120-second timeout.
    - *Input/Output:* Input from `GCP Enabled?` (True); outputs BigQuery response rows to `Normalize GCP Costs`.
    - *Edge Cases/Failure Types:* BigQuery query execution timeout, incorrect table names, or insufficient IAM dataset read permissions.
  - **Normalize GCP Costs**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript data transformation)
    - *Configuration Choices:* Transforms BigQuery schema fields and rows into standard series and driver objects.
    - *Input/Output:* Input from `Query GCP Billing Export`; outputs normalized data to `Detect Day-over-Day Anomalies`.

#### 2.3 Anomaly Detection & Summary
- **Overview:** Evaluates day-over-day cost changes against absolute, percentage, and statistical thresholds, filters out muted or previously processed anomalies, and generates an executive run summary for Slack.
- **Nodes Involved:** `Detect Day-over-Day Anomalies`, `Loop Over Anomalies`, `Build Run Summary`, `Post Run Summary to Slack`
- **Node Details:**
  - **Detect Day-over-Day Anomalies**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript processing block)
    - *Configuration Choices:* Compares target day spend against the previous day and a 7-day historical window. Calculates Z-scores, absolute deltas, percentage shifts, and monthly impact projections. Filters out seen anomalies and muted items based on Google Sheets state.
    - *Input/Output:* Inputs from normalization nodes; outputs arrays of detected anomaly objects to `Loop Over Anomalies`.
  - **Loop Over Anomalies**
    - *Type and Technical Role:* `n8n-nodes-base.splitInBatches` (Iteration controller)
    - *Configuration Choices:* Processes anomaly items iteratively.
    - *Input/Output:* Input from `Detect Day-over-Day Anomalies`. Branch 0 outputs completion summary data to `Build Run Summary`; Branch 1 outputs individual anomaly items for enrichment.
  - **Build Run Summary**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript transformation)
    - *Configuration Choices:* Aggregates total anomalous delta costs and formats a consolidated Markdown text block.
    - *Input/Output:* Input from batch loop completion; outputs message text to `Post Run Summary to Slack`.
  - **Post Run Summary to Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Slack integration)
    - *Configuration Choices:* Posts text messages to the configured Slack channel.
    - *Input/Output:* Input from `Build Run Summary`.
    - *Edge Cases/Failure Types:* Slack API token invalidity or missing channel write scopes.

#### 2.4 AI Root Cause Analysis
- **Overview:** Gathers regional CloudTrail audit events for AWS anomalies and queries OpenAI (`gpt-4o-mini`) to synthesize probable root causes and remediation recommendations.
- **Nodes Involved:** `AWS Region Known?`, `Get Recent CloudTrail Changes`, `Build AI Context`, `AI Explain Cost Spike`
- **Node Details:**
  - **AWS Region Known?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional evaluation)
    - *Configuration Choices:* Evaluates whether `cloudtrail_region` is present and valid.
    - *Input/Output:* Input from `Loop Over Anomalies`; true branch connects to `Get Recent CloudTrail Changes`, false branch connects to `Build AI Context`.
  - **Get Recent CloudTrail Changes**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (REST API invocation)
    - *Configuration Choices:* POST request to regional AWS CloudTrail endpoints (`LookupEvents`) targeting write events (`ReadOnly: false`) within the anomaly timeframe. Continues regular output on error (`onError: continueRegularOutput`).
    - *Input/Output:* Input from `AWS Region Known?` (True); outputs event logs to `Build AI Context`.
    - *Edge Cases/Failure Types:* Missing `cloudtrail:LookupEvents` IAM permissions or unmapped regions.
  - **Build AI Context**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript transformation)
    - *Configuration Choices:* Filters and maps CloudTrail write events (matching actions like `Create`, `Run`, `Modify`, `Scale`) and merges them with anomaly metadata.
    - *Input/Output:* Inputs from batch loop and CloudTrail request; outputs combined JSON context to `AI Explain Cost Spike`.
  - **AI Explain Cost Spike**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (OpenAI LangChain integration)
    - *Configuration Choices:* Uses model `gpt-4o-mini` with temperature `0.2` and JSON response formatting enabled. System prompt instructs the model to act as a senior FinOps engineer correlating usage drivers and CloudTrail events.
    - *Input/Output:* Input from `Build AI Context`; outputs structured AI response objects (`headline`, `probable_cause`, `evidence`, `recommended_actions`, etc.) to `Build Slack Alert`.
    - *Edge Cases/Failure Types:* OpenAI rate limits, token budget limits, or malformed JSON outputs.

#### 2.5 Slack Alert & Audit Logging
- **Overview:** Formats interactive Slack block kits with action buttons containing secure webhook URLs, dispatches the alert, and records the active anomaly into Google Sheets.
- **Nodes Involved:** `Build Slack Alert`, `Post Anomaly Alert to Slack`, `Log Anomaly to Sheet`
- **Node Details:**
  - **Build Slack Alert**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript transformation)
    - *Configuration Choices:* Constructs Slack Block Kit payloads incorporating metadata, AI insights, top cost drivers, and three interactive buttons (`Approve (expected)`, `Ignore 7d`, `Open billing console`) pointing to unique webhook callback URLs with embedded tokens (`id`, `token`, `a`).
    - *Input/Output:* Input from `AI Explain Cost Spike`; outputs formatted block payloads and fallback text to `Post Anomaly Alert to Slack`.
  - **Post Anomaly Alert to Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Slack integration)
    - *Configuration Choices:* Sends block-type messages using JSON block definitions to the target Slack channel ID.
    - *Input/Output:* Input from `Build Slack Alert`; outputs message details to `Log Anomaly to Sheet`.
  - **Log Anomaly to Sheet**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets integration)
    - *Configuration Choices:* Uses operation `appendOrUpdate` matching on `anomaly_id` within the `Anomaly Log` spreadsheet tab.
    - *Input/Output:* Input from `Post Anomaly Alert to Slack`; loops back to `Loop Over Anomalies` to process the next item.
    - *Edge Cases/Failure Types:* Row locking conflicts or spreadsheet quota limits during batch processing.

#### 2.6 Interactive Decision Handling
- **Overview:** Handles inbound HTTP webhooks triggered when users click Slack action buttons, validates security tokens, updates Google Sheets status, returns confirmation/error HTML responses, and posts decision updates back to Slack.
- **Nodes Involved:** `On Approve or Ignore Link Click`, `Look Up Anomaly in Log`, `Validate Action Token`, `Token Valid?`, `Record Decision in Log`, `Show Invalid Link Page`, `Show Decision Saved Page`, `Post Decision to Slack`
- **Node Details:**
  - **On Approve or Ignore Link Click**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Webhook trigger)
    - *Configuration Choices:* Configured with path `cost-anomaly-action` and `responseMode: responseNode`.
    - *Input/Output:* No upstream nodes; outputs HTTP query parameters to `Look Up Anomaly in Log`.
  - **Look Up Anomaly in Log**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets integration)
    - *Configuration Choices:* Looks up rows where `anomaly_id` matches query parameter `id`.
    - *Input/Output:* Input from webhook trigger; outputs matching log row data to `Validate Action Token`.
  - **Validate Action Token**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript security validation)
    - *Configuration Choices:* Validates that the anomaly exists, that the token matches and has not been used, and that the requested action is valid (`approve` or `ignore`). Calculates a 7-day suppression date if ignored.
    - *Input/Output:* Input from `Look Up Anomaly in Log`; outputs validation results to `Token Valid?`.
  - **Token Valid?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional evaluation)
    - *Configuration Choices:* Checks if `valid` equals `true`.
    - *Input/Output:* Input from `Validate Action Token`; true branch routes to `Record Decision in Log`, false branch routes to `Show Invalid Link Page`.
  - **Record Decision in Log**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets integration)
    - *Configuration Choices:* Updates status (`Approved` or `Ignored`), decision timestamp, and suppression date in the row matching `anomaly_id`.
    - *Input/Output:* Input from `Token Valid?` (True); outputs update confirmation to `Show Decision Saved Page`.
  - **Show Invalid Link Page**
    - *Type and Technical Role:* `n8n-nodes-base.respondToWebhook` (HTTP response node)
    - *Configuration Choices:* Returns HTTP status code `403` with an HTML body displaying validation failure reasons.
    - *Input/Output:* Input from `Token Valid?` (False).
  - **Show Decision Saved Page**
    - *Type and Technical Role:` `n8n-nodes-base.respondToWebhook` (HTTP response node)
    - *Configuration Choices:* Returns HTTP status code `200` with an HTML body confirming the action taken.
    - *Input/Output:* Input from `Record Decision in Log`; outputs confirmation data to `Post Decision to Slack`.
  - **Post Decision to Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Slack integration)
    - *Configuration Choices:* Posts a notification message to the designated Slack channel stating the recorded approval or suppression action.
    - *Input/Output:* Input from `Show Decision Saved Page`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note – Overview** | `n8n-nodes-base.stickyNote` | Documentation placeholder | *None* | *None* | ## 💸 FinOps – Cloud Cost Anomaly Agent (AWS + GCP)<br><br>For engineering and finance teams who find out about a runaway NAT gateway, a forgotten GPU instance or a BigQuery job gone wild when the monthly bill lands, weeks after the money is spent.<br><br>### How it works<br>Every morning the workflow pulls yesterday's spend by service and region from AWS Cost Explorer and your GCP billing export. It compares each line with the previous day and the past week, and flags only real spikes that pass a dollar floor, a percentage jump and a z-score.<br><br>For each AWS spike it pulls recent CloudTrail changes in that region. GPT-4o-mini combines them with the usage-type breakdown to suggest the most likely cause. Slack gets an alert with the cost, monthly impact, probable cause and two buttons: mark it expected, or ignore it for seven days. Every anomaly and decision is logged in Google Sheets.<br><br>### Setup steps<br>1. Create a Google Sheet with an **Anomaly_Log** tab.<br>2. Add AWS credentials with `ce:GetCostAndUsage` and `cloudtrail:LookupEvents`, and Google OAuth2 with BigQuery read for GCP.<br>3. Connect Slack and OpenAI.<br>4. In **Set Cost Agent Config**, turn each cloud on or off and set the GCP project, billing table, thresholds and your n8n base URL (used for the buttons).<br>5. Run once manually, then activate. |
| **Sticky Note – Schedule & Config** | `n8n-nodes-base.stickyNote` | Documentation placeholder | *None* | *None* | ## ⏰ Schedule & Config<br>Runs at 9 AM IST, once yesterday's billing data has mostly settled. Turn AWS and GCP on or off and set the thresholds in the config node. |
| **Sticky Note – Pull AWS & GCP Costs** | `n8n-nodes-base.stickyNote` | Documentation placeholder | *None* | *None* | ## ☁️ Pull AWS & GCP Costs<br>AWS Cost Explorer returns 15 days of cost by service and region, plus yesterday's usage-type breakdown. GCP comes from your BigQuery billing export. Each source can be switched off, and both are normalised into one shape. |
| **Sticky Note – Detect & Summarize** | `n8n-nodes-base.stickyNote` | Documentation placeholder | *None* | *None* | ## 📊 Detect & Summarize<br>Flags a service-region spike only when it clears a dollar floor, a percentage jump and a z-score against the past week. Ignored and already-seen anomalies are skipped, and the run ends with a one-line Slack summary. |
| **Sticky Note – AI Root Cause** | `n8n-nodes-base.stickyNote` | Documentation placeholder | *None* | *None* | ## 🧠 AI Root Cause<br>For AWS spikes, recent CloudTrail write events in that region are pulled in, like a new instance or a scaled-up node group. GPT-4o-mini combines them with the usage-type numbers to suggest a probable cause and what to check. |
| **Sticky Note – Slack Alert & Log** | `n8n-nodes-base.stickyNote` | Documentation placeholder | *None* | *None* | ## 💬 Slack Alert & Log<br>Each anomaly gets a Slack message with the numbers, monthly impact if it keeps up, the likely cause and two buttons: Expected or Ignore 7 days. It's logged so it's never alerted twice. |
| **Sticky Note – Decision Link Check** | `n8n-nodes-base.stickyNote` | Documentation placeholder | *None* | *None* | ## 🔗 Decision Link Check<br>The Slack buttons open this webhook. Each link carries a one-time token that's checked against the log, so a reused or forwarded link can't change anything. |
| **Sticky Note – Record & Confirm** | `n8n-nodes-base.stickyNote` | Documentation placeholder | *None* | *None* | ## ✅ Record & Confirm<br>Saves the decision, shows a confirmation page and posts it to Slack. Ignored anomalies stay muted for seven days. Bad links get a clear error page. |
| **Sticky Note – Credentials & Security** | `n8n-nodes-base.stickyNote` | Documentation placeholder | *None* | *None* | ## 🔐 Credentials & Security<br>Use a dedicated read-only AWS IAM user with only Cost Explorer and CloudTrail lookup. Cost Explorer charges a small fee per API call. Approve/Ignore links are one-time, so keep the Slack channel private. |
| **Every Day at 9 AM IST** | `n8n-nodes-base.scheduleTrigger` | Triggers workflow execution daily at 9:00 AM IST | *None* | Set Cost Agent Config | ## ⏰ Schedule & Config<br>Runs at 9 AM IST, once yesterday's billing data has mostly settled. Turn AWS and GCP on or off and set the thresholds in the config node. |
| **Set Cost Agent Config** | `n8n-nodes-base.code` | Initializes configuration parameters, thresholds, and dates | Every Day at 9 AM IST | Read Anomaly Log Sheet | ## ⏰ Schedule & Config<br>Runs at 9 AM IST, once yesterday's billing data has mostly settled. Turn AWS and GCP on or off and set the thresholds in the config node. |
| **Read Anomaly Log Sheet** | `n8n-nodes-base.googleSheets` | Loads existing anomaly history from Google Sheets | Set Cost Agent Config | AWS Enabled? | ## ⏰ Schedule & Config<br>Runs at 9 AM IST, once yesterday's billing data has mostly settled. Turn AWS and GCP on or off and set the thresholds in the config node. |
| **AWS Enabled?** | `n8n-nodes-base.if` | Evaluates if AWS cost checking is enabled | Read Anomaly Log Sheet | Get AWS Cost by Service & Region, GCP Enabled? | ## ☁️ Pull AWS & GCP Costs<br>AWS Cost Explorer returns 15 days of cost by service and region, plus yesterday's usage-type breakdown. GCP comes from your BigQuery billing export. Each source can be switched off, and both are normalised into one shape. |
| **Get AWS Cost by Service & Region** | `n8n-nodes-base.httpRequest` | Fetches daily cost data grouped by service and region from AWS Cost Explorer | AWS Enabled? | Get AWS Cost by Usage Type | ## ☁️ Pull AWS & GCP Costs<br>AWS Cost Explorer returns 15 days of cost by service and region, plus yesterday's usage-type breakdown. GCP comes from your BigQuery billing export. Each source can be switched off, and both are normalised into one shape. |
| **Get AWS Cost by Usage Type** | `n8n-nodes-base.httpRequest` | Fetches daily cost data grouped by service and usage type from AWS Cost Explorer | Get AWS Cost by Service & Region | Normalize AWS Costs | ## ☁️ Pull AWS & GCP Costs<br>AWS Cost Explorer returns 15 days of cost by service and region, plus yesterday's usage-type breakdown. GCP comes from your BigQuery billing export. Each source can be switched off, and both are normalised into one shape. |
| **Normalize AWS Costs** | `n8n-nodes-base.code` | Normalizes AWS cost responses and maps usage types to regions | Get AWS Cost by Usage Type | GCP Enabled? | ## ☁️ Pull AWS & GCP Costs<br>AWS Cost Explorer returns 15 days of cost by service and region, plus yesterday's usage-type breakdown. GCP comes from your BigQuery billing export. Each source can be switched off, and both are normalised into one shape. |
| **GCP Enabled?** | `n8n-nodes-base.if` | Evaluates if GCP billing query is enabled | AWS Enabled?, Normalize AWS Costs | Query GCP Billing Export, Detect Day-over-Day Anomalies | ## ☁️ Pull AWS & GCP Costs<br>AWS Cost Explorer returns 15 days of cost by service and region, plus yesterday's usage-type breakdown. GCP comes from your BigQuery billing export. Each source can be switched off, and both are normalised into one shape. |
| **Query GCP Billing Export** | `n8n-nodes-base.httpRequest` | Queries BigQuery billing export table for GCP cost data | GCP Enabled? | Normalize GCP Costs | ## ☁️ Pull AWS & GCP Costs<br>AWS Cost Explorer returns 15 days of cost by service and region, plus yesterday's usage-type breakdown. GCP comes from your BigQuery billing export. Each source can be switched off, and both are normalised into one shape. |
| **Normalize GCP Costs** | `n8n-nodes-base.code` | Normalizes GCP BigQuery query results | Query GCP Billing Export | Detect Day-over-Day Anomalies | ## ☁️ Pull AWS & GCP Costs<br>AWS Cost Explorer returns 15 days of cost by service and region, plus yesterday's usage-type breakdown. GCP comes from your BigQuery billing export. Each source can be switched off, and both are normalised into one shape. |
| **Detect Day-over-Day Anomalies** | `n8n-nodes-base.code` | Identifies cost anomalies using threshold and statistical filters | Normalize GCP Costs, GCP Enabled? | Loop Over Anomalies | ## 📊 Detect & Summarize<br>Flags a service-region spike only when it clears a dollar floor, a percentage jump and a z-score against the past week. Ignored and already-seen anomalies are skipped, and the run ends with a one-line Slack summary. |
| **Loop Over Anomalies** | `n8n-nodes-base.splitInBatches` | Iterates through detected anomalies one by one | Detect Day-over-Day Anomalies | Build Run Summary, AWS Region Known? | ## 📊 Detect & Summarize<br>Flags a service-region spike only when it clears a dollar floor, a percentage jump and a z-score against the past week. Ignored and already-seen anomalies are skipped, and the run ends with a one-line Slack summary. |
| **Build Run Summary** | `n8n-nodes-base.code` | Formats daily run summary text | Loop Over Anomalies | Post Run Summary to Slack | ## 📊 Detect & Summarize<br>Flags a service-region spike only when it clears a dollar floor, a percentage jump and a z-score against the past week. Ignored and already-seen anomalies are skipped, and the run ends with a one-line Slack summary. |
| **Post Run Summary to Slack** | `n8n-nodes-base.slack` | Posts aggregate run summary message to Slack | Build Run Summary | *None* | ## 📊 Detect & Summarize<br>Flags a service-region spike only when it clears a dollar floor, a percentage jump and a z-score against the past week. Ignored and already-seen anomalies are skipped, and the run ends with a one-line Slack summary. |
| **AWS Region Known?** | `n8n-nodes-base.if` | Checks if anomaly is associated with a valid AWS region | Loop Over Anomalies | Get Recent CloudTrail Changes, Build AI Context | ## 🧠 AI Root Cause<br>For AWS spikes, recent CloudTrail write events in that region are pulled in, like a new instance or a scaled-up node group. GPT-4o-mini combines them with the usage-type numbers to suggest a probable cause and what to check. |
| **Get Recent CloudTrail Changes** | `n8n-nodes-base.httpRequest` | Retrieves recent CloudTrail write event logs for the region | AWS Region Known? | Build AI Context | ## 🧠 AI Root Cause<br>For AWS spikes, recent CloudTrail write events in that region are pulled in, like a new instance or a scaled-up node group. GPT-4o-mini combines them with the usage-type numbers to suggest a probable cause and what to check. |
| **Build AI Context** | `n8n-nodes-base.code` | Merges anomaly metrics and CloudTrail event logs for AI processing | Get Recent CloudTrail Changes, AWS Region Known? | AI Explain Cost Spike | ## 🧠 AI Root Cause<br>For AWS spikes, recent CloudTrail write events in that region are pulled in, like a new instance or a scaled-up node group. GPT-4o-mini combines them with the usage-type numbers to suggest a probable cause and what to check. |
| **AI Explain Cost Spike** | `@n8n/n8n-nodes-langchain.openAi` | Generates root cause explanation and recommendations using gpt-4o-mini | Build AI Context | Build Slack Alert | ## 🧠 AI Root Cause<br>For AWS spikes, recent CloudTrail write events in that region are pulled in, like a new instance or a scaled-up node group. GPT-4o-mini combines them with the usage-type numbers to suggest a probable cause and what to check. |
| **Build Slack Alert** | `n8n-nodes-base.code` | Formats interactive Slack Block Kit message with action links | AI Explain Cost Spike | Post Anomaly Alert to Slack | ## 💬 Slack Alert & Log<br>Each anomaly gets a Slack message with the numbers, monthly impact if it keeps up, the likely cause and two buttons: Expected or Ignore 7 days. It's logged so it's never alerted twice. |
| **Post Anomaly Alert to Slack** | `n8n-nodes-base.slack` | Sends interactive anomaly alert to Slack channel | Build Slack Alert | Log Anomaly to Sheet | ## 💬 Slack Alert & Log<br>Each anomaly gets a Slack message with the numbers, monthly impact if it keeps up, the likely cause and two buttons: Expected or Ignore 7 days. It's logged so it's never alerted twice. |
| **Log Anomaly to Sheet** | `n8n-nodes-base.googleSheets` | Records detected anomaly to Google Sheets log | Post Anomaly Alert to Slack | Loop Over Anomalies | ## 💬 Slack Alert & Log<br>Each anomaly gets a Slack message with the numbers, monthly impact if it keeps up, the likely cause and two buttons: Expected or Ignore 7 days. It's logged so it's never alerted twice. |
| **On Approve or Ignore Link Click** | `n8n-nodes-base.webhook` | Receives inbound HTTP requests from Slack action buttons | *None* | Look Up Anomaly in Log | ## 🔗 Decision Link Check<br>The Slack buttons open this webhook. Each link carries a one-time token that's checked against the log, so a reused or forwarded link can't change anything. |
| **Look Up Anomaly in Log** | `n8n-nodes-base.googleSheets` | Retrieves anomaly record from Google Sheets by anomaly ID | On Approve or Ignore Link Click | Validate Action Token | ## 🔗 Decision Link Check<br>The Slack buttons open this webhook. Each link carries a one-time token that's checked against the log, so a reused or forwarded link can't change anything. |
| **Validate Action Token** | `n8n-nodes-base.code` | Validates one-time security tokens and requested actions | Look Up Anomaly in Log | Token Valid? | ## 🔗 Decision Link Check<br>The Slack buttons open this webhook. Each link carries a one-time token that's checked against the log, so a reused or forwarded link can't change anything. |
| **Token Valid?** | `n8n-nodes-base.if` | Verifies whether the action token is valid | Validate Action Token | Record Decision in Log, Show Invalid Link Page | ## ✅ Record & Confirm<br>Saves the decision, shows a confirmation page and posts it to Slack. Ignored anomalies stay muted for seven days. Bad links get a clear error page. |
| **Record Decision in Log** | `n8n-nodes-base.googleSheets` | Updates decision status and suppression date in Google Sheets | Token Valid? | Show Decision Saved Page | ## ✅ Record & Confirm<br>Saves the decision, shows a confirmation page and posts it to Slack. Ignored anomalies stay muted for seven days. Bad links get a clear error page. |
| **Show Invalid Link Page** | `n8n-nodes-base.respondToWebhook` | Returns HTTP 403 response with error details for invalid links | Token Valid? | *None* | ## ✅ Record & Confirm<br>Saves the decision, shows a confirmation page and posts it to Slack. Ignored anomalies stay muted for seven days. Bad links get a clear error page. |
| **Show Decision Saved Page** | `n8n-nodes-base.respondToWebhook` | Returns HTTP 200 response confirming approval or ignore action | Record Decision in Log | Post Decision to Slack | ## ✅ Record & Confirm<br>Saves the decision, shows a confirmation page and posts it to Slack. Ignored anomalies stay muted for seven days. Bad links get a clear error page. |
| **Post Decision to Slack** | `n8n-nodes-base.slack` | Posts audit notification of user action back to Slack channel | Show Decision Saved Page | *None* | ## ✅ Record & Confirm<br>Saves the decision, shows a confirmation page and posts it to Slack. Ignored anomalies stay muted for seven days. Bad links get a clear error page. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Triggers and Configuration
1. Create a **Schedule Trigger** node (`Every Day at 9 AM IST`) configured with Cron expression `0 9 * * *`.
2. Create a **Code** node (`Set Cost Agent Config`) connected after the schedule trigger. Populate its JavaScript code to return configuration properties (`aws_enabled: true`, `gcp_enabled: false`, thresholds, and date offsets).
3. Create a **Google Sheets** node (`Read Anomaly Log Sheet`) connected after the config node. Set operation to `Read`, document ID to your Google Sheet ID, and sheet name to `Anomaly Log`. Enable `Execute Once`.

#### Step 2: Build Cloud Ingestion & Normalization
4. Create an **If** node (`AWS Enabled?`) connected after the anomaly log read. Configure a boolean condition evaluating `={{ $('Set Cost Agent Config').first().json.aws_enabled }}` to `true`.
5. Connect the True branch to an **HTTP Request** node (`Get AWS Cost by Service & Region`). Set Method to `POST`, URL to `https://ce.us-east-1.amazonaws.com/`, authentication to predefined AWS credential type, and set header `X-Amz-Target` to `AWSInsightsIndexService.GetCostAndUsage`. Configure JSON body for service and region grouping.
6. Connect `Get AWS Cost by Service & Region` to another **HTTP Request** node (`Get AWS Cost by Usage Type`). Set similar AWS credentials and headers, targeting usage type grouping.
7. Connect `Get AWS Cost by Usage Type` to a **Code** node (`Normalize AWS Costs`) to map prefixes to regions.
8. Connect the False branch of `AWS Enabled?` (and output of `Normalize AWS Costs`) to an **If** node (`GCP Enabled?`) evaluating `={{ $('Set Cost Agent Config').first().json.gcp_enabled }}`.
9. Connect the True branch of `GCP Enabled?` to an **HTTP Request** node (`Query GCP Billing Export`). Set Method to `POST`, URL to BigQuery query endpoint using GCP Project ID from config, authentication to `googleOAuth2Api`, and set JSON body for SQL execution.
10. Connect `Query GCP Billing Export` to a **Code** node (`Normalize GCP Costs`) to parse BigQuery results.

#### Step 3: Implement Anomaly Detection and Summarization
11. Connect `Normalize GCP Costs` (and False branch of `GCP Enabled?`) to a **Code** node (`Detect Day-over-Day Anomalies`). Implement statistical calculation logic (mean, standard deviation, Z-score, thresholds) and filtering against seen/muted keys.
12. Connect `Detect Day-over-Day Anomalies` to a **Split In Batches** node (`Loop Over Anomalies`).
13. Connect Branch 0 of the batch loop to a **Code** node (`Build Run Summary`) and then to a **Slack** node (`Post Run Summary to Slack`) to post daily run summaries.

#### Step 4: Configure AI Root Cause Analysis
14. Connect Branch 1 of `Loop Over Anomalies` to an **If** node (`AWS Region Known?`) checking `={{ !!$json.cloudtrail_region }}`.
15. Connect the True branch to an **HTTP Request** node (`Get Recent CloudTrail Changes`). Set URL to `=https://cloudtrail.{{ $json.cloudtrail_region }}.amazonaws.com/`, headers for `LookupEvents`, and set `onError` to `continueRegularOutput`.
16. Connect both branches (CloudTrail request output and False branch of region check) to a **Code** node (`Build AI Context`).
17. Connect `Build AI Context` to an **OpenAI** node (`AI Explain Cost Spike`) using model `gpt-4o-mini` with JSON output enabled and structured system/user prompts.

#### Step 5: Setup Slack Alerting and Google Sheets Audit Logging
18. Connect `AI Explain Cost Spike` to a **Code** node (`Build Slack Alert`) that constructs Slack Block Kit payloads and action webhook links (`/webhook/cost-anomaly-action`).
19. Connect `Build Slack Alert` to a **Slack** node (`Post Anomaly Alert to Slack`) configured for block message types.
20. Connect `Post Anomaly Alert to Slack` to a **Google Sheets** node (`Log Anomaly to Sheet`). Set operation to `Append or Update`, matching columns on `anomaly_id`. Route its output back to `Loop Over Anomalies` to complete the loop.

#### Step 6: Create Interactive Decision Webhook Handler
21. Create a **Webhook** trigger node (`On Approve or Ignore Link Click`) with path `cost-anomaly-action` and `Response Mode` set to `Respond to Webhook Node`.
22. Connect the webhook to a **Google Sheets** node (`Look Up Anomaly in Log`) using filter UI to lookup by `anomaly_id` equal to query parameter `id`.
23. Connect `Look Up Anomaly in Log` to a **Code** node (`Validate Action Token`) to verify tokens and assign suppression expiration dates.
24. Connect `Validate Action Token` to an **If** node (`Token Valid?`) checking `={{ $json.valid }}`.
25. Connect the True branch of `Token Valid?` to a **Google Sheets** node (`Record Decision in Log`) set to update status and suppression date.
26. Connect `Record Decision in Log` to a **Respond to Webhook** node (`Show Decision Saved Page`) returning HTTP `200` with confirmation HTML.
27. Connect `Show Decision Saved Page` to a **Slack** node (`Post Decision to Slack`) to broadcast the action taken.
28. Connect the False branch of `Token Valid?` to a **Respond to Webhook** node (`Show Invalid Link Page`) returning HTTP `403` with error details.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| FinOps Cloud Cost Anomaly Agent architecture for AWS and GCP | Automated daily spend anomaly detection with AI root cause analysis |
| AWS IAM Least Privilege Requirements | Requires `ce:GetCostAndUsage` and `cloudtrail:LookupEvents` permissions |
| GCP BigQuery Export Setup | Requires BigQuery billing export dataset access via Google OAuth2 |
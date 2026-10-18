Monitor refund and chargeback ratio spikes with Google Sheets, Gemini, and Slack

https://n8nworkflows.xyz/workflows/monitor-refund-and-chargeback-ratio-spikes-with-google-sheets--gemini--and-slack-20853


# Monitor refund and chargeback ratio spikes with Google Sheets, Gemini, and Slack

### 1. Workflow Overview

This workflow automates the daily monitoring of e-commerce payment health by tracking sales against refund and chargeback volumes. It calculates a rolling 7-day ratio, compares it against a 90-day historical baseline, logs the metrics to Google Sheets, and uses Google Gemini to generate diagnostic hypotheses and trigger Slack alerts if an anomalous spike is detected.

The logic is organized into the following functional blocks:
- **1.1 Schedule & Configuration:** Triggers daily execution and initializes global settings such as thresholds, Google Sheet identifiers, and Slack channels.
- **1.2 Collect Sales & Refund Data:** Fetches transactional sales records and refund/chargeback records from designated Google Sheets tabs.
- **1.3 Calculate & Detect Anomaly:** Processes raw data using custom JavaScript to derive windowed amounts, ratios, transaction counts, and an explicit anomaly flag.
- **1.4 Diagnose & Alert (Anomaly Branch):** Engages Google Gemini for diagnostic summaries, logs the anomaly snapshot to Google Sheets, and pushes a rich alert to Slack.
- **1.5 Normal Monitoring (Standard Branch):** Handles standard operational states or insufficient data scenarios by formatting and logging records without external alerting.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Schedule & Configuration

##### Overview
Initiates the automated pipeline on a strict daily schedule and populates environment-wide variables for sheet routing, alert destinations, and anomaly multipliers.

##### Nodes Involved
- `Schedule Trigger`
- `Configure Monitor Settings`

##### Node Details

- **Schedule Trigger**
  - **Type and technical role:** `n8n-nodes-base.scheduleTrigger` (Trigger Node)
  - **Configuration choices:** Configured to fire once daily at 09:00 based on the local execution instance timezone.
  - **Key expressions or variables:** None.
  - **Input and output connections:** Input: None (Entry point). Output: Connects to `Configure Monitor Settings`.
  - **Version-specific requirements:** Version 1.3.
  - **Edge cases / failure types:** Failure to trigger if the instance scheduler is inactive.

- **Configure Monitor Settings**
  - **Type and technical role:** `n8n-nodes-base.set` (Data Transformation / Setup Node)
  - **Configuration choices:** Establishes parameters including `thresholdMultiplier` (default: 2), optional `monitoringDateOverride`, Google Spreadsheet identifiers (`spreadsheetId`), tab specifications (`salesSheetName`, `refundsSheetName`, `logSheetName`), and target alerting routes (`slackChannel`).
  - **Key expressions or variables:** Static string and number assignments.
  - **Input and output connections:** Input: `Schedule Trigger`. Output: Connects to `Read Sales Data`.
  - **Version-specific requirements:** Version 3.4.
  - **Edge cases / failure types:** Incorrect Google Sheet identifiers will break downstream integrations.

---

#### Block 1.2: Collect Sales & Refund Data

##### Overview
Pulls all available sales entries and refund/chargeback records from the connected Google Sheets spreadsheet to prepare them for mathematical aggregation.

##### Nodes Involved
- `Read Sales Data`
- `Read Refund Chargeback Data`

##### Node Details

- **Read Sales Data**
  - **Type and technical role:** `n8n-nodes-base.googleSheets` (Integration Node)
  - **Configuration choices:** Retrieves list rows from the configured document ID using list-mode mapping linked to the sales tab (`gid=0`). Ensures empty outputs are handled safely via `alwaysOutputData: true`.
  - **Key expressions or variables:** Uses hardcoded document IDs or mapped spreadsheet properties.
  - **Input and output connections:** Input: `Configure Monitor Settings`. Output: Connects to `Read Refund Chargeback Data`.
  - **Version-specific requirements:** Version 4.7.
  - **Edge cases / failure types:** API rate limits, revoked Google OAuth credentials, or missing columns (`Date`, `Order ID`, `Amount`).

- **Read Refund Chargeback Data**
  - **Type and technical role:** `n8n-nodes-base.googleSheets` (Integration Node)
  - **Configuration choices:** Pulls list data from the refunds tab (`gid=412420719`). Executed once per run with `alwaysOutputData: true`.
  - **Key expressions or variables:** Document ID bindings.
  - **Input and output connections:** Input: `Read Sales Data`. Output: Connects to `Calculate Refund Chargeback Metrics`.
  - **Version-specific requirements:** Version 4.7.
  - **Edge cases / failure types:** Schema mismatches if required columns (`Date`, `Order ID`, `Type`, `Amount`, `Reason`) are missing or misnamed.

---

#### Block 1.3: Calculate & Detect Anomaly

##### Overview
Executes comprehensive deterministic logic to filter windows, compute rolling ratios, compare against historical baselines, and determine if an anomaly threshold has been breached.

##### Nodes Involved
- `Calculate Refund Chargeback Metrics`
- `Refund Ratio Spike?`

##### Node Details

- **Calculate Refund Chargeback Metrics**
  - **Type and technical role:** `n8n-nodes-base.code` (Code Execution Node)
  - **Configuration choices:** Executes a sandboxed JavaScript block parsing dates, scrubbing currency formatting, filtering items into 7-day current and 96-to-7-day historical windows, calculating financial amounts, and deriving transaction ratios.
  - **Key expressions or variables:** Reads configurations dynamically via `$('Configure Monitor Settings').first().json` and raw arrays via `$('Read Sales Data').all()` and `$('Read Refund Chargeback Data').all()`.
  - **Input and output connections:** Input: `Read Refund Chargeback Data`. Output: Connects to `Refund Ratio Spike?`.
  - **Version-specific requirements:** Version 2.
  - **Edge cases / failure types:** Throws exceptions on invalid date overrides; handles missing sales data by assigning fallback statuses (`NO_CURRENT_SALES` or `INSUFFICIENT_BASELINE`).

- **Refund Ratio Spike?**
  - **Type and technical role:** `n8n-nodes-base.if` (Conditional Router Node)
  - **Configuration choices:** Evaluates the boolean value of the anomaly flag generated upstream.
  - **Key expressions or variables:** `={{ $json.anomaly }}`
  - **Input and output connections:** Input: `Calculate Refund Chargeback Metrics`. Output: True branch connects to `Generate Diagnostic Summary`; False branch connects to `Prepare Normal Snapshot`.
  - **Version-specific requirements:** Version 2.3.
  - **Edge cases / failure types:** Type validation errors if upstream data types change.

---

#### Block 1.4: Diagnose & Alert (Anomaly Branch)

##### Overview
Generates an AI-driven diagnosis using Google Gemini when an anomaly is confirmed, records the snapshot into a logging sheet, and fires a formatted notification to Slack.

##### Nodes Involved
- `Generate Diagnostic Summary`
- `Prepare Anomaly Snapshot`
- `Log Ratio Snapshot`
- `Send Refund Chargeback Alert`

##### Node Details

- **Generate Diagnostic Summary**
  - **Type and technical role:** `@n8n/n8n-nodes-langchain.googleGemini` (AI / LLM Node)
  - **Configuration choices:** Uses the `models/gemini-2.5-flash` model with a low temperature (0.3) to enforce deterministic, focused reasoning without hallucinating confirmations of fraud.
  - **Key expressions or variables:** Injects live metrics into system and user prompts using expressions such as `{{ $('Calculate Refund Chargeback Metrics').item.json.currentRatio }}`.
  - **Input and output connections:** Input: `Refund Ratio Spike?` (True). Output: Connects to `Prepare Anomaly Snapshot`.
  - **Sub-workflow reference / integration:** Connects to Google Gemini (PaLM) API credentials.
  - **Version-specific requirements:** Version 1.2.
  - **Edge cases / failure types:** API timeouts, token quota exhaustion, or unhandled model response formatting.

- **Prepare Anomaly Snapshot**
  - **Type and technical role:** `n8n-nodes-base.code` (Data Transformation Node)
  - **Configuration choices:** Extracts and cleans text outputs from the Gemini response object, merging it directly with underlying financial metrics.
  - **Key expressions or variables:** `$input.first().json` and `$('Calculate Refund Chargeback Metrics').first().json`.
  - **Input and output connections:** Input: `Generate Diagnostic Summary`. Output: Connects to `Log Ratio Snapshot`.
  - **Version-specific requirements:** Version 2.
  - **Edge cases / failure types:** Property access errors if the Gemini response schema changes.

- **Log Ratio Snapshot**
  - **Type and technical role:** `n8n-nodes-base.googleSheets` (Integration Node)
  - **Configuration choices:** Appends a new data row to the `Ratio Monitor Log` tab mapping computed financial metrics, status, and anomaly flags.
  - **Key expressions or variables:** Mapped properties such as `={{ $json.status }}` and `={{ $json.anomaly }}`.
  - **Input and output connections:** Input: `Prepare Anomaly Snapshot`. Output: Connects to `Send Refund Chargeback Alert`.
  - **Version-specific requirements:** Version 4.7.
  - **Edge cases / failure types:** Sheet write failures due to locked documents or permission scope errors.

- **Send Refund Chargeback Alert**
  - **Type and technical role:** `n8n-nodes-base.slack` (Messaging Integration Node)
  - **Configuration choices:** Posts a markdown-formatted incident alert to the configured channel (`C0ATZFFKRUH` / `#seller-risk-alerts`).
  - **Key expressions or variables:** Rich template strings referencing metrics and AI diagnostic hypotheses: `={{ $('Calculate Refund Chargeback Metrics').item.json.currentRatio }}`.
  - **Input and output connections:** Input: `Log Ratio Snapshot`. Output: None (Terminal node).
  - **Version-specific requirements:** Version 2.5.
  - **Edge cases / failure types:** Slack API token authorization errors or missing channel write permissions.

---

#### Block 1.5: Normal Monitoring (Standard Branch)

##### Overview
Processes non-anomalous executions or insufficient data states, ensuring complete audit trails are written to the logging sheet without triggering false-positive alerts.

##### Nodes Involved
- `Prepare Normal Snapshot`
- `Log Normal Snapshot`

##### Node Details

- **Prepare Normal Snapshot**
  - **Type and technical role:** `n8n-nodes-base.code` (Data Transformation Node)
  - **Configuration choices:** Attaches a contextual diagnostic summary stating normal operational status or status explanations without invoking an LLM.
  - **Key expressions or variables:** Evaluates `metrics.status` to assign descriptive string messages.
  - **Input and output connections:** Input: `Refund Ratio Spike?` (False). Output: Connects to `Log Normal Snapshot`.
  - **Version-specific requirements:** Version 2.
  - **Edge cases / failure types:** Reference errors if metric properties are undefined.

- **Log Normal Snapshot**
  - **Type and technical role:** `n8n-nodes-base.googleSheets` (Integration Node)
  - **Configuration choices:** Appends baseline operational metrics (`NORMAL`, `NO_CURRENT_SALES`, or `INSUFFICIENT_BASELINE`) to the audit log sheet.
  - **Key expressions or variables:** `={{ $json.status }}` and associated metrics.
  - **Input and output connections:** Input: `Prepare Normal Snapshot`. Output: None (Terminal node).
  - **Version-specific requirements:** Version 4.7.
  - **Edge cases / failure types:** Append failures resulting from worksheet structural alterations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Triggers execution on a daily interval | None | Configure Monitor Settings | Schedule & Configuration<br>Runs the workflow daily at 09:00 and loads the threshold, Google Sheets, and Slack configuration. |
| Configure Monitor Settings | n8n-nodes-base.set | Assigns threshold limits and sheet configurations | Schedule Trigger | Read Sales Data | Schedule & Configuration<br>Runs the workflow daily at 09:00 and loads the threshold, Google Sheets, and Slack configuration. |
| Read Sales Data | n8n-nodes-base.googleSheets | Retrieves sales transactions from Google Sheets | Configure Monitor Settings | Read Refund Chargeback Data | Collect Sales & Refund Data<br>Fetches sales, refunds, and chargebacks from Google Sheets for the monitoring period. |
| Read Refund Chargeback Data | n8n-nodes-base.googleSheets | Retrieves refund and chargeback records | Read Sales Data | Calculate Refund Chargeback Metrics | Collect Sales & Refund Data<br>Fetches sales, refunds, and chargebacks from Google Sheets for the monitoring period. |
| Calculate Refund Chargeback Metrics | n8n-nodes-base.code | Computes rolling window sums, ratios, and anomaly flags | Read Refund Chargeback Data | Refund Ratio Spike? | Calculate & Detect Anomaly<br>Calculates the 7-day refund/chargeback ratio, compares it with the 90-day baseline, and detects spikes above the threshold |
| Refund Ratio Spike? | n8n-nodes-base.if | Routes workflow based on whether an anomaly occurred | Calculate Refund Chargeback Metrics | Generate Diagnostic Summary, Prepare Normal Snapshot | Calculate & Detect Anomaly<br>Calculates the 7-day refund/chargeback ratio, compares it with the 90-day baseline, and detects spikes above the threshold |
| Generate Diagnostic Summary | @n8n/n8n-nodes-langchain.googleGemini | Generates AI hypotheses explaining the anomaly | Refund Ratio Spike? | Prepare Anomaly Snapshot | Diagnose & Alert<br>Gemini explains possible causes, the result is logged, and Slack alerts the seller when an anomaly is detected. Causes are hypotheses, not confirmed fraud. |
| Prepare Anomaly Snapshot | n8n-nodes-base.code | Normalizes AI diagnostic text with metrics | Generate Diagnostic Summary | Log Ratio Snapshot | Diagnose & Alert<br>Gemini explains possible causes, the result is logged, and Slack alerts the seller when an anomaly is detected. Causes are hypotheses, not confirmed fraud. |
| Log Ratio Snapshot | n8n-nodes-base.googleSheets | Appends anomaly snapshot to Google Sheets log | Prepare Anomaly Snapshot | Send Refund Chargeback Alert | Diagnose & Alert<br>Gemini explains possible causes, the result is logged, and Slack alerts the seller when an anomaly is detected. Causes are hypotheses, not confirmed fraud. |
| Send Refund Chargeback Alert | n8n-nodes-base.slack | Sends incident warning to Slack channel | Log Ratio Snapshot | None | Diagnose & Alert<br>Gemini explains possible causes, the result is logged, and Slack alerts the seller when an anomaly is detected. Causes are hypotheses, not confirmed fraud. |
| Prepare Normal Snapshot | n8n-nodes-base.code | Formats normal operational payloads | Refund Ratio Spike? | Log Normal Snapshot | Normal Monitoring<br>Logs normal or insufficient-data results for historical tracking without sending Gemini or Slack alerts. |
| Log Normal Snapshot | n8n-nodes-base.googleSheets | Logs routine operational metrics | Prepare Normal Snapshot | None | Normal Monitoring<br>Logs normal or insufficient-data results for historical tracking without sending Gemini or Slack alerts. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually inside an n8n instance:

1. **Create the Entry Point:**
   - Add a **Schedule Trigger** node. Set its interval rule to run daily at `09:00`.
2. **Configure Global Variables:**
   - Add a **Set** node named `Configure Monitor Settings`.
   - Add string/number assignments: `thresholdMultiplier` (Number: `2`), `monitoringDateOverride` (String: `""`), `spreadsheetId` (String: your Google Sheet ID), `salesSheetName` (String: `Sales`), `refundsSheetName` (String: `Refunds`), `logSheetName` (String: `Ratio Monitor Log`), and `slackChannel` (String: `#seller-risk-alerts`).
   - Connect **Schedule Trigger** to **Configure Monitor Settings**.
3. **Set Up Data Ingestion:**
   - Add a **Google Sheets** node named `Read Sales Data`. Configure authentication using a Google OAuth2 credential. Set Resource to `Sheet` and Operation to `Get` (or list rows), targeting your `Sales` tab. Enable `Always Output Data`.
   - Connect **Configure Monitor Settings** to **Read Sales Data**.
   - Add a second **Google Sheets** node named `Read Refund Chargeback Data`. Configure it to target the `Refunds` tab using the same document ID and authentication. Enable `Always Output Data` and execute once.
   - Connect **Read Sales Data** to **Read Refund Chargeback Data**.
4. **Implement Financial Calculations:**
   - Add a **Code** node named `Calculate Refund Chargeback Metrics`.
   - Paste the deterministic JavaScript snippet responsible for parsing dates, aggregating 7-day current volumes, calculating 96-to-7-day historical baselines, computing transaction ratios, and setting the `anomaly` boolean flag based on the `thresholdMultiplier`.
   - Connect **Read Refund Chargeback Data** to **Calculate Refund Chargeback Metrics**.
5. **Add Conditional Routing:**
   - Add an **If** node named `Refund Ratio Spike?`.
   - Configure a condition where `{{ $json.anomaly }}` equals `true`.
   - Connect **Calculate Refund Chargeback Metrics** to **Refund Ratio Spike?**.
6. **Build the Anomaly Path (True Branch):**
   - Add a **Google Gemini (PaLM)** node named `Generate Diagnostic Summary`. Configure it with a Google Gemini API credential and select `models/gemini-2.5-flash`.
   - Connect the True output of **Refund Ratio Spike?** to **Generate Diagnostic Summary**.
   - Add a **Code** node named `Prepare Anomaly Snapshot` to parse and merge Gemini's markdown output with the numerical metrics object. Connect **Generate Diagnostic Summary** to it.
   - Add a **Google Sheets** node named `Log Ratio Snapshot`. Set operation to `Append` and map parameters (`monitoring_date`, `current_sales_amount`, `current_refund_amount`, `current_chargeback_amount`, `current_ratio`, `historical_baseline`, `spike_multiplier`, `status`, `anomaly`) to their matching JSON properties. Connect **Prepare Anomaly Snapshot** to **Log Ratio Snapshot**.
   - Add a **Slack** node named `Send Refund Chargeback Alert`. Configure a Slack OAuth2 or Bot token credential, select `Channel`, select your target alert channel, and paste the markdown alert template containing metrics and diagnostic summaries. Connect **Log Ratio Snapshot** to **Send Refund Chargeback Alert**.
7. **Build the Normal Path (False Branch):**
   - Add a **Code** node named `Prepare Normal Snapshot` to format non-anomalous status payloads. Connect the False output of **Refund Ratio Spike?** to this node.
   - Add a **Google Sheets** node named `Log Normal Snapshot`. Configure it identically to the anomaly logger, appending routine operational snapshots targeting the `Ratio Monitor Log` tab. Connect **Prepare Normal Snapshot** to **Log Normal Snapshot**.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Refund/Chargeback Ratio Anomaly Monitor for Small Sellers | Core solution architecture designed to safeguard small e-commerce merchants against payment processing penalties. |
| Google Sheets Setup Requirement | Requires a structured workbook containing separate tabs for Sales, Refunds, and Audit Logs (`Ratio Monitor Log`). |
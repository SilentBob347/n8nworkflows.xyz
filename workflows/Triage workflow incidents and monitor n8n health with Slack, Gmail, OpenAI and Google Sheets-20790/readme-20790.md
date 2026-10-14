Triage workflow incidents and monitor n8n health with Slack, Gmail, OpenAI and Google Sheets

https://n8nworkflows.xyz/workflows/triage-workflow-incidents-and-monitor-n8n-health-with-slack--gmail--openai-and-google-sheets-20790


# Triage workflow incidents and monitor n8n health with Slack, Gmail, OpenAI and Google Sheets

### 1. Workflow Overview

This workflow automates the health, security, and performance monitoring of an n8n instance. It runs on a dual schedule (hourly for active incident detection and weekly for operational trend analysis), queries the n8n REST API, calculates metrics (error rates, run counts, latency, and estimated LLM token expenditure), detects anomalies, automatically deactivates runaway or failing workflows, and dispatches rich notifications and metrics records via Slack, Gmail, and Google Sheets.

The workflow logic is divided into the following functional blocks:
- **1.1 Schedule & Configuration Initialization:** Entry points handling hourly guard runs or weekly digest intervals, establishing runtime windows and global monitoring parameters.
- **1.2 Metric Collection & Indexing:** Discovers all active workflows, filters out the monitoring workflow itself, queries execution history, and aggregates performance, error, and AI token consumption metrics.
- **1.3 Execution Branching:** Evaluates whether the current execution run is in hourly "guard" mode or weekly "digest" mode.
- **1.4 Incident Triage (Guard Branch):** Inspects historical incident logs to suppress duplicate alerts via a cooldown period, classifies workflow performance issues into critical or warning states, and retrieves specific error contexts from failed executions.
- **1.5 Trend Analysis & Error Clustering (Digest Branch):** Evaluates historical metrics snapshots against baseline periods to identify regressions, volume spikes, or silent triggers, and parses recent failure logs to cluster root causes.
- **1.6 Remediation, Alerting & Archiving (Guard Branch):** Automatically disables critical workflows (unless protected by tags), posts structured incident alerts to Slack, and logs actions to Google Sheets.
- **1.7 AI Digest, Reporting & History (Digest Branch):** Synthesizes weekly data using an OpenAI language model, emails a formatted HTML report, posts a summary to Slack, and persists current operational metrics to Google Sheets.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Schedule & Configuration Initialization
**Overview:** This block triggers the workflow on an hourly schedule or a weekly cron expression, assigns tracking windows, and defines system-wide configuration thresholds, budgets, and endpoint base URLs.

**Nodes Involved:**
- `Every Hour (Guard)`
- `Every Monday 9 AM (Digest)`
- `Set Guard Mode (1h Window)`
- `Set Digest Mode (7-Day Window)`
- `Set Monitor Config`

**Node Details:**

- **Every Hour (Guard)**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` — Triggers execution every hour.
  - **Configuration:** Rule configured for hourly interval (`field: "hours"`).
  - **Expressions:** None.
  - **Connections:** Output connects to `Set Guard Mode (1h Window)`.
  - **Edge Cases / Failures:** None.

- **Every Monday 9 AM (Digest)**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` — Triggers execution weekly on Mondays at 09:00 AM.
  - **Configuration:** Cron expression rule (`0 9 * * 1`).
  - **Expressions:** None.
  - **Connections:** Output connects to `Set Digest Mode (7-Day Window)`.
  - **Edge Cases / Failures:** None.

- **Set Guard Mode (1h Window)**
  - **Type & Technical Role:** `n8n-nodes-base.set` — Injects mode variables for hourly execution.
  - **Configuration:** Sets `mode = "guard"` and `window_hours = 1`.
  - **Expressions:** Static values.
  - **Connections:** Input from `Every Hour (Guard)`; output connects to `Set Monitor Config`.
  - **Edge Cases / Failures:** None.

- **Set Digest Mode (7-Day Window)**
  - **Type & Technical Role:** `n8n-nodes-base.set` — Injects mode variables for weekly execution.
  - **Configuration:** Sets `mode = "digest"` and `window_hours = 168`.
  - **Expressions:** Static values.
  - **Connections:** Input from `Every Monday 9 AM (Digest)`; output connects to `Set Monitor Config`.
  - **Edge Cases / Failures:** None.

- **Set Monitor Config**
  - **Type & Technical Role:** `n8n-nodes-base.set` — Establishes global threshold parameters.
  - **Configuration:** Defines instance URL, execution limits, runaway thresholds, max error rates, hourly USD token budgets, warning ratios, cooldown periods, slow execution thresholds, default AI models, and digest recipient emails.
  - **Expressions:** Static assignment of configuration parameters.
  - **Connections:** Input from `Set Guard Mode` or `Set Digest Mode`; output connects to `Fetch Active Workflows`.
  - **Edge Cases / Failures:** Invalid base URLs or malformed thresholds will cause subsequent HTTP requests or mathematical evaluations to fail.

---

#### 1.2 Metric Collection & Indexing
**Overview:** This block queries the n8n API to retrieve all active workflows, parses node structures to identify AI workloads, fetches recent execution logs using pagination, and aggregates health and cost metrics.

**Nodes Involved:**
- `Fetch Active Workflows`
- `Build Workflow Index`
- `Fetch Executions per Workflow`
- `Compute Health & Cost Metrics`

**Node Details:**

- **Fetch Active Workflows**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Calls the n8n REST API to list active workflows.
  - **Configuration:** GET request to `{{ $json.n8n_base_url }}/api/v1/workflows`. Query parameters include `active=true` and `limit=250`. Uses generic HTTP Header Authentication (`X-N8N-API-KEY`).
  - **Expressions:** `={{ $json.n8n_base_url }}/api/v1/workflows`
  - **Connections:** Input from `Set Monitor Config`; output connects to `Build Workflow Index`.
  - **Edge Cases / Failures:** Authentication errors if the n8n API key is invalid or lacks permissions. Connection timeouts if the n8n instance is unresponsive.

- **Build Workflow Index**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript code node that filters out the monitor workflow itself and extracts metadata, tags, and AI model configurations.
  - **Configuration:** Evaluates workflow nodes against known AI service signatures (`openai`, `anthropic`, `gemini`, etc.) and determines if workflows are protected by tags (`no-auto-disable` or `critical`).
  - **Expressions:** Accesses upstream configuration using `$('Set Monitor Config').first().json`.
  - **Connections:** Input from `Fetch Active Workflows`; output connects to `Fetch Executions per Workflow`.
  - **Edge Cases / Failures:** Returns empty arrays if workflow data payloads are malformed.

- **Fetch Executions per Workflow**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Fetches execution history for each indexed workflow using cursor-based pagination and batching.
  - **Configuration:** GET request to `{{ $json.base_url }}/api/v1/executions`. Includes batch size parameters (5 items, 300ms interval), pagination limits (max 20 requests), and custom completion expressions based on cursor presence and execution timestamps.
  - **Expressions:** URL uses `={{ $json.base_url }}/api/v1/executions`. Query parameters evaluate `workflowId`, `includeData` (conditional based on AI usage), and limits.
  - **Connections:** Input from `Build Workflow Index`; output connects to `Compute Health & Cost Metrics`.
  - **Edge Cases / Failures:** Rate limits on the n8n API if execution volume is extremely high. Handled by `onError: continueRegularOutput`.

- **Compute Health & Cost Metrics**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript code node processing raw execution pages into structured health metrics and estimated LLM costs.
  - **Configuration:** Iterates through execution records, calculates run counts, failures, success rates, duration percentiles (p95), scans token usage structures recursively, and applies cost models based on specified AI models. Evaluates operational health states against guard thresholds.
  - **Expressions:** Accesses upstream data via `$('Set Monitor Config')` and `$('Build Workflow Index')`.
  - **Connections:** Input from `Fetch Executions per Workflow`; output connects to `Route by Mode`.
  - **Edge Cases / Failures:** Expression execution failures if unexpected payload schemas are encountered in execution data.

---

#### 1.3 Execution Branching
**Overview:** Evaluates the runtime mode variable to route data downstream into either the hourly incident guard pipeline or the weekly trend digest pipeline.

**Nodes Involved:**
- `Route by Mode`

**Node Details:**

- **Route by Mode**
  - **Type & Technical Role:** `n8n-nodes-base.switch` — Routes execution flow based on operational mode.
  - **Configuration:** Evaluates `{{ $json.mode }}`. Output 0 matches `"guard"`; Output 1 matches `"digest"`.
  - **Expressions:** `={{ $json.mode }}`
  - **Connections:** Input from `Compute Health & Cost Metrics`. Output 0 connects to `Read Guard Log Sheet`; Output 1 connects to `Read Metrics History Sheet`.
  - **Edge Cases / Failures:** Unmatched mode values will drop items unless fallback routing is configured.

---

#### 1.4 Incident Triage (Guard Branch)
**Overview:** In guard mode, reads historical incident logs from Google Sheets to suppress repeat warnings during cooldown windows, classifies workflows by severity, and fetches failure details from the n8n API.

**Nodes Involved:**
- `Read Guard Log Sheet`
- `Classify & Dedupe Incidents`
- `Fetch Latest Failed Execution`
- `Extract Error Context`

**Node Details:**

- **Read Guard Log Sheet**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Retrieves historical alert logs to enforce alert cooldowns.
  - **Configuration:** Reads from sheet name `Guard_Log` using configured document ID. `alwaysOutputData` enabled.
  - **Expressions:** Sheet ID mapping.
  - **Connections:** Input from `Route by Mode` (Guard branch); output connects to `Classify & Dedupe Incidents`.
  - **Edge Cases / Failures:** Google API authorization errors or missing worksheet tabs.

- **Classify & Dedupe Incidents**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript node filtering out recently warned workflows based on cooldown periods and tracking 7-day disable frequencies.
  - **Configuration:** Compares log timestamps against `alert_cooldown_hours`, sets severity categories (`critical` or `warning`), and determines required remediation actions.
  - **Expressions:** References metrics and guard log nodes.
  - **Connections:** Input from `Read Guard Log Sheet`; output connects to `Fetch Latest Failed Execution`.
  - **Edge Cases / Failures:** Malformed timestamp strings in log history.

- **Fetch Latest Failed Execution**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Retrieves full execution payload for the most recent failure of a flagged workflow.
  - **Configuration:** GET request to `{{ $json.base_url }}/api/v1/executions/{{ $json.last_error_id }}` with `includeData=true`.
  - **Expressions:** `={{ $json.base_url }}/api/v1/executions/{{ $json.last_error_id }}`
  - **Connections:** Input from `Classify & Dedupe Incidents`; output connects to `Extract Error Context`.
  - **Edge Cases / Failures:** 404 Not Found if execution ID has been pruned by n8n retention policies. Handled by `onError: continueRegularOutput`.

- **Extract Error Context**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript node extracting specific failing node names and error messages from execution payloads.
  - **Configuration:** Traverses execution result data to pinpoint error origins, descriptions, or last executed nodes.
  - **Expressions:** Evaluates item properties.
  - **Connections:** Input from `Fetch Latest Failed Execution`; output connects to `Auto-Disable Runaway?`.
  - **Edge Cases / Failures:** Missing error properties result in fallback strings (`n/a`).

---

#### 1.5 Trend Analysis & Error Clusters (Digest Branch)
**Overview:** In digest mode, reads historical metrics from Google Sheets, calculates week-over-week performance deltas, detects anomalies against 4-week baselines, and clusters error signatures from sampled failures.

**Nodes Involved:**
- `Read Metrics History Sheet`
- `Build Failure Sample List`
- `Fetch Failed Execution Detail`
- `Analyze Trends & Error Clusters`

**Node Details:**

- **Read Metrics History Sheet**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Reads historical snapshots for trend comparisons.
  - **Configuration:** Reads from sheet name `Metrics_History` using configured document ID.
  - **Expressions:** Sheet ID mapping.
  - **Connections:** Input from `Route by Mode` (Digest branch); output connects to `Build Failure Sample List`.
  - **Edge Cases / Failures:** Missing history sheet tabs on initial runs.

- **Build Failure Sample List**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Selects up to 6 workflows with recent failures and flattens their error execution IDs into individual items.
  - **Configuration:** Slices recent error IDs for sampling.
  - **Expressions:** References compute metrics data.
  - **Connections:** Input from `Read Metrics History Sheet`; output connects to `Fetch Failed Execution Detail`.
  - **Edge Cases / Failures:** Empty failure arrays produce placeholder items to keep execution chains active.

- **Fetch Failed Execution Detail**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Fetches execution details for sampled failure IDs.
  - **Configuration:** GET request to `{{ $json.base_url }}/api/v1/executions/{{ $json.execution_id }}` with batching options.
  - **Expressions:** `={{ $json.base_url }}/api/v1/executions/{{ $json.execution_id }}`
  - **Connections:** Input from `Build Failure Sample List`; output connects to `Analyze Trends & Error Clusters`.
  - **Edge Cases / Failures:** Pruned executions return null payloads. Handled by `onError: continueRegularOutput`.

- **Analyze Trends & Error Clusters**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript node comparing current metrics against historical baselines, calculating week-over-week deltas, and normalizing error messages into clustered signatures.
  - **Configuration:** Categorizes errors (credentials, rate limits, timeouts, syntax errors, code exceptions) and flags anomalies (error spikes, silent workflows, cost spikes, latency regressions).
  - **Expressions:** Aggregates multi-node inputs.
  - **Connections:** Input from `Fetch Failed Execution Detail`; output connects to `AI Ops Analyst Summary`.
  - **Edge Cases / Failures:** Insufficient historical baseline data on initial runs.

---

#### 1.6 Remediation, Alerting & Archiving (Guard Branch)
**Overview:** Evaluates incident triage actions to automatically deactivate runaway workflows via the n8n API, posts rich alerts to Slack, and logs incident records to Google Sheets.

**Nodes Involved:**
- `Auto-Disable Runaway?`
- `Deactivate Runaway Workflow`
- `Post Incident Alert to Slack`
- `Log Incident to Sheet`

**Node Details:**

- **Auto-Disable Runaway?**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Checks whether an incident action requires automatic deactivation.
  - **Configuration:** Evaluates `{{ $json.action }}` equals `"auto_disable"`.
  - **Expressions:** `={{ $json.action }}`
  - **Connections:** Input from `Extract Error Context`. True branch connects to `Deactivate Runaway Workflow`; False branch connects to `Post Incident Alert to Slack`.
  - **Edge Cases / Failures:** None.

- **Deactivate Runaway Workflow**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Sends a POST request to the n8n API to deactivate a critical runaway workflow.
  - **Configuration:** POST request to `={{ $json.base_url }}/api/v1/workflows/{{ $json.workflow_id }}/deactivate` using generic header authentication.
  - **Expressions:** `={{ $json.base_url }}/api/v1/workflows/{{ $json.workflow_id }}/deactivate`
  - **Connections:** Input from True branch of `Auto-Disable Runaway?`; output connects to `Post Incident Alert to Slack`.
  - **Edge Cases / Failures:** API permission issues or invalid workflow IDs. Handled by `onError: continueRegularOutput`.

- **Post Incident Alert to Slack**
  - **Type & Technical Role:** `n8n-nodes-base.slack` — Posts formatted incident alerts with error context and remediation status to a designated Slack channel.
  - **Configuration:** Message text constructed via expressions referencing error context nodes. Target select: channel.
  - **Expressions:** Complex template strings accessing `Extract Error Context` properties (severity, workflow name, metrics, failing node, error message).
  - **Connections:** Inputs from False branch of `Auto-Disable Runaway?` and `Deactivate Runaway Workflow`; output connects to `Log Incident to Sheet`.
  - **Edge Cases / Failures:** Slack API rate limits or invalid channel IDs.

- **Log Incident to Sheet**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Appends incident triage details to the `Guard_Log` worksheet.
  - **Configuration:** Operation: append. Maps schema columns (`logged_at`, `workflow_id`, `workflow_name`, `severity`, `action`, `reasons`, `failing_node`, `error_message`) using expressions.
  - **Expressions:** Maps values from `Extract Error Context` and current ISO timestamps (`$now.toISO()`).
  - **Connections:** Input from `Post Incident Alert to Slack`; end of branch.
  - **Edge Cases / Failures:** Spreadsheet permission issues or schema mismatches.

---

#### 1.7 AI Digest, Reporting & History (Digest Branch)
**Overview:** Synthesizes weekly metrics and error clusters using OpenAI, builds formatted HTML digests and Slack summaries, emails reports via Gmail, posts TL;DRs to Slack, and persists metrics snapshots to Google Sheets.

**Nodes Involved:**
- `AI Ops Analyst Summary`
- `Build Weekly Digest`
- `Email Weekly Digest`
- `Post Digest TL;DR to Slack`
- `Prepare Snapshot Rows`
- `Append to Metrics History Sheet`

**Node Details:**

- **AI Ops Analyst Summary**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.openAi` — Generates a structured operational analysis and health score using GPT-4o-mini.
  - **Configuration:** Model: `gpt-4o-mini`. Temperature: 0.2. Enforces JSON output format containing headlines, health scores, key findings, root causes, anomaly actions, and cost actions.
  - **Expressions:** System prompt instructs role as an SRE. User prompt serializes totals, week-over-week deltas, anomalies, error clusters, and top workflow metrics into JSON strings.
  - **Connections:** Input from `Analyze Trends & Error Clusters`; output connects to `Build Weekly Digest`.
  - **Edge Cases / Failures:** OpenAI API rate limits, billing limits, or invalid API keys.

- **Build Weekly Digest**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript code node assembling AI output and analytics data into an HTML email body and a Slack Markdown summary.
  - **Configuration:** Constructs responsive HTML tables, metric cards, anomaly listings, and error cluster breakdowns.
  - **Expressions:** References metrics and AI summary nodes.
  - **Connections:** Input from `AI Ops Analyst Summary`; outputs connect to `Email Weekly Digest` and `Post Digest TL;DR to Slack`.
  - **Edge Cases / Failures:** Parsing errors if AI returns malformed JSON.

- **Email Weekly Digest**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` — Sends the weekly HTML health digest via Gmail.
  - **Configuration:** Configured to send HTML messages (`sendTo`, `subject`, `message`) with attribution disabled.
  - **Expressions:** `={{ $json.to }}` and `={{ $json.html }}`.
  - **Connections:** Input from `Build Weekly Digest`; output connects to `Prepare Snapshot Rows`.
  - **Edge Cases / Failures:** Gmail OAuth token expiration or sending limits.

- **Post Digest TL;DR to Slack**
  - **Type & Technical Role:** `n8n-nodes-base.slack` — Posts the weekly operational digest TL;DR to Slack.
  - **Configuration:** Target select: channel. Channel ID configured.
  - **Expressions:** `={{ $json.slack_text }}`
  - **Connections:** Input from `Build Weekly Digest`; end of branch.
  - **Edge Cases / Failures:** Slack API rate limits or permission errors.

- **Prepare Snapshot Rows**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript node mapping workflow metrics into individual row objects for historical archival, including an aggregate summary row.
  - **Configuration:** Prepares JSON structures matching the `Metrics_History` schema.
  - **Expressions:** References trend analysis data.
  - **Connections:** Input from `Email Weekly Digest`; output connects to `Append to Metrics History Sheet`.
  - **Edge Cases / Failures:** Schema mapping discrepancies.

- **Append to Metrics History Sheet**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Appends current operational metrics snapshots to the `Metrics_History` worksheet for future week-over-week comparisons.
  - **Configuration:** Operation: append. Mapping mode: autoMapInputData.
  - **Expressions:** Input data mapping.
  - **Connections:** Input from `Prepare Snapshot Rows`; end of branch.
  - **Edge Cases / Failures:** Google Sheets API rate limits or invalid spreadsheet IDs.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note – Overview | `n8n-nodes-base.stickyNote` | Documentation overview | None | None | ## 🩺 n8n Instance Health & Cost Monitor... |
| Sticky Note – Schedules & Config | `n8n-nodes-base.stickyNote` | Documentation schedules | None | None | ## ⏰ Schedules & Config... |
| Sticky Note – Collect Metrics | `n8n-nodes-base.stickyNote` | Documentation metrics collection | None | None | ## 📊 Collect Metrics... |
| Sticky Note – Incident Triage | `n8n-nodes-base.stickyNote` | Documentation triage | None | None | ## 🔍 Incident Triage... |
| Sticky Note – Trends & Error Clusters | `n8n-nodes-base.stickyNote` | Documentation trends | None | None | ## 📈 Trends & Error Clusters... |
| Sticky Note – Auto-Disable & Alert | `n8n-nodes-base.stickyNote` | Documentation remediation | None | None | ## 🛑 Auto-Disable & Alert... |
| Sticky Note – AI Digest & History | `n8n-nodes-base.stickyNote` | Documentation AI reporting | None | None | ## 🧠 AI Digest & History... |
| Sticky Note – Credentials & Security | `n8n-nodes-base.stickyNote` | Documentation security | None | None | ## 🔐 Credentials & Security... |
| Every Hour (Guard) | `n8n-nodes-base.scheduleTrigger` | Hourly execution trigger | None | Set Guard Mode (1h Window) | ## ⏰ Schedules & Config |
| Every Monday 9 AM (Digest) | `n8n-nodes-base.scheduleTrigger` | Weekly execution trigger | None | Set Digest Mode (7-Day Window) | ## ⏰ Schedules & Config |
| Set Guard Mode (1h Window) | `n8n-nodes-base.set` | Assigns guard mode parameters | Every Hour (Guard) | Set Monitor Config | ## ⏰ Schedules & Config |
| Set Digest Mode (7-Day Window) | `n8n-nodes-base.set` | Assigns digest mode parameters | Every Monday 9 AM (Digest) | Set Monitor Config | ## ⏰ Schedules & Config |
| Set Monitor Config | `n8n-nodes-base.set` | Defines global thresholds and URLs | Set Guard Mode (1h Window), Set Digest Mode (7-Day Window) | Fetch Active Workflows | ## ⏰ Schedules & Config |
| Fetch Active Workflows | `n8n-nodes-base.httpRequest` | Queries n8n API for active workflows | Set Monitor Config | Build Workflow Index | ## 📊 Collect Metrics |
| Build Workflow Index | `n8n-nodes-base.code` | Indexes workflows and detects AI nodes | Fetch Active Workflows | Fetch Executions per Workflow | ## 📊 Collect Metrics |
| Fetch Executions per Workflow | `n8n-nodes-base.httpRequest` | Fetches workflow execution history | Build Workflow Index | Compute Health & Cost Metrics | ## 📊 Collect Metrics |
| Compute Health & Cost Metrics | `n8n-nodes-base.code` | Calculates operational health and costs | Fetch Executions per Workflow | Route by Mode | ## 📊 Collect Metrics |
| Route by Mode | `n8n-nodes-base.switch` | Branches execution by guard or digest | Compute Health & Cost Metrics | Read Guard Log Sheet, Read Metrics History Sheet | ## 📊 Collect Metrics |
| Read Guard Log Sheet | `n8n-nodes-base.googleSheets` | Reads historical incident logs | Route by Mode | Classify & Dedupe Incidents | ## 🔍 Incident Triage |
| Read Metrics History Sheet | `n8n-nodes-base.googleSheets` | Reads historical metrics snapshots | Route by Mode | Build Failure Sample List | ## 📈 Trends & Error Clusters |
| Classify & Dedupe Incidents | `n8n-nodes-base.code` | Applies cooldowns and severity filters | Read Guard Log Sheet | Fetch Latest Failed Execution | ## 🔍 Incident Triage |
| Build Failure Sample List | `n8n-nodes-base.code` | Samples recent failure execution IDs | Read Metrics History Sheet | Fetch Failed Execution Detail | ## 📈 Trends & Error Clusters |
| Fetch Latest Failed Execution | `n8n-nodes-base.httpRequest` | Fetches failure payload for triage | Classify & Dedupe Incidents | Extract Error Context | ## 🔍 Incident Triage |
| Fetch Failed Execution Detail | `n8n-nodes-base.httpRequest` | Fetches failure payloads for clustering | Build Failure Sample List | Analyze Trends & Error Clusters | ## 📈 Trends & Error Clusters |
| Extract Error Context | `n8n-nodes-base.code` | Extracts failing node and error message | Fetch Latest Failed Execution | Auto-Disable Runaway? | ## 🔍 Incident Triage |
| Analyze Trends & Error Clusters | `n8n-nodes-base.code` | Analyzes baselines and clusters errors | Fetch Failed Execution Detail | AI Ops Analyst Summary | ## 📈 Trends & Error Clusters |
| Auto-Disable Runaway? | `n8n-nodes-base.if` | Checks if auto-deactivation is required | Extract Error Context | Deactivate Runaway Workflow, Post Incident Alert to Slack | ## 🛑 Auto-Disable & Alert |
| AI Ops Analyst Summary | `@n8n/n8n-nodes-langchain.openAi` | Generates weekly AI health summary | Analyze Trends & Error Clusters | Build Weekly Digest | ## 🧠 AI Digest & History |
| Deactivate Runaway Workflow | `n8n-nodes-base.httpRequest` | Deactivates critical runaway workflows | Auto-Disable Runaway? | Post Incident Alert to Slack | ## 🛑 Auto-Disable & Alert |
| Build Weekly Digest | `n8n-nodes-base.code` | Formats email HTML and Slack summaries | AI Ops Analyst Summary | Email Weekly Digest, Post Digest TL;DR to Slack | ## 🧠 AI Digest & History |
| Post Incident Alert to Slack | `n8n-nodes-base.slack` | Posts structured incident alerts to Slack | Auto-Disable Runaway?, Deactivate Runaway Workflow | Log Incident to Sheet | ## 🛑 Auto-Disable & Alert |
| Email Weekly Digest | `n8n-nodes-base.gmail` | Sends weekly HTML digest via email | Build Weekly Digest | Prepare Snapshot Rows | ## 🧠 AI Digest & History |
| Post Digest TL;DR to Slack | `n8n-nodes-base.slack` | Posts weekly TL;DR summary to Slack | Build Weekly Digest | None | ## 🧠 AI Digest & History |
| Log Incident to Sheet | `n8n-nodes-base.googleSheets` | Appends incident details to Guard Log | Post Incident Alert to Slack | None | ## 🛑 Auto-Disable & Alert |
| Prepare Snapshot Rows | `n8n-nodes-base.code` | Prepares metric snapshot rows for sheet | Email Weekly Digest | Append to Metrics History Sheet | ## 🧠 AI Digest & History |
| Append to Metrics History Sheet | `n8n-nodes-base.googleSheets` | Appends metrics history snapshot | Prepare Snapshot Rows | None | ## 🧠 AI Digest & History |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** node named `Every Hour (Guard)`. Configure interval to hours (`1`).
   - Add a **Schedule Trigger** node named `Every Monday 9 AM (Digest)`. Configure cron expression to `0 9 * * 1`.

2. **Configure Mode Setters:**
   - Add a **Set** node named `Set Guard Mode (1h Window)`. Connect `Every Hour (Guard)` to it. Assign string variable `mode = "guard"` and number variable `window_hours = 1`.
   - Add a **Set** node named `Set Digest Mode (7-Day Window)`. Connect `Every Monday 9 AM (Digest)` to it. Assign string variable `mode = "digest"` and number variable `window_hours = 168`.

3. **Configure Global Monitor Parameters:**
   - Add a **Set** node named `Set Monitor Config`. Connect both mode setter nodes to its input.
   - Configure assignments: `n8n_base_url` (string), `max_exec_per_hour` (number, e.g., 300), `min_failures_for_runaway` (number, e.g., 20), `max_error_rate` (number, e.g., 0.8), `hourly_token_budget_usd` (number, e.g., 3), `warn_ratio` (number, e.g., 0.5), `warn_min_failures` (number, e.g., 5), `warn_error_rate` (number, e.g., 0.3), `alert_cooldown_hours` (number, e.g., 6), `slow_p95_seconds` (number, e.g., 120), `default_model` (string, `gpt-4o-mini`), and `digest_email` (string). Enable **Include Other Fields**.

4. **Fetch and Index Workflows:**
   - Add an **HTTP Request** node named `Fetch Active Workflows`. Connect `Set Monitor Config` to it. Set Method to `GET`, URL to `={{ $json.n8n_base_url }}/api/v1/workflows`, query parameters `active=true` and `limit=250`. Configure Authentication to Generic Credential Type -> HTTP Header Auth (`X-N8N-API-KEY`).
   - Add a **Code** node named `Build Workflow Index`. Connect `Fetch Active Workflows` to it. Paste JavaScript logic to filter out the workflow ID and extract AI model metadata, tags, and protection flags.

5. **Collect Executions and Compute Metrics:**
   - Add an **HTTP Request** node named `Fetch Executions per Workflow`. Connect `Build Workflow Index` to it. Set Method to `GET`, URL to `={{ $json.base_url }}/api/v1/executions`, query parameters `workflowId`, `includeData`, and `limit`. Configure pagination options with cursor parameters, max 20 requests, and completion expressions. Set **On Error** to `Continue Regular Output`.
   - Add a **Code** node named `Compute Health & Cost Metrics`. Connect `Fetch Executions per Workflow` to it. Paste JavaScript logic to iterate over execution records, aggregate run counts, failures, durations, token usage, and estimate LLM spend.

6. **Branch by Operational Mode:**
   - Add a **Switch** node named `Route by Mode`. Connect `Compute Health & Cost Metrics` to it. Configure rule to route based on `={{ $json.mode }}`: Output 0 matches `"guard"`; Output 1 matches `"digest"`.

7. **Build Guard Branch (Incident Triage & Remediation):**
   - Add a **Google Sheets** node named `Read Guard Log Sheet`. Connect Output 0 of `Route by Mode` to it. Operation: Get/Read, Sheet Name: `Guard_Log`, Document ID: your sheet ID. Enable **Always Output Data**.
   - Add a **Code** node named `Classify & Dedupe Incidents`. Connect `Read Guard Log Sheet` to it. Paste logic evaluating alert cooldowns and categorizing severity.
   - Add an **HTTP Request** node named `Fetch Latest Failed Execution`. Connect `Classify & Dedupe Incidents` to it. URL: `={{ $json.base_url }}/api/v1/executions/{{ $json.last_error_id }}` with query `includeData=true`. Set **On Error** to `Continue Regular Output`.
   - Add a **Code** node named `Extract Error Context`. Connect `Fetch Latest Failed Execution` to it. Paste extraction logic for failing nodes and error messages.
   - Add an **If** node named `Auto-Disable Runaway?`. Connect `Extract Error Context` to it. Condition: `{{ $json.action }}` equals `"auto_disable"`.
   - Add an **HTTP Request** node named `Deactivate Runaway Workflow`. Connect the True branch of `Auto-Disable Runaway?` to it. Method: `POST`, URL: `={{ $json.base_url }}/api/v1/workflows/{{ $json.workflow_id }}/deactivate`.
   - Add a **Slack** node named `Post Incident Alert to Slack`. Connect the False branch of `Auto-Disable Runaway?` and `Deactivate Runaway Workflow` to it. Configure channel ID and message expressions.
   - Add a **Google Sheets** node named `Log Incident to Sheet`. Connect `Post Incident Alert to Slack` to it. Operation: `Append`, Sheet Name: `Guard_Log`, mapping schema columns (`logged_at`, `workflow_id`, `workflow_name`, `severity`, `action`, `reasons`, `failing_node`, `error_message`).

8. **Build Digest Branch (Trends, AI Analysis & Reporting):**
   - Add a **Google Sheets** node named `Read Metrics History Sheet`. Connect Output 1 of `Route by Mode` to it. Operation: Get/Read, Sheet Name: `Metrics_History`, Document ID: your sheet ID.
   - Add a **Code** node named `Build Failure Sample List`. Connect `Read Metrics History Sheet` to it. Paste sampling logic.
   - Add an **HTTP Request** node named `Fetch Failed Execution Detail`. Connect `Build Failure Sample List` to it. URL: `={{ $json.base_url }}/api/v1/executions/{{ $json.execution_id }}`. Set **On Error** to `Continue Regular Output`.
   - Add a **Code** node named `Analyze Trends & Error Clusters`. Connect `Fetch Failed Execution Detail` to it. Paste baseline comparison, anomaly detection, and clustering logic.
   - Add an **OpenAI** node named `AI Ops Analyst Summary`. Connect `Analyze Trends & Error Clusters` to it. Model: `gpt-4o-mini`, temperature `0.2`, JSON output enabled, system prompt enforcing strict JSON output schema.
   - Add a **Code** node named `Build Weekly Digest`. Connect `AI Ops Analyst Summary` to it. Paste HTML and Slack formatting logic.
   - Add a **Gmail** node named `Email Weekly Digest`. Connect `Build Weekly Digest` to it. Configure recipient, subject, and HTML message body.
   - Add a **Slack** node named `Post Digest TL;DR to Slack`. Connect `Build Weekly Digest` to it. Configure channel ID and slack text expression.
   - Add a **Code** node named `Prepare Snapshot Rows`. Connect `Email Weekly Digest` to it. Paste row mapping logic.
   - Add a **Google Sheets** node named `Append to Metrics History Sheet`. Connect `Prepare Snapshot Rows` to it. Operation: `Append`, Sheet Name: `Metrics_History`, Mapping Mode: `Auto Map Input Data`.

9. **Credentials Setup:**
   - Configure **HTTP Header Auth** credential for n8n API requests (`X-N8N-API-KEY`).
   - Configure **Google Sheets OAuth2 / Service Account** credential.
   - Configure **Slack OAuth2 / Bot Token** credential.
   - Configure **OpenAI API** credential.
   - Configure **Gmail OAuth2** credential.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| n8n Instance Health & Cost Monitor Overview | Monitors active workflows hourly and weekly, calculates error rates and LLM costs, auto-disables runaway workflows, and dispatches alerts via Slack, Gmail, and Google Sheets. |
| Security & Credentials Best Practice | The n8n API key requires read and deactivation privileges; store securely as a credential. Restrict access to the Guard Log and Metrics History Google Sheets as execution payloads may contain sensitive data. |
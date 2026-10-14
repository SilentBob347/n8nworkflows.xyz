Monitor n8n workflow health and OpenAI cost with Slack, Gmail and Sheets

https://n8nworkflows.xyz/workflows/monitor-n8n-workflow-health-and-openai-cost-with-slack--gmail-and-sheets-20789


# Monitor n8n workflow health and OpenAI cost with Slack, Gmail and Sheets

### 1. Workflow Overview

This workflow provides automated instance-level observability, cost tracking, incident triage, and recovery mechanisms for self-hosted n8n instances. It operates in two distinct functional pipelines triggered on schedule: an hourly "Guard" mode for real-time incident detection, automated run-away prevention, AI-driven diagnostics, and Slack alerting; and a weekly "Digest" mode for deep trend analysis, error clustering, and executive summary dispatch via Gmail. Additionally, it features an interactive webhook-based action lane allowing administrators to re-enable, acknowledge, or snooze incidents directly from Slack.

The logical execution blocks include:
- **1.1 Schedule & Configuration Setup:** Establishes execution intervals (hourly guard vs. weekly digest) and initializes global operational parameters (thresholds, API keys, budgets, notification targets).
- **1.2 Metric Collection & Normalization:** Interrogates the n8n REST API to index active workflows, harvest execution logs, compute performance metrics (error rates, p95 durations), and estimate LLM expenditure based on token usage.
- **1.3 Mode Routing & Data Branching:** Splits execution flow depending on whether the trigger originated from the hourly guard or weekly digest schedule.
- **1.4 Guard Branch – Incident Triage & Context Gathering:** Evaluates execution anomalies against threshold rules, queries operational logs for cooldown and suppression states, retrieves failed execution payloads, and inspects underlying node configurations.
- **1.5 Guard Branch – AI Triage Agent & Response Construction:** Leverages an LLM agent supplemented with tool access to historical incidents and baselines to diagnose root causes, assess blast radius, formulate fix steps, and generate structured Slack Block Kit payloads.
- **1.6 Guard Branch – Mitigation, Logging, & Escalation:** Executes automatic workflow deactivation where appropriate (unless protected by tags), posts interactive alerts to Slack, logs incident data to Google Sheets, and pages on-call engineers via Gmail for critical unresolved anomalies.
- **1.7 Digest Branch – Trend Analysis & Error Clustering:** Aggregates multi-week operational snapshots, detects longitudinal performance anomalies, clusters similar error signatures, and generates trend charts.
- **1.8 Digest Branch – AI Health Reporting & Persistence:** Summarizes weekly health metrics via LLM, formats HTML digests and Slack TL;DRs, dispatches reports via Gmail and Slack, and archives historical data points into Google Sheets.
- **1.9 Action Lane – Interactive Remediation Webhook:** Listens for incoming webhook callbacks from Slack interactive buttons, validates single-use security tokens against stored incident records, executes workflow re-activation calls via API, updates operational logs, and returns confirmation interfaces.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Schedule & Configuration Setup
- **Overview:** Initializes the operational pipeline via dual schedule triggers, injects temporal window parameters, and establishes global configuration variables required across subsequent HTTP API requests and data evaluations.
- **Nodes Involved:** `Every Hour (Guard)`, `Every Monday 9 AM (Digest)`, `Set Guard Mode (1h Window)`, `Set Digest Mode (7-Day Window)`, `Set Monitor Config`.
- **Node Details:**
  - **Every Hour (Guard)**
    - Type: `n8n-nodes-base.scheduleTrigger`
    - Technical Role: Initiates hourly monitoring cycles.
    - Configuration: Interval set to fire every hour.
    - Connections: Output to `Set Guard Mode (1h Window)`.
    - Edge Cases: System clock skew or overlapping executions if instance processing stalls.
  - **Every Monday 9 AM (Digest)**
    - Type: `n8n-nodes-base.scheduleTrigger`
    - Technical Role: Initiates weekly audit and reporting cycles.
    - Configuration: Cron expression set to `0 9 * * 1`.
    - Connections: Output to `Set Digest Mode (7-Day Window)`.
    - Edge Cases: Cron evaluation discrepancies under different timezone configurations.
  - **Set Guard Mode (1h Window)**
    - Type: `n8n-nodes-base.set`
    - Technical Role: Defines pipeline runtime parameters for hourly evaluations.
    - Configuration: Sets `mode` to `guard` and `window_hours` to `1`.
    - Connections: Input from `Every Hour (Guard)`; output to `Set Monitor Config`.
  - **Set Digest Mode (7-Day Window)**
    - Type: `n8n-nodes-base.set`
    - Technical Role: Defines pipeline runtime parameters for weekly evaluations.
    - Configuration: Sets `mode` to `digest` and `window_hours` to `168`.
    - Connections: Input from `Every Monday 9 AM (Digest)`; output to `Set Monitor Config`.
  - **Set Monitor Config**
    - Type: `n8n-nodes-base.set`
    - Technical Role: Centralizes environment endpoints, thresholds, token pricing models, and alert limits.
    - Configuration: Assigns base URL, execution limits, runaway failure thresholds, error rate ceilings, LLM hourly budgets, warning ratios, cooldown intervals, slow p95 limits, default models, digest/on-call email addresses, action webhook URLs, and snooze durations. Passes other incoming fields through.
    - Connections: Input from mode-setting nodes; output to `Fetch Active Workflows`.
    - Edge Cases: Misconfigured base URLs or broken webhook targets will cascade downstream failures.

---

#### Block 1.2: Metric Collection & Normalization
- **Overview:** Fetches all active workflows from the n8n API, indexes their execution logs recursively, and computes normalized performance metrics including run counts, failure volumes, p95 durations, and estimated LLM token costs.
- **Nodes Involved:** `Fetch Active Workflows`, `Build Workflow Index`, `Fetch Executions per Workflow`, `Compute Health & Cost Metrics`.
- **Node Details:**
  - **Fetch Active Workflows**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Queries the n8n REST API to retrieve active workflows.
    - Configuration: Uses Generic Credential Type HTTP Header Auth (`X-N8N-API-KEY`). Query parameters request `active=true` with a limit of 250. Implements cursor-based pagination up to 20 max requests.
    - Connections: Input from `Set Monitor Config`; output to `Build Workflow Index`.
    - Edge Cases: API authentication failure (401/403) or pagination timeout under massive workflow inventories.
  - **Build Workflow Index**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Flattens paginated API responses, filters out the monitoring workflow itself, detects AI-enabled nodes, extracts primary LLM models, and identifies protection tags.
    - Configuration: JavaScript snippet processing workflow metadata against regular expressions (`/openai|lmchat|langchain|anthropic|gemini|mistral|ollama|groq/i`) and tags (`/no-auto-disable|critical/i`).
    - Connections: Input from `Fetch Active Workflows`; output to `Fetch Executions per Workflow`.
    - Edge Cases: Malformed workflow JSON payloads or missing node array properties.
  - **Fetch Executions per Workflow**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Pulls execution history for each indexed workflow within the evaluation window.
    - Configuration: URL targets `{{ $json.base_url }}/api/v1/executions`. Uses HTTP Header Auth. Batches requests in groups of 5 with 300ms intervals. Includes cursor-based pagination conditional on window start timestamps and limit constraints (50 for AI workflows, 250 for standard). Configured with `continueRegularOutput` on error.
    - Connections: Input from `Build Workflow Index`; output to `Compute Health & Cost Metrics`.
    - Edge Cases: Rate limiting from the internal n8n API if execution volume is exceptionally high.
  - **Compute Health & Cost Metrics**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Aggregates execution statuses, calculates runtimes, extracts token usage metrics from execution data payloads, computes estimated USD costs based on pre-configured pricing dictionaries, and evaluates operational thresholds.
    - Configuration: JavaScript processing engine evaluating p95 durations, error rates, token input/output counts, and condition checks for guard alerts.
    - Connections: Input from `Fetch Executions per Workflow`; output to `Route by Mode`.
    - Edge Cases: Unrecognized LLM model identifiers defaulting to baseline pricing, or deeply nested token usage structures exceeding recursion depth limits.

---

#### Block 1.3: Mode Routing & Data Branching
- **Overview:** Inspects the runtime mode property and routes the normalized metric payload down either the hourly Guard pipeline or the weekly Digest pipeline.
- **Nodes Involved:** `Route by Mode`.
- **Node Details:**
  - **Route by Mode**
    - Type: `n8n-nodes-base.switch`
    - Technical Role: Conditional branch router.
    - Configuration: Evaluates `{{ $json.mode }}`. Output 0 (“Guard”) matches `guard`; Output 1 (“Digest”) matches `digest`.
    - Connections: Input from `Compute Health & Cost Metrics`; Output 0 connects to `Read Guard Log Sheet`, Output 1 connects to `Read Metrics History Sheet`.
    - Edge Cases: Missing or unexpected mode string values resulting in unrouted items.

---

#### Block 1.4: Guard Branch – Incident Triage & Context Gathering
- **Overview:** Filters operational metrics to identify degraded or runaway workflows, checks alert cooldown states against historical logs, pulls the latest failed execution payload from the API, and extracts granular error context and node configurations.
- **Nodes Involved:** `Read Guard Log Sheet`, `Classify & Dedupe Incidents`, `Fetch Latest Failed Execution`, `Extract Error Context`, `Fetch Workflow Definition`, `Build Triage Context`.
- **Node Details:**
  - **Read Guard Log Sheet**
    - Type: `n8n-nodes-base.googleSheets`
    - Technical Role: Retrieves historical incident logs to evaluate cooldown periods and snooze states.
    - Configuration: Accesses Google Sheet ID `YOUR_GOOGLE_SHEET_ID`, sheet name `Guard_Log`. Always outputs data.
    - Connections: Input from `Route by Mode` (Guard branch); output to `Classify & Dedupe Incidents`.
    - Edge Cases: Google Sheets API connectivity errors or missing columns.
  - **Classify & Dedupe Incidents**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Applies cooldown periods, checks Slack snooze flags, counts recent disable events, and classifies incidents into critical or warning tiers.
    - Configuration: JavaScript processing filter and sorter prioritizing critical severity incidents.
    - Connections: Input from `Read Guard Log Sheet`; output to `Fetch Latest Failed Execution`.
    - Edge Cases: Timestamp parsing anomalies between ISO strings and spreadsheet date formats.
  - **Fetch Latest Failed Execution**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Fetches full execution details for the most recent failure associated with an incident.
    - Configuration: URL targets `{{ $json.base_url }}/api/v1/executions/{{ $json.last_error_id }}`. Uses HTTP Header Auth with `includeData=true`. Configured with `continueRegularOutput` on error.
    - Connections: Input from `Classify & Dedupe Incidents`; output to `Extract Error Context`.
    - Edge Cases: Execution ID pointing to a purged or missing execution record returning a 404.
  - **Extract Error Context**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Parses execution result data to pinpoint the failing node name and clean error descriptions.
    - Configuration: JavaScript function running once per item, extracting error messages from run data or root result data.
    - Connections: Input from `Fetch Latest Failed Execution`; output to `Fetch Workflow Definition`.
    - Edge Cases: Non-standard error object structures yielding generic fallback messages.
  - **Fetch Workflow Definition**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Retrieves the complete structural definition of the failing workflow.
    - Configuration: URL targets `{{ $json.base_url }}/api/v1/workflows/{{ $json.workflow_id }}` using HTTP Header Auth. Configured with `continueRegularOutput` on error.
    - Connections: Input from `Extract Error Context`; output to `Build Triage Context`.
    - Edge Cases: Workflow deleted or inaccessible during API fetch.
  - **Build Triage Context**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Categorizes error types (credentials, rate limits, timeouts, code errors), inspects failing node configurations (parameters, retry settings, credential types), and identifies upstream dependencies.
    - Configuration: JavaScript running once per item, scrubbing secrets to retain credential types only.
    - Connections: Input from `Fetch Workflow Definition`; output to `AI Incident Triage Agent`.
    - Edge Cases: Missing node configuration schemas returning default fallback values.

---

#### Block 1.5: Guard Branch – AI Triage Agent & Response Construction
- **Overview:** Employs an LLM agent equipped with tool access to historical logs and baseline metrics to diagnose incident root causes, evaluate safety parameters, and structure Slack Block Kit notifications.
- **Nodes Involved:** `OpenAI Chat Model`, `Incident History`, `Weekly Baseline`, `AI Incident Triage Agent`, `Triage Output Parser`, `Merge Triage & Build Alert`.
- **Node Details:**
  - **OpenAI Chat Model**
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
    - Technical Role: Provides the foundational language model for the AI triage agent.
    - Configuration: Uses model `gpt-4o-mini` with temperature set to `0.1`. Requires OpenAI API credentials.
    - Connections: Output linked via `ai_languageModel` to `AI Incident Triage Agent`.
    - Edge Cases: OpenAI API rate limits, quota exhaustion, or upstream service outages.
  - **Incident History**
    - Type: `n8n-nodes-base.googleSheetsTool`
    - Technical Role: Exposes past incident logs as an AI tool for the triage agent.
    - Configuration: Targets Google Sheet ID `YOUR_GOOGLE_SHEET_ID`, sheet `Guard_Log`, filtering on `workflow_id`.
    - Connections: Output linked via `ai_tool` to `AI Incident Triage Agent`.
    - Edge Cases: Query parameter mismatch returning empty history sets.
  - **Weekly Baseline**
    - Type: `n8n-nodes-base.googleSheetsTool`
    - Technical Role: Exposes historical weekly performance snapshots as an AI tool for the triage agent.
    - Configuration: Targets Google Sheet ID `YOUR_GOOGLE_SHEET_ID`, sheet `Metrics_History`, filtering on `workflow_id`.
    - Connections: Output linked via `ai_tool` to `AI Incident Triage Agent`.
    - Edge Cases: Missing historical metrics records for newly created workflows.
  - **AI Incident Triage Agent**
    - Type: `@n8n/n8n-nodes-langchain.agent`
    - Technical Role: Executes conversational reasoning loops to diagnose n8n workflow failures.
    - Configuration: Configured as a ReAct/Tool-calling agent with strict system instructions governing credential errors, rate limits, and safe re-enabling rules. Maximum retries set to 2 with failure retry enabled.
    - Connections: Inputs from `Build Triage Context`, `OpenAI Chat Model`, `Incident History`, `Weekly Baseline`, and `Triage Output Parser`. Output to `Merge Triage & Build Alert`.
    - Edge Cases: Agent entering infinite tool-calling loops or failing to conform to structured JSON output schemas.
  - **Triage Output Parser**
    - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`
    - Technical Role: Enforces strict JSON schema compliance on the AI agent's diagnostic output.
    - Configuration: Defines schema requiring `root_cause`, `category`, `evidence`, `fix_steps`, `suggested_node_settings`, `safe_to_reenable`, `reenable_preconditions`, `blast_radius`, `repeat_pattern`, and `confidence`.
    - Connections: Output linked via `ai_outputParser` to `AI Incident Triage Agent`.
    - Edge Cases: LLM generating malformed JSON failing schema validation.
  - **Merge Triage & Build Alert**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Combines triage diagnostics with operational context to generate unique incident IDs, single-use security tokens, Slack interactive button URLs, and Slack Block Kit structures.
    - Configuration: JavaScript generating secure random tokens, constructing deep links, and assembling rich-text block layouts.
    - Connections: Input from `AI Incident Triage Agent`; output to `Auto-Disable Runaway?`.
    - Edge Cases: Expression evaluation errors when handling unpopulated AI triage properties.

---

#### Block 1.6: Guard Branch – Mitigation, Logging, & Escalation
- **Overview:** Automatically deactivates runaway or critical workflows unless protected by tags, posts rich interactive alert cards to Slack, logs incident records to Google Sheets, and dispatches email pages to on-call engineers for high-severity or repeat incidents.
- **Nodes Involved:** `Auto-Disable Runaway?`, `Deactivate Runaway Workflow`, `Finalize Incident`, `Post Incident Alert to Slack`, `Needs Escalation?`, `Log Incident to Sheet`, `Email On-Call Engineer`.
- **Node Details:**
  - **Auto-Disable Runaway?**
    - Type: `n8n-nodes-base.if`
    - Technical Role: Determines whether an incident warrants automatic workflow deactivation.
    - Configuration: Evaluates `{{ $json.action }}` equals `auto_disable`. Uses loose type validation.
    - Connections: Input from `Merge Triage & Build Alert`; Output 0 (True) to `Deactivate Runaway Workflow`, Output 1 (False) to `Finalize Incident`.
  - **Deactivate Runaway Workflow**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Disables the degraded workflow via the n8n REST API.
    - Configuration: POST request to `{{ $json.base_url }}/api/v1/workflows/{{ $json.workflow_id }}/deactivate`. Uses HTTP Header Auth. Configured with `continueRegularOutput` on error.
    - Connections: Input from `Auto-Disable Runaway?` (True branch); output to `Finalize Incident`.
    - Edge Cases: API refusal due to permission boundaries or workflow locking.
  - **Finalize Incident**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Normalizes the state of the incident following deactivation attempts or alert-only evaluation paths.
    - Configuration: JavaScript processing API response codes, setting action results, and flagging escalation requirements if deactivation failed.
    - Connections: Inputs from `Deactivate Runaway Workflow` and `Auto-Disable Runaway?` (False branch); outputs to `Post Incident Alert to Slack` and `Needs Escalation?`.
  - **Post Incident Alert to Slack**
    - Type: `n8n-nodes-base.slack`
    - Technical Role: Broadcasts rich incident alert cards with interactive buttons to designated Slack channels.
    - Configuration: Posts block-kit messages to channel ID `YOUR_SLACK_CHANNEL_ID`. Uses Slack credentials.
    - Connections: Input from `Finalize Incident`; output to `Log Incident to Sheet`.
    - Edge Cases: Slack API rate limits or invalid channel identifiers.
  - **Needs Escalation?**
    - Type: `n8n-nodes-base.if`
    - Technical Role: Evaluates whether an incident requires immediate paging of on-call engineers via email.
    - Configuration: Evaluates boolean `{{ $json.escalate }}` is true.
    - Connections: Input from `Finalize Incident`; Output 0 (True) to `Email On-Call Engineer`.
  - **Log Incident to Sheet**
    - Type: `n8n-nodes-base.googleSheets`
    - Technical Role: Appends a permanent audit record of the incident to the Guard Log spreadsheet.
    - Configuration: Appends row data to Google Sheet ID `YOUR_GOOGLE_SHEET_ID`, sheet `Guard_Log`, mapping incident metadata, AI diagnostics, and security tokens.
    - Connections: Input from `Post Incident Alert to Slack`.
    - Edge Cases: Spreadsheet write locks or schema mismatch errors.
  - **Email On-Call Engineer**
    - Type: `n8n-nodes-base.gmail`
    - Technical Role: Sends high-priority incident notification emails to the on-call engineer.
    - Configuration: Sends email via Gmail credentials to `{{ $('Set Monitor Config').first().json.oncall_email }}` with structured HTML diagnostic summaries.
    - Connections: Input from `Needs Escalation?` (True branch).
    - Edge Cases: SMTP transmission failures or authentication revocations on Gmail OAuth2 credentials.

---

#### Block 1.7: Digest Branch – Trend Analysis & Error Clusters
- **Overview:** Reads historical metric snapshots, calculates week-over-week deltas, performs linear regression failure trend analysis, clusters similar error signatures, and generates visualization chart URLs.
- **Nodes Involved:** `Read Metrics History Sheet`, `Build Failure Sample List`, `Fetch Failed Execution Detail`, `Analyze Trends & Error Clusters`, `Build 8-Week Trend Chart`.
- **Node Details:**
  - **Read Metrics History Sheet**
    - Type: `n8n-nodes-base.googleSheets`
    - Technical Role: Retrieves historical weekly snapshots for longitudinal comparison.
    - Configuration: Accesses Google Sheet ID `YOUR_GOOGLE_SHEET_ID`, sheet `Metrics_History`. Always outputs data.
    - Connections: Input from `Route by Mode` (Digest branch); output to `Build Failure Sample List`.
  - **Build Failure Sample List**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Selects a representative sample of recent failed execution IDs across workflows for error clustering.
    - Configuration: JavaScript extracting up to 6 workflow execution error ID samples, ensuring stream continuity if zero failures occurred.
    - Connections: Input from `Read Metrics History Sheet`; output to `Fetch Failed Execution Detail`.
  - **Fetch Failed Execution Detail**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Retrieves execution payloads for failure sample clustering.
    - Configuration: URL targets `{{ $json.base_url }}/api/v1/executions/{{ $json.execution_id }}`. Uses HTTP Header Auth, batched in groups of 5 with 300ms intervals. Configured with `continueRegularOutput` on error.
    - Connections: Input from `Build Failure Sample List`; output to `Analyze Trends & Error Clusters`.
  - **Analyze Trends & Error Clusters**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Evaluates baseline performance anomalies (error spikes, silent workflows, cost spikes, latency regressions) and clusters error messages into standardized categories.
    - Configuration: JavaScript processing normalization regexes for error signatures and counting cross-workflow error patterns.
    - Connections: Input from `Fetch Failed Execution Detail`; output to `Build 8-Week Trend Chart`.
  - **Build 8-Week Trend Chart**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Compiles an 8-week historical trend dataset and generates an external charting API URL.
    - Configuration: JavaScript calculating linear regression slopes for failure counts and formatting QuickChart.io URLs (`https://quickchart.io/chart...`).
    - Connections: Input from `Analyze Trends & Error Clusters`; output to `AI Ops Analyst Summary`.

---

#### Block 1.8: Digest Branch – AI Health Reporting & Persistence
- **Overview:** Summarizes weekly instance health via an LLM analyst, formats comprehensive HTML email digests and Slack TL;DR payloads, dispatches notifications, and appends new metric snapshots to Google Sheets.
- **Nodes Involved:** `AI Ops Analyst Summary`, `Build Weekly Digest`, `Email Weekly Digest`, `Post Digest TL;DR to Slack`, `Prepare Snapshot Rows`, `Append to Metrics History Sheet`.
- **Node Details:**
  - **AI Ops Analyst Summary**
    - Type: `@n8n/n8n-nodes-langchain.openAi`
    - Technical Role: Generates an AI-driven executive summary and prioritization list based on weekly metrics.
    - Configuration: Uses model `gpt-4o-mini` with temperature `0.2` and strict JSON output formatting. Requires OpenAI credentials.
    - Connections: Input from `Build 8-Week Trend Chart`; output to `Build Weekly Digest`.
  - **Build Weekly Digest**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Transforms AI analysis and metric aggregates into structured HTML email markup and Slack text summaries.
    - Configuration: JavaScript rendering responsive HTML tables, anomaly cards, error cluster lists, and delta badges.
    - Connections: Input from `AI Ops Analyst Summary`; outputs to `Email Weekly Digest` and `Post Digest TL;DR to Slack`.
  - **Email Weekly Digest**
    - Type: `n8n-nodes-base.gmail`
    - Technical Role: Dispatches the complete HTML weekly health digest to administrative stakeholders.
    - Configuration: Sends email via Gmail credentials to `{{ $json.to }}` with subject and HTML body.
    - Connections: Input from `Build Weekly Digest`; output to `Prepare Snapshot Rows`.
  - **Post Digest TL;DR to Slack**
    - Type: `n8n-nodes-base.slack`
    - Technical Role: Posts a high-level summary and alert digest to the primary Slack monitoring channel.
    - Configuration: Sends text message to Slack channel ID `YOUR_SLACK_CHANNEL_ID`. Uses Slack credentials.
    - Connections: Input from `Build Weekly Digest`.
  - **Prepare Snapshot Rows**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Flattens per-workflow metrics and instance totals into discrete rows ready for spreadsheet archival.
    - Configuration: JavaScript mapping workflow execution statistics into individual row objects plus an aggregate `__total__` summary row.
    - Connections: Input from `Email Weekly Digest`; output to `Append to Metrics History Sheet`.
  - **Append to Metrics History Sheet**
    - Type: `n8n-nodes-base.googleSheets`
    - Technical Role: Archives weekly operational snapshots into the Metrics History spreadsheet.
    - Configuration: Appends rows to Google Sheet ID `YOUR_GOOGLE_SHEET_ID`, sheet `Metrics_History`, using auto-mapped input data.
    - Connections: Input from `Prepare Snapshot Rows`.

---

#### Block 1.9: Action Lane – Interactive Remediation Webhook
- **Overview:** Listens for HTTP webhook callbacks triggered by Slack interactive buttons, validates security tokens and action parameters against the incident log, re-activates workflows via the n8n API if requested, updates logs, and renders confirmation pages.
- **Nodes Involved:** `On Slack Action Link Click`, `Look Up Incident in Guard Log`, `Set Action Lane Config`, `Validate Action Token`, `Token Valid?`, `Is Re-enable Request?`, `Show Invalid Link Page`, `Re-activate Workflow via API`, `Update Incident Status in Sheet`, `Show Action Done Page`, `Post Action Confirmation to Slack`.
- **Node Details:**
  - **On Slack Action Link Click**
    - Type: `n8n-nodes-base.webhook`
    - Technical Role: Webhook entry point for Slack button interactions.
    - Configuration: Path set to `n8n-guard-action`, response mode set to responseNode.
    - Connections: Output to `Look Up Incident in Guard Log`.
  - **Look Up Incident in Guard Log**
    - Type: `n8n-nodes-base.googleSheets`
    - Technical Role: Retrieves the matching incident record from the Guard Log spreadsheet using the incident ID query parameter.
    - Configuration: Accesses Google Sheet ID `YOUR_GOOGLE_SHEET_ID`, sheet `Guard_Log`, filtering where `incident_id` equals `{{ $json.query.id }}`. Always outputs data.
    - Connections: Input from `On Slack Action Link Click`; output to `Set Action Lane Config`.
  - **Set Action Lane Config**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Sets operational configuration variables for the action execution lane.
    - Configuration: Returns base URL and snooze duration parameters. Executed once.
    - Connections: Input from `Look Up Incident in Guard Log`; output to `Validate Action Token`.
  - **Validate Action Token**
    - Type: `n8n-nodes-base.code`
    - Technical Role: Validates security tokens, checks expiration or reuse status, and verifies re-enable permissions.
    - Configuration: JavaScript comparing request query tokens (`t`) against stored incident tokens, checking action validity.
    - Connections: Input from `Set Action Lane Config`; output to `Token Valid?`.
  - **Token Valid?**
    - Type: `n8n-nodes-base.if`
    - Technical Role: Branches execution based on security token validity.
    - Configuration: Evaluates boolean `{{ $json.valid }}`.
    - Connections: Input from `Validate Action Token`; Output 0 (True) to `Is Re-enable Request?`, Output 1 (False) to `Show Invalid Link Page`.
  - **Is Re-enable Request?**
    - Type: `n8n-nodes-base.if`
    - Technical Role: Checks whether the requested action is to re-activate a workflow.
    - Configuration: Evaluates `{{ $json.act }}` equals `reenable`.
    - Connections: Input from `Token Valid?` (True branch); Output 0 (True) to `Re-activate Workflow via API`, Output 1 (False) to `Update Incident Status in Sheet`.
  - **Show Invalid Link Page**
    - Type: `n8n-nodes-base.respondToWebhook`
    - Technical Role: Renders an HTML error page when security token validation fails.
    - Configuration: Response code 403, Content-Type `text/html; charset=utf-8`, returning warning HTML body.
    - Connections: Input from `Token Valid?` (False branch).
  - **Re-activate Workflow via API**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Re-activates the disabled workflow via the n8n REST API.
    - Configuration: POST request to `{{ $json.base_url }}/api/v1/workflows/{{ $json.workflow_id }}/activate`. Uses HTTP Header Auth with full response output and `neverError` enabled.
    - Connections: Input from `Is Re-enable Request?` (True branch); output to `Update Incident Status in Sheet`.
  - **Update Incident Status in Sheet**
    - Type: `n8n-nodes-base.googleSheets`
    - Technical Role: Updates the incident status, resolution timestamp, and token consumption state in the Guard Log spreadsheet.
    - Configuration: Updates row in Google Sheet ID `YOUR_GOOGLE_SHEET_ID`, sheet `Guard_Log`, matching on `incident_id`.
    - Connections: Inputs from `Is Re-enable Request?` (False branch) and `Re-activate Workflow via API`; output to `Show Action Done Page`.
  - **Show Action Done Page**
    - Type: `n8n-nodes-base.respondToWebhook`
    - Technical Role: Renders an HTML confirmation page in the administrator's browser.
    - Configuration: Response code 200, Content-Type `text/html; charset=utf-8`, returning success confirmation HTML.
    - Connections: Input from `Update Incident Status in Sheet`; output to `Post Action Confirmation to Slack`.
  - **Post Action Confirmation to Slack**
    - Type: `n8n-nodes-base.slack`
    - Technical Role: Posts an audit confirmation message back to the Slack monitoring channel indicating that an action was performed.
    - Configuration: Sends text message to Slack channel ID `YOUR_SLACK_CHANNEL_ID`. Uses Slack credentials.
    - Connections: Input from `Show Action Done Page`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note – Overview | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## 🩺 n8n Instance Health & Cost Monitor v3... |
| Sticky Note – Schedules & Config | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## ⏰ Schedules & Config... |
| Sticky Note – Collect Metrics | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## 📊 Collect Metrics... |
| Sticky Note – Incident Triage | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## 🔍 Incident Triage... |
| Sticky Note – Trends & Error Clusters | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## 📈 Trends & Error Clusters... |
| Sticky Note – AI Triage Agent | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## 🧠 AI Triage Agent... |
| Sticky Note – AI Digest & History | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## 🗞️ AI Digest & History... |
| Sticky Note – Auto-Disable, Alert & Page | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## 🛑 Auto-Disable, Alert & Page... |
| Sticky Note – Action Link Check | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## 🔗 Action Link Check... |
| Sticky Note – Apply Action & Confirm | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## ✅ Apply Action & Confirm... |
| Sticky Note – Credentials & Security | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## 🔐 Credentials & Security... |
| Every Hour (Guard) | `n8n-nodes-base.scheduleTrigger` | Hourly cron trigger | None | Set Guard Mode (1h Window) | ## ⏰ Schedules & Config... |
| Every Monday 9 AM (Digest) | `n8n-nodes-base.scheduleTrigger` | Weekly cron trigger | None | Set Digest Mode (7-Day Window) | ## ⏰ Schedules & Config... |
| Set Guard Mode (1h Window) | `n8n-nodes-base.set` | Set guard mode variables | Every Hour (Guard) | Set Monitor Config | ## ⏰ Schedules & Config... |
| Set Digest Mode (7-Day Window) | `n8n-nodes-base.set` | Set digest mode variables | Every Monday 9 AM (Digest) | Set Monitor Config | ## ⏰ Schedules & Config... |
| Set Monitor Config | `n8n-nodes-base.set` | Central config assignment | Set Guard Mode (1h Window), Set Digest Mode (7-Day Window) | Fetch Active Workflows | ## ⏰ Schedules & Config... |
| Fetch Active Workflows | `n8n-nodes-base.httpRequest` | Fetch active workflows from API | Set Monitor Config | Build Workflow Index | ## 📊 Collect Metrics... |
| Build Workflow Index | `n8n-nodes-base.code` | Index workflows & AI metadata | Fetch Active Workflows | Fetch Executions per Workflow | ## 📊 Collect Metrics... |
| Fetch Executions per Workflow | `n8n-nodes-base.httpRequest` | Fetch execution logs | Build Workflow Index | Compute Health & Cost Metrics | ## 📊 Collect Metrics... |
| Compute Health & Cost Metrics | `n8n-nodes-base.code` | Calculate run metrics & LLM cost | Fetch Executions per Workflow | Route by Mode | ## 📊 Collect Metrics... |
| Route by Mode | `n8n-nodes-base.switch` | Route guard vs. digest execution | Compute Health & Cost Metrics | Read Guard Log Sheet, Read Metrics History Sheet | ## 📊 Collect Metrics... |
| Read Guard Log Sheet | `n8n-nodes-base.googleSheets` | Read Guard Log from Google Sheets | Route by Mode | Classify & Dedupe Incidents | ## 🔍 Incident Triage... |
| Read Metrics History Sheet | `n8n-nodes-base.googleSheets` | Read Metrics History from Google Sheets | Route by Mode | Build Failure Sample List | ## 📈 Trends & Error Clusters... |
| Classify & Dedupe Incidents | `n8n-nodes-base.code` | Classify & deduplicate incidents | Read Guard Log Sheet | Fetch Latest Failed Execution | ## 🔍 Incident Triage... |
| Build Failure Sample List | `n8n-nodes-base.code` | Sample failed executions | Read Metrics History Sheet | Fetch Failed Execution Detail | ## 📈 Trends & Error Clusters... |
| Fetch Latest Failed Execution | `n8n-nodes-base.httpRequest` | Fetch failing run details | Classify & Dedupe Incidents | Extract Error Context | ## 🔍 Incident Triage... |
| Fetch Failed Execution Detail | `n8n-nodes-base.httpRequest` | Fetch failure sample details | Build Failure Sample List | Analyze Trends & Error Clusters | ## 📈 Trends & Error Clusters... |
| Extract Error Context | `n8n-nodes-base.code` | Extract error node and message | Fetch Latest Failed Execution | Fetch Workflow Definition | ## 🔍 Incident Triage... |
| Analyze Trends & Error Clusters | `n8n-nodes-base.code` | Detect anomalies & cluster errors | Fetch Failed Execution Detail | Build 8-Week Trend Chart | ## 📈 Trends & Error Clusters... |
| Fetch Workflow Definition | `n8n-nodes-base.httpRequest` | Fetch workflow JSON definition | Extract Error Context | Build Triage Context | ## 🔍 Incident Triage... |
| Build 8-Week Trend Chart | `n8n-nodes-base.code` | Generate 8-week trend chart URL | Analyze Trends & Error Clusters | AI Ops Analyst Summary | ## 📈 Trends & Error Clusters... |
| Build Triage Context | `n8n-nodes-base.code` | Assemble AI triage context | Fetch Workflow Definition | AI Incident Triage Agent | ## 🔍 Incident Triage... |
| OpenAI Chat Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM model for triage agent | None | AI Incident Triage Agent | ## 🧠 AI Triage Agent... |
| AI Ops Analyst Summary | `@n8n/n8n-nodes-langchain.openAi` | LLM analyst for weekly digest | Build 8-Week Trend Chart | Build Weekly Digest | ## 📈 Trends & Error Clusters... |
| AI Incident Triage Agent | `@n8n/n8n-nodes-langchain.agent` | AI agent for incident triage | Build Triage Context, OpenAI Chat Model, Incident History, Weekly Baseline, Triage Output Parser | Merge Triage & Build Alert | ## 🧠 AI Triage Agent... |
| Incident History | `n8n-nodes-base.googleSheetsTool` | Tool for past incident history | None | AI Incident Triage Agent | ## 🧠 AI Triage Agent... |
| Build Weekly Digest | `n8n-nodes-base.code` | Format HTML digest & Slack TL;DR | AI Ops Analyst Summary | Email Weekly Digest, Post Digest TL;DR to Slack | ## 🗞️ AI Digest & History... |
| Weekly Baseline | `n8n-nodes-base.googleSheetsTool` | Tool for weekly metrics baseline | None | AI Incident Triage Agent | ## 🧠 AI Triage Agent... |
| Email Weekly Digest | `n8n-nodes-base.gmail` | Send HTML weekly digest email | Build Weekly Digest | Prepare Snapshot Rows | ## 🗞️ AI Digest & History... |
| Post Digest TL;DR to Slack | `n8n-nodes-base.slack` | Post digest TL;DR to Slack | Build Weekly Digest | None | ## 🗞️ AI Digest & History... |
| Triage Output Parser | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforce JSON output schema | None | AI Incident Triage Agent | ## 🧠 AI Triage Agent... |
| Prepare Snapshot Rows | `n8n-nodes-base.code` | Flatten snapshot metrics rows | Email Weekly Digest | Append to Metrics History Sheet | ## 🗞️ AI Digest & History... |
| Merge Triage & Build Alert | `n8n-nodes-base.code` | Build Slack blocks & incident tokens | AI Incident Triage Agent | Auto-Disable Runaway? | ## 🧠 AI Triage Agent... |
| Append to Metrics History Sheet | `n8n-nodes-base.googleSheets` | Append snapshot to history sheet | Prepare Snapshot Rows | None | ## 🗞️ AI Digest & History... |
| Auto-Disable Runaway? | `n8n-nodes-base.if` | Check if auto-disable is needed | Merge Triage & Build Alert | Deactivate Runaway Workflow, Finalize Incident | ## 🛑 Auto-Disable, Alert & Page... |
| Deactivate Runaway Workflow | `n8n-nodes-base.httpRequest` | Deactivate workflow via API | Auto-Disable Runaway? | Finalize Incident | ## 🛑 Auto-Disable, Alert & Page... |
| Finalize Incident | `n8n-nodes-base.code` | Finalize incident record state | Deactivate Runaway Workflow, Auto-Disable Runaway? | Post Incident Alert to Slack, Needs Escalation? | ## 🛑 Auto-Disable, Alert & Page... |
| Post Incident Alert to Slack | `n8n-nodes-base.slack` | Post incident alert card to Slack | Finalize Incident | Log Incident to Sheet | ## 🛑 Auto-Disable, Alert & Page... |
| Needs Escalation? | `n8n-nodes-base.if` | Check if on-call escalation needed | Finalize Incident | Email On-Call Engineer | ## 🛑 Auto-Disable, Alert & Page... |
| Log Incident to Sheet | `n8n-nodes-base.googleSheets` | Log incident to Guard Log sheet | Post Incident Alert to Slack | None | ## 🛑 Auto-Disable, Alert & Page... |
| Email On-Call Engineer | `n8n-nodes-base.gmail` | Email page to on-call engineer | Needs Escalation? | None | ## 🛑 Auto-Disable, Alert & Page... |
| On Slack Action Link Click | `n8n-nodes-base.webhook` | Webhook for Slack action buttons | None | Look Up Incident in Guard Log | ## 🔗 Action Link Check... |
| Look Up Incident in Guard Log | `n8n-nodes-base.googleSheets` | Look up incident by ID in Guard Log | On Slack Action Link Click | Set Action Lane Config | ## 🔗 Action Link Check... |
| Set Action Lane Config | `n8n-nodes-base.code` | Set action lane config values | Look Up Incident in Guard Log | Validate Action Token | ## 🔗 Action Link Check... |
| Validate Action Token | `n8n-nodes-base.code` | Validate token & permissions | Set Action Lane Config | Token Valid? | ## 🔗 Action Link Check... |
| Token Valid? | `n8n-nodes-base.if` | Check if action token is valid | Validate Action Token | Is Re-enable Request?, Show Invalid Link Page | ## 🔗 Action Link Check... |
| Is Re-enable Request? | `n8n-nodes-base.if` | Check if action is reenable | Token Valid? | Re-activate Workflow via API, Update Incident Status in Sheet | ## 🔗 Action Link Check... |
| Show Invalid Link Page | `n8n-nodes-base.respondToWebhook` | Respond with 403 invalid page | Token Valid? | None | ## 🔗 Action Link Check... |
| Re-activate Workflow via API | `n8n-nodes-base.httpRequest` | Re-activate workflow via API | Is Re-enable Request? | Update Incident Status in Sheet | ## ✅ Apply Action & Confirm... |
| Update Incident Status in Sheet | `n8n-nodes-base.googleSheets` | Update incident status in sheet | Is Re-enable Request?, Re-activate Workflow via API | Show Action Done Page | ## ✅ Apply Action & Confirm... |
| Show Action Done Page | `n8n-nodes-base.respondToWebhook` | Respond with success confirmation page | Update Incident Status in Sheet | Post Action Confirmation to Slack | ## ✅ Apply Action & Confirm... |
| Post Action Confirmation to Slack | `n8n-nodes-base.slack` | Post action confirmation to Slack | Show Action Done Page | None | ## ✅ Apply Action & Confirm... |

---

### 4. Reproducing the Workflow from Scratch

#### Prerequisites Setup
1. **n8n API Key:** Generate an API key within your n8n instance settings. Store it as a **Header Auth** credential named `X-N8N-API-KEY`.
2. **External Integrations:** Configure credentials for **Google Sheets OAuth2 / Service Account**, **Slack OAuth2 / Bot Token**, **Gmail OAuth2**, and **OpenAI API Key**.
3. **Google Sheets Setup:** Create a Google Spreadsheet with two tabs named exactly `Guard_Log` and `Metrics_History`. Note the Spreadsheet ID (`YOUR_GOOGLE_SHEET_ID`).
4. **Slack Setup:** Create a private or internal Slack channel for incident alerts and digest updates. Note the Channel ID (`YOUR_SLACK_CHANNEL_ID`).
5. **Workflow Protection Tags:** Apply tags `no-auto-disable` or `critical` to any core workflows that must never be automatically deactivated by the monitor.

---

#### Step-by-Step Node Creation & Connection Guide

##### Pipeline 1: Main Monitoring Pipeline (Guard & Digest)
1. **Create Trigger Nodes:**
   - Add `Every Hour (Guard)` (`n8n-nodes-base.scheduleTrigger`). Set interval to hours.
   - Add `Every Monday 9 AM (Digest)` (`n8n-nodes-base.scheduleTrigger`). Set cron expression to `0 9 * * 1`.
2. **Create Mode Setting Nodes:**
   - Add `Set Guard Mode (1h Window)` (`n8n-nodes-base.set`). Assign string `mode` = `guard`, number `window_hours` = `1`. Connect `Every Hour (Guard)` output to this node.
   - Add `Set Digest Mode (7-Day Window)` (`n8n-nodes-base.set`). Assign string `mode` = `digest`, number `window_hours` = `168`. Connect `Every Monday 9 AM (Digest)` output to this node.
3. **Create Monitor Configuration Node:**
   - Add `Set Monitor Config` (`n8n-nodes-base.set`). Configure assignments for `n8n_base_url`, `max_exec_per_hour` (300), `min_failures_for_runaway` (20), `max_error_rate` (0.8), `hourly_token_budget_usd` (3), `warn_ratio` (0.5), `warn_min_failures` (5), `warn_error_rate` (0.3), `alert_cooldown_hours` (6), `slow_p95_seconds` (120), `default_model` (`gpt-4o-mini`), `digest_email`, `actions_webhook_url`, `oncall_email`, and `snooze_hours` (24). Enable "Include Other Fields". Connect both mode-setting nodes into this node.
4. **Fetch Active Workflows:**
   - Add `Fetch Active Workflows` (`n8n-nodes-base.httpRequest`). Method GET, URL `{{ $json.n8n_base_url }}/api/v1/workflows`. Authenticate with HTTP Header Auth. Set query parameters `active=true` and `limit=250`. Configure cursor-based pagination (max requests 20). Connect `Set Monitor Config` output here.
5. **Build Workflow Index:**
   - Add `Build Workflow Index` (`n8n-nodes-base.code`). Paste JavaScript snippet filtering out self and indexing AI models/tags. Connect from `Fetch Active Workflows`.
6. **Fetch Executions per Workflow:**
   - Add `Fetch Executions per Workflow` (`n8n-nodes-base.httpRequest`). Method GET, URL `{{ $json.base_url }}/api/v1/executions`. Authenticate with HTTP Header Auth. Batch size 5, interval 300ms. Configure pagination with cursor extraction. Set `continueRegularOutput` on error. Connect from `Build Workflow Index`.
7. **Compute Health & Cost Metrics:**
   - Add `Compute Health & Cost Metrics` (`n8n-nodes-base.code`). Paste metric aggregation and LLM pricing script. Connect from `Fetch Executions per Workflow`.
8. **Route by Mode:**
   - Add `Route by Mode` (`n8n-nodes-base.switch`). Create rules evaluating `{{ $json.mode }}` equals `guard` (Output 0) and `digest` (Output 1). Connect from `Compute Health & Cost Metrics`.

##### Pipeline 1A: Guard Lane (Hourly)
9. **Read Guard Log:**
   - Add `Read Guard Log Sheet` (`n8n-nodes-base.googleSheets`). Document ID `YOUR_GOOGLE_SHEET_ID`, Sheet Name `Guard_Log`. Always output data. Connect from `Route by Mode` Output 0.
10. **Classify & Dedupe Incidents:**
    - Add `Classify & Dedupe Incidents` (`n8n-nodes-base.code`). Paste classification script. Connect from `Read Guard Log Sheet`.
11. **Fetch Latest Failed Execution:**
    - Add `Fetch Latest Failed Execution` (`n8n-nodes-base.httpRequest`). URL `{{ $json.base_url }}/api/v1/executions/{{ $json.last_error_id }}`. HTTP Header Auth, query parameter `includeData=true`. `continueRegularOutput` on error. Connect from `Classify & Dedupe Incidents`.
12. **Extract Error Context:**
    - Add `Extract Error Context` (`n8n-nodes-base.code`). Paste error extraction script. Connect from `Fetch Latest Failed Execution`.
13. **Fetch Workflow Definition:**
    - Add `Fetch Workflow Definition` (`n8n-nodes-base.httpRequest`). URL `{{ $json.base_url }}/api/v1/workflows/{{ $json.workflow_id }}`. HTTP Header Auth. `continueRegularOutput` on error. Connect from `Extract Error Context`.
14. **Build Triage Context:**
    - Add `Build Triage Context` (`n8n-nodes-base.code`). Paste context assembly script. Connect from `Fetch Workflow Definition`.
15. **AI Agent Setup:**
    - Add `OpenAI Chat Model` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`). Model `gpt-4o-mini`, temperature `0.1`. Connect `ai_languageModel` to agent.
    - Add `Incident History` (`n8n-nodes-base.googleSheetsTool`). Document ID `YOUR_GOOGLE_SHEET_ID`, Sheet `Guard_Log`, filtering on `workflow_id`. Connect `ai_tool` to agent.
    - Add `Weekly Baseline` (`n8n-nodes-base.googleSheetsTool`). Document ID `YOUR_GOOGLE_SHEET_ID`, Sheet `Metrics_History`, filtering on `workflow_id`. Connect `ai_tool` to agent.
    - Add `Triage Output Parser` (`@n8n/n8n-nodes-langchain.outputParserStructured`). Configure with the specified JSON schema. Connect `ai_outputParser` to agent.
    - Add `AI Incident Triage Agent` (`@n8n/n8n-nodes-langchain.agent`). Prompt type define, text input referencing triage context JSON. Max retries 2 with retry on fail. Connect input from `Build Triage Context`.
16. **Merge Triage & Build Alert:**
    - Add `Merge Triage & Build Alert` (`n8n-nodes-base.code`). Paste Slack Block Kit assembly script. Connect from `AI Incident Triage Agent`.
17. **Auto-Disable Logic:**
    - Add `Auto-Disable Runaway?` (`n8n-nodes-base.if`). Condition: `{{ $json.action }}` equals `auto_disable`. Connect from `Merge Triage & Build Alert`.
    - Add `Deactivate Runaway Workflow` (`n8n-nodes-base.httpRequest`). POST to `{{ $json.base_url }}/api/v1/workflows/{{ $json.workflow_id }}/deactivate`. HTTP Header Auth. `continueRegularOutput` on error. Connect from True output of `Auto-Disable Runaway?`.
    - Add `Finalize Incident` (`n8n-nodes-base.code`). Normalize state from deactivation or alert-only path. Connect from `Deactivate Runaway Workflow` and False output of `Auto-Disable Runaway?`.
18. **Slack Alert & Logging:**
    - Add `Post Incident Alert to Slack` (`n8n-nodes-base.slack`). Channel ID `YOUR_SLACK_CHANNEL_ID`, block kit enabled. Connect from `Finalize Incident`.
    - Add `Log Incident to Sheet` (`n8n-nodes-base.googleSheets`). Document ID `YOUR_GOOGLE_SHEET_ID`, Sheet `Guard_Log`, operation `append`, mapping all incident fields. Connect from `Post Incident Alert to Slack`.
19. **On-Call Escalation:**
    - Add `Needs Escalation?` (`n8n-nodes-base.if`). Condition: `{{ $json.escalate }}` is true. Connect from `Finalize Incident`.
    - Add `Email On-Call Engineer` (`n8n-nodes-base.gmail`). Send to `{{ $('Set Monitor Config').first().json.oncall_email }}` via Gmail credentials. Connect from True output of `Needs Escalation?`.

##### Pipeline 1B: Digest Lane (Weekly)
20. **Read Metrics History:**
    - Add `Read Metrics History Sheet` (`n8n-nodes-base.googleSheets`). Document ID `YOUR_GOOGLE_SHEET_ID`, Sheet `Metrics_History`. Always output data. Connect from `Route by Mode` Output 1.
21. **Failure Sampling & Clustering:**
    - Add `Build Failure Sample List` (`n8n-nodes-base.code`). Connect from `Read Metrics History Sheet`.
    - Add `Fetch Failed Execution Detail` (`n8n-nodes-base.httpRequest`). URL `{{ $json.base_url }}/api/v1/executions/{{ $json.execution_id }}`. Batched 5 / 300ms. `continueRegularOutput` on error. Connect from `Build Failure Sample List`.
    - Add `Analyze Trends & Error Clusters` (`n8n-nodes-base.code`). Connect from `Fetch Failed Execution Detail`.
22. **Trend Chart & AI Summary:**
    - Add `Build 8-Week Trend Chart` (`n8n-nodes-base.code`). Connect from `Analyze Trends & Error Clusters`.
    - Add `AI Ops Analyst Summary` (`@n8n/n8n-nodes-langchain.openAi`). Model `gpt-4o-mini`, temperature `0.2`, JSON output enabled. Connect from `Build 8-Week Trend Chart`.
    - Add `Build Weekly Digest` (`n8n-nodes-base.code`). Connect from `AI Ops Analyst Summary`.
23. **Dispatch Digest & Archive:**
    - Add `Email Weekly Digest` (`n8n-nodes-base.gmail`). Connect from `Build Weekly Digest`.
    - Add `Post Digest TL;DR to Slack` (`n8n-nodes-base.slack`). Channel ID `YOUR_SLACK_CHANNEL_ID`. Connect from `Build Weekly Digest`.
    - Add `Prepare Snapshot Rows` (`n8n-nodes-base.code`). Connect from `Email Weekly Digest`.
    - Add `Append to Metrics History Sheet` (`n8n-nodes-base.googleSheets`). Document ID `YOUR_GOOGLE_SHEET_ID`, Sheet `Metrics_History`, operation `append`. Connect from `Prepare Snapshot Rows`.

##### Pipeline 2: Action Lane (Interactive Webhook)
24. **Webhook Entry Point:**
    - Add `On Slack Action Link Click` (`n8n-nodes-base.webhook`). Path `n8n-guard-action`, response mode `responseNode`.
    - Add `Look Up Incident in Guard Log` (`n8n-nodes-base.googleSheets`). Document ID `YOUR_GOOGLE_SHEET_ID`, Sheet `Guard_Log`, filtering where `incident_id` equals `{{ $json.query.id }}`. Always output data. Connect from webhook.
25. **Validation & Execution:**
    - Add `Set Action Lane Config` (`n8n-nodes-base.code`). Connect from `Look Up Incident in Guard Log`.
    - Add `Validate Action Token` (`n8n-nodes-base.code`). Connect from `Set Action Lane Config`.
    - Add `Token Valid?` (`n8n-nodes-base.if`). Condition: `{{ $json.valid }}` is true. Connect from `Validate Action Token`.
    - Add `Show Invalid Link Page` (`n8n-nodes-base.respondToWebhook`). Response code 403, HTML body. Connect from False output of `Token Valid?`.
    - Add `Is Re-enable Request?` (`n8n-nodes-base.if`). Condition: `{{ $json.act }}` equals `reenable`. Connect from True output of `Token Valid?`.
    - Add `Re-activate Workflow via API` (`n8n-nodes-base.httpRequest`). POST to `{{ $json.base_url }}/api/v1/workflows/{{ $json.workflow_id }}/activate`. HTTP Header Auth, full response, `neverError` enabled. Connect from True output of `Is Re-enable Request?`.
26. **Confirmation & Audit:**
    - Add `Update Incident Status in Sheet` (`n8n-nodes-base.googleSheets`). Document ID `YOUR_GOOGLE_SHEET_ID`, Sheet `Guard_Log`, operation `update`, matching on `incident_id`. Connect from `Re-activate Workflow via API` and False output of `Is Re-enable Request?`.
    - Add `Show Action Done Page` (`n8n-nodes-base.respondToWebhook`). Response code 200, HTML body. Connect from `Update Incident Status in Sheet`.
    - Add `Post Action Confirmation to Slack` (`n8n-nodes-base.slack`). Channel ID `YOUR_SLACK_CHANNEL_ID`. Connect from `Show Action Done Page`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| QuickChart API for 8-week longitudinal failure and error-rate trend graphs. | [QuickChart Documentation](https://quickchart.io/chart) |
| n8n REST API reference for workflow execution management and automated instance governance. | [n8n API Documentation](https://docs.n8n.io/api/) |
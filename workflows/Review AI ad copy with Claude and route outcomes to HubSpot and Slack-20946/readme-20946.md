Review AI ad copy with Claude and route outcomes to HubSpot and Slack

https://n8nworkflows.xyz/workflows/review-ai-ad-copy-with-claude-and-route-outcomes-to-hubspot-and-slack-20946


# Review AI ad copy with Claude and route outcomes to HubSpot and Slack

### 1. Workflow Overview

This workflow is an enterprise-grade automation system designed to receive, validate, deduplicate, and evaluate AI-generated ad copy variants. By leveraging Anthropic's Claude model, it classifies each variant based on channel fit, risk flags, and contextual relevance, routing outcomes dynamically to CRM systems (HubSpot) and communication platforms (Slack). It also features an automated reporting lane to aggregate log metrics, generate AI-driven insights, and distribute performance digests via Gmail, Slack, and Google Sheets.

The architecture is divided into two primary execution pipelines:
- **1.1 Input Reception & Gatekeeping:** Intake via secured webhooks, payload normalization, rigorous field validation, and idempotency checks against an n8n Data Table.
- **1.2 AI Processing & Routing:** Structured prompt construction, communication with the Anthropic Claude API, response validation, fallback execution rules, and dry-run/live-mode branch splitting.
- **1.3 CRM & Chat Execution:** Creation of HubSpot tasks, formatting and dispatching of Slack Block Kit alerts, and persistence of final results to the transaction log.
- **1.4 Analytics & Reporting Lane:** Scheduled and manual triggers, log data extraction, statistical calculation, batch insights generation via Claude, and multi-channel reporting (Gmail, Slack, Google Sheets, HTML export).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Gatekeeping
This block receives incoming ad copy requests, applies global routing and cost variables, enforces basic payload hygiene, and stops malformed or duplicate submissions early.

- **When Ad Variant Received**
  - *Type & Role:* Webhook trigger node using Header Authentication to securely receive incoming HTTP POST requests containing ad copy variant payloads.
  - *Configuration:* Path set to `ad-copy-review`, response handled downstream by a dedicated response node.
  - *Expressions:* None.
  - *Connections:* Input: None (Trigger); Output: `Set Routing Parameters`.
  - *Edge Cases:* Unauthorized access (missing or invalid headers) returns a 401/403 status from n8n's webhook security layer.

- **Set Routing Parameters**
  - *Type & Role:* Set (Edit Fields) node that injects global environment and operational configuration variables into the data stream.
  - *Configuration:* Assigns model identifiers (`claude-haiku-4-5`), token pricing metrics, and default Slack channel names (`campaign-urgent`, `campaign-review`, `campaign-digest`).
  - *Expressions:* Explicit literal assignments for model names and pricing variables.
  - *Connections:* Input: `When Ad Variant Received`; Output: `Validate and Normalize Variant`.
  - *Edge Cases:* Missing variables will propagate downstream; ensure all fields are populated during setup.

- **Validate and Normalize Variant**
  - *Type & Role:* Code (JavaScript) node acting as a deterministic gatekeeper to sanitize incoming payloads and enforce schema requirements.
  - *Configuration:* Validates mandatory fields (`event_id`, `brief_id`, `variant_id`, `headline`, `body`, `cta`, `created_at`), checks upstream quality flags, and constructs an idempotency key (`brief_id:variant_id`).
  - *Expressions:* Standard JavaScript string cleaning and schema mapping.
  - *Connections:* Input: `Set Routing Parameters`; Output: `If Gatekeeper Passes`.
  - *Edge Cases:* Throws validation errors if payload structure is entirely malformed; catches missing fields and populates `gate_reasons`.

- **If Gatekeeper Passes**
  - *Type & Role:* If conditional routing node evaluating whether the variant passed gatekeeper validations.
  - *Configuration:* Evaluates `{{ $json.gate_passed }}` as boolean `true`.
  - *Expressions:* `={{ $json.gate_passed }}`
  - *Connections:* Input: `Validate and Normalize Variant`; Outputs: True path to `Verify Processing Status`, False path to `Handle Rejected Variant`.
  - *Edge Cases:* None.

- **Handle Rejected Variant**
  - *Type & Role:* Code (JavaScript) node that structures a blocked outcome payload for requests failing initial validation.
  - *Configuration:* Assembles a low-priority blocked log object with `status: 'blocked'` and `decision: 'block'` without invoking any AI or external integrations.
  - *Expressions:* JavaScript mapping of gate failure reasons.
  - *Connections:* Input: `If Gatekeeper Passes` (False); Output: `Log Actions to Table`.
  - *Edge Cases:* None.

- **Verify Processing Status**
  - *Type & Role:* n8n Data Table lookup node querying the transaction history for idempotency enforcement.
  - *Configuration:* Retrieves up to 1 record from table `ad_copy_review_log` filtered by `idempotency_key`. Configured with `Continue On Fail` to ensure resilience.
  - *Expressions:* `={{ $json.idempotency_key }}`
  - *Connections:* Input: `If Gatekeeper Passes` (True); Output: `If Already Processed`.
  - *Edge Cases:* Table unavailability or connection drops; handled via `alwaysOutputData` and error continuation.

- **If Already Processed**
  - *Type & Role:* If conditional node checking if an identical brief/variant combination was previously logged.
  - *Configuration:* Checks if `idempotency_key` is present in the lookup result.
  - *Expressions:* `={{ $json.idempotency_key }}`
  - *Connections:* Input: `Verify Processing Status`; Outputs: True path to `Handle Duplicate Variant`, False path to `Build Request for Claude`.
  - *Edge Cases:* None.

- **Handle Duplicate Variant**
  - *Type & Role:* Code (JavaScript) node formatting a duplicate transaction response.
  - *Configuration:* Generates a short-circuit payload indicating the variant was already processed.
  - *Expressions:* JavaScript object mapping.
  - *Connections:* Input: `If Already Processed` (True); Output: `Send Response to Sender`.
  - *Edge Cases:* None.

---

#### 2.2 AI Processing & Routing
This block constructs strict JSON schema-enforced prompts, communicates with the Anthropic API, validates outputs, and routes processing through fallback rules if the AI fails.

- **Build Request for Claude**
  - *Type & Role:* Code (JavaScript) node preparing the system prompt, user payload, and structural JSON schema for the Anthropic API call.
  - *Configuration:* Embeds marketing-ops routing guidelines, channel constraints, risk flag severity criteria, and an output schema restricting model responses to exact enums.
  - *Expressions:* Dynamic template literals embedding variant data, campaign context, and global configuration values.
  - *Connections:* Input: `If Already Processed` (False); Output: `Post to Claude API`.
  - *Edge Cases:* Payload size limitations if ad body copy is exceptionally long.

- **Post to Claude API**
  - *Type & Role:* HTTP Request node communicating with the Anthropic Messages API.
  - *Configuration:* POST request to `https://api.anthropic.com/v1/messages` using `anthropicApi` credentials, a 60-second timeout, up to 3 retries with exponential backoff, and header parameter `anthropic-version: 2023-06-01`.
  - *Expressions:* `={{ JSON.stringify($json.request_body) }}`
  - *Connections:* Input: `Build Request for Claude`; Output: `Check AI Response Validity`.
  - *Edge Cases:* Rate limits (HTTP 429), server errors (HTTP 5xx), or request timeouts. Mitigated by automatic retries and error catch continuation.

- **Check AI Response Validity**
  - *Type & Role:* Code (JavaScript) node validating the structural integrity and semantic correctness of Claude's response.
  - *Configuration:* Inspects stop reasons (`refusal`, `max_tokens`), parses JSON content against allowed enums (`decision`, `priority`, `owner_team`, `slack_channel`), and calculates API token costs.
  - *Expressions:* JavaScript validation parsing.
  - *Connections:* Input: `Post to Claude API`; Output: `If AI Decision Valid`.
  - *Edge Cases:* Malformed JSON outputs, truncated responses, or missing required keys. Sets `ai_ok` to `false` upon detection.

- **If AI Decision Valid**
  - *Type & Role:* If conditional node branching based on AI output validity.
  - *Configuration:* Evaluates `{{ $json.ai_ok }}` as boolean `true`.
  - *Expressions:* `={{ $json.ai_ok }}`
  - *Connections:* Input: `Check AI Response Validity`; Outputs: True path to `Prepare Action Plans`, False path to `Fallback Rule Router`.
  - *Edge Cases:* None.

- **Fallback Rule Router**
  - *Type & Role:* Code (JavaScript) node providing deterministic rule-based fallback routing when the AI output is unusable.
  - *Configuration:* Scans ad copy text using regular expressions for placeholder text (`[website]`, `bit.ly`, `TBD`), unverifiable claims, and false urgency, assigning safe default routing and task parameters.
  - *Expressions:* JavaScript regex matching.
  - *Connections:* Input: `If AI Decision Valid` (False); Output: `Prepare Action Plans`.
  - *Edge Cases:* Regex edge cases on deeply nested or heavily obfuscated text strings.

---

#### 2.3 CRM & Chat Execution
This block transforms routing decisions into execution payloads, evaluates dry-run versus live operational modes, creates external CRM tasks, posts team notifications, and records metrics.

- **Prepare Action Plans**
  - *Type & Role:* Code (JavaScript) node generating formatted CRM payloads and Slack Block Kit message structures.
  - *Configuration:* Maps decisions and priorities to HubSpot task attributes and builds multi-block Slack notifications complete with risk flags and suggested fixes.
  - *Expressions:* JavaScript template mapping utilizing upstream routing parameters.
  - *Connections:* Input: `If AI Decision Valid` (True) or `Fallback Rule Router`; Output: `If Live Mode Active`.
  - *Edge Cases:* Missing optional object attributes handled via null-coalescing.

- **If Live Mode Active**
  - *Type & Role:* If conditional node determining whether external system mutations are permitted.
  - *Configuration:* Evaluates `{{ $json.live }}` as boolean `true`.
  - *Expressions:* `={{ $json.live }}`
  - *Connections:* Input: `Prepare Action Plans`; Outputs: True path to `Create Task in HubSpot`, False path to `Compile Action Results`.
  - *Edge Cases:* None.

- **Create Task in HubSpot**
  - *Type & Role:* HTTP Request node creating tracking tasks within HubSpot CRM.
  - *Configuration:* POST request to `https://api.hubapi.com/crm/v3/objects/tasks` using `hubspotAppToken` credentials, a 30-second timeout, and up to 3 retries.
  - *Expressions:* `={{ JSON.stringify($json.hubspot_payload) }}`
  - *Connections:* Input: `If Live Mode Active` (True); Output: `Post Alert to Slack`.
  - *Edge Cases:* Invalid CRM property mappings or authentication token expiration.

- **Post Alert to Slack**
  - *Type & Role:* HTTP Request node dispatching rich Block Kit alerts to designated Slack channels.
  - *Configuration:* POST request to `https://slack.com/api/chat.postMessage` using `slackApi` credentials, a 30-second timeout, and up to 3 retries.
  - *Expressions:* Dynamically injects the HubSpot task ID into the notification block string via string replacement.
  - *Connections:* Input: `Create Task in HubSpot`; Output: `Compile Action Results`.
  - *Edge Cases:* Channel ID not found or bot token lacking `chat:write` scope permissions.

- **Compile Action Results**
  - *Type & Role:* Code (JavaScript) node consolidating execution outcomes across services into a single database row object.
  - *Configuration:* Aggregates success states, error logs, execution latencies, and financial costs into a unified schema matching the Data Table structure.
  - *Expressions:* Try-catch block retrieval of upstream node outputs.
  - *Connections:* Input: `If Live Mode Active` (False) or `Post Alert to Slack`; Output: `Log Actions to Table`.
  - *Edge Cases:* Partial failures where CRM succeeds but chat delivery fails; records `partial_failure` status accordingly.

- **Log Actions to Table**
  - *Type & Role:* n8n Data Table node persisting execution audit records.
  - *Configuration:* Appends rows to table `ad_copy_review_log` using automatic input data mapping.
  - *Expressions:* Automatic field mapping.
  - *Connections:* Input: `Compile Action Results` or `Handle Rejected Variant`; Output: `Send Response to Sender`.
  - *Edge Cases:* Write conflicts or storage limits on the n8n instance.

- **Send Response to Sender**
  - *Type & Role:* Respond to Webhook node returning the final HTTP response to the webhook caller.
  - *Configuration:* Dynamically sets HTTP response codes (`422` for blocked requests, `200` for processed or duplicate variants).
  - *Expressions:* `={{ $json.status === 'blocked' ? 422 : 200 }}`
  - *Connections:* Input: `Log Actions to Table` or `Handle Duplicate Variant`; Output: None (Terminal node).
  - *Edge Cases:* Connection timeout back to the webhook caller.

---

#### 2.4 Analytics & Reporting Lane
This block handles scheduled and manual triggers, reads historical log data, calculates operational statistics, queries Claude for executive insights, and distributes formatted reports.

- **Manual Report Trigger**
  - *Type & Role:* Manual trigger node allowing on-demand execution of the reporting workflow.
  - *Configuration:* Standard manual run button.
  - *Expressions:* None.
  - *Connections:* Input: None; Output: `Set Report Parameters`.
  - *Edge Cases:* None.

- **Daily Digest at 6 PM**
  - *Type & Role:* Schedule Trigger node executing the reporting workflow automatically every day at 18:00 UTC.
  - *Configuration:* Cron interval set to daily at hour 18, minute 0.
  - *Expressions:* None.
  - *Connections:* Input: None; Output: `Set Report Parameters`.
  - *Edge Cases:* Server timezone discrepancies; ensure n8n instance time matches operational expectations.

- **Set Report Parameters**
  - *Type & Role:* Set (Edit Fields) node configuring reporting boundaries and distribution endpoints.
  - *Configuration:* Defines recipient email (`you@example.com`), lookback window (`24` hours), Slack digest channel (`campaign-digest`), Claude model, and estimated manual minutes saved per variant (`6`).
  - *Expressions:* Literal configuration assignments.
  - *Connections:* Input: `Manual Report Trigger` or `Daily Digest at 6 PM`; Output: `Verify Action Log Table`.
  - *Edge Cases:* Misconfigured recipient email addresses.

- **Verify Action Log Table**
  - *Type & Role:* n8n Data Table setup node ensuring the logging schema exists before read operations occur.
  - *Configuration:* Creates table `ad_copy_review_log` with defined columns if it does not already exist (`createIfNotExists: true`).
  - *Expressions:* None.
  - *Connections:* Input: `Set Report Parameters`; Output: `Read Action Log Data`.
  - *Edge Cases:* Insufficient permissions to create tables on self-hosted instances.

- **Read Action Log Data**
  - *Type & Role:* n8n Data Table read node fetching historical execution logs.
  - *Configuration:* Operation set to `get` with `returnAll: true` from table `ad_copy_review_log`.
  - *Expressions:* None.
  - *Connections:* Input: `Verify Action Log Table`; Output: `Calculate Run Statistics`.
  - *Edge Cases:* Large data tables causing high memory consumption; consider periodic log archiving if volume is extremely high.

- **Calculate Run Statistics**
  - *Type & Role:* Code (JavaScript) node computing operational KPIs, latency percentiles, and risk aggregations.
  - *Configuration:* Filters logs by lookback window or run ID, calculates p50/p95 latency, sums API costs, and extracts sample decision rows.
  - *Expressions:* JavaScript data aggregation and sorting algorithms.
  - *Connections:* Input: `Read Action Log Data`; Output: `Request AI Insights`.
  - *Edge Cases:* Empty datasets returning zero values; guarded against division by zero.

- **Request AI Insights**
  - *Type & Role:* Code (JavaScript) node formatting statistical summaries into an insights prompt for Claude.
  - *Configuration:* Defines a strict JSON schema for executive headlines, recurring patterns, actionable recommendations, and weekly watchlists.
  - *Expressions:* JavaScript JSON stringification of run metrics.
  - *Connections:* Input: `Calculate Run Statistics`; Output: `Post to Insights API`.
  - *Edge Cases:* Token length limitations on massive reporting datasets.

- **Post to Insights API**
  - *Type & Role:* HTTP Request node invoking the Anthropic API for executive analytical insights.
  - *Configuration:* POST request to `https://api.anthropic.com/v1/messages` using `anthropicApi` credentials, 60-second timeout, and 3 retries.
  - *Expressions:* `={{ JSON.stringify($json.request_body) }}`
  - *Connections:* Input: `Request AI Insights`; Output: `Generate Report Document`.
  - *Edge Cases:* API rate limits or refusal responses; handled via safe fallback object creation downstream.

- **Generate Report Document**
  - *Type & Role:* Code (JavaScript) node assembling styled HTML email templates, Slack summary payloads, and raw document structures.
  - *Configuration:* Converts statistical metrics and AI insights into an inline-styled HTML document, a Slack digest message block structure, and export objects.
  - *Expressions:* JavaScript template literal construction with HTML escaping.
  - *Connections:* Input: `Post to Insights API`; Outputs (Fan-out): `Send Report via Gmail`, `Send Slack Digest`, `Export Report as HTML`, `Prepare Sheet Rows`.
  - *Edge Cases:* Malformed AI insight JSON falling back to default messaging.

- **Send Report via Gmail**
  - *Type & Role:* Gmail integration node dispatching the HTML executive report.
  - *Configuration:* Sends email to the configured report address using `gmail` credentials, with automatic attribution disabled.
  - *Expressions:* Recipient: `={{ $('Set Report Parameters').first().json.report_email }}`, Message body: `={{ $json.html }}`, Subject: `={{ $json.subject }}`.
  - *Connections:* Input: `Generate Report Document`; Output: None (Terminal node in branch).
  - *Edge Cases:* Gmail API authentication expiry or sending limits.

- **Send Slack Digest**
  - *Type & Role:* HTTP Request node posting the executive reporting digest to Slack.
  - *Configuration:* POST request to `https://slack.com/api/chat.postMessage` using `slackApi` credentials.
  - *Expressions:* `={{ JSON.stringify($json.slack_payload) }}`
  - *Connections:* Input: `Generate Report Document`; Output: None (Terminal node in branch).
  - *Edge Cases:* Invalid channel configuration or token permissions.

- **Export Report as HTML**
  - *Type & Role:* Convert to File node transforming report markup into a downloadable text artifact.
  - *Configuration:* Converts source property `html` to text with filename `ad_copy_review_report.html`.
  - *Expressions:* `={{ $json.html }}`
  - *Connections:* Input: `Generate Report Document`; Output: `Prepare Sheet Rows`.
  - *Edge Cases:* File system write restrictions in cloud-managed environments.

- **Prepare Sheet Rows**
  - *Type & Role:* Code (JavaScript) node formatting individual log rows for spreadsheet export.
  - *Configuration:* Maps run statistics rows into tidy columnar schema objects suitable for Google Sheets insertion.
  - *Expressions:* JavaScript array mapping.
  - *Connections:* Input: `Export Report as HTML`; Output: `Append to Google Sheet`.
  - *Edge Cases:* Special characters breaking sheet parsing.

- **Append to Google Sheet**
  - *Type & Role:* Google Sheets integration node exporting report records to a shared spreadsheet.
  - *Configuration:* Appends rows using auto-mapped input data via `googleSheets` credentials.
  - *Expressions:* Dynamic spreadsheet and sheet selections via resource locators.
  - *Connections:* Input: `Prepare Sheet Rows`; Output: None (Terminal node).
  - *Edge Cases:* API quotas, permission errors, or mismatched sheet column mappings.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **When Ad Variant Received** | `n8n-nodes-base.webhook` | Webhook intake for ad copy variants. | None | Set Routing Parameters | [Review AI ad copy with Claude and route it to HubSpot and Slack](#) |
| **Set Routing Parameters** | `n8n-nodes-base.set` | Sets global model, pricing, and channel variables. | When Ad Variant Received | Validate and Normalize Variant | [Review AI ad copy with Claude and route it to HubSpot and Slack](#) |
| **Validate and Normalize Variant** | `n8n-nodes-base.code` | Gatekeeper validation and idempotency key creation. | Set Routing Parameters | If Gatekeeper Passes | [Receive and validate variant](#) |
| **If Gatekeeper Passes** | `n8n-nodes-base.if` | Evaluates gatekeeper success status. | Validate and Normalize Variant | Verify Processing Status, Handle Rejected Variant | [Receive and validate variant](#) |
| **Handle Rejected Variant** | `n8n-nodes-base.code` | Builds blocked outcome for invalid variants. | If Gatekeeper Passes | Log Actions to Table | [Receive and validate variant](#) |
| **Verify Processing Status** | `n8n-nodes-base.dataTable` | Checks Data Table for existing idempotency key. | If Gatekeeper Passes | If Already Processed | [Deduplicate ad variants](#) |
| **If Already Processed** | `n8n-nodes-base.if` | Routes duplicate requests away from reprocessing. | Verify Processing Status | Handle Duplicate Variant, Build Request for Claude | [Deduplicate ad variants](#) |
| **Handle Duplicate Variant** | `n8n-nodes-base.code` | Formats duplicate short-circuit response. | If Already Processed | Send Response to Sender | [Deduplicate ad variants](#) |
| **Build Request for Claude** | `n8n-nodes-base.code` | Constructs strict JSON schema prompt for Claude. | If Already Processed | Post to Claude API | [Ask Claude to route](#) |
| **Post to Claude API** | `n8n-nodes-base.httpRequest` | Sends routing prompt to Anthropic Claude API. | Build Request for Claude | Check AI Response Validity | [Ask Claude to route](#) |
| **Check AI Response Validity** | `n8n-nodes-base.code` | Validates AI response structure against schema enums. | Post to Claude API | If AI Decision Valid | [Validate routing decision](#) |
| **If AI Decision Valid** | `n8n-nodes-base.if` | Checks if AI response was valid. | Check AI Response Validity | Prepare Action Plans, Fallback Rule Router | [Validate routing decision](#) |
| **Fallback Rule Router** | `n8n-nodes-base.code` | Deterministic fallback routing when AI fails. | If AI Decision Valid | Prepare Action Plans | [Validate routing decision](#) |
| **Prepare Action Plans** | `n8n-nodes-base.code` | Converts decisions into HubSpot and Slack payloads. | If AI Decision Valid, Fallback Rule Router | If Live Mode Active | [Plan execution actions](#) |
| **If Live Mode Active** | `n8n-nodes-base.if` | Branches between dry-run and live execution modes. | Prepare Action Plans | Create Task in HubSpot, Compile Action Results | [Plan execution actions](#) |
| **Create Task in HubSpot** | `n8n-nodes-base.httpRequest` | Creates tracking task in HubSpot CRM. | If Live Mode Active | Post Alert to Slack | [Create task and alert](#) |
| **Post Alert to Slack** | `n8n-nodes-base.httpRequest` | Posts rich alert message to Slack channels. | Create Task in HubSpot | Compile Action Results | [Create task and alert](#) |
| **Compile Action Results** | `n8n-nodes-base.code` | Consolidates action results into a unified log row. | If Live Mode Active, Post Alert to Slack | Log Actions to Table | [Log and respond](#) |
| **Log Actions to Table** | `n8n-nodes-base.dataTable` | Persists execution audit records to Data Table. | Compile Action Results, Handle Rejected Variant | Send Response to Sender | [Log and respond](#) |
| **Send Response to Sender** | `n8n-nodes-base.respondToWebhook` | Returns final HTTP response to webhook caller. | Log Actions to Table, Handle Duplicate Variant | None | [Log and respond](#) |
| **Manual Report Trigger** | `n8n-nodes-base.manualTrigger` | Manual trigger for on-demand report generation. | None | Set Report Parameters | [Start report run](#) |
| **Daily Digest at 6 PM** | `n8n-nodes-base.scheduleTrigger` | Scheduled trigger for daily automated reporting. | None | Set Report Parameters | [Start report run](#) |
| **Set Report Parameters** | `n8n-nodes-base.set` | Configures reporting lookback and recipient parameters. | Manual Report Trigger, Daily Digest at 6 PM | Verify Action Log Table | [Start report run](#) |
| **Verify Action Log Table** | `n8n-nodes-base.dataTable` | Ensures audit logging table exists. | Set Report Parameters | Read Action Log Data | [Read and aggregate log](#) |
| **Read Action Log Data** | `n8n-nodes-base.dataTable` | Reads historical transaction logs from Data Table. | Verify Action Log Table | Calculate Run Statistics | [Read and aggregate log](#) |
| **Calculate Run Statistics** | `n8n-nodes-base.code` | Computes KPIs, percentiles, and operational metrics. | Read Action Log Data | Request AI Insights | [Read and aggregate log](#) |
| **Request AI Insights** | `n8n-nodes-base.code` | Prepares batch statistics prompt for Claude insights. | Calculate Run Statistics | Post to Insights API | [Generate AI insights](#) |
| **Post to Insights API** | `n8n-nodes-base.httpRequest` | Requests analytical insights from Anthropic Claude. | Request AI Insights | Generate Report Document | [Generate AI insights](#) |
| **Generate Report Document** | `n8n-nodes-base.code` | Generates HTML reports, Slack digests, and exports. | Post to Insights API | Send Report via Gmail, Send Slack Digest, Export Report as HTML, Prepare Sheet Rows | [Build readable report](#) |
| **Send Report via Gmail** | `n8n-nodes-base.gmail` | Delivers HTML executive report via email. | Generate Report Document | None | [Send report digest](#) |
| **Send Slack Digest** | `n8n-nodes-base.httpRequest` | Posts executive digest summary to Slack. | Generate Report Document | None | [Send report digest](#) |
| **Export Report as HTML** | `n8n-nodes-base.convertToFile` | Converts report markup to downloadable text file. | Generate Report Document | Prepare Sheet Rows | [Export report records](#) |
| **Prepare Sheet Rows** | `n8n-nodes-base.code` | Prepares tidy tabular rows for Google Sheets export. | Export Report as HTML | Append to Google Sheet | [Export report records](#) |
| **Append to Google Sheet** | `n8n-nodes-base.googleSheets` | Appends decision export rows to Google Sheets. | Prepare Sheet Rows | None | [Export report records](#) |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, execute the following steps in sequence:

1. **Create Intake Trigger:**
   - Add a **Webhook** node (`When Ad Variant Received`). Set HTTP Method to `POST`, path to `ad-copy-review`, and Authentication to `Header Auth`.
2. **Configure Global Variables:**
   - Add a **Set (Edit Fields)** node (`Set Routing Parameters`). Define string/number assignments for `claude_model` (`claude-haiku-4-5`), Slack channels (`campaign-urgent`, `campaign-review`, `campaign-digest`), and token pricing. Connect input from the Webhook.
3. **Add Gatekeeper Validation:**
   - Add a **Code** node (`Validate and Normalize Variant`). Paste the sanitization and required field verification logic (`event_id`, `brief_id`, `variant_id`, etc.) and compute the `idempotency_key`. Connect input from `Set Routing Parameters`.
4. **Implement Gate Branching:**
   - Add an **If** node (`If Gatekeeper Passes`) evaluating `{{ $json.gate_passed }}`.
   - Connect the True branch to a **Data Table** node (`Verify Processing Status`) set to retrieve records from `ad_copy_review_log` filtered by `idempotency_key` (with `Continue On Fail` enabled).
   - Connect the False branch to a **Code** node (`Handle Rejected Variant`) returning a blocked payload, connected onward to `Log Actions to Table`.
5. **Implement Deduplication Check:**
   - Add an **If** node (`If Already Processed`) checking if `idempotency_key` is non-empty.
   - Connect the True branch to a **Code** node (`Handle Duplicate Variant`), then to `Send Response to Sender`.
   - Connect the False branch to the AI processing block.
6. **Build AI Request & Communication:**
   - Add a **Code** node (`Build Request for Claude`) to construct the system prompt, user payload, and JSON schema.
   - Add an **HTTP Request** node (`Post to Claude API`) pointing to `https://api.anthropic.com/v1/messages`. Configure `anthropicApi` credentials, POST method, header `anthropic-version: 2023-06-01`, and body `={{ JSON.stringify($json.request_body) }}` with 3 retries.
7. **Validate AI Response & Fallback:**
   - Add a **Code** node (`Check AI Response Validity`) to parse and validate Claude's JSON response against allowed enums.
   - Add an **If** node (`If AI Decision Valid`) evaluating `{{ $json.ai_ok }}`.
   - Connect the False branch to a **Code** node (`Fallback Rule Router`) performing regex-based rule routing.
8. **Plan Execution & Live Mode Routing:**
   - Connect both the True branch of `If AI Decision Valid` and the output of `Fallback Rule Router` to a **Code** node (`Prepare Action Plans`) generating HubSpot and Slack payloads.
   - Add an **If** node (`If Live Mode Active`) evaluating `{{ $json.live }}`.
9. **Configure Live Integrations:**
   - Connect the True branch of `If Live Mode Active` to an **HTTP Request** node (`Create Task in HubSpot`) pointing to `https://api.hubapi.com/crm/v3/objects/tasks` using `hubspotAppToken` credentials.
   - Connect HubSpot to an **HTTP Request** node (`Post Alert to Slack`) pointing to `https://slack.com/api/chat.postMessage` using `slackApi` credentials.
   - Connect both the Slack node and the False branch of `If Live Mode Active` to a **Code** node (`Compile Action Results`).
10. **Finalize Processing Lane:**
    - Connect `Compile Action Results` and `Handle Rejected Variant` to a **Data Table** node (`Log Actions to Table`) appending to `ad_copy_review_log`.
    - Connect to a **Respond to Webhook** node (`Send Response to Sender`) returning status `422` for blocked requests and `200` otherwise.
11. **Construct Reporting Lane:**
    - Create a separate branch using a **Manual Trigger** node and a **Schedule Trigger** node (`Daily Digest at 6 PM`) connected to a **Set** node (`Set Report Parameters`) defining report email, lookback hours, and digest channels.
    - Connect to a **Data Table** node (`Verify Action Log Table`) creating `ad_copy_review_log` if missing, followed by a **Data Table** read node (`Read Action Log Data`).
    - Connect to a **Code** node (`Calculate Run Statistics`), then a **Code** node (`Request AI Insights`), and an **HTTP Request** node (`Post to Insights API`) calling Anthropic Claude.
    - Connect to a **Code** node (`Generate Report Document`) formatting HTML output and Slack blocks, and fan out to:
      - **Gmail** node (`Send Report via Gmail`) using `gmail` credentials.
      - **HTTP Request** node (`Send Slack Digest`) posting to Slack.
      - **Convert to File** node (`Export Report as HTML`).
      - **Code** node (`Prepare Sheet Rows`) followed by a **Google Sheets** node (`Append to Google Sheet`) using `googleSheets` credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Workflow designed for automated ad copy governance, compliance checking, CRM synchronization, and multi-channel reporting. | Core architectural purpose |
| Requires active API credentials for Anthropic Claude, HubSpot Private App, Slack Bot Token, Gmail OAuth2/SMTP, and Google Sheets OAuth2. | Credential prerequisites |
| Leverages n8n Data Tables for transactional idempotency logging and historical analytics aggregation. | Storage architecture |
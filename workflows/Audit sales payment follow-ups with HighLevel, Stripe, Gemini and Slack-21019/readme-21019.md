Audit sales payment follow-ups with HighLevel, Stripe, Gemini and Slack

https://n8nworkflows.xyz/workflows/audit-sales-payment-follow-ups-with-highlevel--stripe--gemini-and-slack-21019


# Audit sales payment follow-ups with HighLevel, Stripe, Gemini and Slack

### 1. Workflow Overview

This workflow automates the tracking, auditing, and alerting of unpaid closed-won sales deals. Running on an hourly schedule, it extracts opportunities from GoHighLevel (GHL), checks financial statuses via Stripe, aggregates communication threads and internal tasks, leverages Google Gemini to assess follow-up health, and dispatches prioritized notifications to Slack while logging activities in Google Sheets.

The architecture is divided into the following logical blocks:
- **1.1 Trigger & Opportunity Retrieval:** Fetches closed-won deals from GoHighLevel on an hourly schedule and verifies monetary values and deduplication logs.
- **1.2 CRM & Financial Data Aggregation:** Gathers historical contact communications and tasks from GoHighLevel, and fetches matching customer records and payment charges from Stripe.
- **1.3 AI Context Compilation & Analysis:** Normalizes the collected data into a structured JSON context, passing it to a Gemini-powered LangChain agent to produce a structured severity verdict and a suggested follow-up message.
- **1.4 Alert Processing & Dispatch:** Evaluates the AI verdict, constructs formatted Slack notifications for actionable gaps, and persists the audit logs in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Opportunity Retrieval
- **Overview:** Initiates the audit cycle on an hourly interval, pulls closed-won opportunities, filters out zero-value deals, and cross-references historical tracking logs to ensure fresh processing.
- **Nodes Involved:** 
  - `Every Hour Trigger`
  - `Fetch HighLevel Opportunities`
  - `Check Opportunity Value`
  - `Read Tracked Leads in Sheets`
  - `Filter New Opportunities`

- **Node Details:**
  - **Every Hour Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger) — Starts the execution pipeline every hour.
    - *Configuration:* Interval configured to run hourly (`hours`).
    - *Input/Output:* Output connects to `Fetch HighLevel Opportunities`.
    - *Edge Cases/Failures:* Execution delays if the n8n instance is offline during the scheduled window.
  - **Fetch HighLevel Opportunities**
    - *Type and Technical Role:* `n8n-nodes-base.highLevel` (GoHighLevel) — Retrieves operational sales opportunities from a designated pipeline stage.
    - *Configuration:* Resource: `opportunity`, Operation: `getAll`, Pipeline ID: `smEMRhNOilT5Pm0QZCWO`, Stage ID: `2630bdf2-933e-4b14-a1c7-577412790640`.
    - *Input/Output:* Input from `Every Hour Trigger`; output connects to `Check Opportunity Value`.
    - *Credentials:* Requires a `HighLevel OAuth2 API` credential.
    - *Edge Cases/Failures:* API rate limits, invalid pipeline/stage IDs, or expired OAuth tokens.
  - **Check Opportunity Value**
    - *Type and Technical Role:* `n8n-nodes-base.if` (If) — Evaluates whether fetched opportunities possess a non-zero monetary value.
    - *Configuration:* Condition verifies `{{ $json.monetaryValue }}` is strictly greater than `0`.
    - *Input/Output:* Input from `Fetch HighLevel Opportunities`; output connects to `Read Tracked Leads in Sheets`.
  - **Read Tracked Leads in Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets) — Reads existing notification history to support deduplication.
    - *Configuration:* Operation: Read/Get, Document ID: `1NjNEYyzY6a4BoS1KmLa3WQF3zs1h2d8u0cT4dCcqoiA`, Sheet Name: `Notification Log`. Set with `Execute Once` enabled.
    - *Input/Output:* Input from `Check Opportunity Value`; output connects to `Filter New Opportunities`.
    - *Credentials:* Requires a `Google Service Account` credential.
    - *Edge Cases/Failures:* Sheet missing columns, permissions errors, or missing sheet name.
  - **Filter New Opportunities**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code) — Custom JavaScript that compares incoming opportunity items against previously logged notification records.
    - *Configuration:* Iterates over sheet rows to build an exclusion set and filters out already processed opportunities.
    - *Input/Output:* Input from `Read Tracked Leads in Sheets`; output connects to `Search Contact Conversations`.
    - *Edge Cases/Failures:* Script runtime issues if spreadsheet rows return malformed structures.

---

#### 2.2 CRM & Financial Data Aggregation
- **Overview:** Queries contact-specific interaction threads, active task assignments from GoHighLevel, and pulls customer payment records from Stripe.
- **Nodes Involved:**
  - `Search Contact Conversations`
  - `Fetch Conversation Messages`
  - `Fetch HighLevel Contact Tasks`
  - `Fetch Stripe Customer`
  - `Fetch Stripe Charges`

- **Node Details:**
  - **Search Contact Conversations**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request) — Queries the LeadConnector API for contact conversation threads.
    - *Configuration:* GET request to `https://services.leadconnectorhq.com/conversations/search` with query parameters `locationId` and `contactId`. Custom Header `Version: 2021-04-15`.
    - *Input/Output:* Input from `Filter New Opportunities`; output connects to `Fetch Conversation Messages`.
    - *Credentials:* Requires a `HighLevel OAuth2 API` credential.
    - *Edge Cases/Failures:* 404/400 responses if contact IDs are missing or invalid.
  - **Fetch Conversation Messages**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request) — Retrieves granular message records inside a conversation thread.
    - *Configuration:* GET request to `https://services.leadconnectorhq.com/conversations/{{ $('Search Contact Conversations').item.json.conversations[0].id }}/messages` with limit `100`.
    - *Input/Output:* Input from `Search Contact Conversations`; output connects to `Fetch HighLevel Contact Tasks`.
    - *Credentials:* Requires a `HighLevel OAuth2 API` credential.
    - *Edge Cases/Failures:* Index out-of-bounds error if the conversation array is empty.
  - **Fetch HighLevel Contact Tasks**
    - *Type and Technical Role:* `n8n-nodes-base.highLevel` (GoHighLevel) — Pulls open and closed task records associated with the GHL contact.
    - *Configuration:* Resource: `task`, Operation: `getAll`, Contact ID mapped from upstream.
    - *Input/Output:* Input from `Fetch Conversation Messages`; output connects to `Fetch Stripe Customer`.
    - *Credentials:* Requires a `HighLevel OAuth2 API` credential.
  - **Fetch Stripe Customer**
    - *Type and Technical Role:* `n8n-nodes-base.stripe` (Stripe) — Looks up Stripe customer profiles using an email filter.
    - *Configuration:* Resource: `customer`, Operation: `getAll`, Filter by email (`user@example.com` - *Note: update for production scoping*), limit 1.
    - *Input/Output:* Input from `Fetch HighLevel Contact Tasks`; output connects to `Fetch Stripe Charges`.
    - *Credentials:* Requires a `Stripe API` credential.
    - *Edge Cases/Failures:* Customer not found in Stripe matching the provided static/dynamic email.
  - **Fetch Stripe Charges**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request) — Retrieves charge histories for the resolved Stripe customer ID.
    - *Configuration:* GET request to `https://api.stripe.com/v1/charges` filtering by `customer` ID.
    - *Input/Output:* Input from `Fetch Stripe Customer`; output connects to `Compile Follow-Up Context`.
    - *Credentials:* Requires a `Stripe API` credential.

---

#### 2.3 AI Context Compilation & Analysis
- **Overview:** Consolidates CRM metrics, financial totals, communication metrics, and calculated deterministic flags into a unified context payload sent to a Gemini language model agent.
- **Nodes Involved:**
  - `Compile Follow-Up Context`
  - `Follow-Up Analysis Agent`
  - `Gemini Chat Model`
  - `Parse Agent Output`

- **Node Details:**
  - **Compile Follow-Up Context**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code) — Aggregates and normalizes CRM, task, and payment data, calculating time intervals, payment gaps, and deterministic flags (e.g., `NO_FOLLOW_UP_SINCE_CLOSE`, `PARTIAL_PAYMENT`).
    - *Configuration:* Custom JavaScript block containing SLA definitions, timezone parameters, and payload formatting.
    - *Input/Output:* Input from `Fetch Stripe Charges`; output connects to `Follow-Up Analysis Agent`.
  - **Follow-Up Analysis Agent**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent) — Evaluates deal health based on provided context and custom system instructions.
    - *Configuration:* Prompt injected with JSON stringified context; configured to utilize an output parser.
    - *Input/Output:* Input from `Compile Follow-Up Context`; links to language model and output parser nodes; output connects to `Set Alert Properties`.
  - **Gemini Chat Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (Google Gemini Chat Model) — Provides the underlying generative intelligence.
    - *Configuration:* Model set to `models/gemini-3.1-flash-lite`, temperature `0.2`.
    - *Credentials:* Requires a `Google Palm API` credential.
  - **Parse Agent Output**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Structured Output Parser) — Enforces a strict JSON schema validation on the AI response covering alert needs, severity, scenario, evidence, and actions.
    - *Configuration:* JSON Schema example defining keys like `alert_needed`, `severity`, `scenario`, `headline`, `evidence`, `recommended_action`, `suggested_followup_message`, and `escalate_to_manager`.
    - *Input/Output:* Connects to the AI Agent via `ai_outputParser`.

---

#### 2.4 Alert Processing & Dispatch
- **Overview:** Inspects the AI verdict, filters actions requiring human intervention, formats markdown payloads for Slack, and records dispatched alerts in Google Sheets.
- **Nodes Involved:**
  - `Set Alert Properties`
  - `If Alert Needed`
  - `Build Slack Message`
  - `Post Alert to Slack`
  - `Record Alert in Sheets`

- **Node Details:**
  - **Set Alert Properties**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields) — Maps parsed AI outputs and context values to structured workflow properties.
    - *Configuration:* Assigns properties like `alert_needed`, `severity`, `scenario`, `headline`, `evidence_text`, `recommended_action`, and metadata links.
    - *Input/Output:* Input from `Follow-Up Analysis Agent`; output connects to `If Alert Needed`.
  - **If Alert Needed**
    - *Type and Technical Role:* `n8n-nodes-base.if` (If) — Gates execution based on whether an alert is flagged and severity is active.
    - *Configuration:* Verifies `alert_needed` is `true` and `severity` does not equal `none`.
    - *Input/Output:* Input from `Set Alert Properties`; output connects to `Build Slack Message`.
  - **Build Slack Message**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields) — Constructs a rich, formatted Slack message template incorporating emojis, severity labels, deal summaries, and actions.
    - *Configuration:* Markdown text interpolation using evaluated properties.
    - *Input/Output:* Input from `If Alert Needed`; output connects to `Post Alert to Slack`.
  - **Post Alert to Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Slack) — Publishes the formatted notification to a specific team channel.
    - *Configuration:* Resource: Message, Select: channel, Channel ID: `C0B8VH1M5PX` (general).
    - *Input/Output:* Input from `Build Slack Message`; output connects to `Record Alert in Sheets`.
    - *Credentials:* Requires a `Slack API` credential.
    - *Edge Cases/Failures:* Channel not found or bot permissions missing.
  - **Record Alert in Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets) — Appends or updates the tracking log in Google Sheets.
    - *Configuration:* Operation: `appendOrUpdate`, Matching Columns: `opportunity_id`, Document ID: `1NjNEYyzY6a4BoS1KmLa3WQF3zs1h2d8u0cT4dCcqoiA`, Sheet Name: `Notification Log`. Maps details such as opportunity ID, name, severity, scenario, and notification timestamp.
    - *Input/Output:* Input from `Post Alert to Slack`.
    - *Credentials:* Requires a `Google Service Account` credential.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## GoHighLevel & Stripe Automated Sales Operations Auditor<br><br>### How it works<br><br>Triggers on an hourly schedule to fetch opportunities from GoHighLevel and verify their monetary value.<br>Deduplicates opportunities against existing logs stored in a Google Sheet.<br>Collects historical conversations, tasks, and Stripe billing records for the associated contact.<br>Prepares contextual data and prompts a Gemini-powered AI agent to evaluate follow-up urgency and messaging.<br>Verifies if alert criteria are met, then formats and posts a notification to Slack.<br>Records the dispatched Slack alert back into Google Sheets for tracking.<br><br>### Setup steps<br><br>- - [ ] Connect GoHighLevel credentials for opportunity, conversation, and task retrieval.<br>- - [ ] Connect Stripe API credentials to retrieve customer and charge details.<br>- - [ ] Configure Google Gemini AI credentials in the LangChain chat model node.<br>- - [ ] Connect Google Sheets OAuth credentials and select the appropriate tracking spreadsheet.<br>- - [ ] Configure Slack credentials and specify the destination channel for alerts.<br><br>### Customization<br><br>Adjust the monetary threshold in the filter node or fine-tune prompt instructions and JSON schema inside the AI agent to fit specific sales follow-up criteria.<br><br>@[youtube](Fp2L7VmWtkk) |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Trigger and filter opportunities<br><br>Pulls GoHighLevel opportunities on an hourly schedule and filters them based on monetary value. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Identify unique opportunities<br><br>Cross-references opportunities against Google Sheets history to prevent duplicate handling. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Retrieve contact interactions<br><br>Fetches communication threads, conversation history, and contact tasks from GoHighLevel. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Fetch Stripe customer charges<br><br>Retrieves customer records and payment charge history directly from Stripe. |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Generate AI follow-up insights<br><br>Compiles collected CRM and financial context to evaluate deal urgency and suggest action via Gemini. |
| `Sticky Note6` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Filter and format alerts<br><br>Checks whether an alert is necessary based on AI severity output and formats the Slack payload. |
| `Sticky Note7` | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Dispatch and log alerts<br><br>Sends the generated alert notification to Slack and appends a record to Google Sheets. |
| `Every Hour Trigger` | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow every hour | None | `Fetch HighLevel Opportunities` | |
| `Fetch HighLevel Opportunities` | `n8n-nodes-base.highLevel` | Fetches GHL opportunities | `Every Hour Trigger` | `Check Opportunity Value` | |
| `Check Opportunity Value` | `n8n-nodes-base.if` | Filters opportunities with value > 0 | `Fetch HighLevel Opportunities` | `Read Tracked Leads in Sheets` | |
| `Read Tracked Leads in Sheets` | `n8n-nodes-base.googleSheets` | Reads notification log sheet | `Check Opportunity Value` | `Filter New Opportunities` | |
| `Filter New Opportunities` | `n8n-nodes-base.code` | Removes already notified opportunities | `Read Tracked Leads in Sheets` | `Search Contact Conversations` | |
| `Search Contact Conversations` | `n8n-nodes-base.httpRequest` | Searches GHL contact conversations | `Filter New Opportunities` | `Fetch Conversation Messages` | |
| `Fetch Conversation Messages` | `n8n-nodes-base.httpRequest` | Retrieves messages for conversation | `Search Contact Conversations` | `Fetch HighLevel Contact Tasks` | |
| `Fetch HighLevel Contact Tasks` | `n8n-nodes-base.highLevel` | Retrieves contact tasks | `Fetch Conversation Messages` | `Fetch Stripe Customer` | |
| `Fetch Stripe Customer` | `n8n-nodes-base.stripe` | Looks up customer by email | `Fetch HighLevel Contact Tasks` | `Fetch Stripe Charges` | |
| `Fetch Stripe Charges` | `n8n-nodes-base.httpRequest` | Retrieves customer payment charges | `Fetch Stripe Customer` | `Compile Follow-Up Context` | |
| `Compile Follow-Up Context` | `n8n-nodes-base.code` | Normalizes and constructs audit context | `Fetch Stripe Charges` | `Follow-Up Analysis Agent` | |
| `Follow-Up Analysis Agent` | `@n8n/n8n-nodes-langchain.agent` | Analyzes deal health via Gemini | `Compile Follow-Up Context` | `Set Alert Properties` | |
| `Gemini Chat Model` | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | Provides LLM backend for AI agent | None | `Follow-Up Analysis Agent` | |
| `Parse Agent Output` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Validates AI JSON output schema | None | `Follow-Up Analysis Agent` | |
| `Set Alert Properties` | `n8n-nodes-base.set` | Formats output properties for routing | `Follow-Up Analysis Agent` | `If Alert Needed` | |
| `If Alert Needed` | `n8n-nodes-base.if` | Gates execution based on alert status | `Set Alert Properties` | `Build Slack Message` | |
| `Build Slack Message` | `n8n-nodes-base.set` | Formats alert payload for Slack | `If Alert Needed` | `Post Alert to Slack` | |
| `Post Alert to Slack` | `n8n-nodes-base.slack` | Posts alert message to Slack channel | `Build Slack Message` | `Record Alert in Sheets` | |
| `Record Alert in Sheets` | `n8n-nodes-base.googleSheets` | Logs notification dispatch to Google Sheets | `Post Alert to Slack` | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add a **Schedule Trigger** (`Every Hour Trigger`). Set interval rule to hours.
2. **Add GoHighLevel Opportunity Fetcher:**
   - Add a **GoHighLevel** node (`Fetch HighLevel Opportunities`). Set resource to `opportunity`, operation to `getAll`. Configure pipeline ID (`smEMRhNOilT5Pm0QZCWO`) and stage ID (`2630bdf2-933e-4b14-a1c7-577412790640`). Link input from Trigger. Connect credential (`HighLevel OAuth2 API`).
3. **Add Value Validation Gate:**
   - Add an **If** node (`Check Opportunity Value`). Add condition checking if `{{ $json.monetaryValue }}` is greater than `0`. Link input from GoHighLevel opportunity fetcher.
4. **Add Google Sheets Tracking Reader:**
   - Add a **Google Sheets** node (`Read Tracked Leads in Sheets`). Set operation to Read, select document ID (`1NjNEYyzY6a4BoS1KmLa3WQF3zs1h2d8u0cT4dCcqoiA`) and sheet name (`Notification Log`). Enable `Execute Once`. Link input from If node. Connect `Google Service Account` credential.
5. **Add Deduplication Code Node:**
   - Add a **Code** node (`Filter New Opportunities`). Paste filtering JavaScript logic to compare incoming items against sheet rows. Link input from Google Sheets reader.
6. **Add GHL Conversation Search:**
   - Add an **HTTP Request** node (`Search Contact Conversations`). Set method to GET, URL to `https://services.leadconnectorhq.com/conversations/search`. Configure query parameters `locationId` and `contactId`. Add header `Version: 2021-04-15`. Link input from Code node. Connect `HighLevel OAuth2 API` credential.
7. **Add Conversation Messages Fetcher:**
   - Add an **HTTP Request** node (`Fetch Conversation Messages`). Set method to GET, URL to `https://services.leadconnectorhq.com/conversations/{{ $('Search Contact Conversations').item.json.conversations[0].id }}/messages`. Add query parameter `limit` = `100` and header `Version: 2021-04-15`. Link input from previous HTTP request. Connect `HighLevel OAuth2 API` credential.
8. **Add GHL Task Fetcher:**
   - Add a **GoHighLevel** node (`Fetch HighLevel Contact Tasks`). Set resource to `task`, operation to `getAll`, with contact ID mapped appropriately. Link input from message fetcher. Connect `HighLevel OAuth2 API` credential.
9. **Add Stripe Customer Lookup:**
   - Add a **Stripe** node (`Fetch Stripe Customer`). Set resource to `customer`, operation to `getAll`, filter by email. Link input from Task node. Connect `Stripe API` credential.
10. **Add Stripe Charges Fetcher:**
    - Add an **HTTP Request** node (`Fetch Stripe Charges`). Set method to GET, URL to `https://api.stripe.com/v1/charges` with query parameter `customer`. Link input from Stripe customer node. Connect `Stripe API` credential.
11. **Add Context Compilation Node:**
    - Add a **Code** node (`Compile Follow-Up Context`). Paste data normalization script mapping SLA metrics, payments, tasks, communication stats, and calculating flags. Link input from Stripe charges.
12. **Add AI Agent & LangChain Infrastructure:**
    - Add an **Advanced AI Agent** node (`Follow-Up Analysis Agent`). Set text to evaluate context JSON.
    - Connect a **Google Gemini Chat Model** (`Gemini Chat Model`) to the `ai_languageModel` input. Set model to `models/gemini-3.1-flash-lite` and temperature to `0.2`. Connect `Google Palm API` credential.
    - Connect a **Structured Output Parser** (`Parse Agent Output`) to the `ai_outputParser` input, providing the required JSON schema structure.
    - Link input from Context Compilation code node.
13. **Add Property Setter Node:**
    - Add a **Set** node (`Set Alert Properties`). Assign output fields mapped from the AI execution payload (e.g., `alert_needed`, `severity`, `scenario`, `headline`, `evidence_text`, `recommended_action`, `suggested_followup_message`, `escalate_to_manager`, and context metadata). Link input from AI Agent.
14. **Add Alert Filtering Gate:**
    - Add an **If** node (`If Alert Needed`). Add conditions ensuring `alert_needed` is `true` and `severity` is not `none`. Link input from Property Setter node.
15. **Add Slack Message Builder & Dispatcher:**
    - Add a **Set** node (`Build Slack Message`) to format the rich markdown payload based on severity level and action items. Link input from If node.
    - Add a **Slack** node (`Post Alert to Slack`). Set resource to Message, select channel, and configure target channel ID (`C0B8VH1M5PX`). Link input from Build Slack Message node. Connect `Slack API` credential.
16. **Add Notification Logger:**
    - Add a **Google Sheets** node (`Record Alert in Sheets`). Set operation to `appendOrUpdate`, matching columns to `opportunity_id`, document ID to `1NjNEYyzY6a4BoS1KmLa3WQF3zs1h2d8u0cT4dCcqoiA`, and sheet name to `Notification Log`. Map tracking properties (opportunity ID, name, severity, scenario, action, timestamps, etc.). Link input from Slack node. Connect `Google Service Account` credential.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Video walkthrough detailing architecture and operational design | [YouTube Video Tutorial](https://youtu.be/Fp2L7VmWtkk?si=o_F7GmxOtk0yYRDL) |
Detect revenue leakage and send AI-powered recovery follow-ups via Gmail and Slack

https://n8nworkflows.xyz/workflows/detect-revenue-leakage-and-send-ai-powered-recovery-follow-ups-via-gmail-and-slack-20958


# Detect revenue leakage and send AI-powered recovery follow-ups via Gmail and Slack

### 1. Workflow Overview

This scheduled automation workflow audits operational financial records stored across multiple Google Sheets tabs (Invoices, Work Delivered, Payments, and Follow-ups) to detect, prioritize, and mitigate revenue leakage. It correlates business data to discover anomalies such as uninvoiced work, duplicate billing, billing variances, unposted payments, and overdue accounts. Detected issues are logged back to Google Sheets, processed by an AI agent using Google Gemini to draft professional communications, and finally dispatched either to clients via Gmail or internally via Slack alerts, concluding with an audit summary.

The logic is organized into the following functional blocks:
- **1.1 Schedule & Configuration**: Initializes execution timing and core runtime configuration variables.
- **1.2 Source Data Retrieval**: Sequentially fetches operational records from Google Sheets.
- **1.3 Leakage Detection & Limiting**: Analyzes relational datasets to flag revenue risks, enriches findings with metadata, and limits execution volume per run.
- **1.4 Logging & AI Prompt Preparation**: Records recovery tasks back to Google Sheets and constructs contextual prompts for AI generation.
- **1.5 AI Generation & Parsing**: Invokes the Google Gemini language model via an AI agent to draft professional recovery messaging and parses the raw output into structured data.
- **1.6 Dispatch & Audit**: Evaluates routing rules to send either a customer email via Gmail or an internal alert via Slack, finishing with an audit consolidation node.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Configuration
- **Overview:** Initializes the workflow on a scheduled cadence and establishes global parameters (such as spreadsheet ID, timezone, and aging thresholds) used by downstream data retrieval and analysis nodes.
- **Nodes Involved:** `Scheduled Run Trigger`, `Set Configuration Parameters`.
- **Node Details:**
  - **Scheduled Run Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration Choices:* Configured with default interval settings to trigger pipeline runs automatically.
    - *Key Expressions/Variables:* None.
    - *Connections:* Input: None; Output: `Set Configuration Parameters`.
    - *Edge Cases/Failures:* Missed executions if n8n instance is offline.
  - **Set Configuration Parameters**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration Choices:* Assigns configuration constants: `spreadsheetId`, `timezone` (`Asia/Kolkata`), `gracePeriodDays` (`7`), `draftThresholdDays` (`3`), `staleFollowupDays` (`14`), and `dueSoonDays` (`3`).
    - *Key Expressions/Variables:* Static scalar values.
    - *Connections:* Input: `Scheduled Run Trigger`; Output: `Read Invoices from Sheets`.
    - *Edge Cases/Failures:* Invalid spreadsheet IDs will cause downstream Google Sheets nodes to fail.

---

#### 2.2 Source Data Retrieval
- **Overview:** Sequentially reads operational tabs from the designated Google Sheets document to assemble the complete dataset required for financial correlation.
- **Nodes Involved:** `Read Invoices from Sheets`, `Read Work Delivered from Sheets`, `Read Payments from Sheets`, `Read Follow-ups from Sheets`.
- **Node Details:**
  - **Read Invoices from Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Input)
    - *Configuration Choices:* Reads the "Invoices" sheet using Google Service Account authentication. Executes once to process the document context.
    - *Key Expressions/Variables:* `={{ $('Set Configuration Parameters').first().json.spreadsheetId }}`
    - *Connections:* Input: `Set Configuration Parameters`; Output: `Read Work Delivered from Sheets`.
    - *Edge Cases/Failures:* Authentication revocation, API rate limits, or missing tab name.
  - **Read Work Delivered from Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Input)
    - *Configuration Choices:* Reads the "Work Delivered" sheet using Service Account authentication.
    - *Key Expressions/Variables:* `={{ $('Set Configuration Parameters').first().json.spreadsheetId }}`
    - *Connections:* Input: `Read Invoices from Sheets`; Output: `Read Payments from Sheets`.
    - *Edge Cases/Failures:* Missing columns or sheet schema shifts.
  - **Read Payments from Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Input)
    - *Configuration Choices:* Reads the "Payments" sheet using Service Account authentication.
    - *Key Expressions/Variables:* `={{ $('Set Configuration Parameters').first().json.spreadsheetId }}`
    - *Connections:* Input: `Read Work Delivered from Sheets`; Output: `Read Follow-ups from Sheets`.
    - *Edge Cases/Failures:* Network timeouts or permission errors.
  - **Read Follow-ups from Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Input)
    - *Configuration Choices:* Reads the "Follow-ups" sheet using Service Account authentication.
    - *Key Expressions/Variables:* `={{ $('Set Configuration Parameters').first().json.spreadsheetId }}`
    - *Connections:* Input: `Read Payments from Sheets`; Output: `Detect Revenue Leakage`.
    - *Edge Cases/Failures:* Empty sheet returning unexpected array structures.

---

#### 2.3 Leakage Detection & Limiting
- **Overview:** Executes core business logic via JavaScript to correlate invoices, work items, payments, and follow-ups. It identifies leakage findings, assigns priorities and unique recovery IDs, and caps the processed volume to the first two tasks for the current run.
- **Nodes Involved:** `Detect Revenue Leakage`, `Limit Items to 2 Tasks`.
- **Node Details:**
  - **Detect Revenue Leakage**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Execution)
    - *Configuration Choices:* Custom JavaScript processing cross-tab relationships, calculating aging thresholds, evaluating balance mismatches, and generating structured finding objects.
    - *Key Expressions/Variables:* Reads all items from preceding Google Sheets nodes and configuration parameters.
    - *Connections:* Input: `Read Follow-ups from Sheets`; Output: `Limit Items to 2 Tasks`.
    - *Edge Cases/Failures:* Unhandled date formats or malformed currency strings causing parsing NaN errors.
  - **Limit Items to 2 Tasks**
    - *Type and Technical Role:* `n8n-nodes-base.limit` (Flow Control)
    - *Configuration Choices:* Maximum items set to `2`.
    - *Key Expressions/Variables:* None.
    - *Connections:* Input: `Detect Revenue Leakage`; Outputs: `Append Recovery Tasks to Sheets`, `Create AI Dispatch Prompts`.
    - *Edge Cases/Failures:* Zero findings resulting in empty downstream execution paths.

---

#### 2.4 Logging & AI Prompt Preparation
- **Overview:** Persists limited recovery tasks back into the Google Sheets audit trail while concurrently formatting structured contextual prompts for the AI recovery agent.
- **Nodes Involved:** `Append Recovery Tasks to Sheets`, `Create AI Dispatch Prompts`.
- **Node Details:**
  - **Append Recovery Tasks to Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Output)
    - *Configuration Choices:* Appends rows to the "Follow-ups" sheet using auto-mapped input data and Service Account authentication.
    - *Key Expressions/Variables:* `={{ $('Set Configuration Parameters').first().json.spreadsheetId }}`
    - *Connections:* Input: `Limit Items to 2 Tasks`; Output: None (Terminal branch leg).
    - *Edge Cases/Failures:* Schema mapping mismatches between generated JSON properties and sheet column headers.
  - **Create AI Dispatch Prompts**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Execution)
    - *Configuration Choices:* Iterates over input items to construct formatted string templates containing finding attributes, tone recommendations, and strict JSON response formatting constraints.
    - *Key Expressions/Variables:* `$input.all()`, item properties (`finding_type`, `client_name`, `amount_at_risk`, etc.).
    - *Connections:* Input: `Limit Items to 2 Tasks`; Output: `Revenue Recovery AI Agent`.
    - *Edge Cases/Failures:* Missing optional property keys causing undefined interpolation in prompt text.

---

#### 2.5 AI Generation & Parsing
- **Overview:** Leverages a LangChain-based AI agent powered by a Google Gemini chat model to generate customized recovery messaging, which is subsequently sanitized and parsed into structured JSON.
- **Nodes Involved:** `Revenue Recovery AI Agent`, `Google Gemini AI Model`, `Interpret AI Results`.
- **Node Details:**
  - **Revenue Recovery AI Agent**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI / Advanced AI)
    - *Configuration Choices:* Uses system instructions defining a careful B2B finance operations assistant persona, enforcing strict JSON schema output requirements without fabricating unverified payment data.
    - *Key Expressions/Variables:* `={{ $json.promptText }}`
    - *Connections:* Input: `Create AI Dispatch Prompts`; Output: `Interpret AI Results`; AI Model Input: Connected from `Google Gemini AI Model`.
    - *Edge Cases/Failures:* API rate limiting, token exhaustion, or LLM failure to respect strict JSON output formatting.
  - **Google Gemini AI Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (AI Sub-node)
    - *Configuration Choices:* Configured with model `models/gemini-3.1-flash-lite` and a low temperature setting (`0.2`) for deterministic output. Requires Google Palm API credentials.
    - *Key Expressions/Variables:* None.
    - *Connections:* Output (ai_languageModel): Connected to `Revenue Recovery AI Agent`.
    - *Edge Cases/Failures:* Authentication failure or model deprecation.
  - **Interpret AI Results**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Execution)
    - *Configuration Choices:* JavaScript snippet that strips markdown code blocks from the raw LLM output, extracts the JSON payload using regex, and gracefully falls back to default values if parsing fails.
    - *Key Expressions/Variables:* `$json.output`, `$json.text`, upstream node reference (`$('Create AI Dispatch Prompts')`).
    - *Connections:* Input: `Revenue Recovery AI Agent`; Output: `Check Email Eligibility`.
    - *Edge Cases/Failures:* Completely unstructured text output resulting in fallback error objects.

---

#### 2.6 Dispatch & Audit
- **Overview:** Evaluates communication channels to determine whether to email a client directly via Gmail or post an alert internally to Slack, then aggregates overall run metrics into a final audit summary.
- **Nodes Involved:** `Check Email Eligibility`, `Send Client Email`, `Send Slack Notification`, `Compile Audit Summary`.
- **Node Details:**
  - **Check Email Eligibility**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Evaluates conditions ensuring `audience` equals `CUSTOMER` and `client_email` is not empty.
    - *Key Expressions/Variables:* `={{ $json.audience }}`, `={{ $json.client_email }}`
    - *Connections:* Input: `Interpret AI Results`; True Output: `Send Client Email`; False Output: `Send Slack Notification`.
    - *Edge Cases/Failures:* Incorrect evaluation if data types or casing deviate from expected constants.
  - **Send Client Email**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Communication)
    - *Configuration Choices:* Sends plain text emails using Gmail OAuth2 credentials.
    - *Key Expressions/Variables:* To: `={{ $json.client_email }}`, Subject: `={{ $json.aiSubject }}`, Message: `={{ $json.aiMessage }}`.
    - *Connections:* Input: `Check Email Eligibility` (True); Output: `Compile Audit Summary`.
    - *Edge Cases/Failures:* Invalid recipient email formats, OAuth token expiration, or sending limits exceeded.
  - **Send Slack Notification**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Communication)
    - *Configuration Choices:* Posts formatted rich-text warning blocks to the `#general` channel using Slack API credentials.
    - *Key Expressions/Variables:* Text constructed via template expressions combining priority, finding type, client name, financial exposure, reference ID, and AI actions.
    - *Connections:* Input: `Check Email Eligibility` (False); Output: `Compile Audit Summary`.
    - *Edge Cases/Failures:* Channel permission errors or missing webhook configurations.
  - **Compile Audit Summary**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Execution)
    - *Configuration Choices:* Aggregates execution metrics across all detected findings to calculate total records processed and aggregate revenue at risk.
    - *Key Expressions/Variables:* `$('Detect Revenue Leakage').all()`
    - *Connections:* Inputs: `Send Client Email`, `Send Slack Notification`; Output: None (Terminal node).
    - *Edge Cases/Failures:* Type conversion errors if amount properties contain non-numeric characters.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation & Guide | None | None | ## Revenue Leakage Detector & AI Recovery Pipeline<br><br>### How it works<br><br>This scheduled workflow audits spreadsheet data for revenue leakage by comparing invoices, delivered work, payments, and prior follow-ups. It detects recovery opportunities, logs tasks, then uses an AI agent to draft recovery dispatch content. Depending on eligibility, it sends a client email or posts an internal Slack alert, and finishes by creating an audit summary.<br><br>### Setup steps<br><br>- Configure the Schedule Trigger cadence for how often the revenue audit should run.<br>- Set the Config node values, including spreadsheetId, timezone, gracePeriodDays, and draftThresholdDays.<br>- Connect Google Sheets credentials and ensure the spreadsheet has the expected invoice, work delivered, payment, follow-up, and recovery-task sheets/columns.<br>- Connect Google Gemini credentials for the AI Recovery Agent and verify the selected model is available.<br>- Connect Gmail credentials for client email dispatch and Slack credentials/channel settings for internal alerts.<br><br>### Customization<br><br>Adjust the leakage detection code, limit size, grace-period thresholds, AI prompt construction, and the email/slack eligibility rules to match your billing policy and recovery workflow.<br><br>@[youtube](d74YKhD0P24) |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Documentation & Guide | None | None | ## Schedule and configuration<br><br>Starts the workflow on a schedule and applies core runtime settings such as the target spreadsheet, timezone, grace period, and draft thresholds. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Documentation & Guide | None | None | ## Fetch source records<br><br>Reads the operational source data from Google Sheets, including invoices, delivered work, payments, and existing follow-up records. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Documentation & Guide | None | None | ## Detect recovery tasks<br><br>Analyzes the fetched records to identify revenue leakage and builds recovery tasks, then caps the number of tasks processed in the current run. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Documentation & Guide | None | None | ## Log and prepare prompts<br><br>Branches from the limited recovery tasks to both log task records back to Sheets and prepare AI dispatch prompts for follow-up actions. |
| `Sticky Note5` | `n8n-nodes-base.stickyNote` | Documentation & Guide | None | None | ## Generate AI recovery<br><br>Uses the AI Recovery Agent with the connected Google Gemini chat model to generate recovery messaging or recommended actions from the prepared prompts. |
| `Sticky Note6` | `n8n-nodes-base.stickyNote` | Documentation & Guide | None | None | ## Parse and decide<br><br>Parses the AI output into structured fields and checks whether the case is eligible for a direct customer email or should be handled internally. |
| `Sticky Note7` | `n8n-nodes-base.stickyNote` | Documentation & Guide | None | None | ## Dispatch and audit<br><br>Sends the chosen recovery communication path: either a Gmail client email or an internal Slack alert, then consolidates the outcome in an audit summary. |
| `Set Configuration Parameters` | `n8n-nodes-base.set` | Data Transformation | `Scheduled Run Trigger` | `Read Invoices from Sheets` | ## Schedule and configuration<br><br>Starts the workflow on a schedule and applies core runtime settings such as the target spreadsheet, timezone, grace period, and draft thresholds. |
| `Read Invoices from Sheets` | `n8n-nodes-base.googleSheets` | Data Input | `Set Configuration Parameters` | `Read Work Delivered from Sheets` | ## Fetch source records<br><br>Reads the operational source data from Google Sheets, including invoices, delivered work, payments, and existing follow-up records. |
| `Read Work Delivered from Sheets` | `n8n-nodes-base.googleSheets` | Data Input | `Read Invoices from Sheets` | `Read Payments from Sheets` | ## Fetch source records<br><br>Reads the operational source data from Google Sheets, including invoices, delivered work, payments, and existing follow-up records. |
| `Read Payments from Sheets` | `n8n-nodes-base.googleSheets` | Data Input | `Read Work Delivered from Sheets` | `Read Follow-ups from Sheets` | ## Fetch source records<br><br>Reads the operational source data from Google Sheets, including invoices, delivered work, payments, and existing follow-up records. |
| `Read Follow-ups from Sheets` | `n8n-nodes-base.googleSheets` | Data Input | `Read Payments from Sheets` | `Detect Revenue Leakage` | ## Fetch source records<br><br>Reads the operational source data from Google Sheets, including invoices, delivered work, payments, and existing follow-up records. |
| `Detect Revenue Leakage` | `n8n-nodes-base.code` | Code Execution | `Read Follow-ups from Sheets` | `Limit Items to 2 Tasks` | ## Detect recovery tasks<br><br>Analyzes the fetched records to identify revenue leakage and builds recovery tasks, then caps the number of tasks processed in the current run. |
| `Append Recovery Tasks to Sheets` | `n8n-nodes-base.googleSheets` | Data Output | `Limit Items to 2 Tasks` | None | ## Log and prepare prompts<br><br>Branches from the limited recovery tasks to both log task records back to Sheets and prepare AI dispatch prompts for follow-up actions. |
| `Create AI Dispatch Prompts` | `n8n-nodes-base.code` | Code Execution | `Limit Items to 2 Tasks` | `Revenue Recovery AI Agent` | ## Log and prepare prompts<br><br>Branches from the limited recovery tasks to both log task records back to Sheets and prepare AI dispatch prompts for follow-up actions. |
| `Revenue Recovery AI Agent` | `@n8n/n8n-nodes-langchain.agent` | AI / Advanced AI | `Create AI Dispatch Prompts` | `Interpret AI Results` | ## Generate AI recovery<br><br>Uses the AI Recovery Agent with the connected Google Gemini chat model to generate recovery messaging or recommended actions from the prepared prompts. |
| `Google Gemini AI Model` | `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` | AI Sub-node | None (Connected via AI model input) | `Revenue Recovery AI Agent` | ## Generate AI recovery<br><br>Uses the AI Recovery Agent with the connected Google Gemini chat model to generate recovery messaging or recommended actions from the prepared prompts. |
| `Interpret AI Results` | `n8n-nodes-base.code` | Code Execution | `Revenue Recovery AI Agent` | `Check Email Eligibility` | ## Parse and decide<br><br>Parses the AI output into structured fields and checks whether the case is eligible for a direct customer email or should be handled internally. |
| `Check Email Eligibility` | `n8n-nodes-base.if` | Flow Control | `Interpret AI Results` | `Send Client Email`, `Send Slack Notification` | ## Parse and decide<br><br>Parses the AI output into structured fields and checks whether the case is eligible for a direct customer email or should be handled internally. |
| `Send Slack Notification` | `n8n-nodes-base.slack` | Communication | `Check Email Eligibility` (False) | `Compile Audit Summary` | ## Dispatch and audit<br><br>Sends the chosen recovery communication path: either a Gmail client email or an internal Slack alert, then consolidates the outcome in an audit summary. |
| `Compile Audit Summary` | `n8n-nodes-base.code` | Code Execution | `Send Client Email`, `Send Slack Notification` | None | ## Dispatch and audit<br><br>Sends the chosen recovery communication path: either a Gmail client email or an internal Slack alert, then consolidates the outcome in an audit summary. |
| `Scheduled Run Trigger` | `n8n-nodes-base.scheduleTrigger` | Trigger | None | `Set Configuration Parameters` | ## Schedule and configuration<br><br>Starts the workflow on a schedule and applies core runtime settings such as the target spreadsheet, timezone, grace period, and draft thresholds. |
| `Send Client Email` | `n8n-nodes-base.gmail` | Communication | `Check Email Eligibility` (True) | `Compile Audit Summary` | ## Dispatch and audit<br><br>Sends the chosen recovery communication path: either a Gmail client email or an internal Slack alert, then consolidates the outcome in an audit summary. |
| `Limit Items to 2 Tasks` | `n8n-nodes-base.limit` | Flow Control | `Detect Revenue Leakage` | `Append Recovery Tasks to Sheets`, `Create AI Dispatch Prompts` | ## Detect recovery tasks<br><br>Analyzes the fetched records to identify revenue leakage and builds recovery tasks, then caps the number of tasks processed in the current run. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to manually rebuild the workflow in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`). Leave default interval settings. Name it `Scheduled Run Trigger`.
2. **Add Configuration Setup:**
   - Add a **Set** node (`n8n-nodes-base.set`). Name it `Set Configuration Parameters`.
   - Configure assignments:
     - `spreadsheetId` (String): `1yv38rPH9qQUMb0ObwoHyhwQfwgG3KA9ILmnmPPBaggk`
     - `timezone` (String): `Asia/Kolkata`
     - `gracePeriodDays` (Number): `7`
     - `draftThresholdDays` (Number): `3`
     - `staleFollowupDays` (Number): `14`
     - `dueSoonDays` (Number): `3`
   - Connect `Scheduled Run Trigger` to `Set Configuration Parameters`.
3. **Add Google Sheets Source Readers:**
   - Add four **Google Sheets** nodes (`n8n-nodes-base.googleSheets`), authenticated via Google Service Account (`Google Service Account account` credential):
     - **Read Invoices from Sheets**: Sheet name `Invoices`, Document ID expression `={{ $('Set Configuration Parameters').first().json.spreadsheetId }}`.
     - **Read Work Delivered from Sheets**: Sheet name `Work Delivered`, Document ID expression `={{ $('Set Configuration Parameters').first().json.spreadsheetId }}`.
     - **Read Payments from Sheets**: Sheet name `Payments`, Document ID expression `={{ $('Set Configuration Parameters').first().json.spreadsheetId }}`.
     - **Read Follow-ups from Sheets**: Sheet name `Follow-ups`, Document ID expression `={{ $('Set Configuration Parameters').first().json.spreadsheetId }}`.
   - Connect sequentially: `Set Configuration Parameters` $\rightarrow$ `Read Invoices from Sheets` $\rightarrow$ `Read Work Delivered from Sheets` $\rightarrow$ `Read Payments from Sheets` $\rightarrow$ `Read Follow-ups from Sheets`.
4. **Add Leakage Detection & Limiting:**
   - Add a **Code** node (`n8n-nodes-base.code`), name it `Detect Revenue Leakage`, and paste the JavaScript logic that correlates invoices, work items, payments, and follow-ups to output findings. Connect `Read Follow-ups from Sheets` to this node.
   - Add a **Limit** node (`n8n-nodes-base.limit`), name it `Limit Items to 2 Tasks`, set `Max Items` to `2`. Connect `Detect Revenue Leakage` to it.
5. **Create Logging and Prompt Preparation Branches:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`), name it `Append Recovery Tasks to Sheets`, operation `append`, Sheet name `Follow-ups`, Document ID expression `={{ $('Set Configuration Parameters').first().json.spreadsheetId }}`, using Service Account authentication.
   - Add a **Code** node (`n8n-nodes-base.code`), name it `Create AI Dispatch Prompts`, with JavaScript mapping items to construct prompt text instructing the AI to return strictly formatted JSON with keys `audience`, `subject`, `message_body`, and `recommended_action`.
   - Connect `Limit Items to 2 Tasks` to *both* `Append Recovery Tasks to Sheets` and `Create AI Dispatch Prompts`.
6. **Configure AI Agent & Model:**
   - Add an **Advanced AI Agent** node (`@n8n/n8n-nodes-langchain.agent`), name it `Revenue Recovery AI Agent`. Set `Prompt Type` to `Define`, text expression `={{ $json.promptText }}`, and configure the system message to enforce strict JSON output.
   - Add a **Google Gemini Chat Model** sub-node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`), name it `Google Gemini AI Model`, set model name to `models/gemini-3.1-flash-lite`, temperature to `0.2`, and configure Google Palm API credentials. Connect its AI output to the Agent's model input.
   - Connect `Create AI Dispatch Prompts` to `Revenue Recovery AI Agent`.
7. **Add Result Interpretation & Routing:**
   - Add a **Code** node (`n8n-nodes-base.code`), name it `Interpret AI Results`, with JavaScript to sanitize raw LLM output, parse JSON, and handle fallback errors. Connect `Revenue Recovery AI Agent` to it.
   - Add an **IF** node (`n8n-nodes-base.if`), name it `Check Email Eligibility`. Configure conditions: `audience` equals `CUSTOMER` AND `client_email` is not empty. Connect `Interpret AI Results` to it.
8. **Configure Communications & Audit Summary:**
   - Add a **Gmail** node (`n8n-nodes-base.gmail`), name it `Send Client Email`. Set Operation to send, `Send To` = `={{ $json.client_email }}`, `Subject` = `={{ $json.aiSubject }}`, `Message` = `={{ $json.aiMessage }}`, using Gmail OAuth2 credentials (`Abhiram.bvb`). Connect to the `True` output of `Check Email Eligibility`.
   - Add a **Slack** node (`n8n-nodes-base.slack`), name it `Send Slack Notification`. Set resource/action to post message, channel `#general`, using Slack API credentials (`Slack account 2`), with text formatting expressions combining priorities, exposure amounts, and AI actions. Connect to the `False` output of `Check Email Eligibility`.
   - Add a **Code** node (`n8n-nodes-base.code`), name it `Compile Audit Summary`, with JavaScript aggregating total findings processed and total revenue at risk.
   - Connect *both* `Send Client Email` and `Send Slack Notification` into `Compile Audit Summary`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Video Walkthrough & Reference Tutorial | [YouTube Video Link](https://youtu.be/d74YKhD0P24) |
| Workflow Template Context | Revenue Leakage Detector & AI Recovery Pipeline |
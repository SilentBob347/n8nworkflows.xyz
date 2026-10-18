Recover abandoned quotes with GPT-4o mini, Gmail, Twilio and Google Sheets

https://n8nworkflows.xyz/workflows/recover-abandoned-quotes-with-gpt-4o-mini--gmail--twilio-and-google-sheets-20850


# Recover abandoned quotes with GPT-4o mini, Gmail, Twilio and Google Sheets

### 1. Workflow Overview

This workflow automates the recovery of abandoned customer quotes by running a daily scheduled check, verifying quote statuses against an external CRM via HTTP, generating personalized AI follow-up messages using OpenAI, dispatching them through Gmail or Twilio, and maintaining comprehensive audit logs in Google Sheets. 

The logical execution is grouped into the following blocks:
- **1.1 Initialization and Data Retrieval:** Triggers daily at 10:00 AM, loads global configuration variables, and fetches all open quotes from a Google Sheet.
- **1.2 Filtering and CRM Validation:** Filters quotes to identify those due for follow-up based on their touch count and age, then performs an HTTP verification call to confirm each quote remains open in the source system.
- **1.3 AI Message Generation and Channel Routing:** Evaluates quote status, utilizes an OpenAI language model to compose context-aware follow-up text, and routes the output to either Gmail (for initial touches) or Twilio (for subsequent SMS touches).
- **1.4 Logging and Audit Trail:** Records successfully sent touches back to a Google Sheets tracking tab or logs closed quotes separately if the CRM verification indicates they are no longer active.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Data Retrieval
- **Overview:** Sets up the execution schedule, injects foundational business configuration parameters, and retrieves the raw dataset of open quotes from Google Sheets.
- **Nodes Involved:** 
  - `Schedule - Daily 10AM`
  - `CONFIG - Quotes`
  - `Get Open Quotes - Sheets`

- **Node Details:**
  - **Schedule - Daily 10AM**
    - *Type and Role:* `n8n-nodes-base.scheduleTrigger` (Trigger). Initiates the workflow daily at 10:00 AM.
    - *Configuration:* Interval set to 1 day, triggering at hour 10, minute 0.
    - *Connections:* Input: None. Output: `CONFIG - Quotes`.
    - *Failure Types:* Missed executions if n8n instance is offline.
  - **CONFIG - Quotes**
    - *Type and Role:* `n8n-nodes-base.set` (Data Transformation). Establishes global environment variables.
    - *Configuration:* Assigns `business_name`, `owner_email`, `twilio_number`, and `sheet_id`.
    - *Key Expressions:* Holds static string values representing workspace identifiers.
    - *Connections:* Input: `Schedule - Daily 10AM`. Output: `Get Open Quotes - Sheets`.
    - *Failure Types:* Missing or mistyped configuration keys.
  - **Get Open Quotes - Sheets**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Integration). Reads row data from the source sheet.
    - *Configuration:* Resource: `sheet`, Operation: `read`, Sheet Name: `Quotes`.
    - *Key Expressions:* Document ID dynamically evaluated using `={{ $('CONFIG - Quotes').first().json.sheet_id }}`.
    - *Connections:* Input: `CONFIG - Quotes`. Output: `Filter Due - Code`.
    - *Credentials:* Requires Google Sheets OAuth2 API credentials.
    - *Failure Types:* API rate limits, invalid Sheet ID, missing 'Quotes' tab.

---

#### 2.2 Filtering and CRM Validation
- **Overview:** Evaluates quote dates against a predefined touch schedule, checks opt-out and status fields, and queries the external CRM via HTTP to verify that the target quote is still open.
- **Nodes Involved:**
  - `Filter Due - Code`
  - `Check Quote Status - HTTP`
  - `IF - Still Open?`
  - `Mark Closed - Sheets`

- **Node Details:**
  - **Filter Due - Code**
    - *Type and Role:* `n8n-nodes-base.code` (Logic/Data Transformation). Runs custom JavaScript to determine which quotes require follow-up.
    - *Configuration:* Evaluates `status === 'open'`, `opted_out !== 'yes'`, `touch_count < 3`, and checks time intervals against the schedule `[2 days, 7 days, 14 days]`.
    - *Connections:* Input: `Get Open Quotes - Sheets`. Output: `Check Quote Status - HTTP`.
    - *Failure Types:* JavaScript runtime exceptions due to unexpected date string formats or missing row schemas.
  - **Check Quote Status - HTTP**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (Integration). Queries the CRM endpoint to re-validate quote status.
    - *Configuration:* Method: `GET`, Timeout: 15,000ms, Authentication: None.
    - *Key Expressions:* URL evaluates to `https://YOUR_CRM/quotes/{{ $json.quote_id }}`.
    - *Connections:* Input: `Filter Due - Code`. Output: `IF - Still Open?`.
    - *Failure Types:* Network timeouts, HTTP 4xx/5xx errors, incorrect API base URLs.
  - **IF - Still Open?**
    - *Type and Role:* `n8n-nodes-base.if` (Flow Control). Splits execution based on whether the CRM reports the quote as open.
    - *Configuration:* Condition checks if `{{ $json.status }}` equals `open` using loose type validation.
    - *Connections:* Input: `Check Quote Status - HTTP`. Outputs: Branch True (`Build Follow-Up - AI Agent`), Branch False (`Mark Closed - Sheets`).
    - *Failure Types:* Unexpected payload schemas resulting in evaluation mismatches.
  - **Mark Closed - Sheets**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Integration). Appends records of quotes found closed during re-verification to an audit tab.
    - *Configuration:* Resource: `sheet`, Operation: `append`, Sheet Name: `ClosedLog`.
    - *Key Expressions:* Appends timestamp (`$now.toISO()`) and `quote_id` sourced from `Filter Due - Code`.
    - *Connections:* Input: `IF - Still Open?` (False branch). Output: None.
    - *Credentials:* Google Sheets OAuth2 API.
    - *Failure Types:* Schema mismatches or write permissions errors on the target sheet.

---

#### 2.3 AI Message Generation and Channel Routing
- **Overview:** Leverages an OpenAI language model to draft a personalized follow-up message matching the business voice, then routes delivery through Gmail for email touches or Twilio for SMS touches.
- **Nodes Involved:**
  - `Build Follow-Up - AI Agent`
  - `Chat Model - FollowUp`
  - `IF - Touch 0 (Email)?`
  - `Send Email - Gmail`
  - `Send SMS - Twilio`

- **Node Details:**
  - **Build Follow-Up - AI Agent**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.agent` (AI). Generates contextual messaging using prompt instructions.
    - *Configuration:* System message enforces tone constraints across 3 distinct touch phases, outputting plain message text with required opt-out disclosures for SMS. Max iterations set to 6.
    - *Key Expressions:* Constructs prompt input using customer name, quote amount, and touch index from upstream nodes.
    - *Connections:* Input: `IF - Still Open?` (True branch), `Chat Model - FollowUp` (AI Language Model link). Output: `IF - Touch 0 (Email)?`.
    - *Failure Types:* OpenAI API quota exhaustion, malformed output, or prompt rejection errors.
  - **Chat Model - FollowUp**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Sub-node). Connects the AI agent to the OpenAI LLM provider.
    - *Configuration:* Model set to `gpt-4o-mini`.
    - *Connections:* Output: Linked to `Build Follow-Up - AI Agent`.
    - *Credentials:* OpenAI API Key.
    - *Failure Types:* Invalid API key, rate limits.
  - **IF - Touch 0 (Email)?**
    - *Type and Role:* `n8n-nodes-base.if` (Flow Control). Determines the communication medium based on the touch index.
    - *Configuration:* Evaluates if touch level equals `0`.
    - *Connections:* Input: `Build Follow-Up - AI Agent`. Outputs: True branch (`Send Email - Gmail`), False branch (`Send SMS - Twilio`).
    - *Failure Types:* Type casting mismatches.
  - **Send Email - Gmail**
    - *Type and Role:* `n8n-nodes-base.gmail` (Integration). Dispatches email follow-ups for touch 0.
    - *Configuration:* Resource: `message`, Operation: `send`, Email Type: `text`, Subject: `Quick check-in on your quote`.
    - *Key Expressions:* Recipient evaluated via `={{ $('Filter Due - Code').item.json.email }}`, message body via `={{ $('Build Follow-Up - AI Agent').item.json.output }}`.
    - *Connections:* Input: `IF - Touch 0 (Email)?` (True). Output: `Log Touch - Sheets`.
    - *Credentials:* Gmail OAuth2.
    - *Failure Types:* Token expiration, invalid recipient address, SMTP rejection.
  - **Send SMS - Twilio**
    - *Type and Role:* `n8n-nodes-base.twilio` (Integration). Dispatches SMS follow-ups for touch levels 1 and 2.
    - *Configuration:* Resource: `sms`, Operation: `send`.
    - *Key Expressions:* Destination evaluated via `={{ $('Filter Due - Code').item.json.phone }}`, sender number via global configuration, message text from AI output.
    - *Connections:* Input: `IF - Touch 0 (Email)?` (False) or direct fallback. Output: `Log Touch - Sheets`.
    - *Credentials:* Twilio API.
    - *Failure Types:* Insufficient account funds, invalid phone number formatting, carrier blocking.

---

#### 2.4 Logging and Audit Trail
- **Overview:** Records dispatched touch events back into a dedicated tracking tab in Google Sheets.
- **Nodes Involved:**
  - `Log Touch - Sheets`

- **Node Details:**
  - **Log Touch - Sheets**
    - *Type and Role:* `n8n-nodes-base.googleSheets` (Integration). Appends historical engagement data to the Touches tab.
    - *Configuration:* Resource: `sheet`, Operation: `append`, Sheet Name: `Touches`.
    - *Key Expressions:* Logs ISO timestamp, touch index, channel type (`email` or `sms`), and `quote_id`.
    - *Connections:* Inputs: `Send Email - Gmail` and `Send SMS - Twilio`. Output: None.
    - *Credentials:* Google Sheets OAuth2 API.
    - *Failure Types:* Sheet permission errors or missing column headers.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule - Daily 10AM | n8n-nodes-base.scheduleTrigger | Triggers the workflow daily at 10 AM | None | CONFIG - Quotes | Quote follow-up: daily 10 AM, find open quotes due for a touch (day 2 email, day 7 SMS, day 14 final SMS) → re-check status → AI writes it → send → log. Edit CONFIG - Quotes; create tabs Quotes, Touches, ClosedLog. |
| CONFIG - Quotes | n8n-nodes-base.set | Stores global configuration values | Schedule - Daily 10AM | Get Open Quotes - Sheets | Quote follow-up: daily 10 AM, find open quotes due for a touch (day 2 email, day 7 SMS, day 14 final SMS) → re-check status → AI writes it → send → log. Edit CONFIG - Quotes; create tabs Quotes, Touches, ClosedLog. |
| Get Open Quotes - Sheets | n8n-nodes-base.googleSheets | Fetches rows from the Quotes sheet tab | CONFIG - Quotes | Filter Due - Code | Quote follow-up: daily 10 AM, find open quotes due for a touch (day 2 email, day 7 SMS, day 14 final SMS) → re-check status → AI writes it → send → log. Edit CONFIG - Quotes; create tabs Quotes, Touches, ClosedLog. |
| Filter Due - Code | n8n-nodes-base.code | Identifies quotes due for follow-up via JavaScript | Get Open Quotes - Sheets | Check Quote Status - HTTP | Quote follow-up: daily 10 AM, find open quotes due for a touch (day 2 email, day 7 SMS, day 14 final SMS) → re-check status → AI writes it → send → log. Edit CONFIG - Quotes; create tabs Quotes, Touches, ClosedLog. |
| Check Quote Status - HTTP | n8n-nodes-base.httpRequest | Re-verifies quote status against external CRM | Filter Due - Code | IF - Still Open? | Quote follow-up: daily 10 AM, find open quotes due for a touch (day 2 email, day 7 SMS, day 14 final SMS) → re-check status → AI writes it → send → log. Edit CONFIG - Quotes; create tabs Quotes, Touches, ClosedLog. |
| IF - Still Open? | n8n-nodes-base.if | Evaluates if the CRM quote status is open | Check Quote Status - HTTP | Build Follow-Up - AI Agent, Mark Closed - Sheets | Quote follow-up: daily 10 AM, find open quotes due for a touch (day 2 email, day 7 SMS, day 14 final SMS) → re-check status → AI writes it → send → log. Edit CONFIG - Quotes; create tabs Quotes, Touches, ClosedLog. |
| Mark Closed - Sheets | n8n-nodes-base.googleSheets | Logs quotes found closed to the ClosedLog tab | IF - Still Open? | None | Quote follow-up: daily 10 AM, find open quotes due for a touch (day 2 email, day 7 SMS, day 14 final SMS) → re-check status → AI writes it → send → log. Edit CONFIG - Quotes; create tabs Quotes, Touches, ClosedLog. |
| Build Follow-Up - AI Agent | @n8n/n8n-nodes-langchain.agent | Generates custom follow-up text using AI | IF - Still Open?, Chat Model - FollowUp | IF - Touch 0 (Email)? | Quote follow-up: daily 10 AM, find open quotes due for a touch (day 2 email, day 7 SMS, day 14 final SMS) → re-check status → AI writes it → send → log. Edit CONFIG - Quotes; create tabs Quotes, Touches, ClosedLog. |
| Chat Model - FollowUp | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o-mini language model backend | None | Build Follow-Up - AI Agent | Quote follow-up: daily 10 AM, find open quotes due for a touch (day 2 email, day 7 SMS, day 14 final SMS) → re-check status → AI writes it → send → log. Edit CONFIG - Quotes; create tabs Quotes, Touches, ClosedLog. |
| IF - Touch 0 (Email)? | n8n-nodes-base.if | Routes execution to Email or SMS channels | Build Follow-Up - AI Agent | Send Email - Gmail, Send SMS - Twilio | Quote follow-up: daily 10 AM, find open quotes due for a touch (day 2 email, day 7 SMS, day 14 final SMS) → re-check status → AI writes it → send → log. Edit CONFIG - Quotes; create tabs Quotes, Touches, ClosedLog. |
| Send Email - Gmail | n8n-nodes-base.gmail | Sends initial follow-up emails via Gmail | IF - Touch 0 (Email)? | Log Touch - Sheets | Quote follow-up: daily 10 AM, find open quotes due for a touch (day 2 email, day 7 SMS, day 14 final SMS) → re-check status → AI writes it → send → log. Edit CONFIG - Quotes; create tabs Quotes, Touches, ClosedLog. |
| Send SMS - Twilio | n8n-nodes-base.twilio | Sends follow-up SMS messages via Twilio | IF - Touch 0 (Email)? | Log Touch - Sheets | Quote follow-up: daily 10 AM, find open quotes due for a touch (day 2 email, day 7 SMS, day 14 final SMS) → re-check status → AI writes it → send → log. Edit CONFIG - Quotes; create tabs Quotes, Touches, ClosedLog. |
| Log Touch - Sheets | n8n-nodes-base.googleSheets | Appends a record of sent touches to the Touches tab | Send Email - Gmail, Send SMS - Twilio | None | Quote follow-up: daily 10 AM, find open quotes due for a touch (day 2 email, day 7 SMS, day 14 final SMS) → re-check status → AI writes it → send → log. Edit CONFIG - Quotes; create tabs Quotes, Touches, ClosedLog. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually within n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node named `Schedule - Daily 10AM`.
   - Set the interval to every 1 day, triggering at 10:00 AM.
2. **Add Configuration:**
   - Add a **Set** node named `CONFIG - Quotes`.
   - Create assignments for:
     - `business_name` (String: `YOUR BUSINESS NAME`)
     - `owner_email` (String: `user@example.com`)
     - `twilio_number` (String: Your Twilio phone number)
     - `sheet_id` (String: Your target Google Sheet ID)
   - Connect `Schedule - Daily 10AM` to `CONFIG - Quotes`.
3. **Read Google Sheets Data:**
   - Add a **Google Sheets** node named `Get Open Quotes - Sheets`.
   - Set resource to `sheet`, operation to `read`, and Sheet Name to `Quotes`.
   - Use expression `={{ $('CONFIG - Quotes').first().json.sheet_id }}` for the Document ID.
   - Configure **Google Sheets OAuth2 API** credentials.
   - Connect `CONFIG - Quotes` to this node.
4. **Filter Due Quotes:**
   - Add a **Code** node named `Filter Due - Code`.
   - Paste the JavaScript logic evaluating open status, opt-out flags, touch counts (< 3), and schedule intervals (`[2, 7, 14]`).
   - Connect `Get Open Quotes - Sheets` to this node.
5. **Verify Quote Status via HTTP:**
   - Add an **HTTP Request** node named `Check Quote Status - HTTP`.
   - Set method to `GET` and URL to `https://YOUR_CRM/quotes/{{ $json.quote_id }}`. Set request timeout to 15,000ms.
   - Connect `Filter Due - Code` to this node.
6. **Evaluate Open Status:**
   - Add an **If** node named `IF - Still Open?`.
   - Set condition to check if `{{ $json.status }}` equals `open` with loose validation.
   - Connect `Check Quote Status - HTTP` to this node.
   - **Alternative Branch (False):** Add a **Google Sheets** node named `Mark Closed - Sheets` (resource: `sheet`, operation: `append`, Sheet Name: `ClosedLog`, Document ID expression linked to config sheet ID). Map `date` and `quote_id` fields. Connect the False output of `IF - Still Open?` here.
7. **Configure AI Generation:**
   - Add an **AI Agent** node named `Build Follow-Up - AI Agent`.
   - Set prompt text referencing customer name, amount, and touch level. Configure the system message to enforce business voice and touch behaviors.
   - Add a **OpenAI Chat Model** node named `Chat Model - FollowUp` configured with model `gpt-4o-mini` and OpenAI API credentials. Link its output to the AI Agent's language model input.
   - Connect the True output of `IF - Still Open?` to the AI Agent.
8. **Route Channels:**
   - Add an **If** node named `IF - Touch 0 (Email)?`.
   - Set condition to check if touch level equals `0`.
   - Connect `Build Follow-Up - AI Agent` to this node.
9. **Dispatch Messages:**
   - **Email Channel:** Add a **Gmail** node named `Send Email - Gmail` (resource: `message`, operation: `send`, plain text). Set recipient to `={{ $('Filter Due - Code').item.json.email }}` and message to `={{ $('Build Follow-Up - AI Agent').item.json.output }}`. Configure Gmail OAuth2 credentials. Connect the True output of `IF - Touch 0 (Email)?` here.
   - **SMS Channel:** Add a **Twilio** node named `Send SMS - Twilio` (resource: `sms`, operation: `send`). Set recipient to `={{ $('Filter Due - Code').item.json.phone }}`, sender to config Twilio number, and message to AI output. Configure Twilio credentials. Connect the False output of `IF - Touch 0 (Email)?` here.
10. **Log Touches:**
    - Add a **Google Sheets** node named `Log Touch - Sheets` (resource: `sheet`, operation: `append`, Sheet Name: `Touches`).
    - Map columns: `date` (`$now.toISO()`), `touch`, `channel` (evaluated via ternary/expression checking email vs sms execution path), and `quote_id`. Configure Google Sheets OAuth2 credentials.
    - Connect both `Send Email - Gmail` and `Send SMS - Twilio` outputs to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheet Setup Requirements | Ensure your Google Sheet contains three separate tabs named exactly: `Quotes`, `Touches`, and `ClosedLog`. |
| Spreadsheet Schema Expectations | The `Quotes` tab requires columns including `quote_id`, `name`, `phone`, `email`, `amount`, `sent_date`, `touch_count`, `status`, and `opted_out`. |
| Credential Provisioning | All credential slots act as placeholders and must be populated with active accounts for Google Sheets, OpenAI, Gmail, and Twilio prior to activation. |
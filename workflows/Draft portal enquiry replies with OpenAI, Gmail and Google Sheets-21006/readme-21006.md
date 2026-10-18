Draft portal enquiry replies with OpenAI, Gmail and Google Sheets

https://n8nworkflows.xyz/workflows/draft-portal-enquiry-replies-with-openai--gmail-and-google-sheets-21006


# Draft portal enquiry replies with OpenAI, Gmail and Google Sheets

### 1. Workflow Overview

This workflow is designed for UK estate and letting agents (as well as similar service businesses) to automate the initial handling of property portal or website form enquiries. Its primary purpose is to parse incoming lead emails from platforms like Rightmove, Zoopla, or OnTheMarket, extract structured lead details using OpenAI, perform strict validation checks, log the information into Google Sheets, and create a Gmail draft reply for manual review before any communication is sent to the client.

The workflow is categorized into the following logical blocks:
- **1.1 Input Reception & Configuration:** Watches for incoming emails with a specific Gmail label and establishes global agency settings and AI behavioral instructions.
- **1.2 AI Processing & Sanitization:** Prepares the email content, strips HTML markup, and sends a structured prompt to OpenAI utilizing JSON schema enforcement.
- **1.3 Validation & Logging:** Evaluates the AI output against strict security and data validation rules, logs the lead details to Google Sheets, and determines the appropriate routing path.
- **1.4 Draft Generation:** Evaluates whether a draft requires creation and generates the reviewable Gmail draft with standardized branding, signatures, and compliant links.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** This block triggers the workflow upon receiving a relevant portal enquiry email and injects essential configuration parameters and system prompts that dictate agency identity and AI operational boundaries.
- **Nodes Involved:** 
  - `Portal enquiry (Gmail)`
  - `Settings`
  - `AI instructions`

- **Node Details:**
  - **Portal enquiry (Gmail)**
    - *Type and Technical Role:* `n8n-nodes-base.gmailTrigger` (Trigger Node)
    - *Configuration Choices:* Polls every minute for emails matching the filter query `label:portal-leads`, checking both read and unread statuses.
    - *Key Expressions or Variables:* Filter query: `label:portal-leads`
    - *Input and Output Connections:* Input: None (Trigger); Output: `Settings`
    - *Version-specific Requirements:* Version 1.4
    - *Edge Cases or Potential Failure Types:* Gmail API authentication expiry or polling delays.
  - **Settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation Node)
    - *Configuration Choices:* Manually assigns global string variables for agency metadata, including agency name, agent name, booking links, privacy notice URLs, HTML signatures, auto-reply notes, market type, blocked domains, and data retention duration.
    - *Key Expressions or Variables:* Custom variables: `AGENCY_NAME`, `AGENT_NAME`, `BOOKING_LINK`, `PRIVACY_NOTICE_URL`, `SIGNATURE_HTML`, `AUTO_REPLY_NOTE`, `MARKET`, `BLOCKED_DOMAINS`, `RETENTION_MONTHS`.
    - *Input and Output Connections:* Input: `Portal enquiry (Gmail)`; Output: `AI instructions`
    - *Version-specific Requirements:* Version 3.4
    - *Edge Cases or Potential Failure Types:* Missing or malformed URLs in configuration values.
  - **AI instructions**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation Node)
    - *Configuration Choices:* Defines the core LLM system prompt (`SYSTEM_PROMPT`) specifying output rules, conversational tone, classification rubrics, and strict data separation boundaries, along with the designated model identifier (`AI_MODEL`).
    - *Key Expressions or Variables:* Custom variables: `AI_MODEL`, `SYSTEM_PROMPT`.
    - *Input and Output Connections:* Input: `Settings`; Output: `Prepare enquiry`
    - *Version-specific Requirements:* Version 3.4
    - *Edge Cases or Potential Failure Types:* Overly complex prompts leading to unexpected JSON schema drift if parameters are altered incorrectly.

#### 2.2 AI Processing & Sanitization
- **Overview:** This block cleans and sanitizes the incoming email body, constructs a structured payload combining agency settings and enquiry data, and executes an API call to OpenAI with strict JSON schema constraints.
- **Nodes Involved:**
  - `Prepare enquiry`
  - `AI: draft reply (OpenAI)`

- **Node Details:**
  - **Prepare enquiry**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Execution Node)
    - *Configuration Choices:* Executes custom JavaScript for each item. Strips HTML tags, decodes HTML entities, truncates text to 8,000 characters, builds a structured user prompt payload, and defines an OpenAI JSON schema enforcing strict output properties.
    - *Key Expressions or Variables:* `=$json`, `$('Portal enquiry (Gmail)').item.json`
    - *Input and Output Connections:* Input: `AI instructions`; Output: `AI: draft reply (OpenAI)`
    - *Version-specific Requirements:* Version 2
    - *Edge Cases or Potential Failure Types:* Malformed HTML structures causing regex stripping issues or exceeding token payload bounds.
  - **AI: draft reply (OpenAI)**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration Node)
    - *Configuration Choices:* Sends a POST request to OpenAI's chat completions endpoint with a 120-second timeout, configured with automatic retries (up to 3 tries with a 5000ms delay) using predefined OpenAI credentials.
    - *Key Expressions or Variables:* `={{ JSON.stringify($json.requestBody) }}`
    - *Input and Output Connections:* Input: `Prepare enquiry`; Output: `Check AI output`
    - *Version-specific Requirements:* Version 4.5
    - *Edge Cases or Potential Failure Types:* Rate limits (HTTP 429), API authentication failures, upstream OpenAI timeouts, or malformed JSON schema responses.

#### 2.3 Validation & Logging
- **Overview:** This block processes the AI's response, enforces rigorous security and safety compliance checks, maps the validated data to a standardized schema, logs the lead in Google Sheets, and assesses workflow routing requirements.
- **Nodes Involved:**
  - `Check AI output`
  - `Log lead (Google Sheets)`
  - `Draft for review?`

- **Node Details:**
  - **Check AI output**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Code Execution Node)
    - *Configuration Choices:* Executes custom JavaScript to parse the LLM output, validate enquirer email authenticity against blocked domains and header/body presence, check for unauthorized links or contact details in the AI response, calculate data retention deletion dates, and assemble structured logging variables.
    - *Key Expressions or Variables:* `$('AI instructions').item.json`, `$('Prepare enquiry').item.json`, `$json`
    - *Input and Output Connections:* Input: `AI: draft reply (OpenAI)`; Output: `Log lead (Google Sheets)`
    - *Version-specific Requirements:* Version 2
    - *Edge Cases or Potential Failure Types:* Unparseable AI JSON strings or mismatched email headers resulting in flagged exceptions.
  - **Log lead (Google Sheets)**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet Integration Node)
    - *Configuration Choices:* Appends a new row to the specified Google Sheet tab (`Leads`) using automatic data mapping with raw cell formatting and extra data handling set to ignore.
    - *Key Expressions or Variables:* Spreadsheet URL reference and mapped item properties from the preceding code node.
    - *Input and Output Connections:* Input: `Check AI output`; Output: `Draft for review?`
    - *Version-specific Requirements:* Version 4.7
    - *Edge Cases or Potential Failure Types:* Google Sheets API quota limits, missing spreadsheet columns, or permission/authorization errors.
  - **Draft for review?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching Node)
    - *Configuration Choices:* Evaluates whether the internal action flag equals `draft`.
    - *Key Expressions or Variables:* `={{ $('Check AI output').item.json._action }}` evaluated against `draft`.
    - *Input and Output Connections:* Input: `Log lead (Google Sheets)`; Output: True branch connects to `Save draft for review`.
    - *Version-specific Requirements:* Version 2.3
    - *Edge Cases or Potential Failure Types:* Loose type validation mismatches if workflow state variables change unexpectedly.

#### 2.4 Draft Generation
- **Overview:** This block creates a reviewable Gmail draft populated with verified template content, secure styling, agency signatures, and mandatory compliance notices.
- **Nodes Involved:**
  - `Save draft for review`

- **Node Details:**
  - **Save draft for review**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Integration Node)
    - *Configuration Choices:* Creates a Gmail draft resource using HTML email formatting, setting the recipient address, subject line, and HTML body derived from validated output parameters.
    - *Key Expressions or Variables:* 
      - Send To: `={{ $('Check AI output').item.json._to }}`
      - Subject: `={{ $('Check AI output').item.json._subject }}`
      - Message: `={{ $('Check AI output').item.json._html }}`
    - *Input and Output Connections:* Input: `Draft for review? (True branch)`; Output: None (Terminal Node)
    - *Version-specific Requirements:* Version 2.2
    - *Edge Cases or Potential Failure Types:* Gmail API authorization expiration, missing recipient addresses, or exceeding Gmail draft payload limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Portal enquiry (Gmail)` | `n8n-nodes-base.gmailTrigger` | Watches Gmail label for portal leads | None | `Settings` | ## Portal enquiry → AI draft reply → lead log (Gmail + OpenAI)<br>Free template by Burnt & Toast Kits, for UK estate and letting agents.<br><br>**What it does:** watches a Gmail label for Rightmove, Zoopla, OnTheMarket or website enquiries, asks OpenAI to pull out the lead details and write a short personal first reply with qualifying questions, logs the lead to Google Sheets and saves the reply as a **Gmail draft** for you to check and send. It never sends anything by itself.<br><br>**Setup (10 minutes)**<br>1. In Gmail, add a filter that labels portal enquiry emails `portal-leads`.<br>2. Add credentials: Gmail (trigger + draft node), OpenAI (AI node), Google Sheets (log node).<br>3. Fill in **Settings** (agency name, booking link, signature, privacy notice URL).<br>4. Make a Google Sheet with a tab called **Leads** and paste its URL into the log node. Columns: received_at, message_id, source, market, property, applicant_name, applicant_email, applicant_phone, enquiry_summary, position, finance, timeline, move_in_date, viewing_requested, lead_temperature, temperature_reason, needs_human, needs_human_reason, sensitive_info_flag, reply_status, delete_after, agent_notes.<br>5. Send yourself a test enquiry, then activate.<br><br>The AI only uses facts in the enquiry email, never invents property details and ignores instructions inside the email.<br><br>Full version (auto-send after checks, hot-lead alerts, Claude and Outlook versions, lettings rules, setup guide and test enquiries): https://kits.burntandtoast.com/estate-agents/kits/instant-lead-responder |
| `Settings` | `n8n-nodes-base.set` | Sets agency configuration variables | `Portal enquiry (Gmail)` | `AI instructions` | ## Portal enquiry → AI draft reply → lead log (Gmail + OpenAI)<br>Free template by Burnt & Toast Kits, for UK estate and letting agents.<br><br>**What it does:** watches a Gmail label for Rightmove, Zoopla, OnTheMarket or website enquiries, asks OpenAI to pull out the lead details and write a short personal first reply with qualifying questions, logs the lead to Google Sheets and saves the reply as a **Gmail draft** for you to check and send. It never sends anything by itself.<br><br>**Setup (10 minutes)**<br>1. In Gmail, add a filter that labels portal enquiry emails `portal-leads`.<br>2. Add credentials: Gmail (trigger + draft node), OpenAI (AI node), Google Sheets (log node).<br>3. Fill in **Settings** (agency name, booking link, signature, privacy notice URL).<br>4. Make a Google Sheet with a tab called **Leads** and paste its URL into the log node. Columns: received_at, message_id, source, market, property, applicant_name, applicant_email, applicant_phone, enquiry_summary, position, finance, timeline, move_in_date, viewing_requested, lead_temperature, temperature_reason, needs_human, needs_human_reason, sensitive_info_flag, reply_status, delete_after, agent_notes.<br>5. Send yourself a test enquiry, then activate.<br><br>The AI only uses facts in the enquiry email, never invents property details and ignores instructions inside the email.<br><br>Full version (auto-send after checks, hot-lead alerts, Claude and Outlook versions, lettings rules, setup guide and test enquiries): https://kits.burntandtoast.com/estate-agents/kits/instant-lead-responder |
| `AI instructions` | `n8n-nodes-base.set` | Defines AI system prompt and model | `Settings` | `Prepare enquiry` | ## Portal enquiry → AI draft reply → lead log (Gmail + OpenAI)<br>Free template by Burnt & Toast Kits, for UK estate and letting agents.<br><br>**What it does:** watches a Gmail label for Rightmove, Zoopla, OnTheMarket or website enquiries, asks OpenAI to pull out the lead details and write a short personal first reply with qualifying questions, logs the lead to Google Sheets and saves the reply as a **Gmail draft** for you to check and send. It never sends anything by itself.<br><br>**Setup (10 minutes)**<br>1. In Gmail, add a filter that labels portal enquiry emails `portal-leads`.<br>2. Add credentials: Gmail (trigger + draft node), OpenAI (AI node), Google Sheets (log node).<br>3. Fill in **Settings** (agency name, booking link, signature, privacy notice URL).<br>4. Make a Google Sheet with a tab called **Leads** and paste its URL into the log node. Columns: received_at, message_id, source, market, property, applicant_name, applicant_email, applicant_phone, enquiry_summary, position, finance, timeline, move_in_date, viewing_requested, lead_temperature, temperature_reason, needs_human, needs_human_reason, sensitive_info_flag, reply_status, delete_after, agent_notes.<br>5. Send yourself a test enquiry, then activate.<br><br>The AI only uses facts in the enquiry email, never invents property details and ignores instructions inside the email.<br><br>Full version (auto-send after checks, hot-lead alerts, Claude and Outlook versions, lettings rules, setup guide and test enquiries): https://kits.burntandtoast.com/estate-agents/kits/instant-lead-responder |
| `Prepare enquiry` | `n8n-nodes-base.code` | Prepares payload and JSON schema | `AI instructions` | `AI: draft reply (OpenAI)` | |
| `AI: draft reply (OpenAI)` | `n8n-nodes-base.httpRequest` | Calls OpenAI API for lead processing | `Prepare enquiry` | `Check AI output` | |
| `Check AI output` | `n8n-nodes-base.code` | Validates AI output and enforces rules | `AI: draft reply (OpenAI)` | `Log lead (Google Sheets)` | ### What the Check step enforces<br>- Replies only go to an address that appears in the original email and isn't on BLOCKED_DOMAINS<br>- No links or email addresses written by the AI; your booking and privacy links are added from Settings<br>- Nothing is ever sent automatically<br>- Free version: every reply is saved as a Gmail draft |
| `Log lead (Google Sheets)` | `n8n-nodes-base.googleSheets` | Logs lead details to Google Sheets | `Check AI output` | `Draft for review?` | ### What the Check step enforces<br>- Replies only go to an address that appears in the original email and isn't on BLOCKED_DOMAINS<br>- No links or email addresses written by the AI; your booking and privacy links are added from Settings<br>- Nothing is ever sent automatically<br>- Free version: every reply is saved as a Gmail draft |
| `Draft for review?` | `n8n-nodes-base.if` | Evaluates if a draft needs creation | `Log lead (Google Sheets)` | `Save draft for review` | ### What the Check step enforces<br>- Replies only go to an address that appears in the original email and isn't on BLOCKED_DOMAINS<br>- No links or email addresses written by the AI; your booking and privacy links are added from Settings<br>- Nothing is ever sent automatically<br>- Free version: every reply is saved as a Gmail draft |
| `Save draft for review` | `n8n-nodes-base.gmail` | Creates a Gmail draft for manual review | `Draft for review?` | None | ### What the Check step enforces<br>- Replies only go to an address that appears in the original email and isn't on BLOCKED_DOMAINS<br>- No links or email addresses written by the AI; your booking and privacy links are added from Settings<br>- Nothing is ever sent automatically<br>- Free version: every reply is saved as a Gmail draft |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Trigger:**
   - Add a **Gmail Trigger** node (`n8n-nodes-base.gmailTrigger`).
   - Set parameters: `Simple` = `false`, Filter Query (`q`) = `label:portal-leads`, Read Status = `both`, Poll Times = Every minute.
   - Configure valid Gmail OAuth2 credentials.

2. **Configure Agency Settings:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Settings`.
   - Connect `Portal enquiry (Gmail)` to `Settings`.
   - Configure manual string assignments for:
     - `AGENCY_NAME` (e.g., `Your Agency Ltd`)
     - `AGENT_NAME` (e.g., `The Sales Team`)
     - `BOOKING_LINK` (e.g., `https://calendly.com/your-agency/viewing`)
     - `PRIVACY_NOTICE_URL` (e.g., `https://www.youragency.co.uk/privacy`)
     - `SIGNATURE_HTML` (e.g., `<p>Kind regards,<br>The Sales Team<br>Your Agency Ltd · 01234 567890</p>`)
     - `AUTO_REPLY_NOTE` (e.g., `This is our first reply to your enquiry. A member of our team will follow up personally.`)
     - `MARKET` (`auto`)
     - `BLOCKED_DOMAINS` (e.g., `youragency.co.uk, rightmove.co.uk, zoopla.co.uk, onthemarket.com`)
     - `RETENTION_MONTHS` (`12`)

3. **Set Up AI Instructions:**
   - Add a second **Set** node (`n8n-nodes-base.set`) named `AI instructions`.
   - Connect `Settings` to `AI instructions`.
   - Define string variables:
     - `AI_MODEL`: `gpt-6-luna` (or your preferred OpenAI model identifier)
     - `SYSTEM_PROMPT`: Paste the complete system instructions detailing the assistant role, market classification, reply constraints, JSON schema properties, and lead temperature rubric as provided in the source configuration.

4. **Prepare the Enquiry Payload:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Prepare enquiry`.
   - Connect `AI instructions` to `Prepare enquiry`.
   - Set execution mode to `Run Once for Each Item` and populate the JavaScript code to strip HTML tags, extract email headers, format the user message payload, and attach the strict OpenAI JSON schema.

5. **Execute the OpenAI Request:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `AI: draft reply (OpenAI)`.
   - Connect `Prepare enquiry` to `AI: draft reply (OpenAI)`.
   - Configure parameters:
     - Method: `POST`
     - URL: `https://api.openai.com/v1/chat/completions`
     - Authentication: Predefined credential type -> OpenAI API (`openAiApi`)
     - Specify Body: `JSON`
     - JSON Body: `={{ JSON.stringify($json.requestBody) }}`
     - Options: Timeout = `120000` ms, Retry on Fail = `true`, Max Tries = `3`, Wait Between Tries = `5000` ms.

6. **Validate and Process Output:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Check AI output`.
   - Connect `AI: draft reply (OpenAI)` to `Check AI output`.
   - Set execution mode to `Run Once for Each Item` and populate the JavaScript code that parses the AI response, validates enquirer email presence and domain safety, checks for unauthorized links/contacts, calculates retention deletion dates, and outputs standardized mapping fields.

7. **Log Lead to Google Sheets:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Log lead (Google Sheets)`.
   - Connect `Check AI output` to `Log lead (Google Sheets)`.
   - Configure parameters:
     - Operation: `Append`
     - Document ID: Paste your Google Spreadsheet URL (`https://docs.google.com/spreadsheets/d/PASTE-YOUR-SHEET-ID/edit`)
     - Sheet Name: `Leads`
     - Mapping Mode: Auto-map input data
     - Options: Cell Format = `RAW`, Handling Extra Data = `Ignore It`.
   - Configure valid Google Sheets OAuth2 credentials. Ensure the sheet contains columns matching: `received_at`, `message_id`, `source`, `market`, `property`, `applicant_name`, `applicant_email`, `applicant_phone`, `enquiry_summary`, `position`, `finance`, `timeline`, `move_in_date`, `viewing_requested`, `lead_temperature`, `temperature_reason`, `needs_human`, `needs_human_reason`, `sensitive_info_flag`, `reply_status`, `delete_after`, `agent_notes`.

8. **Evaluate Routing Condition:**
   - Add an **If** node (`n8n-nodes-base.if`) named `Draft for review?`.
   - Connect `Log lead (Google Sheets)` to `Draft for review?`.
   - Configure conditions:
     - Left Value: `={{ $('Check AI output').item.json._action }}`
     - Operator: `Equals`
     - Right Value: `draft`

9. **Create Gmail Review Draft:**
   - Add a **Gmail** node (`n8n-nodes-base.gmail`) named `Save draft for review`.
   - Connect the True output branch of `Draft for review?` to `Save draft for review`.
   - Configure parameters:
     - Resource: `Draft`
     - Operation: `Create`
     - Email Type: `HTML`
     - Send To: `={{ $('Check AI output').item.json._to }}`
     - Subject: `={{ $('Check AI output').item.json._subject }}`
     - Message: `={{ $('Check AI output').item.json._html }}`
   - Configure valid Gmail OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Created by Burnt & Toast Kits | Project creator and template publisher |
| Instant Lead Responder (Full Version) | [Instant Lead Responder Kit](https://kits.burntandtoast.com/estate-agents/kits/instant-lead-responder) |
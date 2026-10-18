Send cold outreach from lead files and classify Gmail replies with OpenRouter

https://n8nworkflows.xyz/workflows/send-cold-outreach-from-lead-files-and-classify-gmail-replies-with-openrouter-20944


# Send cold outreach from lead files and classify Gmail replies with OpenRouter

### 1. Workflow Overview

This workflow automates a complete B2B cold email outreach lifecycle using n8n Data Tables, Gmail, and an OpenRouter-powered Large Language Model. Its primary purpose is to ingest lead lists via an uploaded file, send scheduled or manually triggered first-touch cold emails with personalization, monitor incoming inbox replies, aggregate entire conversation threads, and use AI to evaluate whether a prospect has shown genuine interest for human sales handoff.

The workflow logic is categorized into four distinct functional blocks:

- **1.1 Lead Ingestion & Storage:** Receives uploaded lead files via an online form, parses the file contents, and upserts the extracted leads into an n8n Data Table to prevent duplicate records.
- **1.2 Outreach Dispatch & Tracking:** Initiates outbound campaigns either automatically on a daily schedule or manually, merges lead data with centralized email copy templates, transmits emails through Gmail, logs transmission records, and marks contacted leads to prevent duplicate outreach.
- **1.3 Reply Detection & Thread Retrieval:** Listens for incoming Gmail replies, matches sender addresses to known leads, records incoming messages, and fetches the entire chronological thread history while filtering out outbound system messages.
- **1.4 AI Classification & Handoff:** Passes filtered conversation messages to an OpenRouter LLM agent constrained by a structured output schema, evaluates whether the lead is interested, and flags qualifying leads for human handoff.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Ingestion & Storage
- **Overview:** This block handles incoming contact files from external users or team members, extracts tabular rows, and maintains a centralized leads Data Table using upsert logic to ensure data cleanliness.
- **Nodes Involved:** 
  - `When a lead file is uploaded`
  - `read the lead file`
  - `store the leads`
- **Node Details:**
  - **`When a lead file is uploaded`**
    - *Type and technical role:* Form Trigger (`n8n-nodes-base.formTrigger`, v2.6). Acts as the ingestion entry point for external file uploads.
    - *Configuration choices:* Configured with a form title ("Lead Imports") and a single file upload field (`Import File`).
    - *Key expressions or variables:* None.
    - *Input and output connections:* Input: Webhook/HTTP Trigger; Output: `read the lead file`.
    - *Version-specific requirements:* v2.6. Requires public webhook execution access if accessed externally.
    - *Edge cases or potential failure types:* Uploads exceeding size limits or unsupported file formats (e.g., non-CSV/XLSX) will cause parsing errors.
  - **`read the lead file`**
    - *Type and technical role:* Extract From File (`n8n-nodes-base.extractFromFile`, v1.1). Parses binary file data into structured JSON items.
    - *Configuration choices:* Uses binary property `Import_File`.
    - *Key expressions or variables:* None.
    - *Input and output connections:* Input: `When a lead file is uploaded`; Output: `store the leads`.
    - *Version-specific requirements:* v1.1.
    - *Edge cases or potential failure types:* Malformed spreadsheets or missing columns (`name`, `email`) will cause downstream mapping errors.
  - **`store the leads`**
    - *Type and technical role:* Data Table (`n8n-nodes-base.dataTable`, v1.1). Upserts lead records.
    - *Configuration choices:* Operation set to `upsert`, matching existing records on the `email` column. Sets initial status column value to `new_leads`. Maps `name` and `email` from incoming fields.
    - *Key expressions:*. `={{ $json.name }}`, `={{ $json.email }}`.
    - *Input and output connections:* Input: `read the lead file`; Output: None (Terminal node for this branch).
    - *Edge cases or potential failure types:* Database connection drops or schema mismatches (missing table columns) will cause execution failure.

#### 2.2 Outreach Dispatch & Tracking
- **Overview:** This block manages the execution of cold email campaigns, pulling uncontacted leads, merging them with centralized messaging copy, dispatching emails via Gmail, logging dispatch metadata, and updating campaign statuses.
- **Nodes Involved:**
  - `Every morning`
  - `Run the send pass`
  - `config: the offer`
  - `read leads not yet contacted`
  - `[cred] Gmail - send the outreach email`
  - `record what was sent`
  - `mark the lead contacted`
- **Node Details:**
  - **`Every morning`**
    - *Type and technical role:* Schedule Trigger (`n8n-nodes-base.scheduleTrigger`, v1.3). Automatically triggers the campaign on a recurring schedule.
    - *Configuration choices:* Interval rule set to every hour (or daily depending on cron/interval fine-tuning).
    - *Key expressions:* None.
    - *Input and output connections:* Input: None; Output: `config: the offer`.
  - **`Run the send pass`**
    - *Type and technical role:* Manual Trigger (`n8n-nodes-base.manualTrigger`, v1). Allows manual testing and on-demand campaign execution.
    - *Configuration choices:* Default manual execution setup.
    - *Input and output connections:* Input: None; Output: `config: the offer`.
  - **`config: the offer`**
    - *Type and technical role:* Set (`n8n-nodes-base.set`, v3.4). Central configuration repository for email copy and titles.
    - *Configuration choices:* Raw JSON mode outputting `email_title` and `email_content`.
    - *Key expressions:* None.
    - *Input and output connections:* Input: `Every morning`, `Run the send pass`; Output: `read leads not yet contacted`.
  - **`read leads not yet contacted`**
    - *Type and technical role:* Data Table (`n8n-nodes-base.dataTable`, v1.1). Queries leads awaiting outreach.
    - *Configuration choices:* Operation set to `get`, filtering records where `status` equals `new_leads`.
    - *Key expressions:* None.
    - *Input and output connections:* Input: `config: the offer`; Output: `[cred] Gmail - send the outreach email`.
  - **`[cred] Gmail - send the outreach email`**
    - *Type and technical role:* Gmail (`n8n-nodes-base.gmail`, v2.2). Sends outbound emails via the connected Gmail account.
    - *Configuration choices:* Email type set to text, recipient mapped to lead email, subject and body pulled from the configuration node with dynamic placeholder replacement.
    - *Key expressions:* 
      - Send To: `={{ $json.email }}`
      - Subject: `={{ $('config: the offer').item.json.email_title }}`
      - Message: `={{ $('config: the offer').item.json.email_content.replaceAll("[name]",$json.name ) }}`
    - *Input and output connections:* Input: `read leads not yet contacted`; Output: `record what was sent`.
    - *Credentials:* Uses `gmailOAuth2` credential ("Gmail account").
    - *Edge cases or potential failure types:* Invalid recipient email addresses, rate limiting, or revoked OAuth tokens will halt execution.
  - **`record what was sent`**
    - *Type and technical role:* Data Table (`n8n-nodes-base.dataTable`, v1.1). Logs sent communication details into the inbox data table.
    - *Configuration choices:* Operation set to `insert`/`upsert` mapping `email`, `title`, `content`, `inbox_type` ("sent"), `external_id`, `success_sent`, `thread_external_id`, and `history_external_id`.
    - *Key expressions:*
      - Email: `={{ $('read leads not yet contacted').item.json.email }}`
      - External ID: `={{ $json.id }}`
      - Success Sent: `={{ $json.labelIds[0] === 'SENT' }}`
      - Thread ID: `={{ $json.threadId }}`
    - *Input and output connections:* Input: `[cred] Gmail - send the outreach email`; Output: `mark the lead contacted`.
  - **`mark the lead contacted`**
    - *Type and technical role:* Data Table (`n8n-nodes-base.dataTable`, v1.1). Updates lead statuses post-dispatch.
    - *Configuration choices:* Operation set to `update`, updating lead status to `first_offer` based on the lead record ID.
    - *Key expressions:* Filter key value `={{ $('read leads not yet contacted').item.json.id }}`.
    - *Input and output connections:* Input: `record what was sent`; Output: None (Terminal node).

####  2.3 Reply Detection & Thread Retrieval
- **Overview:** This block monitors inbound email traffic, links replies to existing leads by parsing sender headers, records received messages, and aggregates the complete conversation thread history.
- **Nodes Involved:**
  - `When a reply arrives`
  - `match the reply to a lead`
  - `store the reply`
  - `read the whole thread`
  - `keep only the prospect's messages`
- **Node Details:**
  - **`When a reply arrives`**
    - *Type and technical role:* Gmail Trigger (`n8n-nodes-base.gmailTrigger`, v1.4). Polls the Gmail inbox for new incoming messages.
    - *Configuration choices:* Polling frequency set to every minute.
    - *Input and output connections:* Input: Polling trigger; Output: `match the reply to a lead`.
    - *Credentials:* Uses `gmailOAuth2` credential ("Gmail account").
  - **`match the reply to a lead`**
    - *Type and technical role:* Data Table (`n8n-nodes-base.dataTable`, v1.1). Validates if the sender exists in the leads table.
    - *Configuration choices:* Operation set to `get`, limiting results to 1, filtering by email extracted from the message header.
    - *Key expressions:* Filter `email` matching `={{ $json.From.match(/<([^>]+)>/)?.[1] }}`.
    - *Input and output connections:* Input: `When a reply arrives`; Output: `store the reply`.
  - **`store the reply`**
    - *Type and technical role:* Data Table (`n8n-nodes-base.dataTable`, v1.1). Logs the inbound reply into the inbox data table.
    - *Configuration choices:* Operation set to insert/upsert, recording metadata, snippet content (cleaning quoted reply history), thread IDs, and history IDs.
    - *Key expressions:*
      - Content: `={{ $('When a reply arrives').item.json.snippet.replace(/\s*On\s.+?wrote:.*$/s, "").trim() }}`
      - External ID: `={{ $('When a reply arrives').item.json.id }}`
    - *Input and output connections:* Input: `match the reply to a lead`; Output: `read the whole thread`.
  - **`read the whole thread`**
    - *Type and technical role:* Data Table (`n8n-nodes-base.dataTable`, v1.1). Retrieves historical messages associated with the lead and thread ID.
    - *Configuration choices:* Operation set to `get` (limit 20), filtering where email and thread ID match, filtering for successfully sent/received items, ordered by history external ID.
    - *Key expressions:*
      - Email filter: `={{ $json.email }}`
      - Thread ID filter: `={{ $json.thread_external_id }}`
    - *Input and output connections:* Input: `store the reply`; Output: `keep only the prospect's messages`.
  - **`keep only the prospect's messages`**
    - *Type and technical role:* Code (`n8n-nodes-base.code`, v2). Formats and filters thread arrays to isolate prospect interactions.
    - *Configuration choices:* Custom JavaScript processing input array items in reverse chronological order, tagging sender type as `'User'` or `'System'`.
    - *Key expressions/Code logic:* Iterates over `$input.all().reverse()` to map sender roles.
    - *Input and output connections:* Input: `read the whole thread`; Output: `classify the reply`.

#### 2.4 AI Classification & Handoff
- **Overview:** Evaluates conversation threads using an OpenRouter LLM agent constrained by a structured JSON output schema, determining prospect interest levels and triggering human sales handoffs when criteria are met.
- **Nodes Involved:**
  - `classify the reply`
  - `OpenRouter model`
  - `reply classification schema`
  - `interested?`
  - `flag the lead for handoff`
- **Node Details:**
  - **`classify the reply`**
    - *Type and technical role:* Advanced AI Agent (`@n8n/n8n-nodes-langchain.agent`, v3.1). Orchestrates prompt execution with connected language models and output parsers.
    - *Configuration choices:* Prompt configured to evaluate user messages for sales interest indicators based on provided JSON thread strings. Enforces structured output parsing.
    - *Key expressions:* Text prompt interpolates `={{ JSON.stringify($json.messages) }}`.
    - *Input and output connections:* Inputs: `keep only the prospect's messages`, `OpenRouter model` (AI Language Model), `reply classification schema` (AI Output Parser); Output: `interested?`.
  - **`OpenRouter model`**
    - *Type and technical role:* OpenRouter Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`, v1). Provides LLM capabilities via OpenRouter.
    - *Configuration choices:* Standard parameter defaults.
    - *Input and output connections:* Output connects to `classify the reply` (`ai_languageModel`).
    - *Credentials:* Uses `openRouterApi` credential ("OpenRouter Manexus").
  - **`reply classification schema`**
    - *Type and technical role:* Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`, v1.3). Enforces strict JSON output schemas on the AI agent response.
    - *Configuration choices:* JSON Schema example defining `alreadyInterested` as a boolean (`{ "alreadyInterested": true }`).
    - *Input and output connections:* Output connects to `classify the reply` (`ai_outputParser`).
  - **`interested?`**
    - *Type and technical role:* If (`n8n-nodes-base.if`, v2.3). Evaluates the boolean result from the AI classification.
    - *Configuration choices:* Condition checks if `{{ $json.output.alreadyInterested }}` equals `true`.
    - *Input and output connections:* Input: `classify the reply`; Output (True branch): `flag the lead for handoff`.
  - **`flag the lead for handoff`**
    - *Type and technical role:* Data Table (`n8n-nodes-base.dataTable`, v1.1). Updates lead record status upon confirmed interest.
    - *Configuration choices:* Operation set to `update`, setting `status` to `handoff` and destination to `teams`.
    - *Key expressions:* Filter key value matches `={{ $('match the reply to a lead').item.json.id }}`.
    - *Input and output connections:* Input: `interested?` (True branch); Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `When a lead file is uploaded` | `n8n-nodes-base.formTrigger` | Ingests lead spreadsheet uploads. | Webhook | `read the lead file` | **1. Import the leads**<br>Upload a CSV or spreadsheet through the form. **`read the lead file`** parses it and **`store the leads`** upserts each row.<br><br>Upsert, not insert: re-uploading a corrected file updates those people instead of creating a second copy of your list. |
| `read the lead file` | `n8n-nodes-base.extractFromFile` | Parses binary lead files into JSON items. | `When a lead file is uploaded` | `store the leads` | **1. Import the leads**<br>Upload a CSV or spreadsheet through the form. **`read the lead file`** parses it and **`store the leads`** upserts each row.<br><br>Upsert, not insert: re-uploading a corrected file updates those people instead of creating a second copy of your list. |
| `store the leads` | `n8n-nodes-base.dataTable` | Upserts parsed lead rows into the Data Table. | `read the lead file` | None | **1. Import the leads**<br>Upload a CSV or spreadsheet through the form. **`read the lead file`** parses it and **`store the leads`** upserts each row.<br><br>Upsert, not insert: re-uploading a corrected file updates those people instead of creating a second copy of your list. |
| `Every morning` | `n8n-nodes-base.scheduleTrigger` | Triggers daily campaign dispatch. | None | `config: the offer` | |
| `Run the send pass` | `n8n-nodes-base.manualTrigger` | Manually triggers campaign dispatch. | None | `config: the offer` | |
| `config: the offer` | `n8n-nodes-base.set` | Centralizes email subject and body copy. | `Every morning`, `Run the send pass` | `read leads not yet contacted` | **2. Send the first touch**<br><br>**`config: the offer`** holds the offer, subject and body in one node - it is the only place you edit copy.<br><br>**`read leads not yet contacted`** is what makes the pass repeatable, and the row is marked **after** the send succeeds. Marking first would silently drop anyone the send failed for. |
| `read leads not yet contacted` | `n8n-nodes-base.dataTable` | Queries leads with `new_leads` status. | `config: the offer` | `[cred] Gmail - send the outreach email` | **2. Send the first touch**<br><br>**`config: the offer`** holds the offer, subject and body in one node - it is the only place you edit copy.<br><br>**`read leads not yet contacted`** is what makes the pass repeatable, and the row is marked **after** the send succeeds. Marking first would silently drop anyone the send failed for. |
| `[cred] Gmail - send the outreach email` | `n8n-nodes-base.gmail` | Dispatches cold emails via Gmail. | `read leads not yet contacted` | `record what was sent` | **2. Send the first touch**<br><br>**`config: the offer`** holds the offer, subject and body in one node - it is the only place you edit copy.<br><br>**`read leads not yet contacted`** is what makes the pass repeatable, and the row is marked **after** the send succeeds. Marking first would silently drop anyone the send failed for. |
| `record what was sent` | `n8n-nodes-base.dataTable` | Logs sent emails to the inbox Data Table. | `[cred] Gmail - send the outreach email` | `mark the lead contacted` | **2. Send the first touch**<br><br>**`config: the offer`** holds the offer, subject and body in one node - it is the only place you edit copy.<br><br>**`read leads not yet contacted`** is what makes the pass repeatable, and the row is marked **after** the send succeeds. Marking first would silently drop anyone the send failed for. |
| `mark the lead contacted` | `n8n-nodes-base.dataTable` | Updates lead status to `first_offer`. | `record what was sent` | None | **2. Send the first touch**<br><br>**`config: the offer`** holds the offer, subject and body in one node - it is the only place you edit copy.<br><br>**`read leads not yet contacted`** is what makes the pass repeatable, and the row is marked **after** the send succeeds. Marking first would silently drop anyone the send failed for. |
| `When a reply arrives` | `n8n-nodes-base.gmailTrigger` | Polls inbox for incoming prospect replies. | Polling Trigger | `match the reply to a lead` | **3. Read the thread, not the message**<br><br>A reply is matched back to a lead, stored, and then the **whole thread** is pulled.<br><br>**`keep only the prospect's messages`** drops your own sent copy before the model sees it. Leaving it in is how a reply classifier talks itself into scoring every thread as interested - it is reading your pitch back to itself. |
| `match the reply to a lead` | `n8n-nodes-base.dataTable` | Matches reply sender to lead records. | `When a reply arrives` | `store the reply` | **3. Read the thread, not the message**<br><br>A reply is matched back to a lead, stored, and then the **whole thread** is pulled.<br><br>**`keep only the prospect's messages`** drops your own sent copy before the model sees it. Leaving it in is how a reply classifier talks itself into scoring every thread as interested - it is reading your pitch back to itself. |
| `store the reply` | `n8n-nodes-base.dataTable` | Logs received replies in the inbox Data Table. | `match the reply to a lead` | `read the whole thread` | **3. Read the thread, not the message**<br><br>A reply is matched back to a lead, stored, and then the **whole thread** is pulled.<br><br>**`keep only the prospect's messages`** drops your own sent copy before the model sees it. Leaving it in is how a reply classifier talks itself into scoring every thread as interested - it is reading your pitch back to itself. |
| `read the whole thread` | `n8n-nodes-base.dataTable` | Retrieves chronological thread history. | `store the reply` | `keep only the prospect's messages` | **3. Read the thread, not the message**<br><br>A reply is matched back to a lead, stored, and then the **whole thread** is pulled.<br><br>**`keep only the prospect's messages`** drops your own sent copy before the model sees it. Leaving it in is how a reply classifier talks itself into scoring every thread as interested - it is reading your pitch back to itself. |
| `keep only the prospect's messages` | `n8n-nodes-base.code` | Filters out outbound messages from history. | `read the whole thread` | `classify the reply` | **3. Read the thread, not the message**<br><br>A reply is matched back to a lead, stored, and then the **whole thread** is pulled.<br><br>**`keep only the prospect's messages`** drops your own sent copy before the model sees it. Leaving it in is how a reply classifier talks itself into scoring every thread as interested - it is reading your pitch back to itself. |
| `classify the reply` | `@n8n/n8n-nodes-langchain.agent` | AI Agent analyzing conversation sentiment. | `keep only the prospect's messages`, `OpenRouter model`, `reply classification schema` | `interested?` | |
| `OpenRouter model` | `@n8n/n8n-nodes-langchain.lmChatOpenRouter` | Connects AI agent to OpenRouter LLM. | None | `classify the reply` | |
| `reply classification schema` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces JSON output structure (`alreadyInterested`). | None | `classify the reply` | |
| `interested?` | `n8n-nodes-base.if` | Evaluates if `alreadyInterested` is true. | `classify the reply` | `flag the lead for handoff` | |
| `flag the lead for handoff` | `n8n-nodes-base.dataTable` | Updates lead status to `handoff`. | `interested?` | None | **4. Typed classification, then a human**<br><br>**`classify the reply`** is pinned by **`reply classification schema`**, so the verdict is a typed field rather than a sentence somebody has to interpret.<br><br>An interested prospect is **flagged for handoff**, not auto-replied to. The first real conversation is the one worth a person. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create Data Tables:**
   - Create a Data Table named `email_outreach_automation_leads` with columns: `name` (string), `email` (string), `status` (string), and `handoff` (string).
   - Create a second Data Table named `email_outreach_automation_email_inboxes` with columns: `email` (string), `inbox_type` (string), `title` (string), `content` (string), `success_sent` (boolean), `external_id` (string), `thread_external_id` (string), `receive_process_status` (string), and `history_external_id` (string).

2. **Build Block 1 (Lead Ingestion):**
   - **Node 1:** Create `When a lead file is uploaded` (`n8n-nodes-base.formTrigger`, v2.6). Set form title to "Lead Imports" and add a file upload field labeled `Import File`.
   - **Node 2:** Create `read the lead file` (`n8n-nodes-base.extractFromFile`, v1.1). Set binary property name to `Import_File`. Connect `When a lead file is uploaded` to this node.
   - **Node 3:** Create `store the leads` (`n8n-nodes-base.dataTable`, v1.1). Set operation to `upsert`, match on column `email`, map `name` and `email` from incoming data, and set default `status` to `new_leads`. Connect `read the lead file` to this node.

3. **Build Block 2 (Outreach Dispatch):**
   - **Node 4:** Create `Every morning` (`n8n-nodes-base.scheduleTrigger`, v1.3). Set interval rule to hourly execution.
   - **Node 5:** Create `Run the send pass` (`n8n-nodes-base.manualTrigger`, v1).
   - **Node 6:** Create `config: the offer` (`n8n-nodes-base.set`, v3.4). Set mode to raw JSON and supply campaign title and body text containing `[name]` placeholders. Connect both `Every morning` and `Run the send pass` here.
   - **Node 7:** Create `read leads not yet contacted` (`n8n-nodes-base.dataTable`, v1.1). Set operation to `get` with filter condition `status` equals `new_leads`. Connect `config: the offer` to this node.
   - **Node 8:** Create `[cred] Gmail - send the outreach email` (`n8n-nodes-base.gmail`, v2.2). Configure recipient, subject, and message expressions. Configure and select your `gmailOAuth2` credentials. Connect `read leads not yet contacted` here.
   - **Node 9:** Create `record what was sent` (`n8n-nodes-base.dataTable`, v1.1). Set operation to insert/upsert logging metadata (`email`, `inbox_type: sent`, `success_sent`, `thread_external_id`, etc.). Connect `[cred] Gmail - send the outreach email` here.
   - **Node 10:** Create `mark the lead contacted` (`n8n-nodes-base.dataTable`, v1.1). Set operation to `update` setting `status` to `first_offer` filtered by the lead ID. Connect `record what was sent` here.

4. **Build Block 3 (Reply Detection & Thread Retrieval):**
   - **Node 11:** Create `When a reply arrives` (`n8n-nodes-base.gmailTrigger`, v1.4). Set polling interval to every minute and assign your `gmailOAuth2` credentials.
   - **Node 12:** Create `match the reply to a lead` (`n8n-nodes-base.dataTable`, v1.1). Set operation to `get` (limit 1) filtering lead emails matching the extracted sender header. Connect `When a reply arrives` here.
   - **Node 13:** Create `store the reply` (`n8n-nodes-base.dataTable`, v1.1). Set operation to insert/upsert inbound reply details, cleaning quoted reply text in snippet fields. Connect `match the reply to a lead` here.
   - **Node 14:** Create `read the whole thread` (`n8n-nodes-base.dataTable`, v1.1). Set operation to `get` (limit 20) filtering by email and thread ID, ordered by history external ID. Connect `store the reply` here.
   - **Node 15:** Create `keep only the prospect's messages` (`n8n-nodes-base.code`, v2). Add JavaScript code to reverse array items and format messages into User/System roles. Connect `read the whole thread` here.

5. **Build Block 4 (AI Classification & Handoff):**
   - **Node 16:** Create `classify the reply` (`@n8n/n8n-nodes-langchain.agent`, v3.1). Set prompt text to evaluate serialized prospect messages for sales interest.
   - **Node 17:** Create `OpenRouter model` (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`, v1). Configure and assign your `openRouterApi` credentials. Connect its output to `classify the reply` (`ai_languageModel`).
   - **Node 18:** Create `reply classification schema` (`@n8n/n8n-nodes-langchain.outputParserStructured`, v1.3). Define JSON schema example with `alreadyInterested` boolean. Connect its output to `classify the reply` (`ai_outputParser`).
   - **Node 19:** Create `interested?` (`n8n-nodes-base.if`, v2.3). Set condition checking if `{{ $json.output.alreadyInterested }}` equals `true`. Connect `classify the reply` to this node.
   - **Node 20:** Create `flag the lead for handoff` (`n8n-nodes-base.dataTable`, v1.1). Set operation to `update` setting `status` to `handoff` and `handoff` to `teams` filtered by lead ID. Connect the True branch of `interested?` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Send cold outreach from an uploaded lead file and detect replies in Gmail | Workflow template name and architecture overview |
| Gmail OAuth2 Configuration | Required for both outbound email dispatch (`[cred] Gmail - send the outreach email`) and inbound monitoring (`When a reply arrives`) |
| OpenRouter API Integration | Powers the AI reply classification agent (`OpenRouter model` using OpenRouter credentials) |
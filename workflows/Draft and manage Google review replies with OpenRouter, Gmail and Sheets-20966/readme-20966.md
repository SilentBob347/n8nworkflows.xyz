Draft and manage Google review replies with OpenRouter, Gmail and Sheets

https://n8nworkflows.xyz/workflows/draft-and-manage-google-review-replies-with-openrouter--gmail-and-sheets-20966


# Draft and manage Google review replies with OpenRouter, Gmail and Sheets

### 1. Workflow Overview

This workflow automates the retrieval, sentiment analysis, drafting, logging, and approval management of Google Business Profile reviews. Running on an hourly schedule, it prevents duplicate processing, interfaces with OpenRouter to produce contextual and on-brand public responses, records data in Google Sheets, emails drafts to business owners, and supports optional auto-posting alongside robust error handling.

The workflow is organized into the following logical blocks:
- **1.1 Input Reception & Settings:** Triggers execution on an hourly interval and establishes configuration variables (business name, owner email, IDs, and brand voice).
- **1.2 Data Retrieval & Filtering:** Fetches existing logs from Google Sheets and pulls the latest reviews from the Google Business Profile API, filtering out already-processed items and existing replies.
- **1.3 AI Processing & Validation:** Sends new reviews to an OpenRouter language model to classify sentiment and urgency, while using a custom code node to safely parse and validate outputs against broken payloads.
- **1.4 Logging, Approval & Optional Posting:** Appends review data and draft replies to Google Sheets, dispatches a notification email via Gmail, and conditionally executes direct API publishing if auto-posting is active.
- **1.5 Error Handling:** Captures exceptions from HTTP, Gmail, or posting tasks, writes diagnostics to an error tracking sheet, and alerts the administrator.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Settings
**Overview:** This block initiates the workflow on an hourly schedule and centralizes global parameters required across subsequent HTTP requests and data operations.

**Nodes Involved:**
- `Check for New Reviews Every Hour`
- `Workflow Settings`

**Node Details:**
- **Check for New Reviews Every Hour**
  - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.2) — Acts as the entry point, firing the workflow automatically at hourly intervals.
  - *Configuration Choices:* Interval rule set to every 1 hour.
  - *Key Expressions/Variables:* None.
  - *Connections:* Output connects to `Workflow Settings`.
  - *Version Requirements:* v1.2+.
  - *Edge Cases/Failures:* Missed executions if n8n instance is offline during the trigger window.
- **Workflow Settings**
  - *Type & Technical Role:* `n8n-nodes-base.set` (v3.4) — Defines static configuration properties used globally across the workflow.
  - *Configuration Choices:* Assigns custom parameters (`businessName`, `ownerEmail`, `accountId`, `locationId`, `sheetId`, `brandVoice`, `autoPostReplies`).
  - *Key Expressions/Variables:* String and boolean constants.
  - *Connections:* Input from `Check for New Reviews Every Hour`; output connects to `Read Already-Logged Reviews`.
  - *Version Requirements:* v3.4+.
  - *Edge Cases/Failures:* Missing or incorrect placeholder configuration (e.g., leaving `REPLACE_WITH_ACCOUNT_ID`) will cause downstream API and Sheet errors.

---

#### Block 1.2: Data Retrieval & Filtering
**Overview:** This block reads previously recorded records from Google Sheets to prevent duplicate processing, fetches recent reviews from the Google Business Profile API, and filters out older or already-replied items.

**Nodes Involved:**
- `Read Already-Logged Reviews`
- `Fetch Latest Google Reviews`
- `Keep Only New Reviews`

**Node Details:**
- **Read Already-Logged Reviews**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (v4.6) — Retrieves existing rows from the "Reviews" spreadsheet tab.
  - *Configuration Choices:* Document ID sourced from workflow settings; sheet name set to "Reviews"; configured to execute once and always output data.
  - *Key Expressions/Variables:* `={{ $json.sheetId }}`
  - *Connections:* Input from `Workflow Settings`; output connects to `Fetch Latest Google Reviews`.
  - *Version Requirements:* v4.6+.
  - *Edge Cases/Failures:* Google Sheets API rate limits or missing spreadsheet tab names.
- **Fetch Latest Google Reviews**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) — Calls the Google My Business API to retrieve up to 50 recent reviews sorted by update time.
  - *Configuration Choices:* GET request with predefined Google OAuth2 credentials; error output routed via `continueErrorOutput`.
  - *Key Expressions/Variables:* `https://mybusiness.googleapis.com/v4/accounts/{{ $('Workflow Settings').first().json.accountId }}/locations/{{ $('Workflow Settings').first().json.locationId }}/reviews?pageSize=50&orderBy=updateTime%20desc`
  - *Connections:* Input from `Read Already-Logged Reviews`; output branches to `Keep Only New Reviews` (main) and `Prepare Error Log` (error).
  - *Version Requirements:* v4.2+.
  - *Edge Cases/Failures:* Expired OAuth2 tokens, invalid account/location IDs, or API downtime.
- **Keep Only New Reviews**
  - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript code node that compares fetched reviews against already-logged IDs and filters out items that already contain owner replies.
  - *Configuration Choices:* Custom ES6 script parsing star ratings and cross-referencing input datasets.
  - *Key Expressions/Variables:* Accesses `Read Already-Logged Reviews` and `Fetch Latest Google Reviews`.
  - *Connections:* Input from `Fetch Latest Google Reviews`; output connects to `OpenRouter — Classify + Draft Reply`.
  - *Version Requirements:* v2+.
  - *Edge Cases/Failures:* Malformed JSON structures returned by the Google API.

---

#### Block 1.3: AI Processing & Validation
**Overview:** This block passes filtered reviews to OpenRouter via a LangChain chain node using a chat model, classifies sentiment and urgency, drafts a response, and securely parses the output.

**Nodes Involved:**
- `OpenRouter — Classify + Draft Reply`
- `OpenRouter Chat Model`
- `Validate AI Draft Safely`

**Node Details:**
- **OpenRouter — Classify + Draft Reply**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (v1.7) — AI chain node that formats a structured prompt combining brand voice, business name, and review metadata.
  - *Configuration Choices:* Prompt type set to define; expects strict JSON output format containing sentiment, urgency, summary, and reply fields.
  - *Key Expressions/Variables:* `{{ $('Workflow Settings').first().json.businessName }}`, `{{ $('Workflow Settings').first().json.brandVoice }}`, `{{ $json.reviewer }}`, `{{ $json.rating }}`, `{{ $json.comment }}`
  - *Connections:* Input from `Keep Only New Reviews`; AI model connected via `OpenRouter Chat Model`; output connects to `Validate AI Draft Safely`.
  - *Version Requirements:* v1.7+.
  - *Edge Cases/Failures:* API timeouts, refusal errors, or non-JSON responses from the LLM.
- **OpenRouter Chat Model**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenRouter` (v1) — Language model provider configuration node.
  - *Configuration Choices:* Model configured to `openrouter/free`.
  - *Key Expressions/Variables:* Uses OpenRouter API credentials.
  - *Connections:* Connected to `OpenRouter — Classify + Draft Reply` via `ai_languageModel`.
  - *Version Requirements:* v1+.
  - *Edge Cases/Failures:* Invalid API keys or model unavailability.
- **Validate AI Draft Safely**
  - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript code node executing per item to parse LLM text output into structured JSON with fallback values.
  - *Configuration Choices:* Mode set to run once for each item.
  - *Key Expressions/Variables:* Accesses `Keep Only New Reviews` and raw text response.
  - *Connections:* Input from `OpenRouter — Classify + Draft Reply`; output connects to `Log Review and Draft in Sheets`.
  - *Version Requirements:* v2+.
  - *Edge Cases/Failures:* Unparseable text strings triggering fallback defaults.

---

#### Block 1.4: Logging, Approval & Optional Posting
**Overview:** This block saves the processed review and AI-generated draft to Google Sheets, emails the owner for approval, and optionally publishes the reply directly via the Google API if auto-posting is enabled.

**Nodes Involved:**
- `Log Review and Draft in Sheets`
- `Email Owner the Draft for Approval`
- `Is Auto-Posting Turned On?`
- `Post Reply to Google Review`

**Node Details:**
- **Log Review and Draft in Sheets**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (v4.6) — Appends review metadata, generated replies, and initial statuses to the "Reviews" spreadsheet tab.
  - *Configuration Choices:* Operation set to append; column mapping explicitly defined.
  - *Key Expressions/Variables:* `={{ $json.createdAt }}`, `={{ $json.rating }}`, `={{ $('Workflow Settings').first().json.autoPostReplies ? 'Auto-posted' : 'Awaiting approval' }}`, `={{ $json.comment }}`, `={{ $json.urgency }}`, `={{ $json.reviewer }}`, `={{ $json.reviewId }}`, `={{ $json.sentiment }}`, `={{ $json.reply }}`
  - *Connections:* Input from `Validate AI Draft Safely`; output connects to `Email Owner the Draft for Approval`.
  - *Version Requirements:* v4.6+.
  - *Edge Cases/Failures:* Missing columns or schema mismatches in Google Sheets.
- **Email Owner the Draft for Approval**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (v2.1) — Sends an email notification to the business owner containing review details and the suggested reply.
  - *Configuration Choices:* Text email type; error output routed via `continueErrorOutput`.
  - *Key Expressions/Variables:* `={{ $('Workflow Settings').first().json.ownerEmail }}`, dynamic rating, reviewer name, sentiment, urgency, comment, and reply text.
  - *Connections:* Input from `Log Review and Draft in Sheets`; outputs branch to `Is Auto-Posting Turned On?` (main) and `Prepare Error Log` (error).
  - *Version Requirements:* v2.1+.
  - *Edge Cases/Failures:* Gmail authentication revocation or recipient bounce.
- **Is Auto-Posting Turned On?**
  - *Type & Technical Role:* `n8n-nodes-base.if` (v2.2) — Conditional branching node checking whether auto-posting is active.
  - *Configuration Choices:* Evaluates `autoPostReplies` boolean property.
  - *Key Expressions/Variables:* `={{ $('Workflow Settings').first().json.autoPostReplies }}`
  - *Connections:* Input from `Email Owner the Draft for Approval`; true branch connects to `Post Reply to Google Review`.
  - *Version Requirements:* v2.2+.
  - *Edge Cases/Failures:* Type mismatch if boolean evaluation is misconfigured.
- **Post Reply to Google Review**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) — Publishes the approved draft reply back to the Google Business Profile API.
  - *Configuration Choices:* PUT request with JSON payload; error output routed via `continueErrorOutput`.
  - *Key Expressions/Variables:* `https://mybusiness.googleapis.com/v4/{{ $('Validate AI Draft Safely').item.json.reviewName }}/reply`, `={{ JSON.stringify({ comment: $('Validate AI Draft Safely').item.json.reply }) }}`
  - *Connections:* Input from `Is Auto-Posting Turned On?` (true branch); error output connects to `Prepare Error Log`.
  - *Version Requirements:* v4.2+.
  - *Edge Cases/Failures:* Insufficient OAuth2 scopes (`business.manage` required) or closed/invalid review identifiers.

---

#### Block 1.5: Error Handling
**Overview:** This block captures failures across HTTP, Gmail, and posting nodes, structures error metadata, logs the incident to a dedicated Google Sheets tab, and alerts the administrator.

**Nodes Involved:**
- `Prepare Error Log`
- `Log Error in Google Sheets`
- `Alert Owner — Workflow Error`

**Node Details:**
- **Prepare Error Log**
  - *Type & Technical Role:* `n8n-nodes-base.set` (v3.4) — Normalizes error outputs and execution context into a structured format.
  - *Configuration Choices:* Assigns timestamp, node name, error message, and execution ID.
  - *Key Expressions/Variables:* `={{ $now.toISO() }}`, `={{ $json.error ? ($json.error.node ? $json.error.node.name : 'HTTP / Gmail step') : 'Unknown' }}`, `={{ $execution.id }}`
  - *Connections:* Input from error branches of `Fetch Latest Google Reviews`, `Email Owner the Draft for Approval`, and `Post Reply to Google Review`; output connects to `Log Error in Google Sheets`.
  - *Version Requirements:* v3.4+.
  - *Edge Cases/Failures:* Unhandled deep nested error objects.
- **Log Error in Google Sheets**
  - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (v4.6) — Appends error details to the "Errors" spreadsheet tab.
  - *Configuration Choices:* Operation set to append; automatic column mapping.
  - *Key Expressions/Variables:* `={{ $('Workflow Settings').first().json.sheetId }}`
  - *Connections:* Input from `Prepare Error Log`; output connects to `Alert Owner — Workflow Error`.
  - *Version Requirements:* v4.6+.
  - *Edge Cases/Failures:* Missing "Errors" tab in the target spreadsheet.
- **Alert Owner — Workflow Error**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (v2.1) — Sends an alert email to the business owner detailing the workflow failure.
  - *Configuration Choices:* Text email type.
  - *Key Expressions/Variables:* `={{ $('Workflow Settings').first().json.ownerEmail }}`, `={{ $('Prepare Error Log').item.json.Node }}`, `={{ $('Prepare Error Log').item.json.Error }}`
  - *Connections:* Input from `Log Error in Google Sheets`.
  - *Version Requirements:* v2.1+.
  - *Edge Cases/Failures:* Gmail authentication failure during error notification dispatch.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Check for New Reviews Every Hour | scheduleTrigger | Triggers workflow execution hourly | None | Workflow Settings | ## Google Review Reply Drafter<br>**Never leave a Google review unanswered again.**<br><br>### How it works<br>1. Every hour, new Google reviews are fetched for your location.<br>2. Reviews already in your sheet are skipped.<br>3. An AI model rates sentiment and urgency and drafts an on-brand reply.<br>4. Each review and draft is saved in Google Sheets.<br>5. The draft is emailed to you for approval. Reviews of 3 stars or less are marked URGENT.<br>6. Optional: turn on auto-posting to publish replies directly. |
| Workflow Settings | set | Defines global workflow configuration parameters | Check for New Reviews Every Hour | Read Already-Logged Reviews | ## 1. Load settings and logged reviews<br>Your business details are set and the review IDs already in the sheet are read so nothing is processed twice.<br><br>**You need to:** fill in business name, owner email, account ID, location ID, sheet ID and brand voice in **Workflow Settings**. |
| Read Already-Logged Reviews | googleSheets | Reads existing records from sheet to prevent duplicates | Workflow Settings | Fetch Latest Google Reviews | ## 1. Load settings and logged reviews<br>Your business details are set and the review IDs already in the sheet are read so nothing is processed twice.<br><br>**You need to:** fill in business name, owner email, account ID, location ID, sheet ID and brand voice in **Workflow Settings**. |
| Fetch Latest Google Reviews | httpRequest | Fetches recent reviews from Google My Business API | Read Already-Logged Reviews | Keep Only New Reviews, Prepare Error Log | ## 2. Fetch and filter reviews<br>The Google Business Profile API returns the latest reviews. Only new reviews without an existing owner reply move on.<br><br>**You need to:** connect a Google OAuth2 credential with the business.manage scope. |
| Prepare Error Log | set | Structures error data and execution context | Fetch Latest Google Reviews, Email Owner the Draft for Approval, Post Reply to Google Review | Log Error in Google Sheets | ## 6. Error handling<br>If fetching, emailing or posting fails, the error is saved in the `Errors` tab and emailed to you with the node and execution ID.<br><br>**You need to:** create the `Errors` tab with the headers Timestamp, Node, Error, Execution ID. |
| Log Error in Google Sheets | googleSheets | Appends error details to the Errors sheet tab | Prepare Error Log | Alert Owner — Workflow Error | ## 6. Error handling<br>If fetching, emailing or posting fails, the error is saved in the `Errors` tab and emailed to you with the node and execution ID.<br><br>**You need to:** create the `Errors` tab with the headers Timestamp, Node, Error, Execution ID. |
| Alert Owner — Workflow Error | gmail | Emails error alert to business owner | Log Error in Google Sheets | None | ## 6. Error handling<br>If fetching, emailing or posting fails, the error is saved in the `Errors` tab and emailed to you with the node and execution ID.<br><br>**You need to:** create the `Errors` tab with the headers Timestamp, Node, Error, Execution ID. |
| Keep Only New Reviews | code | Filters out already-logged reviews and replied reviews | Fetch Latest Google Reviews | OpenRouter — Classify + Draft Reply | ## 2. Fetch and filter reviews<br>The Google Business Profile API returns the latest reviews. Only new reviews without an existing owner reply move on.<br><br>**You need to:** connect a Google OAuth2 credential with the business.manage scope. |
| OpenRouter — Classify + Draft Reply | chainLlm | Generates sentiment classification and reply draft via AI | Keep Only New Reviews | Validate AI Draft Safely | ## 3. AI classifies and drafts the reply<br>The AI returns sentiment, urgency, a summary and a reply under 90 words. The code node checks the answer and uses a safe default reply if the output is broken.<br><br>**You need to:** add your OpenRouter credential and pick any chat model that can return JSON. |
| OpenRouter Chat Model | lmChatOpenRouter | Provides OpenRouter model configuration | None | OpenRouter — Classify + Draft Reply | ## 3. AI classifies and drafts the reply<br>The AI returns sentiment, urgency, a summary and a reply under 90 words. The code node checks the answer and uses a safe default reply if the output is broken.<br><br>**You need to:** add your OpenRouter credential and pick any chat model that can return JSON. |
| Validate AI Draft Safely | code | Validates LLM output and applies safe fallback values | OpenRouter — Classify + Draft Reply | Log Review and Draft in Sheets | ## 3. AI classifies and drafts the reply<br>The AI returns sentiment, urgency, a summary and a reply under 90 words. The code node checks the answer and uses a safe default reply if the output is broken.<br><br>**You need to:** add your OpenRouter credential and pick any chat model that can return JSON. |
| Log Review and Draft in Sheets | googleSheets | Records review data and draft replies in Google Sheets | Validate AI Draft Safely | Email Owner the Draft for Approval | ## 4. Log and send for approval<br>Every review is added to the `Reviews` tab, then the draft is emailed to you. 3 stars or less gets an URGENT subject.<br><br>**You need to:** create the `Reviews` tab with the exact headers and connect Gmail. |
| Email Owner the Draft for Approval | gmail | Emails review draft to owner for approval | Log Review and Draft in Sheets | Is Auto-Posting Turned On?, Prepare Error Log | ## 4. Log and send for approval<br>Every review is added to the `Reviews` tab, then the draft is emailed to you. 3 stars or less gets an URGENT subject.<br><br>**You need to:** create the `Reviews` tab with the exact headers and connect Gmail. |
| Is Auto-Posting Turned On? | if | Checks if automatic posting is enabled | Email Owner the Draft for Approval | Post Reply to Google Review | ## 5. Optional auto-posting<br>Off by default. When on, the reply is published to Google through the updateReply endpoint.<br><br>**You need to:** set `autoPostReplies` to true in **Workflow Settings** only after you trust the drafts. |
| Post Reply to Google Review | httpRequest | Publishes approved reply to Google Business Profile API | Is Auto-Posting Turned On? | Prepare Error Log | ## 5. Optional auto-posting<br>Off by default. When on, the reply is published to Google through the updateReply endpoint.<br><br>**You need to:** set `autoPostReplies` to true in **Workflow Settings** only after you trust the drafts. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`Check for New Reviews Every Hour`).
   - Configure the interval rule to trigger every `1` hours.

2. **Add Settings Node:**
   - Create a **Set** node named `Workflow Settings`.
   - Add the following string assignments: `businessName` (e.g., "Your Business Name"), `ownerEmail` (e.g., "user@example.com"), `accountId` (e.g., "REPLACE_WITH_ACCOUNT_ID"), `locationId` (e.g., "REPLACE_WITH_LOCATION_ID"), `sheetId` (e.g., "REPLACE_WITH_GOOGLE_SHEET_ID"), and `brandVoice` (e.g., "Warm, professional and brief. Thank the customer by first name. Never offer refunds or discounts in public.").
   - Add a boolean assignment: `autoPostReplies` set to `false`.
   - Connect `Check for New Reviews Every Hour` to `Workflow Settings`.

3. **Read Existing Logs:**
   - Add a **Google Sheets** node named `Read Already-Logged Reviews`.
   - Configure Document ID to `={{ $json.sheetId }}` and Sheet Name to `Reviews`. Set options to execute once and always output data.
   - Connect `Workflow Settings` to this node.
   - Configure Google Sheets OAuth2 credentials.

4. **Fetch Google Reviews:**
   - Add an **HTTP Request** node named `Fetch Latest Google Reviews`.
   - Set method to GET, URL to `https://mybusiness.googleapis.com/v4/accounts/{{ $('Workflow Settings').first().json.accountId }}/locations/{{ $('Workflow Settings').first().json.locationId }}/reviews?pageSize=50&orderBy=updateTime%20desc`, and authentication to Predefined Credential Type (Google OAuth2 API with scope `https://www.googleapis.com/auth/business.manage`).
   - Enable error output routing (`continueErrorOutput`).
   - Connect `Read Already-Logged Reviews` to this node.

5. **Filter New Reviews:**
   - Add a **Code** node named `Keep Only New Reviews`.
   - Insert JavaScript to filter out reviews already present in the Google Sheet and those with existing owner replies.
   - Connect the main output of `Fetch Latest Google Reviews` to this node.

6. **Configure AI Processing:**
   - Add an **Advanced AI > Chain LLM** node named `OpenRouter — Classify + Draft Reply`.
   - Set prompt type to define, and supply the prompt requesting JSON containing sentiment, urgency, summary, and reply.
   - Add an **OpenRouter Chat Model** sub-node, configure model to `openrouter/free`, and link it via the `ai_languageModel` input. Set up OpenRouter API credentials.
   - Connect `Keep Only New Reviews` to the chain node.

7. **Validate AI Output:**
   - Add a **Code** node named `Validate AI Draft Safely`.
   - Set mode to run once for each item, and include fallback logic for broken JSON responses and sentiment mapping.
   - Connect `OpenRouter — Classify + Draft Reply` to this node.

8. **Log Review and Send Approval Email:**
   - Add a **Google Sheets** node named `Log Review and Draft in Sheets`. Set operation to append, document ID to `={{ $('Workflow Settings').first().json.sheetId }}`, sheet name to `Reviews`, and map columns (`Review ID`, `Date`, `Reviewer`, `Rating`, `Comment`, `Sentiment`, `Urgency`, `Draft Reply`, `Status`).
   - Add a **Gmail** node named `Email Owner the Draft for Approval`. Set recipient to `={{ $('Workflow Settings').first().json.ownerEmail }}`, subject based on urgency/rating, message body with review and suggested reply, and enable error output routing (`continueErrorOutput`). Configure Gmail OAuth2 credentials.
   - Connect `Validate AI Draft Safely` -> `Log Review and Draft in Sheets` -> `Email Owner the Draft for Approval`.

9. **Configure Optional Auto-Posting:**
   - Add an **If** node named `Is Auto-Posting Turned On?`.
   - Set condition to evaluate `={{ $('Workflow Settings').first().json.autoPostReplies }}` as true.
   - Add an **HTTP Request** node named `Post Reply to Google Review`. Set method to PUT, URL to `https://mybusiness.googleapis.com/v4/{{ $('Validate AI Draft Safely').item.json.reviewName }}/reply`, body content type to JSON, body payload to `={{ JSON.stringify({ comment: $('Validate AI Draft Safely').item.json.reply }) }}`, and configure Google OAuth2 credentials with error output routing (`continueErrorOutput`).
   - Connect `Email Owner the Draft for Approval` (main output) -> `Is Auto-Posting Turned On?` -> `Post Reply to Google Review` (true branch).

10. **Configure Error Handling:**
    - Add a **Set** node named `Prepare Error Log` to format timestamp, node name, error message, and execution ID.
    - Add a **Google Sheets** node named `Log Error in Google Sheets` to append errors to the `Errors` tab (`Timestamp`, `Node`, `Error`, `Execution ID`).
    - Add a **Gmail** node named `Alert Owner — Workflow Error` to email error details to `={{ $('Workflow Settings').first().json.ownerEmail }}`.
    - Connect the error outputs of `Fetch Latest Google Reviews`, `Email Owner the Draft for Approval`, and `Post Reply to Google Review` to `Prepare Error Log`.
    - Connect `Prepare Error Log` -> `Log Error in Google Sheets` -> `Alert Owner — Workflow Error`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Review Reply Drafter Setup Guide | Initial workflow setup instructions covering Google Sheets tabs (`Reviews` and `Errors`), OAuth2 scope configuration (`https://www.googleapis.com/auth/business.manage`), and OpenRouter integration. |
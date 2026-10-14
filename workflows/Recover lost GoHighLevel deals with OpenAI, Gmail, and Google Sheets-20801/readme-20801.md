Recover lost GoHighLevel deals with OpenAI, Gmail, and Google Sheets

https://n8nworkflows.xyz/workflows/recover-lost-gohighlevel-deals-with-openai--gmail--and-google-sheets-20801


# Recover lost GoHighLevel deals with OpenAI, Gmail, and Google Sheets

### 1. Workflow Overview

This workflow automates the daily discovery and recovery assessment of lost sales opportunities originating from GoHighLevel (GHL). Its primary purpose is to systematically analyze why deals were lost, evaluate current customer engagement and deal longevity, and intelligently route each record toward automated winback outreach, long-term nurturing, exclusion, or human managerial review.

The logic is organized into five functional blocks:
- **1.1 Input Reception & Filtering:** Triggers on a daily schedule, queries the GoHighLevel opportunities search API for lost deals within a specified pipeline and location, and filters the results to retain only those whose status changed within the last 24 hours.
- **1.2 Signal Evaluation:** Deconstructs each filtered deal in parallel across three independent dimensions: duration (timing), lead behavior/tags (engagement), and presence of a recorded loss reason (objection). The evaluated signals are then combined back into a single payload.
- **1.3 AI Intelligence & Decision Making:** Passes the multi-signal context to an OpenAI Chat Model governed by a structured output parser. The model acts as a deal recovery strategist, outputting a precise action category (`SEND_WINBACK_OFFER`, `SCHEDULE_30_DAY_FOLLOWUP`, `DO_NOT_CONTACT`, or `ESCALATE_TO_MANAGER`), along with reasoning and an optional plain-text winback email.
- **1.4 Action Routing & Execution:** Uses a switch node to branch execution based on the AI's determined action:
  - Sending a winback email via Gmail.
  - Logging outcomes and calculating 30-day follow-up dates in Google Sheets.
  - Escalating contradictory or high-risk signals to a manager via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Filtering
- **Overview:** Initiates the automation on a daily cron schedule, queries GoHighLevel for recently lost deals, breaks the array into individual items, and filters out older records to prevent duplicate processing.
- **Nodes Involved:** 
  - `Daily Lost Deal Scan`
  - `Fetch Lost Opportunities`
  - `Split Opportunities`
  - `Lost in Last 24h`

- **Node Details:**
  - **Daily Lost Deal Scan**
    - *Type and technical role:* Schedule Trigger (`n8n-nodes-base.scheduleTrigger`, v1.4). Starts execution daily at hour 08:00.
    - *Configuration choices:* Interval configured to trigger at hour `8`.
    - *Key expressions or variables:* None.
    - *Input/Output:* Output connects to `Fetch Lost Opportunities`.
    - *Edge cases:* System downtime during the scheduled hour will skip the run (no built-in backfill).
  - **Fetch Lost Opportunities**
    - *Type and technical role:* HTTP Request (`n8n-nodes-base.httpRequest`, v4.2). Communicates with the LeadConnector/GoHighLevel API to search opportunities.
    - *Configuration choices:* Uses Generic Credential Type (`httpHeaderAuth`) with query parameters: `location_id` (`YOUR_GHL_LOCATION_ID`), `pipeline_id` (`YOUR_PIPELINE_ID`), `status` (`lost`), and `limit` (`100`). Sends custom header `Version: 2021-07-28`.
    - *Key expressions or variables:* None (placeholders must be updated).
    - *Input/Output:* Input from `Daily Lost Deal Scan`; output to `Split Opportunities`.
    - *Edge cases:* 401 Unauthorized if the bearer token is invalid or expired; rate limits or pagination limits if more than 100 lost deals occur in 24 hours.
  - **Split Opportunities**
    - *Type and technical role:* Item Lists / Split Out (`n8n-nodes-base.splitOut`, v1). Flattens the array of opportunities into separate items for individual processing.
    - *Configuration choices:* Field to split out set to `opportunities`.
    - *Key expressions or variables:* None.
    - *Input/Output:* Input from `Fetch Lost Opportunities`; output to `Lost in Last 24h`.
    - *Edge cases:* Fails if the API returns an empty or unexpected payload structure where the `opportunities` array is missing.
  - **Lost in Last 24h**
    - *Type and technical role:* Filter (`n8n-nodes-base.filter`, v2.2). Evaluates item timestamps and drops deals that changed status more than 24 hours ago.
    - *Configuration choices:* Uses loose type validation with a combinator `and`. Condition compares `$json.lastStatusChangeAt` as *after* `{{ $now.minus({ hours: 24 }).toISO() }}`.
    - *Key expressions or variables:* `{{ $now.minus({ hours: 24 }).toISO() }}`
    - *Input/Output:* Input from `Split Opportunities`; output branches to the three signal nodes.
    - *Edge cases:* Missing or null `lastStatusChangeAt` properties will cause evaluation failures or drop the item.

---

#### 1.2 Signal Evaluation
- **Overview:** Evaluates three distinct operational axes—deal duration, contact tags, and objection logging—in parallel across three separate code nodes before consolidating them back into unified items.
- **Nodes Involved:**
  - `Signal 1 - Timing`
  - `Signal 2 - Engagement`
  - `Signal 3 - Objection`
  - `Combine All Signals`

- **Node Details:**
  - **Signal 1 - Timing**
    - *Type and technical role:* Code (`n8n-nodes-base.code`, v2). Calculates how many days the deal was open between creation and its last status change.
    - *Configuration choices:* Mode set to `runOnceForEachItem`. JavaScript execution calculates age in days and assigns categorical signals (`DIED_SAME_DAY`, `DIED_FAST`, `DIED_SLOW`, `NORMAL_CYCLE`). Extracts contact details and computes `contactFirstName`.
    - *Key expressions or variables:* Operates on raw item fields (`deal.createdAt`, `deal.lastStatusChangeAt`, `deal.contact`).
    - *Input/Output:* Input from `Lost in Last 24h`; output to `Combine All Signals` (Input Index 0).
    - *Edge cases:* Invalid date strings can yield `NaN` age values, defaulting to `UNKNOWN`.
  - **Signal 2 - Engagement**
    - *Type and technical role:* Code (`n8n-nodes-base.code`, v2). Inspects contact tags to determine if the lead remains active or has opted out.
    - *Configuration choices:* Mode set to `runOnceForEachItem`. Checks for tags matching `email-opened-recently`, `site-visit-recent`, `unsubscribed`, or `do-not-contact`.
    - *Key expressions or variables:* `$json.contact?.tags`
    - *Input/Output:* Input from `Lost in Last 24h`; output to `Combine All Signals` (Input Index 1).
    - *Edge cases:* Missing `contact` or `tags` properties handled via optional chaining (`?.`).
  - **Signal 3 - Objection**
    - *Type and technical role:* Code (`n8n-nodes-base.code`, v2). Checks whether a lost reason ID was recorded on the opportunity.
    - *Configuration choices:* Mode set to `runOnceForEachItem`. Assigns `REASON_LOGGED` or `NO_REASON_LOGGED`.
    - *Key expressions or variables:* `$json.lostReasonId`
    - *Input/Output:* Input from `Lost in Last 24h`; output to `Combine All Signals` (Input Index 2).
    - *Edge cases:* Null or undefined `lostReasonId` values default safely to `NO_REASON_LOGGED`.
  - **Combine All Signals**
    - *Type and technical role:* Merge (`n8n-nodes-base.merge`, v3.2). Reconciles data streams from the three parallel signal nodes per deal.
    - *Configuration choices:* Mode set to `combine` using `combineByPosition` across `3` inputs.
    - *Key expressions or variables:* None.
    - *Input/Output:* Inputs from `Signal 1`, `Signal 2`, and `Signal 3`; output to `AI Recovery Strategist`.
    - *Edge cases:* Item count mismatches across parallel branches can cause misaligned merges if items are dropped unevenly.

---

#### 1.3 AI Intelligence & Decision Making
- **Overview:** Submits the consolidated deal context and signals to an OpenAI language model, which evaluates strategic rules and enforces a structured JSON schema output containing the recovery decision, reasoning, and generated email content.
- **Nodes Involved:**
  - `AI Recovery Strategist`
  - `OpenAI Model`
  - `Recovery Decision Schema`

- **Node Details:**
  - **AI Recovery Strategist**
    - *Type and technical role:* Advanced AI Agent (`@n8n/n8n-nodes-langchain.agent`, v3.1). Acts as the central orchestrator for prompt evaluation, connecting language model capabilities with output structure validation.
    - *Configuration choices:* Prompt type set to `define`. System message enforces deal recovery strategy guidelines, action criteria, constraints (e.g., no winback email if no email address exists), and text formatting rules.
    - *Key expressions or variables:* 
      - `=Lost deal: {{ $json.dealName }}`
      - `Contact first name: {{ $json.contactFirstName }}`
      - `Has email address: {{ $json.contactEmail ? 'yes' : 'no' }}`
      - Signals evaluation context variables.
    - *Input/Output:* Input from `Combine All Signals`; connected to `OpenAI Model` (AI language model) and `Recovery Decision Schema` (AI output parser); output connects to `Route by Action`.
    - *Edge cases:* Model hallucinations or schema validation errors if the LLM output violates the required JSON structure.
  - **OpenAI Model**
    - *Type and technical role:* OpenAi Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`, v1.3). Provides the underlying LLM engine for the AI agent.
    - *Configuration choices:* Model selected: `gpt-5-mini`.
    - *Key expressions or variables:* None (requires OpenAI API credentials).
    - *Input/Output:* Connects to `AI Recovery Strategist`.
    - *Edge cases:* API rate limits, insufficient account quota, or upstream service outages from OpenAI.
  - **Recovery Decision Schema**
    - *Type and technical role:* Structured Output Parser (`@n8n/n8n-nodes-langchain.outputParserStructured`, v1.3). Enforces a strict JSON schema for the AI agent's response.
    - *Configuration choices:* JSON schema example defines properties for `recoveryAction`, `reasoning`, `emailSubject`, and `emailBody`.
    - *Key expressions or variables:* None.
    - *Input/Output:* Connects to `AI Recovery Strategist`.
    - *Edge cases:* Failure to parse responses that deviate from the schema template.

---

#### 1.4 Action Routing & Execution
- **Overview:** Branches the workflow based on the AI's selected recovery action, executing Gmail outreach, Google Sheets logging, or manager escalations accordingly.
- **Nodes Involved:**
  - `Route by Action`
  - `Send Winback Email`
  - `Log to Recovery Sheet`
  - `Escalate to Manager`

- **Node Details:**
  - **Route by Action**
    - *Type and technical role:* Switch (`n8n-nodes-base.switch`, v3.4). Evaluates the AI's decision and routes execution to the appropriate downstream path.
    - *Configuration choices:* Configured with rules matching `SEND_WINBACK_OFFER`, `SCHEDULE_30_DAY_FOLLOWUP`, and `DO_NOT_CONTACT`. Fallback output renamed to `escalate` (handling `ESCALATE_TO_MANAGER`).
    - *Key expressions or variables:* `={{ $json.output.recoveryAction }}`
    - *Input/Output:* Input from `AI Recovery Strategist`; 4 output branches connecting to email and logging nodes.
    - *Edge cases:* Unrecognized action values fall through to the escalation route.
  - **Send Winback Email**
    - *Type and technical role:* Gmail (`n8n-nodes-base.gmail`, v2.2). Sends the AI-generated winback offer to the lead.
    - *Configuration choices:* Email type set to `text`. Appends attribution disabled.
    - *Key expressions or variables:*
      - Send To: `={{ $('Combine All Signals').item.json.contactEmail }}`
      - Subject: `={{ $json.output.emailSubject }}`
      - Message: `={{ $json.output.emailBody }}\n\nBest,\nYOUR_NAME`
    - *Input/Output:* Input from branch 1 of `Route by Action`.
    - *Edge cases:* Authentication revocation (OAuth2), sending failures due to invalid recipient email addresses, or missing signature placeholders.
  - **Log to Recovery Sheet**
    - *Type and technical role:* Google Sheets (`n8n-nodes-base.googleSheets`, v4.7). Appends deal recovery outcomes and metadata to a tracking spreadsheet.
    - *Configuration choices:* Operation set to `append`. Mapping mode set to `defineBelow`. Columns mapped: `Date`, `Deal ID`, `Deal Name`, `Contact`, `Email`, `Action`, `Follow Up Date`, and `Reasoning`.
    - *Key expressions or variables:*
      - Date: `={{ $now.toFormat('yyyy-MM-dd') }}`
      - Follow Up Date: `={{ $json.output.recoveryAction === 'SCHEDULE_30_DAY_FOLLOWUP' ? $now.plus({ days: 30 }).toFormat('yyyy-MM-dd') : '' }}`
    - *Input/Output:* Inputs from branches 2 and 3 of `Route by Action` (`SCHEDULE_30_DAY_FOLLOWUP` and `DO_NOT_CONTACT`).
    - *Edge cases:* Sheet column mismatch, missing Google Sheets credentials, or permission errors on the target spreadsheet.
  - **Escalate to Manager**
    - *Type and technical role:* Gmail (`n8n-nodes-base.gmail`, v2.2). Sends an escalation notification email containing deal context and AI reasoning when signals conflict.
    - *Configuration choices:* Recipient set to `user@example.com` (must be updated). Email type set to `text`.
    - *Key expressions or variables:*
      - Send To: `user@example.com`
      - Subject: `=Lost deal needs a decision: {{ $('Combine All Signals').item.json.dealName }}`
      - Message references deal identifiers, contact details, individual signal states, and AI reasoning.
    - *Input/Output:* Input from the fallback/escalation branch of `Route by Action`.
    - *Edge cases:* Invalid manager email address or Gmail authorization errors.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note - Overview** | n8n-nodes-base.stickyNote | Documentation and setup instructions | None | None | ## Lost Deal Recovery (GoHighLevel + AI)... [See setup & customize instructions in JSON] |
| **Sticky Note - Fetch** | n8n-nodes-base.stickyNote | Block 1 overview note | None | None | ### 1. Pull lost deals\nGets lost deals from your pipeline and keeps only the ones that changed status in the last 24h, so nobody gets emailed twice. |
| **Sticky Note - Signals** | n8n-nodes-base.stickyNote | Block 2 overview note | None | None | ### 2. Check three signals\nEach check runs on its own and doesn't know about the others. The merge puts them back together per deal. |
| **Sticky Note - AI** | n8n-nodes-base.stickyNote | Block 3 overview note | None | None | ### 3. AI decides\nPicks one action and writes the email if it's a winback. Contradicting signals go to a human. |
| **Sticky Note - Actions** | n8n-nodes-base.stickyNote | Block 4 overview note | None | None | ### 4. Take action\n30 day follow ups and do not contact both go to the same sheet, the action is in its own column. |
| **Daily Lost Deal Scan** | n8n-nodes-base.scheduleTrigger | Triggers workflow daily at 08:00 | None | Fetch Lost Opportunities | |
| **Fetch Lost Opportunities** | n8n-nodes-base.httpRequest | Queries GHL opportunities search API | Daily Lost Deal Scan | Split Opportunities | |
| **Split Opportunities** | n8n-nodes-base.splitOut | Flattens opportunity arrays into items | Fetch Lost Opportunities | Lost in Last 24h | |
| **Lost in Last 24h** | n8n-nodes-base.filter | Filters deals changed in past 24 hours | Split Opportunities | Signal 1, Signal 2, Signal 3 | |
| **Signal 1 - Timing** | n8n-nodes-base.code | Evaluates deal duration and timing | Lost in Last 24h | Combine All Signals | |
| **Signal 2 - Engagement** | n8n-nodes-base.code | Evaluates contact tags and engagement | Lost in Last 24h | Combine All Signals | |
| **Signal 3 - Objection** | n8n-nodes-base.code | Evaluates presence of lost reason ID | Lost in Last 24h | Combine All Signals | |
| **Combine All Signals** | n8n-nodes-base.merge | Merges multi-signal evaluation branches | Signal 1, Signal 2, Signal 3 | AI Recovery Strategist | |
| **AI Recovery Strategist** | @n8n/n8n-nodes-langchain.agent | AI Agent determining recovery action | Combine All Signals | Route by Action | |
| **OpenAI Model** | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o-mini language model | None | AI Recovery Strategist | |
| **Recovery Decision Schema** | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON output schema for AI | None | AI Recovery Strategist | |
| **Route by Action** | n8n-nodes-base.switch | Routes execution by AI recovery action | AI Recovery Strategist | Send Winback Email, Log to Recovery Sheet, Escalate to Manager | |
| **Send Winback Email** | n8n-nodes-base.gmail | Sends winback offer email via Gmail | Route by Action | None | |
| **Log to Recovery Sheet** | n8n-nodes-base.googleSheets | Logs deal outcome to Google Sheets | Route by Action | None | |
| **Escalate to Manager** | n8n-nodes-base.gmail | Sends escalation email to manager | Route by Action | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`Daily Lost Deal Scan`). Set the interval rule to trigger at hour `8`.
2. **Add the GHL API Request Node:**
   - Add an **HTTP Request** node (`Fetch Lost Opportunities`).
   - Set method to `GET`, URL to `https://services.leadconnectorhq.com/opportunities/search`.
   - Configure Authentication as **Generic Credential Type** (`httpHeaderAuth`). Provide your GoHighLevel Private Integration token as `Bearer <token>`.
   - Add query parameters: `location_id` (`YOUR_GHL_LOCATION_ID`), `pipeline_id` (`YOUR_PIPELINE_ID`), `status` (`lost`), and `limit` (`100`).
   - Add header parameter: `Version` = `2021-07-28`.
   - Connect `Daily Lost Deal Scan` to `Fetch Lost Opportunities`.
3. **Split and Filter Opportunities:**
   - Add an **Item Lists** (`Split Out`) node (`Split Opportunities`) with field to split out set to `opportunities`. Connect `Fetch Lost Opportunities` to it.
   - Add a **Filter** node (`Lost in Last 24h`). Set condition to check if `$json.lastStatusChangeAt` is *after* `{{ $now.minus({ hours: 24 }).toISO() }}` using loose type validation. Connect `Split Opportunities` to it.
4. **Create the Parallel Signal Evaluation Nodes:**
   - Add three **Code** nodes (`Signal 1 - Timing`, `Signal 2 - Engagement`, `Signal 3 - Objection`), all configured with mode `runOnceForEachItem`.
   - Paste the respective JavaScript code snippets for timing age calculations, tag checking (`email-opened-recently`, `unsubscribed`, etc.), and `lostReasonId` detection from the source configuration into each node.
   - Connect the output of `Lost in Last 24h` to all three signal nodes.
5. **Merge Signals:**
   - Add a **Merge** node (`Combine All Signals`) configured in `combine` mode using `combineByPosition` with `3` inputs.
   - Connect `Signal 1` to Input 0, `Signal 2` to Input 1, and `Signal 3` to Input 2.
6. **Configure the AI Agent and Sub-Components:**
   - Add an **Advanced AI Agent** node (`AI Recovery Strategist`). Set prompt type to `define` with system instructions defining the deal recovery strategist persona and decision rules. Set the agent text prompt to evaluate the incoming deal name, contact name, email presence, and the three signal metrics.
   - Add an **OpenAI Chat Model** node (`OpenAI Model`), select model `gpt-5-mini`, and connect its AI language model output to the agent. Configure OpenAI credentials.
   - Add a **Structured Output Parser** node (`Recovery Decision Schema`) with a JSON schema example containing `recoveryAction`, `reasoning`, `emailSubject`, and `emailBody`. Connect its output to the agent's parser input.
   - Connect `Combine All Signals` to the `AI Recovery Strategist`.
7. **Add the Routing Switch:**
   - Add a **Switch** node (`Route by Action`). Set expression to `={{ $json.output.recoveryAction }}`.
   - Define rules for:
     1. `SEND_WINBACK_OFFER`
     2. `SCHEDULE_30_DAY_FOLLOWUP`
     3. `DO_NOT_CONTACT`
   - Set the fallback output name to `escalate` (`ESCALATE_TO_MANAGER`).
   - Connect `AI Recovery Strategist` to `Route by Action`.
8. **Configure Action Execution Nodes:**
   - **Send Winback Email (Gmail):** Add a **Gmail** node. Set recipient to `={{ $('Combine All Signals').item.json.contactEmail }}`, subject to `={{ $json.output.emailSubject }}`, and message body to `={{ $json.output.emailBody }}\n\nBest,\nYOUR_NAME`. Connect to branch 1 of the switch. Configure Gmail OAuth2 credentials.
   - **Log to Recovery Sheet (Google Sheets):** Add a **Google Sheets** node. Set operation to `append`, mapping mode to `defineBelow`. Map columns (`Date`, `Deal ID`, `Deal Name`, `Contact`, `Email`, `Action`, `Follow Up Date`, `Reasoning`) using expressions for formatting dates and extracting action metadata. Connect switch branches 2 and 3 to this node. Configure Google Sheets credentials.
   - **Escalate to Manager (Gmail):** Add a second **Gmail** node. Set recipient to your manager's email address, subject to include the deal name, and message to output deal details, signal states, and AI reasoning. Connect to the fallback switch branch. Configure Gmail credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| GoHighLevel API Documentation | [LeadConnector API Reference](https://highlevel.stoplight.io/) |
| n8n Advanced AI Documentation | [n8n LangChain Nodes Guide](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.agent/) |
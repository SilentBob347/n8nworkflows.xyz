Triage complaints and draft cited replies with Gmail, Airtable and Claude

https://n8nworkflows.xyz/workflows/triage-complaints-and-draft-cited-replies-with-gmail--airtable-and-claude-20860


# Triage complaints and draft cited replies with Gmail, Airtable and Claude

### 1. Workflow Overview

This workflow automates the intake, classification, acknowledgement, policy-grounded drafting, and human-in-the-loop review of incoming customer complaints via Gmail, Airtable, and Anthropic Claude. Its primary objective is to ensure that every complaint is rapidly acknowledged with a reference number and addressed with a policy-cited response, while maintaining strict guardrails against ungrounded AI hallucinations and requiring explicit human sign-off before sending any final correspondence.

The logical blocks are structured as follows:
- **1.1 Input Reception & Hand-off:** Polls the Gmail inbox, extracts incoming email metadata, and spawns independent sub-workflow executions for isolated concurrent processing.
- **1.2 Reading & Routing:** Normalizes message bodies, screens out automated mail/newsletters, and cross-references Airtable cases to determine whether an email represents a new case, a follow-up, or skipped noise.
- **1.3 Classification & Acknowledgement:** Uses Anthropic Claude Haiku to classify complaints and assess severity, generates a structured Airtable case record, sends a formal automated acknowledgement to the complainant, and logs the receipt.
- **1.4 Policy Grounding & Citation Check:** Fetches approved policies and tone examples from Airtable, uses Anthropic Claude Opus to draft a response tied strictly to document markers, and programmatically validates citation integrity with a single automated retry loop.
- **1.5 Review & Approval:** Formats an interactive review email containing compliance warnings, citations, and ungrounded points, dispatching it to a handler or manager via a Gmail interactive form.
- **1.6 Execution, Decision Recording & Audit:** Processes the reviewer's choice (send, hold, personal reply, or timeout), executes the final email dispatch if approved, updates case statuses in Airtable, and writes detailed audit trail entries.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Hand-off
- **Overview:** Monitors the designated support inbox at one-minute intervals and isolates each newly detected message into an independent, asynchronous sub-workflow execution to prevent concurrent processing bottlenecks.
- **Nodes Involved:** `Check the Complaints Inbox`, `Hand Off Each Email`, `Handle One Email`
- **Node Details:**
  - `Check the Complaints Inbox`
    - *Type and Technical Role:* `n8n-nodes-base.gmailTrigger` (Trigger)
    - *Configuration Choices:* Polls every minute using Gmail query filter `in:inbox`.
    - *Key Expressions or Variables:* Simple output format enabled.
    - *Input/Output Connections:* Input: None (Trigger); Output: `Hand Off Each Email`.
    - *Edge Cases/Failures:* OAuth token expiration or API rate limits during high-volume polling.
  - `Hand Off Each Email`
    - *Type and Technical Role:* `n8n-nodes-base.executeWorkflow` (Action)
    - *Configuration Choices:* Executes the current workflow (`$workflow.id`) in `"each"` mode with `waitForSubWorkflow` set to `false`.
    - *Key Expressions:` `={{ $workflow.id }}`
    - *Input/Output Connections:* Input: `Check the Complaints Inbox`; Output: None (Spawns independent child execution).
    - *Edge Cases/Failures:* Sub-workflow recursion limits or memory exhaustion under heavy burst traffic.
  - `Handle One Email`
    - *Type and Technical Role:* `n8n-nodes-base.executeWorkflowTrigger` (Trigger)
    - *Configuration Choices:* Accepts passthrough input data from the parent hand-off action.
    - *Input/Output Connections:* Input: None (Child Trigger); Output: `Config`.
    - *Edge Cases/Failures:* Missing payload data passed from parent execution context.

---

#### 2.2 Reading & Routing
- **Overview:** Centralizes global configuration parameters, fetches the complete MIME message content from Gmail, normalizes HTML/text bodies, screens out automated or internal noise, and determines case routing via Airtable lookup.
- **Nodes Involved:** `Config`, `Get Full Email`, `Read Email`, `Find Existing Case`, `Route Email`, `What Is It?`, `Log Follow-up`, `Tell Handler: Follow-up`, `Not for the Desk`
- **Node Details:**
  - `Config`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data transformation / Setup)
    - *Configuration Choices:* Defines global business parameters (Airtable base ID, inbox address, handler details, reference prefixes, SLA days, tone rules, and acknowledgement templates). Validates placeholder strings before execution.
    - *Key Expressions:* Accesses `$input.first().json` for message and thread IDs.
    - *Input/Output Connections:* Input: `Handle One Email`; Output: `Get Full Email`.
    - *Edge Cases/Failures:* Throws an explicit JavaScript error if default placeholder strings (`appYOURBASEID`, `example.com`) are not replaced during setup.
  - `Get Full Email`
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Data Retrieval)
    - *Configuration Choices:* Retrieves full message details with up to 3 retry attempts on failure.
    - *Key Expressions:* `={{ $json.message_id }}`
    - *Input/Output Connections:* Input: `Config`; Output: `Read Email`.
    - *Edge Cases/Failures:* API timeouts or missing message IDs.
  - `Read Email`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Parsing / Filtering)
    - *Configuration Choices:* Strips HTML tags, decodes HTML entities, sanitizes body length, parses headers, matches reference prefixes (`COMP-YYYY-######`), and flags invalid senders (auto-replies, bounces, newsletters, self-sent mail).
    - *Key Expressions:* Accesses `$('Config').first().json` and raw Gmail payload properties.
    - *Input/Output Connections:* Input: `Get Full Email`; Output: `Find Existing Case`.
    - *Edge Cases/Failures:* Malformed MIME structures or unusually large email payloads exceeding slice limits.
  - `Find Existing Case`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Query)
    - *Configuration Choices:* Searches the `Cases` table for matching Gmail thread IDs, quoted references, or sender email addresses where status is neither `"Sent"` nor `"Handled personally"`.
    - *Key Expressions:* `={{ $('Config').first().json.airtable_base_id }}` and formulas checking thread/reference/email fields.
    - *Input/Output Connections:* Input: `Read Email`; Output: `Route Email`.
    - *Edge Cases/Failures:* Airtable API rate limits or formula syntax errors due to unescaped quotation marks in sender addresses.
  - `Route Email`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Routing Logic)
    - *Configuration Choices:* Evaluates skip reasons and existing case matches to assign a route of `SKIP`, `FOLLOW_UP`, or `NEW`.
    - *Input/Output Connections:* Input: `Find Existing Case`; Output: `What Is It?`.
  - `What Is It?`
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Conditional Branching)
    - *Configuration Choices:* Routes execution based on the string value of `$json.route` into branches: `"New"`, `"Follow-up"`, or `"Skip"`.
    - *Input/Output Connections:* Input: `Route Email`; Outputs: `Classify` (New), `Log Follow-up` (Follow-up), `Not for the Desk` (Skip).
  - `Log Follow-up`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Write)
    - *Configuration Choices:* Creates an audit record in the `Audit` table with action `"Follow-up received"`.
    - *Input/Output Connections:* Input: `What Is It?` (Follow-up branch); Output: `Tell Handler: Follow-up`.
  - `Tell Handler: Follow-up`
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Notification)
    - *Configuration Choices:* Sends a plaintext notification email to the configured handler about the inbound follow-up message without triggering a second automated acknowledgement.
    - *Key Expressions:* `={{ $('Config').first().json.handler_email }}` and trimmed email bodies.
    - *Input/Output Connections:* Input: `Log Follow-up`; Output: None.
  - `Not for the Desk`
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Terminal Node)
    - *Configuration Choices:* Terminating node for skipped or filtered messages.
    - *Input/Output Connections:* Input: `What Is It?` (Skip branch); Output: None.

---

#### 2.3 Classification & Acknowledgement
- **Overview:** Evaluates new complaints using Anthropic Claude Haiku with structured output parsing, generates a new case record in Airtable, dispatches an automated formal acknowledgement to the complainant, and logs the initial event.
- **Nodes Involved:** `Classify`, `Claude Haiku`, `Classification Format`, `Is It a Complaint?`, `Not a Complaint`, `Build Case`, `Create Case`, `Send Acknowledgement`, `Mark Acknowledged`, `Log Received`
- **Node Details:**
  - `Classify` & Model/Parser Nodes (`Claude Haiku`, `Classification Format`)
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm`, `@n8n/n8n-nodes-langchain.lmChatAnthropic`, `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Processing)
    - *Configuration Choices:* Utilizes Claude Haiku (`claude-haiku-5-5`) with a strict JSON schema enforcing boolean complaint validation, severity categorization (`Low`, `Standard`, `Serious`), severity reasoning, and summary points.
    - *Input/Output Connections:* Input: `What Is It?` (New branch) and `Claude Haiku` / `Classification Format`; Output: `Is It a Complaint?`.
    - *Edge Cases/Failures:* AI model refusals, structural schema violations, or unexpected markdown formatting in LLM outputs.
  - `Is It a Complaint?`
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration Choices:* Evaluates whether `output.is_complaint === true`.
    - *Input/Output Connections:* Input: `Classify`; Outputs: `Build Case` (True), `Not a Complaint` (False).
  - `Not a Complaint`
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Terminal Node)
    - *Configuration Choices:* Terminating node for classified non-complaint inquiries or spam.
    - *Input/Output Connections:* Input: `Is It a Complaint?` (False branch); Output: None.
  - `Build Case`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Calculates working-day SLA deadlines, retention end dates, unique case reference numbers (`PREFIX-YYYY-######`), and interpolates the automated acknowledgement template.
    - *Input/Output Connections:* Input: `Is It a Complaint?` (True branch); Output: `Create Case`.
  - `Create Case`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Write)
    - *Configuration Choices:* Inserts a new record into the `Cases` table with initial status `"New"`, mapping summary, severity, reference, respondent timelines, and message identifiers.
    - *Input/Output Connections:* Input: `Build Case`; Output: `Send Acknowledgement`.
  - `Send Acknowledgement`
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Reply)
    - *Configuration Choices:* Sends a reply email to the complainant using the pre-compiled acknowledgement text threaded to the original message ID.
    - *Key Expressions:* `={{ $('Build Case').first().json.acknowledgement }}`
    - *Input/Output Connections:* Input: `Create Case`; Output: `Mark Acknowledged`.
  - `Mark Acknowledged`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Update)
    - *Configuration Choices:* Updates the case record status in Airtable to `"Acknowledged"`.
    - *Input/Output Connections:* Input: `Send Acknowledgement`; Output: `Log Received`.
  - `Log Received`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Write)
    - *Configuration Choices:* Creates an audit entry in the `Audit` table noting system intake, severity, and respond-by target dates.
    - *Input/Output Connections:* Input: `Mark Acknowledged`; Output: `Fetch Documents`.

---

#### 2.4 Policy Grounding & Citation Check
- **Overview:** Gathers approved policy documents and writing style examples from Airtable, instructs Anthropic Claude Opus to draft a response constrained exclusively to cited library IDs, and executes programmatic verification of citation markers with an automated retry fallback.
- **Nodes Involved:** `Fetch Documents`, `Fetch Examples`, `Assemble Grounding`, `Draft Reply`, `Claude Opus`, `Draft Format`, `Check Draft`, `Redraft?`, `Prepare Redraft`
- **Node Details:**
  - `Fetch Documents`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Query)
    - *Configuration Choices:* Searches the `Documents` table where `{Status}='Approved'`. Configured with `executeOnce: true`.
    - *Input/Output Connections:* Input: `Log Received`; Output: `Fetch Examples`.
  - `Fetch Examples`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Query)
    - *Configuration Choices:* Searches the `Examples` table where `{Status}='Approved'` to supply tone guidelines. Configured with `executeOnce: true`.
    - *Input/Output Connections:* Input: `Fetch Documents`; Output: `Assemble Grounding`.
  - `Assemble Grounding`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Preparation)
    - *Configuration Choices:* Packages approved documents into a structured JSON library corpus, formats thread contents, and prepares tone guidelines for the downstream LLM.
    - *Input/Output Connections:* Input: `Fetch Examples`; Output: `Draft Reply`.
  - `Draft Reply` & Model/Parser Nodes (`Claude Opus`, `Draft Format`)
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm`, `@n8n/n8n-nodes-langchain.lmChatAnthropic`, `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Processing)
    - *Configuration Choices:* Uses Claude Opus (`claude-opus-5-5`) with max tokens set to 4096. Enforces strict grounding rules, inline citations (`[1]`), placeholder generation for missing internal facts (`[to confirm]`), and structured output parsing.
    - *Input/Output Connections:* Inputs: `Assemble Grounding` (or `Prepare Redraft` on retry) and `Claude Opus` / `Draft Format`; Output: `Check Draft`.
    - *Edge Cases/Failures:* Token limit truncation or refusal to comply with strict citation constraints.
  - `Check Draft`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Validation Logic)
    - *Configuration Choices:* Programmatically verifies that every citation marker maps to a valid approved library ID, checks that all draft markers have corresponding citation entries, and flags unreferenced citations. Determines whether a retry is warranted (`attempt < 2`).
    - *Input/Output Connections:* Input: `Draft Reply`; Output: `Redraft?`.
  - `Redraft?`
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration Choices:* Evaluates `{{ $json.retry === true }}`.
    - *Input/Output Connections:* Input: `Check Draft`; Outputs: `Prepare Redraft` (True), `Prepare Review` (False).
  - `Prepare Redraft`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Combines original grounding context with accumulated feedback errors for submission to a second generation attempt.
    - *Input/Output Connections:* Input: `Redraft?` (True branch); Output: `Draft Reply`.

---

#### 2.5 Review & Approval
- **Overview:** Prepares HTML-formatted approval emails for handlers or managers (routing to managers if compensation is proposed or severity is Serious), saves the draft and citations to Airtable, logs the drafting action, and dispatches an interactive email form awaiting human review.
- **Nodes Involved:** `Prepare Review`, `Save Draft`, `Log Drafted`, `Ask for Approval`, `Read Decision`, `Send It?`
- **Node Details:**
  - `Prepare Review`
    - *Type and Technical Role:* `n8n-nodes-base.code` (HTML Formatting)
    - *Configuration Choices:* Strips inline citation markers (`[n]`) from the reply text, compiles grounding problem logs, formats citation sources and unsupported points, and determines whether the review routes to the manager or handler.
    - *Input/Output Connections:* Input: `Redraft?` (False branch / Final check); Output: `Save Draft`.
  - `Save Draft`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Update)
    - *Configuration Choices:* Updates the Airtable case record with status `"Awaiting approval"`, approver role, formatted citations, draft body with markers, unsupported points, and compensation flags.
    - *Input/Output Connections:* Input: `Prepare Review`; Output: `Log Drafted`.
  - `Log Drafted`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Write)
    - *Configuration Choices:* Creates an audit entry recording that the draft was submitted for approval.
    - *Input/Output Connections:* Input: `Save Draft`; Output: `Ask for Approval`.
  - `Ask for Approval`
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Interactive Form Email)
    - *Configuration Choices:* Sends an email requesting review using Gmail's custom form response feature (`responseType: "customForm"`). Configures form fields for decision dropdown (`"Send this reply"`, `"Hold, I need to look into it"`, `"I will reply myself"`), reply textarea, and internal notes. Sets wait timeout limits based on `approval_wait_days`.
    - *Key Expressions:* Recipient address, HTML body, form configuration labels.
    - *Input/Output Connections:* Input: `Log Drafted`; Output: `Read Decision`.
    - *Edge Cases/Failures:* Form submission timeouts or approver failure to submit within the designated resume window.
  - `Read Decision`
    - *Type and Technical Role:* `n8n-nodes-base.code` (Decision Parsing)
    - *Configuration Choices:* Parses form responses. Enforces safety guardrails: if `"Send"` is chosen but the reply is empty or contains unresolved `[to confirm]` placeholders, the route is overridden to `"HOLD"`. Detects timeout (`NO_ANSWER`). Maps final routes (`SEND`, `HOLD`, `SELF`, `NO_ANSWER`) to case statuses.
    - *Input/Output Connections:* Input: `Ask for Approval`; Output: `Send It?`.
  - `Send It?`
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration Choices:* Evaluates `{{ $json.route === 'SEND' }}`.
    - *Input/Output Connections:* Input: `Read Decision`; Outputs: `Mark Sending` (True), `Record Decision` (False).

---

#### 2.6 Execution, Decision Recording & Audit
- **Overview:** Executes final email replies when approved (with error fallback to handlers), or logs hold/self-handling decisions, records final audit trails, and chases handlers if approval requests time out or are placed on hold.
- **Nodes Involved:** `Mark Sending`, `Send Reply`, `Mark Sent`, `Log Sent`, `Tell Handler: Not Sent`, `Record Decision`, `Log Decision`, `Chase Needed?`, `Chase Handler`
- **Node Details:**
  - `Mark Sending`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Update)
    - *Configuration Choices:* Updates case status in Airtable to `"Sending"`.
    - *Input/Output Connections:* Input: `Send It?` (True branch); Output: `Send Reply`.
  - `Send Reply`
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Dispatch)
    - *Configuration Choices:* Sends the approved reply email threaded to the original message ID. Configured with error output redirection (`onError: "continueErrorOutput"`).
    - *Key Expressions:* `={{ $('Read Decision').first().json.reply_text }}`
    - *Input/Output Connections:* Input: `Mark Sending`; Outputs: `Mark Sent` (Main output), `Tell Handler: Not Sent` (Error output).
    - *Edge Cases/Failures:* SMTP/Gmail API transmission failures resulting in diverted error execution.
  - `Mark Sent`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Update)
    - *Configuration Choices:* Updates case status to `"Sent"`, records completion timestamp (`$now.toISO()`), and saves final reply body.
    - *Input/Output Connections:* Input: `Send Reply` (Main output); Output: `Log Sent`.
  - `Log Sent`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Write)
    - *Configuration Choices:* Creates an audit entry recording successful reply dispatch.
    - *Input/Output Connections:* Input: `Mark Sent`; Output: None.
  - `Tell Handler: Not Sent`
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Notification)
    - *Configuration Choices:* Notifies the handler via email that an approved reply failed to transmit automatically and requires manual sending.
    - *Input/Output Connections:* Input: `Send Reply` (Error output); Output: None.
  - `Record Decision`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Update)
    - *Configuration Choices:* Updates case status in Airtable based on non-send decisions (`On hold`, `Handled personally`, or `No decision`).
    - *Input/Output Connections:* Input: `Send It?` (False branch); Output: `Log Decision`.
  - `Log Decision`
    - *Type and Technical Role:* `n8n-nodes-base.airtable` (Database Write)
    - *Configuration Choices:* Creates an audit entry detailing the reviewer's decision reason and internal notes.
    - *Input/Output Connections:* Input: `Record Decision`; Output: `Chase Needed?`.
  - `Chase Needed?`
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Branching)
    - *Configuration Choices:* Evaluates whether the route is `NO_ANSWER` or `HOLD` and contains a reason requiring escalation.
    - *Input/Output Connections:* Input: `Log Decision`; Outputs: `Chase Handler` (True), None (False).
  - `Chase Handler`
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Notification)
    - *Configuration Choices:* Sends an urgent notification email to the handler detailing why a complaint requires intervention or has timed out.
    - *Input/Output Connections:* Input: `Chase Needed?` (True branch); Output: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## AI complaints desk with cited replies and email approval... |
| Section: intake | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 1. One run per email |
| Section: route | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 2. Read and route |
| Section: acknowledge | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 3. Classify, open the case, acknowledge |
| Section: draft | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 4. Ground the reply, check every citation |
| Section: review | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 5. Review by email |
| Section: send | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 6. Send and record |
| Check the Complaints Inbox | `n8n-nodes-base.gmailTrigger` | Trigger incoming mail | None | Hand Off Each Email | |
| Hand Off Each Email | `n8n-nodes-base.executeWorkflow` | Sub-workflow spawn | Check the Complaints Inbox | None | |
| Handle One Email | `n8n-nodes-base.executeWorkflowTrigger` | Sub-workflow entry | None | Config | |
| Config | `n8n-nodes-base.code` | Global configuration | Handle One Email | Get Full Email | |
| Get Full Email | `n8n-nodes-base.gmail` | Fetch message MIME | Config | Read Email | |
| Read Email | `n8n-nodes-base.code` | Normalization & filtering | Get Full Email | Find Existing Case | |
| Find Existing Case | `n8n-nodes-base.airtable` | Case deduplication query | Read Email | Route Email | |
| Route Email | `n8n-nodes-base.code` | Routing decision | Find Existing Case | What Is It? | |
| What Is It? | `n8n-nodes-base.switch` | Conditional branch | Route Email | Classify, Log Follow-up, Not for the Desk | |
| Not for the Desk | `n8n-nodes-base.noOp` | Terminal node | What Is It? | None | |
| Log Follow-up | `n8n-nodes-base.airtable` | Audit logging | What Is It? | Tell Handler: Follow-up | |
| Tell Handler: Follow-up | `n8n-nodes-base.gmail` | Handler notification | Log Follow-up | None | |
| Classify | `@n8n/n8n-nodes-langchain.chainLlm` | LLM intake classification | What Is It?, Claude Haiku, Classification Format | Is It a Complaint? | |
| Claude Haiku | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | AI Model (Haiku) | None | Classify | |
| Classification Format | `@n8n/n8n-nodes-langchain.outputParserStructured` | Structured JSON output | None | Classify | |
| Is It a Complaint? | `n8n-nodes-base.if` | Complaint validation check | Classify | Build Case, Not a Complaint | |
| Not a Complaint | `n8n-nodes-base.noOp` | Terminal node | Is It a Complaint? | None | |
| Build Case | `n8n-nodes-base.code` | Case record builder | Is It a Complaint? | Create Case | |
| Create Case | `n8n-nodes-base.airtable` | Airtable case creation | Build Case | Send Acknowledgement | |
| Send Acknowledgement | `n8n-nodes-base.gmail` | Acknowledgement email | Create Case | Mark Acknowledged | |
| Mark Acknowledged | `n8n-nodes-base.airtable` | Update case status | Send Acknowledgement | Log Received | |
| Log Received | `n8n-nodes-base.airtable` | Audit logging | Mark Acknowledged | Fetch Documents | |
| Fetch Documents | `n8n-nodes-base.airtable` | Retrieve approved policies | Log Received | Fetch Examples | |
| Fetch Examples | `n8n-nodes-base.airtable` | Retrieve tone examples | Fetch Documents | Assemble Grounding | |
| Assemble Grounding | `n8n-nodes-base.code` | Grounding compilation | Fetch Examples | Draft Reply | |
| Draft Reply | `@n8n/n8n-nodes-langchain.chainLlm` | LLM policy-cited drafting | Assemble Grounding, Claude Opus, Draft Format, Prepare Redraft | Check Draft | |
| Claude Opus | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | AI Model (Opus) | None | Draft Reply | |
| Draft Format | `@n8n/n8n-nodes-langchain.outputParserStructured` | Structured JSON output | None | Draft Reply | |
| Check Draft | `n8n-nodes-base.code` | Citation contract validation | Draft Reply | Redraft? | |
| Redraft? | `n8n-nodes-base.if` | Retry evaluation check | Check Draft | Prepare Redraft, Prepare Review | |
| Prepare Redraft | `n8n-nodes-base.code` | Redraft payload prep | Redraft? | Draft Reply | |
| Prepare Review | `n8n-nodes-base.code` | Approval email formatting | Redraft? | Save Draft | |
| Save Draft | `n8n-nodes-base.airtable` | Update case with draft | Prepare Review | Log Drafted | |
| Log Drafted | `n8n-nodes-base.airtable` | Audit logging | Save Draft | Ask for Approval | |
| Ask for Approval | `n8n-nodes-base.gmail` | Interactive review form | Log Drafted | Read Decision | |
| Read Decision | `n8n-nodes-base.code` | Form decision parser | Ask for Approval | Send It? | |
| Send It? | `n8n-nodes-base.if` | Approval check | Read Decision | Mark Sending, Record Decision | |
| Mark Sending | `n8n-nodes-base.airtable` | Update status to sending | Send It? | Send Reply | |
| Send Reply | `n8n-nodes-base.gmail` | Dispatch final reply | Mark Sending | Mark Sent, Tell Handler: Not Sent | |
| Mark Sent | `n8n-nodes-base.airtable` | Update status to sent | Send Reply | Log Sent | |
| Log Sent | `n8n-nodes-base.airtable` | Audit logging | Mark Sent | None | |
| Tell Handler: Not Sent | `n8n-nodes-base.gmail` | Failure notification | Send Reply | None | |
| Record Decision | `n8n-nodes-base.airtable` | Update non-send status | Send It? | Log Decision | |
| Log Decision | `n8n-nodes-base.airtable` | Audit logging | Record Decision | Chase Needed? | |
| Chase Needed? | `n8n-nodes-base.if` | Escalation check | Log Decision | Chase Handler, None | |
| Chase Handler | `n8n-nodes-base.gmail` | Handler escalation notice | Chase Needed? | None | |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Airtable Base & Tables Setup
1. Create an Airtable base and note its Base ID (starting with `app`).
2. Create four tables with exact naming:
   - **Cases**: Required fields include `Reference` (Text), `Status` (Single Select/Text), `Subject` (Text), `Summary` (Long Text), `Severity` (Single Select/Text), `Severity reason` (Long Text), `Complainant name` (Text), `Complainant email` (Text), `Gmail thread id` (Text), `Gmail message id` (Text), `Respond by` (Date), `Retention until` (Date), `Draft reply` (Long Text), `Citations` (Long Text), `Unsupported points` (Long Text), `Compensation proposed` (Checkbox), `Approver` (Text), `Sent at` (DateTime), `Final reply` (Long Text).
   - **Documents**: Required fields include `Name` (Text), `Version` (Text), `Content` (Long Text), `Status` (Single Select set to `"Approved"` for active policies).
   - **Examples**: Required fields include `Example reply` (Long Text), `Status` (Single Select set to `"Approved"`).
   - **Audit**: Required fields include `Reference` (Text), `At` (DateTime), `Actor` (Text), `Action` (Text), `Detail` (Long Text).

#### Step 2: Credentials Configuration
1. Configure **Gmail OAuth2** credentials with access to the support inbox.
2. Configure **Airtable API** credentials linked to your Airtable account.
3. Configure **Anthropic (Claude)** credentials with a valid API key.

#### Step 3: Core Trigger & Sub-Workflow Hand-off Nodes
1. Create a **Gmail Trigger** node named `Check the Complaints Inbox`:
   - Set poll times to every minute.
   - Set query filter to `in:inbox`.
2. Create an **Execute Workflow** node named `Hand Off Each Email`:
   - Set mode to `"each"`. Set `waitForSubWorkflow` to `false`.
   - Set workflow ID expression to `={{ $workflow.id }}`.
   - Connect `Check the Complaints Inbox` to `Hand Off Each Email`.
3. Create an **Execute Workflow Trigger** node named `Handle One Email`.

#### Step 4: Intake & Routing Nodes
1. Create a **Code** node named `Config`:
   - Insert the configuration JavaScript object defining `airtable_base_id`, `inbox_address`, `handler_email`, `reference_prefix`, SLAs, tone rules, and the multi-line `acknowledgement` template.
   - Connect `Handle One Email` to `Config`.
2. Create a **Gmail** node named `Get Full Email`:
   - Set operation to `Get`, message ID to `={{ $json.message_id }}`, max retries to 3.
   - Connect `Config` to `Get Full Email`.
3. Create a **Code** node named `Read Email`:
   - Insert body normalization, HTML tag stripping, regex reference matching, and automated mail screening logic.
   - Connect `Get Full Email` to `Read Email`.
4. Create an **Airtable** node named `Find Existing Case`:
   - Set operation to `Search`, base ID to `={{ $('Config').first().json.airtable_base_id }}`, table to `Cases`.
   - Set filter formula to check matching thread IDs, quoted references, or sender emails.
   - Connect `Read Email` to `Find Existing Case`.
5. Create a **Code** node named `Route Email`:
   - Assign route (`SKIP`, `FOLLOW_UP`, or `NEW`).
   - Connect `Find Existing Case` to `Route Email`.
6. Create a **Switch** node named `What Is It?`:
   - Create 3 output branches evaluating `={{ $json.route }}` against `"NEW"`, `"FOLLOW_UP"`, and `"SKIP"`.
   - Connect `Route Email` to `What Is It?`.
7. For the Follow-up branch:
   - Create Airtable node `Log Follow-up` (Audit table creation).
   - Create Gmail node `Tell Handler: Follow-up` (Plaintext handler notification).
   - Connect `What Is It?` -> `Log Follow-up` -> `Tell Handler: Follow-up`.
8. For the Skip branch:
   - Create **NoOp** node `Not for the Desk`.
   - Connect `What Is It?` -> `Not for the Desk`.

#### Step 5: Classification & Acknowledgement Nodes
1. Create an **AI Agent / LLM Chain** node named `Classify`:
   - Attach an **Anthropic Chat Model** node named `Claude Haiku` configured with model `claude-haiku-5-5`.
   - Attach a **Structured Output Parser** node named `Classification Format` with a JSON schema defining `is_complaint`, `complainant_name`, `severity` (`Low`, `Standard`, `Serious`), `severity_reason`, and `summary_points`.
   - Connect `What Is It?` (New branch) to `Classify`.
2. Create an **If** node named `Is It a Complaint?`:
   - Condition: `={{ $json.output.is_complaint === true }}`.
   - Connect `Classify` to `Is It a Complaint?`.
3. Create a **NoOp** node named `Not a Complaint` connected to the false output of `Is It a Complaint?`.
4. Create a **Code** node named `Build Case`:
   - Calculates working-day respondent dates, retention periods, unique case reference numbers, and interpolates acknowledgement templates.
   - Connect `Is It a Complaint?` (True branch) to `Build Case`.
5. Create an **Airtable** node named `Create Case`:
   - Operation: `Create`, Table: `Cases`, mapping status, summary, severity, reference, respondent timelines, and IDs.
   - Connect `Build Case` to `Create Case`.
6. Create a **Gmail** node named `Send Acknowledgement`:
   - Operation: `Reply`, Message ID: `={{ $('Route Email').first().json.message_id }}`, Message: `={{ $('Build Case').first().json.acknowledgement }}`.
   - Connect `Create Case` to `Send Acknowledgement`.
7. Create an **Airtable** node named `Mark Acknowledged`:
   - Operation: `Update`, Table: `Cases`, updating status to `"Acknowledged"`.
   - Connect `Send Acknowledgement` to `Mark Acknowledged`.
8. Create an **Airtable** node named `Log Received`:
   - Operation: `Create`, Table: `Audit`, logging system receipt.
   - Connect `Mark Acknowledged` to `Log Received`.

#### Step 6: Policy Grounding & Citation Check Nodes
1. Create an Airtable node `Fetch Documents`:
   - Operation: `Search`, Table: `Documents`, Filter: `={Status}='Approved'`, `executeOnce: true`.
   - Connect `Log Received` to `Fetch Documents`.
2. Create an Airtable node `Fetch Examples`:
   - Operation: `Search`, Table: `Examples`, Filter: `={Status}='Approved'`, `executeOnce: true`.
   - Connect `Fetch Documents` to `Fetch Examples`.
3. Create a **Code** node named `Assemble Grounding`:
   - Packages approved documents and style examples into a structured JSON corpus.
   - Connect `Fetch Examples` to `Assemble Grounding`.
4. Create an **AI Agent / LLM Chain** node named `Draft Reply`:
   - Attach an **Anthropic Chat Model** node named `Claude Opus` configured with model `claude-opus-5-5` and max tokens 4096.
   - Attach a **Structured Output Parser** node named `Draft Format` with a JSON schema defining `draft_body`, `citations` (marker, claim, source_id, locator, quote), `unsupported_points`, `compensation_proposed`, `compensation_detail`, and `notes`.
   - Connect `Assemble Grounding` (and `Prepare Redraft`) to `Draft Reply`.
5. Create a **Code** node named `Check Draft`:
   - Programmatically verifies citation contracts and determines if a retry is needed (`retry: true`, `attempt < 2`).
   - Connect `Draft Reply` to `Check Draft`.
6. Create an **If** node named `Redraft?`:
   - Condition: `={{ $json.retry === true }}`.
   - Connect `Check Draft` to `Redraft?`.
7. Create a **Code** node named `Prepare Redraft`:
   - Combines grounding corpus with accumulated feedback errors.
   - Connect `Redraft?` (True branch) to `Draft Reply`.

#### Step 7: Review & Approval Nodes
1. Create a **Code** node named `Prepare Review`:
   - Strips inline markers (`[n]`), compiles HTML approval emails, formats citation lists, and determines manager routing thresholds.
   - Connect `Redraft?` (False branch) to `Prepare Review`.
2. Create an **Airtable** node named `Save Draft`:
   - Operation: `Update`, Table: `Cases`, updating status to `"Awaiting approval"`, saving draft body, citations, and unsupported points.
   - Connect `Prepare Review` to `Save Draft`.
3. Create an Airtable node `Log Drafted` (Audit table creation).
   - Connect `Save Draft` to `Log Drafted`.
4. Create a **Gmail** node named `Ask for Approval`:
   - Operation: `Send and Wait`, Response Type: `Custom Form`.
   - Configure form fields for Decision (dropdown: Send this reply, Hold, I will reply myself), Reply (textarea pre-filled with `reply_text`), and Internal note (textarea). Set wait time limits based on `wait_days`.
   - Connect `Log Drafted` to `Ask for Approval`.
5. Create a **Code** node named `Read Decision`:
   - Parses form data, enforces guardrails against empty replies or unresolved `[to confirm]` placeholders, and categorizes routes (`SEND`, `HOLD`, `SELF`, `NO_ANSWER`).
   - Connect `Ask for Approval` to `Read Decision`.
6. Create an **If** node named `Send It?`:
   - Condition: `={{ $json.route === 'SEND' }}`.
   - Connect `Read Decision` to `Send It?`.

#### Step 8: Execution, Decision Recording & Audit Nodes
1. For the True branch of `Send It?`:
   - Create Airtable node `Mark Sending` (Updates status to `"Sending"`).
   - Create Gmail node `Send Reply`: Operation `Reply`, Message ID `={{ $('Route Email').first().json.message_id }}`, Message `={{ $('Read Decision').first().json.reply_text }}`, with error output redirection (`onError: "continueErrorOutput"`).
   - Create Airtable node `Mark Sent`: Updates status to `"Sent"`, saves timestamp and final reply.
   - Create Airtable node `Log Sent` (Audit table creation).
   - Create Gmail node `Tell Handler: Not Sent` connected to the error output of `Send Reply`.
   - Connect: `Send It?` -> `Mark Sending` -> `Send Reply` -> (`Mark Sent` -> `Log Sent`, and error branch -> `Tell Handler: Not Sent`).
2. For the False branch of `Send It?`:
   - Create Airtable node `Record Decision` (Updates status to decision value).
   - Create Airtable node `Log Decision` (Audit table creation).
   - Create an **If** node named `Chase Needed?`: Condition evaluating `NO_ANSWER` or `HOLD` routes with reasons.
   - Create Gmail node `Chase Handler` connected to the true branch of `Chase Needed?`.
   - Connect: `Send It?` -> `Record Decision` -> `Log Decision` -> `Chase Needed?` -> `Chase Handler`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| AI complaints desk with cited replies and email approval workflow description and operational guardrails. | Provided by workflow author in embedded sticky note documentation. |
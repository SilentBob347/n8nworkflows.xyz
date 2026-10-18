Draft client-approved email replies with OpenAI and email

https://n8nworkflows.xyz/workflows/draft-client-approved-email-replies-with-openai-and-email-20885


# Draft client-approved email replies with OpenAI and email

### 1. Workflow Overview

This workflow automates the process of drafting client reply emails using an OpenAI language model based on incoming tasks/messages. It ensures that no response is sent to a recipient without explicit approval from a designated client representative. The system is designed for agencies and freelancers requiring an intermediary authorization gate that does not necessitate an n8n account for the approver.

The workflow logic is categorized into the following logical blocks:
- **1.1 Input Reception & Validation:** Ingests POST webhook requests containing message tasks and validates payload structures against global configuration parameters.
- **1.2 Client Database Lookup & Logging:** Queries the `approval_clients` data table to fetch client-specific configurations, verifies client statuses, and initializes tracking logs within the `approval_requests` data table.
- **1.3 AI Prompt Construction & Generation:** Formulates dynamic prompts using initial tasks or revised change requests, leveraging an OpenAI Chat model via an AI Agent node to generate contextually accurate draft replies, which are subsequently secured using SHA-256 fingerprinting.
- **1.4 Approval Lifecycle & Form Management:** Determines whether approval is mandatory, dispatches HTML review emails containing secure form resume links, and enforces time-bound waiting cycles with automated reminder phases.
- **1.5 Decision Processing & Routing:** Evaluates reviewer inputs (approval, edits, requested revisions, or rejections), handles obsolete form submissions, and routes states accordingly (iterating prompt generation if changes are requested).
- **1.6 Execution & Safeguards:** Cryptographically verifies approved text against final dispatch buffers, executes outward messaging, updates action logs, and captures system errors to notify workflow administrators.
- **1.7 Status & Monitoring Utilities:** Exposes secondary endpoints including a GET webhook for querying pending approvals and a scheduled cron trigger that dispatches a daily morning digest of waiting requests.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Validation

##### Overview
This block receives incoming webhook requests containing draft instructions, assigns baseline environment parameters, and sanitizes payload properties to ensure data integrity before further processing.

##### Nodes Involved
- `New draft request`
- `Settings`
- `Validate request`
- `Valid request?`
- `Reject: bad request`

##### Node Details

###### New draft request
- **Type and Technical Role:** `n8n-nodes-base.webhook` (Webhook Trigger)
- **Configuration Choices:** Configured to listen for HTTP `POST` requests on path `client-approval/draft` using Header Authentication and responding via the response node.
- **Key Expressions or Variables:** N/A (Entry point)
- **Input and Output Connections:** Input: External HTTP source; Output: `Settings`
- **Version-Specific Requirements:** v2.1
- **Edge Cases or Potential Failure Types:** Authentication failures return 401/403; malformed or missing headers.
- **Sub-Workflow Reference:** None

###### Settings
- **Type and Technical Role:** `n8n-nodes-base.set` (Edit Fields / Set)
- **Configuration Choices:** Defines global workflow variables including `company_name`, `sender`, `owner_email`, `default_expiry_hours` (48), `reminder_before_hours` (12), `max_rounds` (3), `max_task_chars` (8000), `max_iterations` (3), `max_output_tokens` (700), and the base `system_message`.
- **Key Expressions or Variables:** Static string and number assignments.
- **Input and Output Connections:** Input: `New draft request`; Output: `Validate request`
- **Version-Specific Requirements:** v3.4
- **Edge Cases or Potential Failure Types:** Type mismatch if values are modified improperly.
- **Sub-Workflow Reference:** None

###### Validate request
- **Type and Technical Role:** `n8n-nodes-base.code` (Code)
- **Configuration Choices:** Executes custom JavaScript to inspect incoming payload properties (`client_id`, `task`, `recipient_email`, `subject`, `reference`), validating lengths and regex email formats.
- **Key Expressions or Variables:** `$('Settings').first().json`, `$('New draft request').first().json.body`
- **Input and Output Connections:** Input: `Settings`; Output: `Valid request?`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** Throws evaluation errors if payload properties are completely undefined.
- **Sub-Workflow Reference:** None

###### Valid request?
- **Type and Technical Role:** `n8n-nodes-base.if` (If)
- **Configuration Choices:** Evaluates boolean condition `{{ $json.valid }}`.
- **Key Expressions or Variables:** `={{ $json.valid }}`
- **Input and Output Connections:** Input: `Validate request`; True Output: `Look up client`, False Output: `Reject: bad request`
- **Version-Specific Requirements:** v2.2
- **Edge Cases or Potential Failure Types:** Boolean type mismatch.
- **Sub-Workflow Reference:** None

###### Reject: bad request
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` (Respond to Webhook)
- **Configuration Choices:** Responds with HTTP Status `400` and a JSON body containing error details.
- **Key Expressions or Variables:** `={{ { error: 'invalid_request', message: $json.error } }}`
- **Input and Output Connections:** Input: `Valid request?` (False); Output: None (Terminal)
- **Version-Specific Requirements:** v1.5
- **Edge Cases or Potential Failure Types:** Response connection missing if triggered outside webhook context.
- **Sub-Workflow Reference:** None

---

#### Block 1.2: Client Database Lookup & Logging

##### Overview
This block cross-references the validated client identifier against the `approval_clients` data table, validates operational privileges, and provisions a tracking row within the `approval_requests` data table.

##### Nodes Involved
- `Look up client`
- `Reject: client table unavailable`
- `Check client`
- `Client can use this?`
- `Reject: unknown or inactive client`
- `Create request row`
- `Reject: request log unavailable`
- `Accept request`

##### Node Details

###### Look up client
- **Type and Technical Role:** `n8n-nodes-base.dataTable` (Data Table)
- **Configuration Choices:** Operation `get` with condition filtering where `client_id` equals `{{ $json.client_id }}`. Error handling configured to continue on error output.
- **Key Expressions or Variables:** `={{ $json.client_id }}`
- **Input and Output Connections:** Input: `Valid request?` (True); Output: Main -> `Check client`, Error -> `Reject: client table unavailable`
- **Version-Specific Requirements:** v1.1
- **Edge Cases or Potential Failure Types:** Data table inaccessible or missing.
- **Sub-Workflow Reference:** None

###### Reject: client table unavailable
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` (Respond to Webhook)
- **Configuration Choices:** Responds with HTTP `503` and a JSON error payload indicating unavailable client settings.
- **Key Expressions or Variables:** Static JSON string.
- **Input and Output Connections:** Input: `Look up client` (Error); Output: None (Terminal)
- **Version-Specific Requirements:** v1.5
- **Edge Cases or Potential Failure Types:** None.
- **Sub-Workflow Reference:** None

###### Check client
- **Type and Technical Role:** `n8n-nodes-base.code` (Code)
- **Configuration Choices:** JavaScript execution consolidating configuration settings, client metadata, and request inputs into a standardized unified state object.
- **Key Expressions or Variables:** `$('Settings').first().json`, `$('Validate request').first().json`, `$input.first().json`
- **Input and Output Connections:** Input: `Look up client`; Output: `Client can use this?`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** Missing data table rows resulting in unhandled properties.
- **Sub-Workflow Reference:** None

###### Client can use this?
- **Type and Technical Role:** `n8n-nodes-base.if` (If)
- **Configuration Choices:** Evaluates boolean condition `{{ $json.ok }}`.
- **Key Expressions or Variables:** `={{ $json.ok }}`
- **Input and Output Connections:** Input: `Check client`; True Output: `Create request row`, False Output: `Reject: unknown or inactive client`
- **Version-Specific Requirements:** v2.2
- **Edge Cases or Potential Failure Types:** Invalid client validation flags.
- **Sub-Workflow Reference:** None

###### Reject: unknown or inactive client
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` (Respond to Webhook)
- **Configuration Choices:** Responds with dynamic HTTP status (`404` or `403`) and an error payload.
- **Key Expressions or Variables:** `={{ $json.http_status }}`, `={{ $json.error }}`
- **Input and Output Connections:** Input: `Client can use this?` (False); Output: None (Terminal)
- **Version-Specific Requirements:** v1.5
- **Edge Cases or Potential Failure Types:** Undefined HTTP status codes.
- **Sub-Workflow Reference:** None

###### Create request row
- **Type and Technical Role:** `n8n-nodes-base.dataTable` (Data Table)
- **Configuration Choices:** Operation `insert` into data table `approval_requests` mapping state attributes (`round`, `status`, `history`, `subject`, `approver`, `client_id`, `reference`, `request_id`). Error handling continues on error output.
- **Key Expressions or Variables:** `={{ $json.round }}`, `={{ $json.status }}`, `={{ $json.request_id }}`
- **Input and Output Connections:** Input: `Client can use this?` (True); Output: Main -> `Accept request`, Error -> `Reject: request log unavailable`
- **Version-Specific Requirements:** v1.1
- **Edge Cases or Potential Failure Types:** Table schema mismatch or write permissions failure.
- **Sub-Workflow Reference:** None

###### Reject: request log unavailable
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` (Respond to Webhook)
- **Configuration Choices:** Responds with HTTP `503` indicating request logging table write failures.
- **Key Expressions or Variables:** Static JSON response.
- **Input and Output Connections:** Input: `Create request row` (Error); Output: None (Terminal)
- **Version-Specific Requirements:** v1.5
- **Edge Cases or Potential Failure Types:** None.
- **Sub-Workflow Reference:** None

###### Accept request
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` (Respond to Webhook)
- **Configuration Choices:** Responds immediately with HTTP `202` (Accepted) and request metadata to the calling client system.
- **Key Expressions or Variables:** `={{ $('Check client').first().json.request_id }}`
- **Input and Output Connections:** Input: `Create request row` (Success); Output: `Build the prompt`
- **Version-Specific Requirements:** v1.5
- **Edge Cases or Potential Failure Types:** Webhook response timing out if upstream tasks lag.
- **Sub-Workflow Reference:** None

---

#### Block 1.3: AI Prompt Construction & Generation

##### Overview
This block constructs the prompt template depending on the execution round (initial task vs. revision requests), executes the OpenAI language model via an AI agent, verifies the generated output, and generates an SHA-256 cryptographic fingerprint of the draft text.

##### Nodes Involved
- `Build the prompt`
- `Draft the reply`
- `Chat model`
- `Check the draft`
- `Fingerprint the draft`

##### Node Details

###### Build the prompt
- **Type and Technical Role:** `n8n-nodes-base.code` (Code)
- **Configuration Choices:** Assembles prompt strings incorporating client context, base tasks, and previous draft revisions alongside change requests when `round > 1`.
- **Key Expressions or Variables:** `$json.cfg`, `$json.task`, `$json.previous_draft`, `$json.change_request`
- **Input and Output Connections:** Input: `Accept request` or `What was decided?` (Changes requested loop); Output: `Draft the reply`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** Missing state references if loop conditions mismatch.
- **Sub-Workflow Reference:** None

###### Draft the reply
- **Type and Technical Role:** `@n8n/n8n-nodes-langchain.agent` (AI Agent)
- **Configuration Choices:** Text generation agent configured with max iterations and custom system message parameters combining global settings and client-specific instructions. Error handling continues on error output.
- **Key Expressions or Variables:** `={{ $json.prompt }}`, `={{ $json.cfg.max_iterations }}`, `={{ $json.cfg.system_message }}`
- **Input and Output Connections:** Input: `Build the prompt`, `Chat model` (AI Language Model); Output: Main -> `Check the draft`, Error -> `Prepare failure notice`
- **Version-Specific Requirements:** v3.1
- **Edge Cases or Potential Failure Types:** API rate limits, upstream OpenAI token failures, empty outputs.
- **Sub-Workflow Reference:** None

###### Chat model
- **Type and Technical Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (OpenAI Chat Model)
- **Configuration Choices:** Configures model `gpt-5-mini` with token constraints derived from settings.
- **Key Expressions or Variables:** `={{ $('Settings').first().json.max_output_tokens }}`
- **Input and Output Connections:** Input: None; Output: `Draft the reply` (AI Language Model connection)
- **Version-Specific Requirements:** v1.3
- **Edge Cases or Potential Failure Types:** Invalid OpenAI API credentials.
- **Sub-Workflow Reference:** None

###### Check the draft
- **Type and Technical Role:** `n8n-nodes-base.code` (Code)
- **Configuration Choices:** Validates that the agent output contains non-empty text, stripping extraneous carriage returns and throwing an explicit error if the draft is blank. Error handling continues on error output.
- **Key Expressions or Variables:** `$('Build the prompt').last().json`, `$json.output`
- **Input and Output Connections:** Input: `Draft the reply`; Output: Main -> `Fingerprint the draft`, Error -> `Prepare failure notice`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** Malformed text formatting from language model.
- **Sub-Workflow Reference:** None

###### Fingerprint the draft
- **Type and Technical Role:** `n8n-nodes-base.crypto` (Crypto)
- **Configuration Choices:** Generates an SHA-256 hexadecimal hash (`draft_sha256`) of the generated draft text.
- **Key Expressions or Variables:** `={{ $json.draft }}`
- **Input and Output Connections:** Input: `Check the draft`; Output: `Approval required?`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** Null string input values.
- **Sub-Workflow Reference:** None

---

#### Block 1.4: Approval Lifecycle & Form Management

##### Overview
This block checks whether client approval is required. If bypassed by settings, it prepares an immediate approval state; otherwise, it calculates expiration deadlines and reminder schedules, compiles HTML emails with secure resume links, dispatches them via email, updates data logs, and waits for human interaction.

##### Nodes Involved
- `Approval required?`
- `Approved by client setting`
- `Start approval round`
- `Prepare the approval email`
- `Email the approver`
- `Mark as waiting`
- `Hold for decision`
- `Wait for the approver`

##### Node Details

###### Approval required?
- **Type and Technical Role:** `n8n-nodes-base.if` (If)
- **Configuration Choices:** Evaluates condition `{{ $json.client.approval_required }}`.
- **Key Expressions or Variables:** `={{ $json.client.approval_required }}`
- **Input and Output Connections:** Input: `Fingerprint the draft`; True Output: `Start approval round`, False Output: `Approved by client setting`
- **Version-Specific Requirements:** v2.2
- **Edge Cases or Potential Failure Types:** Undefined client flags defaulting unexpectedly.
- **Sub-Workflow Reference:** None

###### Approved by client setting
- **Type and Technical Role:** `n8n-nodes-base.code` (Code)
- **Configuration Choices:** Bypasses human review by immediately setting the outcome status to approved when client settings dictate `approval_required = false`.
- **Key Expressions or Variables:** `$json`
- **Input and Output Connections:** Input: `Approval required?` (False); Output: `Fingerprint the answer`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** None.
- **Sub-Workflow Reference:** None

###### Start approval round
- **Type and Technical Role:** `n8n-nodes-base.code` (Code)
- **Configuration Choices:** Calculates round expiration timestamps and intermediate reminder trigger times based on configuration settings.
- **Key Expressions or Variables:** `$json`, `$json.cfg.expiry_hours`, `$json.cfg.reminder_before_hours`
- **Input and Output Connections:** Input: `Approval required?` (True); Output: `Prepare the approval email`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** Date arithmetic discrepancies on invalid timezones.
- **Sub-Workflow Reference:** None

###### Prepare the approval email
- **Type and Technical Role:** `n8n-nodes-base.code` (Code)
- **Configuration Choices:** Constructs the HTML body for approver emails, escaping special characters and inserting placeholder variables for task summaries and proposed replies.
- **Key Expressions or Variables:** `$json`, `$json.expires_at`, `$json.draft`
- **Input and Output Connections:** Input: `Start approval round`; Output: `Email the approver`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** HTML rendering glitches on unconventional email clients.
- **Sub-Workflow Reference:** None

###### Email the approver
- **Type and Technical Role:** `n8n-nodes-base.emailSend` (Send Email)
- **Configuration Choices:** Sends HTML email via SMTP to the client approver, injecting the execution form resume URL into the designated placeholder. Error handling continues on error output.
- **Key Expressions or Variables:** `={{ $json.email_html.replace('%%LINK%%', $execution.resumeFormUrl) }}`, `={{ $json.email_subject }}`, `={{ $json.client.approver_email }}`
- **Input and Output Connections:** Input: `Prepare the approval email`; Output: Main -> `Mark as waiting`, Error -> `Prepare failure notice`
- **Version-Specific Requirements:** v2.1
- **Edge Cases or Potential Failure Types:** SMTP server auth failures, invalid recipient email formats.
- **Sub-Workflow Reference:** None

###### Mark as waiting
- **Type and Technical Role:** `n8n-nodes-base.dataTable` (Data Table)
- **Configuration Choices:** Operation `update` on table `approval_requests` updating status to waiting, round details, expiry dates, and drafts. Error handling continues on error output.
- **Key Expressions or Variables:** `={{ $('Prepare the approval email').last().json.request_id }}`
- **Input and Output Connections:** Input: `Email the approver`; Output: Main -> `Hold for decision`, Error -> `Prepare failure notice`
- **Version-Specific Requirements:** v1.1
- **Edge Cases or Potential Failure Types:** Row mismatch or database lock timeouts.
- **Sub-Workflow Reference:** None

###### Hold for decision
- **Type and Technical Role:** `n8n-nodes-base.code` (Code)
- **Configuration Choices:** Maintains state integrity across asynchronous wait intervals, accounting for stale form feedback or recurrent reminder states.
- **Key Expressions or Variables:** `$('Prepare the approval email').last().json`, `$input.first().json`
- **Input and Output Connections:** Input: `Mark as waiting`; Output: `Wait for the approver`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** State drift during prolonged holds.
- **Sub-Workflow Reference:** None

###### Wait for the approver
- **Type and Technical Role:** `n8n-nodes-base.wait` (Wait)
- **Configuration Choices:** Configured to resume via `form` submission with a specified time limit (`limitType: atSpecifiedTime`, max date time bounded by `$json.wait_until`). Renders form fields including task HTML, editable reply textareas, decision dropdowns, comment areas, and hidden SHA-256 fingerprint fields.
- **Key Expressions or Variables:** `={{ $json.wait_until }}`, `={{ $json.draft }}`
- **Input and Output Connections:** Input: `Hold for decision`; Output: `Read the decision`
- **Version-Specific Requirements:** v1.1
- **Edge Cases or Potential Failure Types:** Expiry timeouts, missing form response data.
- **Sub-Workflow Reference:** None

---

#### Block 1.5: Decision Processing & Routing

##### Overview
This block processes form submissions or deadline expirations, validates SHA-256 form fingerprints to discard stale submissions from previous versions, categorizes outcomes (approval, changes requested, rejection, or expiration), and routes the execution flow accordingly.

##### Nodes Involved
- `Read the decision`
- `Route the answer`

##### Node Details

###### Read the decision
- **Type and Technical Role:** `n8n-nodes-base.code` (Code)
- **Configuration Choices:** Evaluates incoming form payloads or timeout events, checking submission timestamps, verifying form draft fingerprints against active state hashes, and parsing decision choices.
- **Key Expressions or Variables:** `$('Hold for decision').last().json`, `$input.first().json`
- **Input and Output Connections:** Input: `Wait for the approver`; Output: `Route the answer`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** Malformed form field objects.
- **Sub-Workflow Reference:** None

###### Route the answer
- **Type and Technical Role:** `n8n-nodes-base.switch` (Switch)
- **Configuration Choices:** Evaluates `$json.outcome` strings to route execution streams (`stale` to `Hold for decision`, `remind` to `Prepare the approval email`, and fallback `Decided` path).
- **Key Expressions or Variables:** `={{ $json.outcome }}`
- **Input and Output Connections:** Input: `Read the decision`; Output: Output 1 (Older form) -> `Hold for decision`, Output 2 (Remind) -> `Prepare the approval email`, Fallback (Decided) -> `Fingerprint the answer`
- **Version-Specific Requirements:** v3.2
- **Edge Cases or Potential Failure Types:** Unhandled outcome states.
- **Sub-Workflow Reference:** None

---

#### Block 1.6: Execution & Safeguards

##### Overview
This block records finalized decisions into data tables, verifies text integrity via secondary cryptographic hashing to prevent tampering, executes outward message dispatch, updates completion statuses, and catches execution exceptions to alert administrative owners.

##### Nodes Involved
- `Fingerprint the answer`
- `Record the decision`
- `Decision recorded`
- `What was decided?`
- `Email the owner: not approved`
- `Text to send`
- `Fingerprint the text to send`
- `Same text as approved?`
- `Send the approved reply`
- `Record: reply sent`
- `Prepare failure notice`
- `Record the failure`
- `Email the owner: failed`

##### Node Details

###### Fingerprint the answer
- **Type and Technical Role:** `n8n-nodes-base.crypto` (Crypto)
- **Configuration Choices:** Hashes approved text fields using SHA-256 hexadecimal encoding (`approved_sha256`).
- **Key Expressions or Variables:** `={{ $json.approved_text ?? '' }}`
- **Input and Output Connections:** Input: `Route the answer` (Fallback) or `Approved by client setting`; Output: `Record the decision`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** Empty text strings on rejections.
- **Sub-Workflow Reference:** None

###### Record the decision
- **Type and Technical Role:** `n8n-nodes-base.dataTable` (Data Table)
- **Configuration Choices:** Operation `update` on `approval_requests` recording decision outcomes, comments, timestamps, approver metadata, and approved SHA hashes. Error handling continues on error output.
- **Key Expressions or Variables:** `={{ $json.request_id }}`, `={{ $json.status }}`
- **Input and Output Connections:** Input: `Fingerprint the answer`; Output: Main -> `Decision recorded`, Error -> `Prepare failure notice`
- **Version-Specific Requirements:** v1.1
- **Edge Cases or Potential Failure Types:** Database write locks.
- **Sub-Workflow Reference:** None

###### Decision recorded
- **Type and Technical Role:** `n8n-nodes-base.code` (Code)
- **Configuration Choices:** Resumes state object propagation following successful table writes.
- **Key Expressions or Variables:** `$('Fingerprint the answer').last().json`
- **Input and Output Connections:** Input: `Record the decision`; Output: `What was decided?`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** Upstream state loss.
- **Sub-Workflow Reference:** None

###### What was decided?
- **Type and Technical Role:** `n8n-nodes-base.switch` (Switch)
- **Configuration Choices:** Switches execution based on `$json.outcome` values (`approved` proceeds to dispatch, `changes` loops back to prompt generation, fallback handles rejections/expirations).
- **Key Expressions or Variables:** `={{ $json.outcome }}`
- **Input and Output Connections:** Input: `Decision recorded`; Output 1 (Approved) -> `Text to send`, Output 2 (Changes requested) -> `Build the prompt`, Fallback -> `Email the owner: not approved`
- **Version-Specific Requirements:** v3.2
- **Edge Cases or Potential Failure Types:** Unrecognized decision strings.
- **Sub-Workflow Reference:** None

###### Email the owner: not approved
- **Type and Technical Role:** `n8n-nodes-base.emailSend` (Send Email)
- **Configuration Choices:** Sends text notification email to workflow owner detailing rejections, expirations, or revision limit breaches. Error handling continues on regular output.
- **Key Expressions or Variables:** `={{ $('Settings').first().json.owner_email }}`, `={{ $json.decision }}`
- **Input and Output Connections:** Input: `What was decided?` (Fallback); Output: None (Terminal)
- **Version-Specific Requirements:** v2.1
- **Edge Cases or Potential Failure Types:** SMTP dispatch failures.
- **Sub-Workflow Reference:** None

###### Text to send
- **Type and Technical Role:** `n8n-nodes-base.set` (Edit Fields / Set)
- **Configuration Choices:** Assigns approved text to `send_text` parameter while retaining other properties.
- **Key Expressions or Variables:** `={{ $json.approved_text }}`
- **Input and Output Connections:** Input: `What was decided?` (Approved); Output: `Fingerprint the text to send`
- **Version-Specific Requirements:** v3.4
- **Edge Cases or Potential Failure Types:** Variable propagation issues.
- **Sub-Workflow Reference:** None

###### Fingerprint the text to send
- **Type and Technical Role:** `n8n-nodes-base.crypto` (Crypto)
- **Configuration Choices:** Generates an SHA-256 hexadecimal hash (`sent_sha256`) of the final text slated for dispatch.
- **Key Expressions or Variables:** `={{ $json.send_text }}`
- **Input and Output Connections:** Input: `Text to send`; Output: `Same text as approved?`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** Null values.
- **Sub-Workflow Reference:** None

###### Same text as approved?
- **Type and Technical Role:** `n8n-nodes-base.if` (If)
- **Configuration Choices:** Validates that `sent_sha256` strictly matches `approved_sha256`.
- **Key Expressions or Variables:** `={{ $json.sent_sha256 }}`, `={{ $json.approved_sha256 }}`
- **Input and Output Connections:** Input: `Fingerprint the text to send`; True Output: `Send the approved reply`, False Output: `Prepare failure notice`
- **Version-Specific Requirements:** v2.2
- **Edge Cases or Potential Failure Types:** Hash mismatch due to whitespace alterations.
- **Sub-Workflow Reference:** None

###### Send the approved reply
- **Type and Technical Role:** `n8n-nodes-base.emailSend` (Send Email)
- **Configuration Choices:** Dispatches the approved reply text via SMTP to the original recipient email address. Error handling continues on error output.
- **Key Expressions or Variables:** `={{ $json.send_text }}`, `={{ $json.recipient_email }}`, `={{ $json.cfg.sender }}`
- **Input and Output Connections:** Input: `Same text as approved?` (True); Output: Main -> `Record: reply sent`, Error -> `Prepare failure notice`
- **Version-Specific Requirements:** v2.1
- **Edge Cases or Potential Failure Types:** SMTP delivery failures, invalid destination addresses.
- **Sub-Workflow Reference:** None

###### Record: reply sent
- **Type and Technical Role:** `n8n-nodes-base.dataTable` (Data Table)
- **Configuration Choices:** Operation `update` on `approval_requests` marking request status as closed, action status as done, updating history logs, and setting `sent_sha256`. Error handling continues on regular output.
- **Key Expressions or Variables:** `={{ $('Fingerprint the text to send').last().json.request_id }}`
- **Input and Output Connections:** Input: `Send the approved reply` (Success); Output: None (Terminal)
- **Version-Specific Requirements:** v1.1
- **Edge Cases or Potential Failure Types:** Data table update failures.
- **Sub-Workflow Reference:** None

###### Prepare failure notice
- **Type and Technical Role:** `n8n-nodes-base.code` (Code)
- **Configuration Choices:** Extracts error messages and execution states from failed execution paths, structuring administrative alerts.
- **Key Expressions or Variables:** `$input.first().json`, `$('Fingerprint the answer')`, `$('Prepare the approval email')`
- **Input and Output Connections:** Input: Error outputs from `Draft the reply`, `Check the draft`, `Email the approver`, `Mark as waiting`, `Record the decision`, `Same text as approved?`, and `Send the approved reply`; Output: `Record the failure`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** Missing state references if failure occurs pre-initialization.
- **Sub-Workflow Reference:** None

###### Record the failure
- **Type and Technical Role:** `n8n-nodes-base.dataTable` (Data Table)
- **Configuration Choices:** Operation `update` on `approval_requests` recording failure states, decisions, and action status. Error handling continues on regular output.
- **Key Expressions or Variables:** `={{ $json.request_id }}`
- **Input and Output Connections:** Input: `Prepare failure notice`; Output: `Email the owner: failed`
- **Version-Specific Requirements:** v1.1
- **Edge Cases or Potential Failure Types:** Database connectivity drops.
- **Sub-Workflow Reference:** None

###### Email the owner: failed
- **Type and Technical Role:** `n8n-nodes-base.emailSend` (Send Email)
- **Configuration Choices:** Sends text notification email to workflow owner concerning execution failures or blocked sends. Error handling continues on regular output.
- **Key Expressions or Variables:** `={{ $('Settings').first().json.owner_email }}`
- **Input and Output Connections:** Input: `Record the failure`; Output: None (Terminal)
- **Version-Specific Requirements:** v2.1
- **Edge Cases or Potential Failure Types:** SMTP errors.
- **Sub-Workflow Reference:** None

---

#### Block 1.7: Status & Monitoring Utilities

##### Overview
This block exposes utility endpoints including a GET webhook to query pending approval requests and a cron-based schedule trigger that compiles and emails a daily digest of all active waiting requests.

##### Nodes Involved
- `Pending requests`
- `Load waiting requests`
- `Summarise waiting requests`
- `Return waiting requests`
- `Reject: status unavailable`
- `Every morning`
- `Digest settings`
- `Load waiting requests for digest`
- `Build the digest`
- `Email the daily digest`

##### Node Details

###### Pending requests
- **Type and Technical Role:** `n8n-nodes-base.webhook` (Webhook Trigger)
- **Configuration Choices:** Configured to listen for HTTP `GET` requests on path `client-approval/pending` using Header Authentication.
- **Key Expressions or Variables:** N/A (Trigger)
- **Input and Output Connections:** Input: External HTTP GET request; Output: `Load waiting requests`
- **Version-Specific Requirements:** v2.1
- **Edge Cases or Potential Failure Types:** Invalid API authentication headers.
- **Sub-Workflow Reference:** None

###### Load waiting requests
- **Type and Technical Role:** `n8n-nodes-base.dataTable` (Data Table)
- **Configuration Choices:** Operation `get` with filtering condition where `status` equals `waiting`, returning all matched rows. Error handling continues on error output.
- **Key Expressions or Variables:** Static filter condition.
- **Input and Output Connections:** Input: `Pending requests`; Output: Main -> `Summarise waiting requests`, Error -> `Reject: status unavailable`
- **Version-Specific Requirements:** v1.1
- **Edge Cases or Potential Failure Types:** Database unavailability.
- **Sub-Workflow Reference:** None

###### Summarise waiting requests
- **Type and Technical Role:** `n8n-nodes-base.code` (Code)
- **Configuration Choices:** Parses query parameters for optional client filtering, filters active waiting table rows, and sorts entries by expiration timestamps.
- **Key Expressions or Variables:** `$('Pending requests').first().json.query?.client_id`
- **Input and Output Connections:** Input: `Load waiting requests`; Output: `Return waiting requests`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** Malformed query string parameters.
- **Sub-Workflow Reference:** None

###### Return waiting requests
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` (Respond to Webhook)
- **Configuration Choices:** Responds with HTTP `200` and a JSON body containing waiting request summaries.
- **Key Expressions or Variables:** `={{ $json }}`
- **Input and Output Connections:** Input: `Summarise waiting requests`; Output: None (Terminal)
- **Version-Specific Requirements:** v1.5
- **Edge Cases or Potential Failure Types:** Response connection missing.
- **Sub-Workflow Reference:** None

###### Reject: status unavailable
- **Type and Technical Role:** `n8n-nodes-base.respondToWebhook` (Respond to Webhook)
- **Configuration Choices:** Responds with HTTP `503` and JSON error body when waiting requests cannot be loaded.
- **Key Expressions or Variables:** Static JSON response.
- **Input and Output Connections:** Input: `Load waiting requests` (Error); Output: None (Terminal)
- **Version-Specific Requirements:** v1.5
- **Edge Cases or Potential Failure Types:** None.
- **Sub-Workflow Reference:** None

###### Every morning
- **Type and Technical Role:** `n8n-nodes-base.scheduleTrigger` (Schedule Trigger)
- **Configuration Choices:** Configured to trigger daily at hour `8` (instance timezone).
- **Key Expressions or Variables:** N/A (Trigger)
- **Input and Output Connections:** Input: None; Output: `Digest settings`
- **Version-Specific Requirements:** v1.2
- **Edge Cases or Potential Failure Types:** Timezone shifts.
- **Sub-Workflow Reference:** None

###### Digest settings
- **Type and Technical Role:** `n8n-nodes-base.set` (Edit Fields / Set)
- **Configuration Choices:** Sets owner email and sender address parameters for digest emails.
- **Key Expressions or Variables:** Static variable assignments.
- **Input and Output Connections:** Input: `Every morning`; Output: `Load waiting requests for digest`
- **Version-Specific Requirements:** v3.4
- **Edge Cases or Potential Failure Types:** Unassigned email addresses.
- **Sub-Workflow Reference:** None

###### Load waiting requests for digest
- **Type and Technical Role:** `n8n-nodes-base.dataTable` (Data Table)
- **Configuration Choices:** Operation `get` fetching all rows where `status` equals `waiting`.
- **Key Expressions or Variables:** Static condition.
- **Input and Output Connections:** Input: `Digest settings`; Output: `Build the digest`
- **Version-Specific Requirements:** v1.1
- **Edge Cases or Potential Failure Types:** Database errors.
- **Sub-Workflow Reference:** None

###### Build the digest
- **Type and Technical Role:** `n8n-nodes-base.code` (Code)
- **Configuration Choices:** Formulates consolidated text summary lines for all pending waiting requests, returning empty results if no requests are active.
- **Key Expressions or Variables:** `$input.all()`
- **Input and Output Connections:** Input: `Load waiting requests for digest`; Output: `Email the daily digest`
- **Version-Specific Requirements:** v2
- **Edge Cases or Potential Failure Types:** Null row datasets.
- **Sub-Workflow Reference:** None

###### Email the daily digest
- **Type and Technical Role:** `n8n-nodes-base.emailSend` (Send Email)
- **Configuration Choices:** Dispatches daily digest email via SMTP to the workflow owner. Error handling continues on regular output.
- **Key Expressions or Variables:** `={{ $json.text }}`
- **Input and Output Connections:** Input: `Build the digest`; Output: None (Terminal)
- **Version-Specific Requirements:** v2.1
- **Edge Cases or Potential Failure Types:** SMTP connection failures.
- **Sub-Workflow Reference:** None

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| New draft request | n8n-nodes-base.webhook | Webhook Trigger | External HTTP source | Settings | Draft replies with an AI agent and send them after client approval / 1 · Receive and check |
| Settings | n8n-nodes-base.set | Edit Fields / Set | New draft request | Validate request | Draft replies with an AI agent and send them after client approval / 1 · Receive and check |
| Validate request | n8n-nodes-base.code | Code | Settings | Valid request? | Draft replies with an AI agent and send them after client approval / 1 · Receive and check |
| Valid request? | n8n-nodes-base.if | If | Validate request | Look up client, Reject: bad request | Draft replies with an AI agent and send them after client approval / 1 · Receive and check |
| Reject: bad request | n8n-nodes-base.respondToWebhook | Respond to Webhook | Valid request? | None | Draft replies with an AI agent and send them after client approval / 1 · Receive and check |
| Look up client | n8n-nodes-base.dataTable | Data Table | Valid request? | Check client, Reject: client table unavailable | Draft replies with an AI agent and send them after client approval / 1 · Receive and check |
| Reject: client table unavailable | n8n-nodes-base.respondToWebhook | Respond to Webhook | Look up client | None | Draft replies with an AI agent and send them after client approval / 1 · Receive and check |
| Check client | n8n-nodes-base.code | Code | Look up client | Client can use this? | Draft replies with an AI agent and send them after client approval / 1 · Receive and check |
| Client can use this? | n8n-nodes-base.if | If | Check client | Create request row, Reject: unknown or inactive client | Draft replies with an AI agent and send them after client approval / 1 · Receive and check |
| Reject: unknown or inactive client | n8n-nodes-base.respondToWebhook | Respond to Webhook | Client can use this? | None | Draft replies with an AI agent and send them after client approval / 1 · Receive and check |
| Create request row | n8n-nodes-base.dataTable | Data Table | Client can use this? | Accept request, Reject: request log unavailable | Draft replies with an AI agent and send them after client approval / 1 · Receive and check |
| Reject: request log unavailable | n8n-nodes-base.respondToWebhook | Respond to Webhook | Create request row | None | Draft replies with an AI agent and send them after client approval / 1 · Receive and check |
| Accept request | n8n-nodes-base.respondToWebhook | Respond to Webhook | Create request row | Build the prompt | Draft replies with an AI agent and send them after client approval / 1 · Receive and check |
| Build the prompt | n8n-nodes-base.code | Code | Accept request, What was decided? | Draft the reply | Draft replies with an AI agent and send them after client approval / 2 · Draft with the AI agent |
| Draft the reply | @n8n/n8n-nodes-langchain.agent | AI Agent | Build the prompt, Chat model | Check the draft, Prepare failure notice | Draft replies with an AI agent and send them after client approval / 2 · Draft with the AI agent |
| Chat model | @n8n/n8n-nodes-langchain.lmChatOpenAi | OpenAI Chat Model | None | Draft the reply | Draft replies with an AI agent and send them after client approval / 2 · Draft with the AI agent |
| Check the draft | n8n-nodes-base.code | Code | Draft the reply | Fingerprint the draft, Prepare failure notice | Draft replies with an AI agent and send them after client approval / 2 · Draft with the AI agent |
| Fingerprint the draft | n8n-nodes-base.crypto | Crypto | Check the draft | Approval required? | Draft replies with an AI agent and send them after client approval / 2 · Draft with the AI agent |
| Approval required? | n8n-nodes-base.if | If | Fingerprint the draft | Start approval round, Approved by client setting | Draft replies with an AI agent and send them after client approval / 2 · Draft with the AI agent |
| Approved by client setting | n8n-nodes-base.code | Code | Approval required? | Fingerprint the answer | Draft replies with an AI agent and send them after client approval / 2 · Draft with the AI agent |
| Start approval round | n8n-nodes-base.code | Code | Approval required? | Prepare the approval email | Draft replies with an AI agent and send them after client approval / 3 · Ask the approver and wait |
| Prepare the approval email | n8n-nodes-base.code | Code | Start approval round, Route the answer | Email the approver | Draft replies with an AI agent and send them after client approval / 3 · Ask the approver and wait |
| Email the approver | n8n-nodes-base.emailSend | Send Email | Prepare the approval email | Mark as waiting, Prepare failure notice | Draft replies with an AI agent and send them after client approval / 3 · Ask the approver and wait |
| Mark as waiting | n8n-nodes-base.dataTable | Data Table | Email the approver | Hold for decision, Prepare failure notice | Draft replies with an AI agent and send them after client approval / 3 · Ask the approver and wait |
| Hold for decision | n8n-nodes-base.code | Code | Mark as waiting, Route the answer | Wait for the approver | Draft replies with an AI agent and send them after client approval / 3 · Ask the approver and wait |
| Wait for the approver | n8n-nodes-base.wait | Wait | Hold for decision | Read the decision | Draft replies with an AI agent and send them after client approval / 3 · Ask the approver and wait |
| Read the decision | n8n-nodes-base.code | Code | Wait for the approver | Route the answer | Draft replies with an AI agent and send them after client approval / 3 · Ask the approver and wait |
| Route the answer | n8n-nodes-base.switch | Switch | Read the decision | Hold for decision, Prepare the approval email, Fingerprint the answer | Draft replies with an AI agent and send them after client approval / 3 · Ask the approver and wait |
| Fingerprint the answer | n8n-nodes-base.crypto | Crypto | Route the answer, Approved by client setting | Record the decision | Draft replies with an AI agent and send them after client approval / 4 · Record the decision |
| Record the decision | n8n-nodes-base.dataTable | Data Table | Fingerprint the answer | Decision recorded, Prepare failure notice | Draft replies with an AI agent and send them after client approval / 4 · Record the decision |
| Decision recorded | n8n-nodes-base.code | Code | Record the decision | What was decided? | Draft replies with an AI agent and send them after client approval / 4 · Record the decision |
| What was decided? | n8n-nodes-base.switch | Switch | Decision recorded | Text to send, Build the prompt, Email the owner: not approved | Draft replies with an AI agent and send them after client approval / 4 · Record the decision |
| Email the owner: not approved | n8n-nodes-base.emailSend | Send Email | What was decided? | None | Draft replies with an AI agent and send them after client approval / 4 · Record the decision |
| Text to send | n8n-nodes-base.set | Edit Fields / Set | What was decided? | Fingerprint the text to send | Draft replies with an AI agent and send them after client approval / 5 · Do the approved action |
| Fingerprint the text to send | n8n-nodes-base.crypto | Crypto | Text to send | Same text as approved? | Draft replies with an AI agent and send them after client approval / 5 · Do the approved action |
| Same text as approved? | n8n-nodes-base.if | If | Fingerprint the text to send | Send the approved reply, Prepare failure notice | Draft replies with an AI agent and send them after client approval / 5 · Do the approved action |
| Send the approved reply | n8n-nodes-base.emailSend | Send Email | Same text as approved? | Record: reply sent, Prepare failure notice | Draft replies with an AI agent and send them after client approval / 5 · Do the approved action |
| Record: reply sent | n8n-nodes-base.dataTable | Data Table | Send the approved reply | None | Draft replies with an AI agent and send them after client approval / 5 · Do the approved action |
| Prepare failure notice | n8n-nodes-base.code | Code | Draft the reply, Check the draft, Email the approver, Mark as waiting, Record the decision, Same text as approved?, Send the approved reply | Record the failure | Draft replies with an AI agent and send them after client approval / 6 · If anything fails |
| Record the failure | n8n-nodes-base.dataTable | Data Table | Prepare failure notice | Email the owner: failed | Draft replies with an AI agent and send them after client approval / 6 · If anything fails |
| Email the owner: failed | n8n-nodes-base.emailSend | Send Email | Record the failure | None | Draft replies with an AI agent and send them after client approval / 6 · If anything fails |
| Pending requests | n8n-nodes-base.webhook | Webhook Trigger | External HTTP source | Load waiting requests | Draft replies with an AI agent and send them after client approval / 7 · What is waiting? |
| Load waiting requests | n8n-nodes-base.dataTable | Data Table | Pending requests | Summarise waiting requests, Reject: status unavailable | Draft replies with an AI agent and send them after client approval / 7 · What is waiting? |
| Summarise waiting requests | n8n-nodes-base.code | Code | Load waiting requests | Return waiting requests | Draft replies with an AI agent and send them after client approval / 7 · What is waiting? |
| Return waiting requests | n8n-nodes-base.respondToWebhook | Respond to Webhook | Summarise waiting requests | None | Draft replies with an AI agent and send them after client approval / 7 · What is waiting? |
| Reject: status unavailable | n8n-nodes-base.respondToWebhook | Respond to Webhook | Load waiting requests | None | Draft replies with an AI agent and send them after client approval / 7 · What is waiting? |
| Every morning | n8n-nodes-base.scheduleTrigger | Schedule Trigger | None | Digest settings | Draft replies with an AI agent and send them after client approval / 8 · Daily digest |
| Digest settings | n8n-nodes-base.set | Edit Fields / Set | Every morning | Load waiting requests for digest | Draft replies with an AI agent and send them after client approval / 8 · Daily digest |
| Load waiting requests for digest | n8n-nodes-base.dataTable | Data Table | Digest settings | Build the digest | Draft replies with an AI agent and send them after client approval / 8 · Daily digest |
| Build the digest | n8n-nodes-base.code | Code | Load waiting requests for digest | Email the daily digest | Draft replies with an AI agent and send them after client approval / 8 · Daily digest |
| Email the daily digest | n8n-nodes-base.emailSend | Send Email | Build the digest | None | Draft replies with an AI agent and send them after client approval / 8 · Daily digest |

---

### 4. Reproducing the Workflow from Scratch

To manually rebuild the workflow from scratch, execute the following steps:

1. **Create Data Tables**:
   - Create a Data Table named `approval_clients` with columns: `client_id` (string), `client_name` (string), `approver_email` (string), `instructions` (string), `expiry_hours` (number), `approval_required` (boolean), `active` (boolean). Add at least one active client record.
   - Create a Data Table named `approval_requests` with columns: `request_id` (string), `client_id` (string), `reference` (string), `subject` (string), `status` (string), `decision` (string), `action_status` (string), `approver` (string), `decided_by` (string), `decided_at` (string), `expires_at` (string), `draft_sha256` (string), `approved_sha256` (string), `sent_sha256` (string), `comment` (string), `history` (string), `round` (number), `edited_by_approver` (boolean).

2. **Set Up Core Trigger and Settings**:
   - Create a **Webhook** node named `New draft request` configured for POST requests on path `client-approval/draft` with Header Authentication.
   - Create a **Set** node named `Settings` and assign agency defaults (`company_name`, `sender`, `owner_email`, `default_expiry_hours`, `reminder_before_hours`, `max_rounds`, `max_task_chars`, `max_iterations`, `max_output_tokens`, `system_message`). Connect `New draft request` -> `Settings`.
   - Create a **Code** node named `Validate request` and connect `Settings` -> `Validate request`.
   - Create an **If** node named `Valid request?` and connect `Validate request` -> `Valid request?`.
   - Create a **Respond to Webhook** node named `Reject: bad request` (Response code 400). Connect `Valid request?` (False) -> `Reject: bad request`.

3. **Set Up Client Lookup and Request Logging**:
   - Create a **Data Table** node named `Look up client` (Operation: Get, Table: `approval_clients`, filter `client_id` = `{{ $json.client_id }}`). Connect `Valid request?` (True) -> `Look up client`.
   - Create a **Respond to Webhook** node named `Reject: client table unavailable` (Response code 503). Connect `Look up client` (Error output) -> `Reject: client table unavailable`.
   - Create a **Code** node named `Check client`. Connect `Look up client` (Main output) -> `Check client`.
   - Create an **If** node named `Client can use this?`. Connect `Check client` -> `Client can use this?`.
   - Create a **Respond to Webhook** node named `Reject: unknown or inactive client` (Response code dynamic). Connect `Client can use this?` (False) -> `Reject: unknown or inactive client`.
   - Create a **Data Table** node named `Create request row` (Operation: Insert, Table: `approval_requests`). Connect `Client can use this?` (True) -> `Create request row`.
   - Create a **Respond to Webhook** node named `Reject: request log unavailable` (Response code 503). Connect `Create request row` (Error output) -> `Reject: request log unavailable`.
   - Create a **Respond to Webhook** node named `Accept request` (Response code 202). Connect `Create request row` (Main output) -> `Accept request`.

4. **Configure AI Agent and Prompt Generation**:
   - Create a **Code** node named `Build the prompt`. Connect `Accept request` -> `Build the prompt`.
   - Create an **AI Agent** node named `Draft the reply`. Connect `Build the prompt` -> `Draft the reply`.
   - Create an **OpenAI Chat Model** node (`gpt-5-mini`) and connect it to the `ai_languageModel` input of `Draft the reply`.
   - Create a **Code** node named `Check the draft`. Connect `Draft the reply` (Main output) -> `Check the draft`.
   - Create a **Crypto** node named `Fingerprint the draft` (SHA-256 hash of `$json.draft`). Connect `Check the draft` -> `Fingerprint the draft`.
   - Create an **If** node named `Approval required?`. Connect `Fingerprint the draft` -> `Approval required?`.
   - Create a **Code** node named `Approved by client setting`. Connect `Approval required?` (False) -> `Approved by client setting`.

5. **Configure Approval and Wait Logic**:
   - Create a **Code** node named `Start approval round`. Connect `Approval required?` (True) -> `Start approval round`.
   - Create a **Code** node named `Prepare the approval email`. Connect `Start approval round` -> `Prepare the approval email`.
   - Create a **Send Email** node named `Email the approver` (SMTP credentials). Connect `Prepare the approval email` -> `Email the approver`.
   - Create a **Data Table** node named `Mark as waiting` (Operation: Update `approval_requests`). Connect `Email the approver` (Main output) -> `Mark as waiting`.
   - Create a **Code** node named `Hold for decision`. Connect `Mark as waiting` -> `Hold for decision`.
   - Create a **Wait** node named `Wait for the approver` (Resume: Form, Limit type: Specified time). Connect `Hold for decision` -> `Wait for the approver`.
   - Create a **Code** node named `Read the decision`. Connect `Wait for the approver` -> `Read the decision`.
   - Create a **Switch** node named `Route the answer`. Connect `Read the decision` -> `Route the answer`. Route `stale` back to `Hold for decision`, `remind` to `Prepare the approval email`, and fallback to `Fingerprint the answer`.

6. **Configure Decision Recording and Dispatch**:
   - Create a **Crypto** node named `Fingerprint the answer` (SHA-256 hash). Connect `Route the answer` (Fallback) and `Approved by client setting` -> `Fingerprint the answer`.
   - Create a **Data Table** node named `Record the decision` (Operation: Update `approval_requests`). Connect `Fingerprint the answer` -> `Record the decision`.
   - Create a **Code** node named `Decision recorded`. Connect `Record the decision` (Main output) -> `Decision recorded`.
   - Create a **Switch** node named `What was decided?`. Connect `Decision recorded` -> `What was decided?`. Route `approved` to `Text to send`, `changes` to `Build the prompt`, and fallback to `Email the owner: not approved`.
   - Create a **Send Email** node named `Email the owner: not approved`. Connect `What was decided?` (Fallback) -> `Email the owner: not approved`.
   - Create a **Set** node named `Text to send`. Connect `What was decided?` (Approved) -> `Text to send`.
   - Create a **Crypto** node named `Fingerprint the text to send` (SHA-256 hash). Connect `Text to send` -> `Fingerprint the text to send`.
   - Create an **If** node named `Same text as approved?`. Connect `Fingerprint the text to send` -> `Same text as approved?`.
   - Create a **Send Email** node named `Send the approved reply` (SMTP credentials). Connect `Same text as approved?` (True) -> `Send the approved reply`.
   - Create a **Data Table** node named `Record: reply sent` (Operation: Update `approval_requests`). Connect `Send the approved reply` (Main output) -> `Record: reply sent`.

7. **Configure Error Handling & Utilities**:
   - Create a **Code** node named `Prepare failure notice`. Connect error outputs from `Draft the reply`, `Check the draft`, `Email the approver`, `Mark as waiting`, `Record the decision`, `Same text as approved?`, and `Send the approved reply` -> `Prepare failure notice`.
   - Create a **Data Table** node named `Record the failure` (Operation: Update `approval_requests`). Connect `Prepare failure notice` -> `Record the failure`.
   - Create a **Send Email** node named `Email the owner: failed`. Connect `Record the failure` -> `Email the owner: failed`.
   - Create a **Webhook** node named `Pending requests` (GET, path `client-approval/pending`, Header Auth).
   - Create a **Data Table** node named `Load waiting requests` (Operation: Get waiting records). Connect `Pending requests` -> `Load waiting requests`.
   - Create a **Code** node named `Summarise waiting requests`. Connect `Load waiting requests` (Main output) -> `Summarise waiting requests`.
   - Create a **Respond to Webhook** node named `Return waiting requests`. Connect `Summarise waiting requests` -> `Return waiting requests`.
   - Create a **Respond to Webhook** node named `Reject: status unavailable` (Response code 503). Connect `Load waiting requests` (Error output) -> `Reject: status unavailable`.
   - Create a **Schedule Trigger** node named `Every morning` (trigger at 08:00).
   - Create a **Set** node named `Digest settings`. Connect `Every morning` -> `Digest settings`.
   - Create a **Data Table** node named `Load waiting requests for digest` (Operation: Get waiting records). Connect `Digest settings` -> `Load waiting requests for digest`.
   - Create a **Code** node named `Build the digest`. Connect `Load waiting requests for digest` -> `Build the digest`.
   - Create a **Send Email** node named `Email the daily digest` (SMTP credentials). Connect `Build the digest` -> `Email the daily digest`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Prepared with AI assistance. Editorial responsibility: PowerQuant ApS. | Credits and workflow origination notes. |
| Approver identity verification relies on secure unique email resume links rather than verified user account logins. | Security architecture note for form links. |
| Data table schemas require exact column definitions for `approval_clients` and `approval_requests` to prevent runtime mapping exceptions. | Database prerequisite requirements. |
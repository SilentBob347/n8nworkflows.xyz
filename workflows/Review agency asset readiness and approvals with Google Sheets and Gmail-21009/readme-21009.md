Review agency asset readiness and approvals with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/review-agency-asset-readiness-and-approvals-with-google-sheets-and-gmail-21009


# Review agency asset readiness and approvals with Google Sheets and Gmail

### 1. Workflow Overview

This workflow is a manual, deterministic asset readiness checker and version approval auditor designed for creative, web, or marketing agency coordinators. Its primary purpose is to read a structured Google Sheets workbook containing project definitions, item trackers, version-specific decisions, and a send ledger, validate the data against strict validation rules, and calculate project readiness alongside an owner follow-up queue. 

The workflow operates in two distinct operational modes controlled via configuration:
1. **Preview Mode (Default):** Evaluates all input data, checks business logic, calculates missing assets and pending reviews, and returns a human-readable preview digest without writing to the ledger or sending external communications.
2. **Send-Test Mode (Explicitly Approved):** Validates strict operator flags, appends a durable cryptographic reservation to the send ledger, re-verifies source data and ledger state, sends a single test notification via Gmail, and records the delivery outcome back into the ledger.

#### Logical Blocks
- **1.1 Initialization and Validation:** Triggers execution manually, sets up runtime parameters, and validates the presence of a target Google Sheets ID.
- **1.2 Data Ingestion:** Sequentially reads project, item, decision, and ledger data from the configured Google Sheets workbook.
- **1.3 Core Evaluation and Routing:** Executes comprehensive JavaScript validation and business rule logic, producing either a preview summary or routing the execution to the guarded sending pipeline.
- **1.4 Guarded Send-Test and Reconciliation (Optional):** Appends a pre-send reservation, rechecks all source sheets and ledger integrity, dispatches an approved synthetic test message through Gmail, and verifies the recorded delivery outcome.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Initialization and Validation
This block initiates the workflow manually, injects baseline operational configuration variables (such as timestamps, timezones, and execution modes), and ensures required workbook identifiers are present before calling external services.

- **Run one approved test**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` (Manual Trigger)
  - **Configuration Choices:** Initiates execution on demand with no scheduled triggers or webhooks.
  - **Key Expressions/Variables:** None.
  - **Connections:** Output connects to `Pilot configuration`.
  - **Version Requirements:** v1.
  - **Edge Cases / Failure Types:** None (manual execution).

- **Pilot configuration**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Code)
  - **Configuration Choices:** Executes once for all items, setting up environment defaults including ISO timestamp (`now`), timezone (`Asia/Makassar`), `spreadsheet_id` (empty placeholder by default), `mode` (`preview`), and safety flags (`allow_test_send`, `single_operator_confirmed`).
  - **Key Expressions/Variables:** `new Date().toISOString()`, `Asia/Makassar`, `preview`.
  - **Connections:** Input from `Run one approved test`; output connects to `Validate pilot setup`.
  - **Version Requirements:** v2.
  - **Edge Cases / Failure Types:** Improper clock skew if not synchronized via system time.

- **Validate pilot setup**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Code)
  - **Configuration Choices:** Validates that `spreadsheet_id` is supplied as a non-empty string.
  - **Key Expressions/Variables:** `const c=$input.first().json;`
  - **Connections:** Input from `Pilot configuration`; output connects to `Read projects`.
  - **Version Requirements:** v2.
  - **Edge Cases / Failure Types:** Throws `SETUP BLOCKED` if `spreadsheet_id` is blank.

---

#### Block 1.2: Data Ingestion
This block sequentially reads the four core tabs (`projects`, `items`, `decisions`, and `send_ledger`) from the specified Google Sheets document. Each node is executed once and configured to fail the workflow if an error occurs.

- **Read projects**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets Read)
  - **Configuration Choices:** Reads data from the `projects` sheet using OAuth2 authentication. Configured with `executeOnce: true` and `onError: stopWorkflow`. Output formatting parses dates as formatted strings and general values as unformatted.
  - **Key Expressions/Variables:** `={{ $('Pilot configuration').first().json.spreadsheet_id }}`
  - **Connections:** Input from `Validate pilot setup`; output connects to `Read items`.
  - **Version Requirements:** v4.7.
  - **Edge Cases / Failure Types:** Authentication failures, revoked OAuth tokens, or missing sheet name.

- **Read items**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets Read)
  - **Configuration Choices:** Reads data from the `items` sheet using OAuth2 authentication. Configured with `executeOnce: true` and `onError: stopWorkflow`.
  - **Key Expressions/Variables:** `={{ $('Pilot configuration').first().json.spreadsheet_id }}`
  - **Connections:** Input from `Read projects`; output connects to `Read decisions`.
  - **Version Requirements:** v4.7.
  - **Edge Cases / Failure Types:** Sheet missing, row limit exceeded (max 100 rows per table).

- **Read decisions**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets Read)
  - **Configuration Choices:** Reads data from the `decisions` sheet using OAuth2 authentication. Configured with `executeOnce: true` and `onError: stopWorkflow`.
  - **Key Expressions/Variables:** `={{ $('Pilot configuration').first().json.spreadsheet_id }}`
  - **Connections:** Input from `Read items`; output connects to `Read ledger`.
  - **Version Requirements:** v4.7.
  - **Edge Cases / Failure Types:** Malformed decision records, invalid date formats.

- **Read ledger**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets Read)
  - **Configuration Choices:** Reads data from the `send_ledger` sheet using OAuth2 authentication. Configured with `executeOnce: true` and `onError: stopWorkflow`.
  - **Key Expressions/Variables:** `={{ $('Pilot configuration').first().json.spreadsheet_id }}`
  - **Connections:** Input from `Read decisions`; output connects to `Gate reviewed preview`.
  - **Version Requirements:** v4.7.
  - **Edge Cases / Failure Types:** Ledger validation failure due to corrupt tracking events.

---

#### Block 1.3: Core Evaluation and Routing
This block acts as the analytical and routing core. It processes all ingested sheet rows through an embedded deterministic JavaScript evaluation engine (`Tracker` and `Preview`), validates business rules, computes project readiness and owner follow-up queues, and routes execution based on the selected mode.

- **Gate reviewed preview**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Code)
  - **Configuration Choices:** Encapsulates the complete deterministic evaluation library (`Tracker` and `Preview`). Validates IDs, boolean flags, dates, receipt timestamps, and version-specific decision rules. If `mode === 'preview'`, returns the rendered preview object. If in send mode, generates and verifies a send plan.
  - **Key Expressions/Variables:** Uses `$(name).all()` to aggregate rows from all four upstream Google Sheets nodes.
  - **Connections:** Input from `Read ledger`; output connects to `Route approved send only`.
  - **Version Requirements:** v2.
  - **Edge Cases / Failure Types:** Throws `PREVIEW BLOCKED` or `SEND BLOCKED` if schema limits, input errors, or invalid clocks are detected.

- **Route approved send only**
  - **Type & Technical Role:** `n8n-nodes-base.if` (If / Conditional Routing)
  - **Configuration Choices:** Evaluates whether `send_allowed === true` and `preview_only === false` using strict type validation.
  - **Key Expressions/Variables:** `={{ $json.send_allowed === true && $json.preview_only === false }}`
  - **Connections:** Input from `Gate reviewed preview`. True branch connects to `Reserve before sending`; False branch connects to `Return preview only`.
  - **Version Requirements:** v2.2.
  - **Edge Cases / Failure Types:** Routing misconfiguration if boolean flags are improperly formatted.

- **Return preview only**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Code)
  - **Configuration Choices:** Terminal preview node that validates preview state consistency and outputs the final human-readable report.
  - **Key Expressions/Variables:** Reads input JSON and `Pilot configuration`.
  - **Connections:** Input from the false branch of `Route approved send only`.
  - **Version Requirements:** v2.
  - **Edge Cases / Failure Types:** Throws `PREVIEW ROUTE BLOCKED` if unexpected input reaches this node.

---

#### Block 1.4: Guarded Send-Test and Reconciliation
When explicitly approved, this block manages the atomic transaction simulation: writing a pre-send reservation to the ledger, re-reading all source data and ledger state to ensure no concurrent modifications occurred, sending a single test email via Gmail, and recording the outcome.

- **Reserve before sending**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets Append)
  - **Configuration Choices:** Appends a new row to the `send_ledger` sheet with a `reserved` event status, attempt ID, delivery key, timestamp, and metadata note. Uses RAW cell formatting.
  - **Key Expressions/Variables:** `={{ $json.plan.reservation.at }}`, `={{ $json.plan.reservation.note }}`, `={{ $json.plan.reservation.event }}`, `={{ $json.plan.reservation.event_id }}`, `={{ $json.plan.reservation.attempt_id }}`, `={{ $json.plan.reservation.delivery_key }}`, `={{ $json.plan.reservation.provider_message_id }}`
  - **Connections:** Input from the true branch of `Route approved send only`; output connects to `Recheck ledger`.
  - **Version Requirements:** v4.7.
  - **Edge Cases / Failure Types:** Append failure due to Sheet API rate limits or permission drops.

- **Recheck ledger**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets Read)
  - **Configuration Choices:** Re-reads the `send_ledger` tab to verify the reservation was successfully committed and remains unique.
  - **Key Expressions/Variables:** `={{ $('Pilot configuration').first().json.spreadsheet_id }}`
  - **Connections:** Input from `Reserve before sending`; output connects to `Recheck projects`.
  - **Version Requirements:** v4.7.
  - **Edge Cases / Failure Types:** Stale cache reads.

- **Recheck projects**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets Read)
  - **Configuration Choices:** Re-reads the `projects` tab to ensure source data has not changed since initial evaluation.
  - **Key Expressions/Variables:** `={{ $('Pilot configuration').first().json.spreadsheet_id }}`
  - **Connections:** Input from `Recheck ledger`; output connects to `Recheck items`.
  - **Version Requirements:** v4.7.
  - **Edge Cases / Failure Types:** Concurrent sheet modifications.

- **Recheck items**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets Read)
  - **Configuration Choices:** Re-reads the `items` tab for freshness validation.
  - **Key Expressions/Variables:** `={{ $('Pilot configuration').first().json.spreadsheet_id }}`
  - **Connections:** Input from `Recheck projects`; output connects to `Recheck decisions`.
  - **Version Requirements:** v4.7.
  - **Edge Cases / Failure Types:** Sheet read timeout.

- **Recheck decisions**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets Read)
  - **Configuration Choices:** Re-reads the `decisions` tab for freshness validation.
  - **Key Expressions/Variables:** `={{ $('Pilot configuration').first().json.spreadsheet_id }}`
  - **Connections:** Input from `Recheck items`; output connects to `Verify fresh data and reservation`.
  - **Version Requirements:** v4.7.
  - **Edge Cases / Failure Types:** Sheet read error.

- **Verify fresh data and reservation**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Code)
  - **Configuration Choices:** Executes `Tracker.verifyReservation` and verifies that current tracker evaluations match the approved snapshot and that the ledger reservation is uncontested.
  - **Key Expressions/Variables:** `Tracker.verifyReservation()`, `Tracker.evaluate()`.
  - **Connections:** Input from `Recheck decisions`; output connects to `Send only approved test inbox`.
  - **Version Requirements:** v2.
  - **Edge Cases / Failure Types:** Throws `SEND BLOCKED` if tracker state changed after reservation.

- **Send only approved test inbox**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` (Gmail Send)
  - **Configuration Choices:** Dispatched as a text email using OAuth2 authentication. Configured with `disabled: true` by default for safety. Uses plain text messaging with attribution appending disabled.
  - **Key Expressions/Variables:** `={{ $json.recipient }}`, `={{ $json.subject }}`, `={{ $json.message }}`
  - **Connections:** Input from `Verify fresh data and reservation`; output connects to `Classify provider response`.
  - **Version Requirements:** v2.1.
  - **Edge Cases / Failure Types:** SMTP/Gmail API rejections, recipient address mismatch, invalid OAuth scopes.

- **Classify provider response**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Code)
  - **Configuration Choices:** Evaluates the Gmail API response object to determine whether the provider accepted the message (`sent`) or if the outcome is ambiguous (`uncertain`).
  - **Key Expressions/Variables:** `Tracker.sendOutcome()`.
  - **Connections:** Input from `Send only approved test inbox`; output connects to `Record sent or uncertain`.
  - **Version Requirements:** v2.
  - **Edge Cases / Failure Types:** Node error classification if API call fails.

- **Record sent or uncertain**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets Append)
  - **Configuration Choices:** Appends the terminal outcome event (`sent` or `uncertain`) along with provider message IDs and audit notes back into the `send_ledger` sheet.
  - **Key Expressions/Variables:** Mapped from `Classify provider response` output variables (`at`, `note`, `event`, `event_id`, `attempt_id`, `delivery_key`, `provider_message_id`).
  - **Connections:** Input from `Classify provider response`; output connects to `Read back outcome`.
  - **Version Requirements:** v4.7.
  - **Edge Cases / Failure Types:** Write failures to the ledger tab.

- **Read back outcome**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Google Sheets Read)
  - **Configuration Choices:** Re-reads the `send_ledger` tab to confirm that the terminal outcome event has been durably recorded.
  - **Key Expressions/Variables:** `={{ $('Pilot configuration').first().json.spreadsheet_id }}`
  - **Connections:** Input from `Record sent or uncertain`; output connects to `Verify durable outcome`.
  - **Version Requirements:** v4.7.
  - **Edge Cases / Failure Types:** Sheet read error.

- **Verify durable outcome**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Code)
  - **Configuration Choices:** Validates ledger state to ensure the terminal event is present and correctly linked to the attempt ID.
  - **Key Expressions/Variables:** `Tracker.ledgerState()`.
  - **Connections:** Input from `Read back outcome`; output is terminal.
  - **Version Requirements:** v2.
  - **Edge Cases / Failure Types:** Throws `OUTCOME UNVERIFIED` if the outcome is missing from the ledger.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run one approved test | `manualTrigger` | Manual entry point | None | Pilot configuration | Deliberate test-send gate |
| Pilot configuration | `code` | Sets baseline configuration | Run one approved test | Validate pilot setup | Deliberate test-send gate |
| Validate pilot setup | `code` | Blocks execution if spreadsheet ID is blank | Pilot configuration | Read projects | Deliberate test-send gate |
| Read projects | `googleSheets` | Loads project definitions | Validate pilot setup | Read items | Deliberate test-send gate |
| Read items | `read` (Sheets) | Loads item records | Read projects | Read decisions | Deliberate test-send gate |
| Read decisions | `read` (Sheets) | Loads decision records | Read items | Read ledger | Deliberate test-send gate |
| Read ledger | `read` (Sheets) | Loads send ledger history | Read decisions | Gate reviewed preview | Deliberate test-send gate |
| Gate reviewed preview | `code` | Evaluates rules and generates preview/plan | Read ledger | Route approved send only | Deliberate test-send gate |
| Route approved send only | `if` | Routes preview vs. approved send | Gate reviewed preview | Reserve before sending, Return preview only | Deliberate test-send gate |
| Return preview only | `code` | Outputs readable preview report | Route approved send only | None | Deliberate test-send gate |
| Reserve before sending | `append` (Sheets) | Appends pre-send reservation | Route approved send only | Recheck ledger | Durable pre-send ledger |
| Recheck ledger | `read` (Sheets) | Verifies ledger reservation | Reserve before sending | Recheck projects | Durable pre-send ledger |
| Recheck projects | `read` (Sheets) | Re-reads projects for freshness | Recheck ledger | Recheck items | Durable pre-send ledger |
| Recheck items | `read` (Sheets) | Re-reads items for freshness | Recheck projects | Recheck decisions | Durable pre-send ledger |
| Recheck decisions | `read` (Sheets) | Re-reads decisions for freshness | Recheck items | Verify fresh data and reservation | Durable pre-send ledger |
| Verify fresh data and reservation | `code` | Validates data freshness and reservation | Recheck decisions | Send only approved test inbox | Durable pre-send ledger |
| Send only approved test inbox | `gmail` | Sends approved test email (disabled by default) | Verify fresh data and reservation | Classify provider response | Uncertain means stop |
| Classify provider response | `code` | Classifies Gmail API response | Send only approved test inbox | Record sent or uncertain | Uncertain means stop |
| Record sent or uncertain | `append` (Sheets) | Appends terminal send outcome | Classify provider response | Read back outcome | Uncertain means stop |
| Read back outcome | `read` (Sheets) | Reads ledger to verify outcome | Record sent or uncertain | Verify durable outcome | Uncertain means stop |
| Verify durable outcome | `code` | Verifies durable outcome recording | Read back outcome | None | Uncertain means stop |
| Deliberate test-send gate | `stickyNote` | Workflow documentation and overview | None | None | Deliberate test-send gate |
| Durable pre-send ledger | `stickyNote` | Schema specifications and sample data | None | None | Durable pre-send ledger |
| Uncertain means stop | `stickyNote` | Operating guidelines and security notes | None | None | Uncertain means stop |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Point:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run one approved test`.

2. **Add Configuration Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Pilot configuration`.
   - Set mode to `runOnceForAllItems` and language to `javaScript`.
   - Paste the configuration object initializing `now`, `time_zone` (`Asia/Makassar`), `spreadsheet_id` (empty string ''), `mode` (`preview`), and safety flags (`allow_test_send: false`, `single_operator_confirmed: false`, etc.).

3. **Add Validation Node:**
   - Add a **Code** node named `Validate pilot setup`.
   - Add logic verifying that `spreadsheet_id` is a non-empty string. Connect `Pilot configuration` to `Validate pilot setup`.

4. **Set Up Google Sheets Reading Nodes:**
   - Create four **Google Sheets** nodes (`n8n-nodes-base.googleSheets`) configured with resource `sheet`, operation `read`, and your authorized OAuth2 credential.
   - **Node 1 (`Read projects`):** Set sheet name to `projects`. Set `spreadsheet_id` expression to `={{ $('Pilot configuration').first().json.spreadsheet_id }}`. Set `onError` to `stopWorkflow`.
   - **Node 2 (`Read items`):** Set sheet name to `items`. Connect `Read projects` output to `Read items`.
   - **Node 3 (`Read decisions`):** Set sheet name to `decisions`. Connect `Read items` output to `Read decisions`.
   - **Node 4 (`Read ledger`):** Set sheet name to `send_ledger`. Connect `Read decisions` output to `Read ledger`.

5. **Add Core Evaluation Engine:**
   - Add a **Code** node named `Gate reviewed preview`.
   - Implement the deterministic evaluation library (`Tracker` and `Preview`) and handler to process rows from `Read projects`, `Read items`, `Read decisions`, and `Read ledger`.
   - Connect `Read ledger` to `Gate reviewed preview`.

6. **Add Routing Logic:**
   - Add an **If** node (`n8n-nodes-base.if`) named `Route approved send only`.
   - Configure strict condition: Left Value `={{ $json.send_allowed === true && $json.preview_only === false }}` equals boolean `true`.
   - Connect `Gate reviewed preview` to `Route approved send only`.

7. **Build Preview Path:**
   - Add a **Code** node named `Return preview only`.
   - Connect the `false` output of `Route approved send only` to `Return preview only`.

8. **Build Send-Test and Reservation Path:**
   - Add a **Google Sheets** node named `Reserve before sending` (Operation: `append`, Sheet: `send_ledger`). Connect the `true` output of `Route approved send only` here.
   - Add sequential **Google Sheets** read nodes: `Recheck ledger`, `Recheck projects`, `Recheck items`, and `Recheck decisions`.
   - Add a **Code** node named `Verify fresh data and reservation` to validate snapshot consistency and ledger uniqueness.
   - Add a **Gmail** node (`n8n-nodes-base.gmail`) named `Send only approved test inbox`. Set resource to `message`, operation to `send`, configure recipients/subject/message via expressions, and **keep the node disabled (`disabled: true`)**.
   - Add a **Code** node named `Classify provider response` to parse the API result.
   - Add a **Google Sheets** node named `Record sent or uncertain` (Operation: `append`, Sheet: `send_ledger`).
   - Add a **Google Sheets** node named `Read back outcome` (Operation: `read`, Sheet: `send_ledger`).
   - Add a **Code** node named `Verify durable outcome` as the final verification step.

9. **Configure Workflow Settings:**
   - Set global timezone to `Asia/Makassar`, execution order to `v1`, and execution timeout to `120` seconds. Keep the workflow inactive.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Synthetic Single-Operator Pilot Documentation | Detailed overview of agency asset readiness tracking and audit constraints. |
| Self-Contained Spreadsheet Setup Schema | Required headers, data types, and copyable sample rows for `projects`, `items`, `decisions`, and `send_ledger`. |
| Guarded Send-Test & Reconciliation Guidelines | Operational steps for safely executing optional synthetic email tests and handling ambiguous delivery outcomes. |
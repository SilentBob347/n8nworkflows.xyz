Sync Lemlist email and LinkedIn events with Breakcold and Google Sheets

https://n8nworkflows.xyz/workflows/sync-lemlist-email-and-linkedin-events-with-breakcold-and-google-sheets-20995


# Sync Lemlist email and LinkedIn events with Breakcold and Google Sheets

### 1. Workflow Overview

The **Sync Lemlist email and LinkedIn events with Breakcold and Google Sheets** workflow automates the tracking of outbound sales touchpoints. It ingests real-time webhook payloads from Lemlist (covering both email and LinkedIn interactions), normalizes the payload attributes, validates security and lead association, updates lead timelines in Breakcold CRM, and writes an immutable audit record to Google Sheets.

The architecture breaks down into four sequential logical blocks:

- **1.1 Input Ingestion & Normalization:** Receives HTTP POST webhooks from Lemlist, standardizes event properties, timestamps, sequence steps, and extracts CRM identifiers (`breakcold_lead_id` and `contact_id`).
- **1.2 Filtering & Security:** Validates incoming payloads to ensure they pertain to actual leads and optionally enforces a shared webhook secret verification.
- **1.3 Routing & CRM Sync:** Evaluates event types against an allowlist of 12 supported interactions. Validates the presence of a Breakcold identifier, then executes an authenticated HTTP request to append a custom activity onto the lead’s timeline in Breakcold.
- **1.4 Audit Logging & Error Handling:** Normalizes the outcome of the CRM operation (Success, API Error, Missing ID, or Unrecognized Event Type) and appends a structured record into a Google Sheets tracking table without interrupting execution on CRM failures.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Ingestion & Normalization

##### Overview
This block captures incoming raw webhook traffic from Lemlist and processes the nested payload structure into a predictable, flat schema for downstream logic.

##### Nodes Involved
- `Lemlist Webhook`
- `Normalize Event`

##### Node Details

- **Lemlist Webhook**
  - **Type & Technical Role:** `n8n-nodes-base.webhook` (v2.1). Acts as the primary entry point, listening for incoming HTTP POST requests.
  - **Configuration Choices:** Uses HTTP method `POST` on the webhook path `lemlist-events`.
  - **Key Expressions / Variables:** Generates a production/test webhook URL based on the instance configuration.
  - **Input / Output:** Inputs from external Lemlist service; outputs raw body, headers, and query parameters to `Normalize Event`.
  - **Edge Cases & Failure Types:** Unauthenticated or improperly configured endpoints in Lemlist result in dropped requests or 404 errors.

- **Normalize Event**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2). Custom JavaScript transformer that flattens data structures, converts various timestamp formats to epoch milliseconds, extracts custom variables (`breakcold_lead_id`, `contact_id`), evaluates sequence step indices, and checks webhook secrets.
  - **Configuration Choices:** Contains a local configuration block (`SECRET`, `CLIENT_NAME`) that must be adjusted per installation. Iterates through all incoming items safely.
  - **Key Expressions / Variables:** Reads `body.lead`, `body.customVariables`, and headers such as `x-lemlist-secret`.
  - **Input / Output:** Input from `Lemlist Webhook`; output supplies normalized JSON objects containing fields like `event_type`, `occurred_at_ms`, `lead_email`, `breakcold_id`, `client`, and `secret_ok`.
  - **Edge Cases & Failure Types:** Malformed JSON bodies falling back to empty objects; unparseable timestamps defaulting to `Date.now()`.

---

#### 2.2 Filtering & Security

##### Overview
This block filters out noise by dropping non-lead payloads and enforces authorization checks if a shared webhook secret is configured.

##### Nodes Involved
- `Is Lead Event?`
- `Secret Check`

##### Node Details

- **Is Lead Event?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (v2.3). Conditional router designed to discard background or system-level webhook pings that lack core lead identifiers.
  - **Configuration Choices:** Evaluates strict boolean logic on the expression `{{ !!($json.campaign_id || $json.breakcold_id || $json.contact_id || $json.lead_email) }}`.
  - **Input / Output:** Input from `Normalize Event`; outputs `true` branch to `Secret Check`. (False branch terminates execution silently).
  - **Edge Cases & Failure Types:** Webhooks missing all metadata identifiers are dropped immediately.

- **Secret Check**
  - **Type & Technical Role:** `n8n-nodes-base.if` (v2.3). Security barrier validating payload authenticity against a predefined secret.
  - **Configuration Choices:** Evaluates loose type validation on `{{ $json.secret_ok }}`.
  - **Input / Output:** Input from `Is Lead Event?`; outputs `true` branch to `Event Type Router`.
  - **Edge Cases & Failure Types:** Mismatched webhook secrets will halt execution if `SECRET` is populated in the normalization script.

---

#### 2.3 Routing & CRM Sync

##### Overview
This block classifies events into supported categories, verifies the presence of a Breakcold CRM identifier, and sends structured custom activity data to the Breakcold REST API.

##### Nodes Involved
- `Event Type Router`
- `Has Breakcold ID?`
- `Breakcold: Post Custom Activity`

##### Node Details

- **Event Type Router**
  - **Type & Technical Role:** `n8n-nodes-base.switch` (v3.4). Multi-output router sorting incoming events by type.
  - **Configuration Choices:** Evaluates `{{ $json.event_type }}` against 12 recognized email and LinkedIn event names (`emailsSent`, `emailsOpened`, `emailsClicked`, `emailsReplied`, `emailsBounced`, `emailsUnsubscribed`, `linkedinSent`, `linkedinOpened`, `linkedinReplied`, `linkedinInviteDone`, `linkedinInviteAccepted`, `linkedinVisitDone`). Unrecognized events route to the fallback output.
  - **Input / Output:** Input from `Secret Check`; outputs recognized events to `Has Breakcold ID?` and unrecognized events to `Mark Unrecognized Event`.

- **Has Breakcold ID?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (v2.3). Validation gate ensuring the lead record is mapped to a Breakcold entity before attempting an API call.
  - **Configuration Choices:** Checks if `{{ $json.breakcold_id }}` is not empty.
  - **Input / Output:** Input from `Event Type Router`; routes valid IDs to `Breakcold: Post Custom Activity` and missing IDs to `Mark Unmatched – No Breakcold ID`.

- **Breakcold: Post Custom Activity**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (v4.4). REST API integration node communicating with Breakcold.
  - **Configuration Choices:** Sends a `POST` request to `https://rest.breakcold.com/api/v1/records/{{ $json.breakcold_id }}/custom-activities`. Configured with generic header authentication (`Authorization: Bearer <API_KEY>`). Includes retry logic (max 3 tries with a 2-second interval) and error redirection via *Continue (using error output)*.
  - **Key Expressions / Variables:** Constructs a JSON payload mapping `title` (event type + source), `descriptionPlain` (campaign, contact, step, email), and `startAt` (timestamp in milliseconds).
  - **Input / Output:** Input from `Has Breakcold ID?`; outputs successful responses to `Mark Matched – Success` and error outputs to `Mark Unmatched – Breakcold Error`.
  - **Edge Cases & Failure Types:** Rate limiting (HTTP 429), invalid bearer tokens (HTTP 401), or missing record IDs (HTTP 404) are caught by error routing without stopping the workflow.

---

#### 2.4 Audit Logging & Error Handling

##### Overview
This block collects results from all execution paths—successes, CRM errors, missing identifiers, and unrecognized event types—and appends structured tracking entries into Google Sheets.

##### Nodes Involved
- `Mark Matched – Success`
- `Mark Unmatched – Breakcold Error`
- `Mark Unmatched – No Breakcold ID`
- `Mark Unrecognized Event`
- `Append to Google Sheets (Events)`

##### Node Details

- **Mark Matched – Success**
  - **Type & Technical Role:** `n8n-nodes-base.set` (v3.4). Data preparation node formatting successful CRM synchronizations.
  - **Configuration Choices:** Sets `matched` to `true`, `error` to empty, and resolves ISO timestamps, channel identification (`email` vs `linkedin`), campaign metadata, and contact references.
  - **Input / Output:** Input from `Breakcold: Post Custom Activity` (success branch); output flows to `Append to Google Sheets (Events)`.

- **Mark Unmatched – Breakcold Error**
  - **Type & Technical Role:** `n8n-nodes-base.set` (v3.4). Data preparation node capturing API failure codes.
  - **Configuration Choices:** Sets `matched` to `false` and extracts error codes dynamically from the HTTP response payload or defaults to `breakcold_request_failed`.
  - **Input / Output:** Input from `Breakcold: Post Custom Activity` (error branch); output flows to `Append to Google Sheets (Events)`.

- **Mark Unmatched – No Breakcold ID**
  - **Type & Technical Role:** `n8n-nodes-base.set` (v3.4). Data preparation node for leads lacking CRM mapping.
  - **Configuration Choices:** Sets `matched` to `false` and writes the error string `no_breakcold_id`.
  - **Input / Output:** Input from `Has Breakcold ID?` (false branch); output flows to `Append to Google Sheets (Events)`.

- **Mark Unrecognized Event**
  - **Type & Technical Role:** `n8n-nodes-base.set` (v3.4). Data preparation node for unsupported webhook event types.
  - **Configuration Choices:** Sets `matched` to `false` and writes the error string `unrecognized_event_type`.
  - **Input / Output:** Input from `Event Type Router` (fallback branch); output flows to `Append to Google Sheets (Events)`.

- **Append to Google Sheets (Events)**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (v4.7). Destination integration node writing audit records.
  - **Configuration Choices:** Performs an `append` operation on the target spreadsheet using the `Events` sheet name. Maps fields explicitly (`timestamp`, `contact_id`, `campaign_id`, `client`, `channel`, `event_type`, `sequence_step`, `matched`, `error`). Configured with automatic retry logic (up to 5 attempts with a 3-second delay).
  - **Input / Output:** Input from all four `Mark` transformation nodes; writes final rows to the specified Google Sheet.
  - **Edge Cases & Failure Types:** Google API rate limits or invalid sheet names trigger automatic retries.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Lemlist Webhook** | n8n-nodes-base.webhook | Receives incoming POST webhooks from Lemlist | None | Normalize Event | Sync Lemlist email & LinkedIn activity to Breakcold CRM and log every event to Google Sheets. Outbound agencies and sales teams running campaigns in Lemlist can use this workflow to push every email and LinkedIn touchpoint (sends, opens, clicks, replies, bounces, invites accepted and more) to the matching lead's timeline in Breakcold in real time. Every event is also logged to a Google Sheet, giving you a full audit trail of campaign engagement and flagging events that could not be synced, so gaps get fixed instead of lost. |
| **Normalize Event** | n8n-nodes-base.code | Normalizes raw payloads, extracts metadata, custom variables, and timestamps | Lemlist Webhook | Is Lead Event? | Normalize Event: config & payload mapping. Edit only the CONFIG block at the top of the code: CLIENT_NAME written to the client column, SECRET optional webhook validation token. Lemlist sequenceStep is zero-indexed. |
| **Is Lead Event?** | n8n-nodes-base.if | Filters out non-lead operational webhooks | Normalize Event | Secret Check | Sync Lemlist email & LinkedIn activity to Breakcold CRM and log every event to Google Sheets... |
| **Secret Check** | n8n-nodes-base.if | Validates optional webhook shared secret | Is Lead Event? | Event Type Router | Sync Lemlist email & LinkedIn activity to Breakcold CRM and log every event to Google Sheets... |
| **Event Type Router** | n8n-nodes-base.switch | Routes events by type (12 recognized types vs fallback) | Secret Check | Has Breakcold ID?, Mark Unrecognized Event | Sync Lemlist email & LinkedIn activity to Breakcold CRM and log every event to Google Sheets... |
| **Has Breakcold ID?** | n8n-nodes-base.if | Validates whether the lead contains a Breakcold CRM identifier | Event Type Router | Breakcold: Post Custom Activity, Mark Unmatched – No Breakcold ID | Sync Lemlist email & LinkedIn activity to Breakcold CRM and log every event to Google Sheets... |
| **Breakcold: Post Custom Activity** | n8n-nodes-base.httpRequest | Posts custom activity timeline events to the Breakcold REST API | Has Breakcold ID? | Mark Matched – Success, Mark Unmatched – Breakcold Error | Breakcold sync & error handling. Endpoint: `POST https://rest.breakcold.com/api/v1/records/{breakcold_lead_id}/custom-activities`. Auth: Header Auth credential. Error handling uses continueErrorOutput to ensure failed CRM calls do not block execution. |
| **Mark Matched – Success** | n8n-nodes-base.set | Formats success payload for the audit log | Breakcold: Post Custom Activity | Append to Google Sheets (Events) | Sync Lemlist email & LinkedIn activity to Breakcold CRM and log every event to Google Sheets... |
| **Mark Unmatched – Breakcold Error** | n8n-nodes-base.set | Formats error payload when the Breakcold API fails | Breakcold: Post Custom Activity | Append to Google Sheets (Events) | Breakcold sync & error handling... |
| **Mark Unmatched – No Breakcold ID** | n8n-nodes-base.set | Formats payload when lead lacks a Breakcold identifier | Has Breakcold ID? | Append to Google Sheets (Events) | Sync Lemlist email & LinkedIn activity to Breakcold CRM and log every event to Google Sheets... |
| **Mark Unrecognized Event** | n8n-nodes-base.set | Formats payload for unsupported event types | Event Type Router | Append to Google Sheets (Events) | Sync Lemlist email & LinkedIn activity to Breakcold CRM and log every event to Google Sheets... |
| **Append to Google Sheets (Events)** | n8n-nodes-base.googleSheets | Appends all event outcomes to an audit tracking sheet | Mark Matched – Success, Mark Unmatched – Breakcold Error, Mark Unmatched – No Breakcold ID, Mark Unrecognized Event | None | Sync Lemlist email & LinkedIn activity to Breakcold CRM and log every event to Google Sheets... |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create a Webhook Node**
   - Add a **Webhook** node named `Lemlist Webhook`.
   - Set HTTP Method to `POST`.
   - Define path as `lemlist-events`.

2. **Create the Normalization Code Node**
   - Add a **Code** node named `Normalize Event`.
   - Connect `Lemlist Webhook` output to `Normalize Event`.
   - Paste the normalization script into the JavaScript editor, updating `CLIENT_NAME` and optionally `SECRET`.

3. **Add Lead Validation Condition**
   - Add an **If** node named `Is Lead Event?`.
   - Connect `Normalize Event` output to `Is Lead Event?`.
   - Configure condition expression: `{{ !!($json.campaign_id || $json.breakcold_id || $json.contact_id || $json.lead_email) }}`.

4. **Add Secret Verification Condition**
   - Add an **If** node named `Secret Check`.
   - Connect `Is Lead Event?` (true branch) to `Secret Check`.
   - Configure condition expression: `{{ $json.secret_ok }}`.

5. **Add Event Routing Switch**
   - Add a **Switch** node named `Event Type Router`.
   - Connect `Secret Check` (true branch) to `Event Type Router`.
   - Create 12 string equality rules matching `{{ $json.event_type }}` against: `emailsSent`, `emailsOpened`, `emailsClicked`, `emailsReplied`, `emailsBounced`, `emailsUnsubscribed`, `linkedinSent`, `linkedinOpened`, `linkedinReplied`, `linkedinInviteDone`, `linkedinInviteAccepted`, `linkedinVisitDone`. Set fallback output name to `Unrecognized`.

6. **Add Breakcold ID Validation Gate**
   - Add an **If** node named `Has Breakcold ID?`.
   - Connect `Event Type Router` (Recognized output) to `Has Breakcold ID?`.
   - Configure condition: `breakcold_id` is not empty.

7. **Configure Breakcold HTTP Request Node**
   - Add an **HTTP Request** node named `Breakcold: Post Custom Activity`.
   - Connect `Has Breakcold ID?` (true branch) to `Breakcold: Post Custom Activity`.
   - Set Method to `POST`, URL to `https://rest.breakcold.com/api/v1/records/{{ $json.breakcold_id }}/custom-activities`.
   - Set Authentication to **Header Auth**, creating a credential with Name `Authorization` and Value `Bearer YOUR_BREAKCOLD_API_KEY`.
   - Add header: `Content-Type: application/json`.
   - Enable JSON body specification with the mapping: 
     `{{ { "title": $json.event_type + " (Lemlist)", "descriptionPlain": "campaign_id=" + $json.campaign_id + " contact_id=" + $json.contact_id + " step=" + $json.sequence_step + " email=" + $json.lead_email, "startAt": $json.occurred_at_ms } }}`.
   - Configure error handling (*On Error*) to **Continue (using error output)** and enable retries (3 tries, 2000ms wait).

8. **Create Audit Log Data Preparation Nodes**
   - Add four **Set** (Edit Fields) nodes:
     - **Mark Matched – Success**: Sets `matched` to `true`, `error` to `""`, along with timestamp, contact ID, campaign ID, client name, channel calculation (`email` or `linkedin`), event type, and sequence step.
     - **Mark Unmatched – Breakcold Error**: Sets `matched` to `false`, and `error` extracted from `{{ $json.error?.code || ... }}`.
     - **Mark Unmatched – No Breakcold ID**: Sets `matched` to `false`, and `error` to `"no_breakcold_id"`.
     - **Mark Unrecognized Event**: Sets `matched` to `false`, and `error` to `"unrecognized_event_type"`.

9. **Connect Data Preparation Nodes**
   - Connect `Breakcold: Post Custom Activity` (main success output) to `Mark Matched – Success`.
   - Connect `Breakcold: Post Custom Activity` (error output) to `Mark Unmatched – Breakcold Error`.
   - Connect `Has Breakcold ID?` (false branch) to `Mark Unmatched – No Breakcold ID`.
   - Connect `Event Type Router` (Unrecognized output) to `Mark Unrecognized Event`.

10. **Configure Google Sheets Destination Node**
    - Add a **Google Sheets** node named `Append to Google Sheets (Events)`.
    - Connect the outputs of all four `Mark` nodes into `Append to Google Sheets (Events)`.
    - Authenticate using your Google account credentials.
    - Set Operation to `Append`, select your target Document, and choose the sheet named `Events`.
    - Map columns explicitly: `timestamp`, `contact_id`, `campaign_id`, `client`, `channel`, `event_type`, `sequence_step`, `matched`, and `error`. Enable retry on fail (5 attempts, 3000ms delay).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Lemlist Webhook Setup** | Configure webhooks under Lemlist settings (`Settings > Integrations > Webhooks`) pointing to the production URL generated by the `Lemlist Webhook` node. |
| **Breakcold API Rate Limits** | Breakcold enforces a limit of 120 requests/min per route. The HTTP request node is configured with retry intervals to accommodate traffic bursts. |
| **Google Sheets Schema Requirement** | The target spreadsheet must contain a tab named `Events` with row 1 headers matching: `timestamp`, `contact_id`, `campaign_id`, `client`, `channel`, `event_type`, `sequence_step`, `matched`, `error`. |
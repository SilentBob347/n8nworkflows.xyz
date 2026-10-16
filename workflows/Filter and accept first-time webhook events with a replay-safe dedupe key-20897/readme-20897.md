Filter and accept first-time webhook events with a replay-safe dedupe key

https://n8nworkflows.xyz/workflows/filter-and-accept-first-time-webhook-events-with-a-replay-safe-dedupe-key-20897


# Filter and accept first-time webhook events with a replay-safe dedupe key

### 1. Workflow Overview

This workflow acts as an API gateway shield that receives incoming HTTP POST webhooks, validates their structure, normalizes critical identifiers, and filters out duplicate deliveries (replay attacks or retries) to ensure that downstream services process each unique event precisely once.

The workflow logic is categorized into three functional blocks:
- **1.1 Input Reception & Validation:** Captures raw webhook payloads via an HTTP endpoint, executes programmatic validation against required envelope formats, and routes the execution based on validation outcomes.
- **1.2 Replay Protection & Success Path:** Deduplicates incoming valid requests using a stateful memory check across previous executions (`source:event_id`), formats a success payload, and responds with an HTTP 202 Accepted status for first-time deliveries.
- **1.3 Error Handling & Rejection Path:** Captures invalid event structures, formats a structured error response payload, and returns an HTTP 422 Unprocessable Entity status.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
- **Overview:** This block receives the raw webhook payload, performs syntax and business validation checks on the event envelope (such as checking `event_id`/`eventId` and `source`/`origin`), and branches the flow depending on whether the data conforms to required specifications.
- **Nodes Involved:**
  - `When Event Received`
  - `Validate Event Contents`
  - `If Event is Valid`
- **Node Details:**
  - **When Event Received**
    - *Type & Technical Role:* `n8n-nodes-base.webhook` (Trigger)
    - *Configuration:* Listens for incoming HTTP `POST` requests on path `webhook-replay-shield` using manual response control (`responseNode`).
    - *Key Expressions:* None (relies on path settings).
    - *Input/Output:* Output connects to `Validate Event Contents`.
    - *Version:* 2.1
    - *Edge Cases/Failures:* HTTP timeout if downstream nodes take too long to respond; invalid routing if the path is requested via incorrect HTTP methods (e.g., GET).
  - **Validate Event Contents**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation / Validation)
    - *Configuration:* Executes JavaScript to inspect the payload body. Checks for allowed field lengths, proper regex patterns for event IDs and sources, and a maximum threshold of top-level properties.
    - *Key Expressions:* Accesses input via `const input = $input.first().json;`.
    - *Input/Output:* Inputs from `When Event Received`; output connects to `If Event is Valid`.
    - *Version:* 2.0
    - *Edge Cases/Failures:* Throws JavaScript execution errors if the input body is not a valid JSON object or contains unexpected types.
  - **If Event is Valid**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Evaluates a strict boolean check against the `valid` property produced by the preceding code node.
    - *Key Expressions:* `={{ $json.valid }}` (Right value: `true`).
    - *Input/Output:* Inputs from `Validate Event Contents`; True branch connects to `Deduplicate by Event Keys`, False branch connects to `Handle Invalid Event Error`.
    - *Version:* 2.2
    - *Edge Cases/Failures:* Expression evaluation failure if previous node output schema changes unexpectedly.

---

#### 2.2 Replay Protection & Success Path
- **Overview:** This block tracks unique event identifiers across workflow executions to identify duplicate submissions, formats a success metadata object, and returns an HTTP 202 response to the client.
- **Nodes Involved:**
  - `Deduplicate by Event Keys`
  - `Generate First Response Code`
  - `Send 202 Status for First Seen`
- **Node Details:**
  - **Deduplicate by Event Keys**
    - *Type & Technical Role:* `n8n-nodes-base.removeDuplicates` (Filter / Stateful Deduplication)
    - *Configuration:* Removes items with previously seen key values using node-level history scope with a history size limit of 10,000 items.
    - *Key Expressions:* `={{ $json.eventKey }}`
    - *Input/Output:* Inputs from `If Event is Valid` (True branch); output connects to `Generate First Response Code`.
    - *Version:* 2.0
    - *Edge Cases/Failures:* Memory exhaustion if history size is set excessively high; key collisions if different event sources generate overlapping IDs without proper prefixing.
  - **Generate First Response Code**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration:* Extracts event metadata and builds a standardized success JSON response payload.
    - *Key Expressions:* Extracts `{eventId, source, eventKey}` from `$input.first().json`.
    - *Input/Output:* Inputs from `Deduplicate by Event Keys`; output connects to `Send 202 Status for First Seen`.
    - *Version:* 2.0
    - *Edge Cases/Failures:* Reference errors if required properties are missing from the input stream.
  - **Send 202 Status for First Seen**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Response Handler)
    - *Configuration:* Sends an HTTP response with status code `202` and responds with the JSON body provided by the upstream code node.
    - *Key Expressions:* `={{ $json }}`
    - *Input/Output:* Inputs from `Generate First Response Code`; no downstream output.
    - *Version:* 1.5
    - *Edge Cases/Failures:* Webhook connection drops before the HTTP response is transmitted back to the client.

---

#### 2.3 Error Handling & Rejection Path
- **Overview:** This block processes failed validations, prepares a structured error description payload, and returns an HTTP 422 Unprocessable Entity status to the calling client.
- **Nodes Involved:**
  - `Handle Invalid Event Error`
  - `Send 422 Status for Invalid Event`
- **Node Details:**
  - **Handle Invalid Event Error**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration:* Extracts error metadata from the invalid event branch and formats a rejection payload object.
    - *Key Expressions:* Extracts `{error}` from `$input.first().json`.
    - *Input/Output:* Inputs from `If Event is Valid` (False branch); output connects to `Send 422 Status for Invalid Event`.
    - *Version:* 2.0
    - *Edge Cases/Failures:* Undefined error propagation if validation structure is modified.
  - **Send 422 Status for Invalid Event**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Response Handler)
    - *Configuration:* Sends an HTTP response with status code `422` and returns the formatted error JSON body.
    - *Key Expressions:* `={{ $json }}`
    - *Input/Output:* Inputs from `Handle Invalid Event Error`; no downstream output.
    - *Version:* 1.5
    - *Edge Cases/Failures:* Client disconnects prematurely.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation & setup instructions | None | None | Webhook Replay Shield - Safe Single-Output Replay Filter<br><br>### How it works<br><br>This workflow receives webhook events, validates that the event envelope is well formed, and branches based on validity. Valid events pass through a replay shield that remembers previously seen event keys so only first deliveries continue to the success response. Invalid events are formatted into an error payload and returned with a 422 response.<br><br>### Setup steps<br><br>- Configure the webhook trigger URL and ensure the upstream service sends events to it.<br>- Review the validation code to match the expected event envelope, required fields, timestamps, and any signature or secret checks.<br>- Configure the duplicate-removal node to use the correct event key or idempotency key, and set an appropriate retention window for replay protection.<br>- Verify the Respond to Webhook nodes are using the intended HTTP status codes and response bodies.<br><br>### Customization<br><br>Adjust the idempotency key, validation rules, retention period, and response payloads/status codes to match the webhook provider’s retry and replay semantics. |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Block documentation label | None | None | ## Receive and validate event<br><br>Receives the incoming webhook, checks the event envelope, and routes the flow based on whether the event is valid. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Block documentation label | None | None | ## Accept first delivery<br><br>Handles the valid-event branch by remembering seen event keys, preparing the first-delivery response, and returning a 202 response for accepted events. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Block documentation label | None | None | ## Reject invalid events<br><br>Handles invalid events by creating a rejection payload and returning a 422 webhook response. |
| `When Event Received` | `n8n-nodes-base.webhook` | Receives incoming POST webhooks | None | `Validate Event Contents` | Receive and validate event |
| `Validate Event Contents` | `n8n-nodes-base.code` | Validates envelope structure and normalizes IDs | `When Event Received` | `If Event is Valid` | Receive and validate event |
| `If Event is Valid` | `n8n-nodes-base.if` | Routes execution based on validation flag | `Validate Event Contents` | `Deduplicate by Event Keys`, `Handle Invalid Event Error` | Receive and validate event |
| `Deduplicate by Event Keys` | `n8n-nodes-base.removeDuplicates` | Filters out duplicate event keys seen in previous executions | `If Event is Valid` | `Generate First Response Code` | Accept first delivery |
| `Generate First Response Code` | `n8n-nodes-base.code` | Prepares success metadata payload | `Deduplicate by Event Keys` | `Send 202 Status for First Seen` | Accept first delivery |
| `Handle Invalid Event Error` | `n8n-nodes-base.code` | Formats validation error payload | `If Event is Valid` | `Send 422 Status for Invalid Event` | Reject invalid events |
| `Send 202 Status for First Seen` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 202 response for accepted events | `Generate First Response Code` | None | Accept first delivery |
| `Send 422 Status for Invalid Event` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 422 response for rejected events | `Handle Invalid Event Error` | None | Reject invalid events |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node named `When Event Received` (TypeVersion: `2.1`).
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `webhook-replay-shield`.
   - Set **Response Mode** to `Response Node`.

2. **Add Event Validation Logic:**
   - Add a **Code** node named `Validate Event Contents` (TypeVersion: `2.0`).
   - Connect `When Event Received` output to `Validate Event Contents`.
   - Insert JavaScript code to extract and validate `event_id`/`eventId` and `source`/`origin` properties against defined regular expression constraints and maximum top-level property limits. Return an object containing `valid`, `eventId`, `source`, `eventKey`, and `error`.

3. **Add Conditional Branching:**
   - Add an **If** node named `If Event is Valid` (TypeVersion: `2.2`).
   - Connect `Validate Event Contents` output to `If Event is Valid`.
   - Set condition: Left Value `={{ $json.valid }}`, Operator `Boolean: True`.

4. **Configure the Success Path (Replay Shield & Response):**
   - Add a **Remove Duplicates** node named `Deduplicate by Event Keys` (TypeVersion: `2.0`).
   - Connect the **True** output of `If Event is Valid` to `Deduplicate by Event Keys`.
   - Configure **Operation** to `Remove items seen in previous executions`, set **Scope** to `Node`, set **History Size** to `10000`, and set **Deduplication Value** to `={{ $json.eventKey }}`.
   - Add a **Code** node named `Generate First Response Code` (TypeVersion: `2.0`).
   - Connect `Deduplicate by Event Keys` output to `Generate First Response Code`.
   - Insert JavaScript code to build a success metadata response object containing `ok: true`, `status: "accepted"`, and deduplication keys.
   - Add a **Respond to Webhook** node named `Send 202 Status for First Seen` (TypeVersion: `1.5`).
   - Connect `Generate First Response Code` output to `Send 202 Status for First Seen`.
   - Configure **Response Code** to `202`, **Respond With** to `JSON`, and **Response Body** to `={{ $json }}`.

5. **Configure the Error Path (Rejection):**
   - Add a **Code** node named `Handle Invalid Event Error` (TypeVersion: `2.0`).
   - Connect the **False** output of `If Event is Valid` to `Handle Invalid Event Error`.
   - Insert JavaScript code to extract the `error` object and format a rejection response.
   - Add a **Respond to Webhook** node named `Send 422 Status for Invalid Event` (TypeVersion: `1.5`).
   - Connect `Handle Invalid Event Error` output to `Send 422 Status for Invalid Event`.
   - Configure **Response Code** to `422`, **Respond With** to `JSON`, and **Response Body** to `={{ $json }}`.

6. **Final Settings:**
   - Configure workflow settings timezone to `UTC` and execution order to `v1`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Workflow metadata and disclaimer | The provided text originates exclusively from an automated n8n workflow and complies fully with content safety policies. |
| Execution settings | Timezone is configured to UTC with execution order v1. |
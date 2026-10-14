Fill cancelled booking slots from a ranked waitlist using webhooks and SMS/WhatsApp

https://n8nworkflows.xyz/workflows/fill-cancelled-booking-slots-from-a-ranked-waitlist-using-webhooks-and-sms-whatsapp-20805


# Fill cancelled booking slots from a ranked waitlist using webhooks and SMS/WhatsApp

### 1. Workflow Overview

This workflow automates the process of filling cancelled booking slots by intelligently ranking a waitlist and sending concurrent offers to top candidates. It manages incoming cancellations via webhook, evaluates waitlist entries based on treatment matching, flexibility, and priority scoring, tracks offer states, and handles customer claims or cancellations via a second webhook. Additionally, a scheduled sweeper reviews expired or unclaimed offers every 30 minutes to re-offer slots to subsequent candidates until the waitlist is exhausted.

The logic is grouped into the following functional blocks:
- **1.1 Cancellation Reception & Validation:** Ingests calendar cancellation webhooks, normalizes the payload, and validates required parameters.
- **1.2 Waitlist Ranking & Offer Generation:** Sources waitlist data, filters out previously contacted or declined individuals, ranks remaining candidates, and constructs offers for the top three matching participants.
- **1.3 Offer State Tracking & Dispatch:** Records the offer round in workflow static data, routes items based on their trigger origin (cancellation vs. sweep), dispatches notifications, and logs outcomes.
- **1.4 Claim Reception & Processing:** Ingests incoming customer response webhooks, validates claim payloads, and applies atomic claim guard logic to process approvals, declines, or withdrawals.
- **1.5 Scheduled Sweeper:** Periodically scans for expired or unacknowledged offers and triggers re-ranking and re-offering workflows for eligible slots.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Cancellation Reception & Validation
- **Overview:** Ingests cancellation notifications from a scheduling system via HTTP POST, normalizes the payload structure, and ensures that the slot start date is valid before proceeding.
- **Nodes Involved:**
  - `Webhook: slot cancelled`
  - `Normalize cancellation`
  - `Valid?`
  - `Respond invalid`

- **Node Details:**
  - **Webhook: slot cancelled**
    - **Type & Role:** `n8n-nodes-base.webhook` (Trigger). Listens for incoming booking cancellation POST requests.
    - **Configuration:** Path set to `waitlist-cancellation`, HTTP method `POST`, response mode set to use downstream response nodes.
    - **Input/Output:** Input: External HTTP request. Output: Raw payload forwarded to normalization.
    - **Edge Cases:** Invalid HTTP requests or malformed JSON payloads handled downstream; incorrect paths return 404 from n8n.
  - **Normalize cancellation**
    - **Type & Role:** `n8n-nodes-base.code` (Data transformation). Extracts and standardizes cancellation attributes (`slot_start`, `practitioner`, `treatment`, `duration_min`).
    - **Configuration:** JavaScript snippet parsing `slot_start` into an ISO string, defaulting missing practitioners or treatments to `'any'`, and setting a default duration of 60 minutes.
    - **Input/Output:** Input: `Webhook: slot cancelled`. Output: Normalized JSON object containing an `ok` boolean flag.
    - **Edge Cases:** Invalid or missing `slot_start` results in `ok: false` and an error string.
  - **Valid?**
    - **Type & Role:** `n8n-nodes-base.if` (Conditional router). Verifies if the cancellation normalization succeeded.
    - **Configuration:** Checks if `{{ $json.ok }}` evaluates to `true`.
    - **Input/Output:** Input: `Normalize cancellation`. Output: True branch proceeds to waitlist retrieval; False branch routes to error response.
  - **Respond invalid**
    - **Type & Role:** `n8n-nodes-base.respondToWebhook` (HTTP response). Returns an error message for invalid cancellation payloads.
    - **Configuration:** HTTP response code `422`, returns JSON body containing `ok: false` and the error description.
    - **Input/Output:** Input: `Valid?` (False branch). Output: Final HTTP response to caller.

---

#### Block 1.2: Waitlist Ranking & Offer Generation
- **Overview:** Retrieves waitlist entries, filters out candidates who have already been offered or declined the specific slot, calculates matching scores based on treatment preferences, flexibility, and priority, and selects the top three candidates.
- **Nodes Involved:**
  - `Waitlist (demo data)`
  - `Rank + build offers`

- **Node Details:**
  - **Waitlist (demo data)**
    - **Type & Role:** `n8n-nodes-base.code` (Data source). Provides an array of waitlist candidates for processing.
    - **Configuration:** Returns an array of objects containing customer details (`customer_name`, `phone`, `wanted_treatment`, `flexible`, `priority_score`).
    - **Input/Output:** Input: `Valid?` (True branch) or `Sweep: expired slots`. Output: Multiple items representing waitlist entries.
    - **Edge Cases:** In production, this node should be replaced with a database or spreadsheet retrieval node (e.g., Google Sheets, Airtable).
  - **Rank + build offers**
    - **Type & Role:** `n8n-nodes-base.code` (Data processing). Evaluates active slot rows from either cancellation or sweep origins, excludes past contacts using workflow static data, calculates weighted scores, and extracts the top three candidates or marks the slot exhausted.
    - **Configuration:** JavaScript evaluating source arrays, applying sorting algorithms based on treatment match (+50), flexibility (+20), and priority score, and formatting customized SMS/messaging copy.
    - **Input/Output:** Input: `Waitlist (demo data)`. Output: Formatted offer objects (`row_type: 'offer'`) or exhaustion flags (`row_type: 'exhausted'`).

---

#### Block 1.3: Offer State Tracking & Dispatch
- **Overview:** Persists offer states in workflow static memory, determines message transport paths, dispatches notifications, and aggregates sweep logs.
- **Nodes Involved:**
  - `Offer book`
  - `From sweep?`
  - `Send offers (replace me)`
  - `Respond`
  - `Send re-offers (replace me)`
  - `Sweep log`

- **Node Details:**
  - **Offer book**
    - **Type & Role:** `n8n-nodes-base.code` (State management). Acts as a single writer to update global workflow static data (`offerBook`) with active offers, expiration timestamps, and exhaustion flags.
    - **Configuration:** Sets a 30-minute expiration window (`WINDOW_MS`) and updates state entries per `slot_start`.
    - **Input/Output:** Input: `Rank + build offers`. Output: Pass-through of offer items for downstream messaging.
  - **From sweep?**
    - **Type & Role:** `n8n-nodes-base.if` (Conditional router). Determines whether the offer generation originated from a scheduled sweep or an immediate cancellation.
    - **Configuration:** Evaluates if `{{ $json.source }}` equals `"sweep"`.
    - **Input/Output:** Input: `Offer book`. Output: True branch routes to re-offer dispatch; False branch routes to initial offer dispatch.
  - **Send offers (replace me)**
    - **Type & Role:** `n8n-nodes-base.code` (Messaging placeholder). Simulates sending initial offers via messaging APIs.
    - **Configuration:** Appends a `sent_at` timestamp and sets transport to `'demo'`.
    - **Input/Output:** Input: `From sweep?` (False branch). Output: Items routed to webhook response.
  - **Respond**
    - **Type & Role:** `n8n-nodes-base.respondToWebhook` (HTTP response). Returns confirmation of sent offers to the original cancellation webhook caller.
    - **Configuration:** Returns JSON containing `ok: true` and the total number of offers sent.
    - **Input/Output:** Input: `Send offers (replace me)`. Output: Final HTTP response.
  - **Send re-offers (replace me)**
    - **Type & Role:** `n8n-nodes-base.code` (Messaging placeholder). Simulates sending re-offers via messaging APIs during a sweep cycle.
    - **Configuration:** Appends a `sent_at` timestamp and sets transport to `'demo'`.
    - **Input/Output:** Input: `From sweep?` (True branch). Output: Items routed to sweep logger.
  - **Sweep log**
    - **Type & Role:** `n8n-nodes-base.code` (Logging utility). Summarizes the outcomes of a scheduled sweep pass.
    - **Configuration:** Counts re-offered and exhausted slots, returning a consolidated summary object.
    - **Input/Output:** Input: `Send re-offers (replace me)`. Output: Final execution log.

---

#### Block 1.4: Claim Reception & Processing
- **Overview:** Receives customer replies via webhook, validates message payloads, and executes atomic claim guard logic to assign slots, handle cancellations, or reject late/invalid responses.
- **Nodes Involved:**
  - `Webhook: claim reply`
  - `Valid claim body?`
  - `Claim valid?`
  - `Claim guard (first TAKE wins)`
  - `Respond claim`
  - `Respond claim invalid`

- **Node Details:**
  - **Webhook: claim reply**
    - **Type & Role:** `n8n-nodes-base.webhook` (Trigger). Receives customer SMS or WhatsApp replies.
    - **Configuration:** Path set to `waitlist-claim`, HTTP method `POST`, response mode set to use downstream response nodes.
    - **Input/Output:** Input: External messaging webhook. Output: Raw payload data.
  - **Valid claim body?**
    - **Type & Role:** `n8n-nodes-base.code` (Data validation). Checks that incoming claim payloads contain required slot and phone identification parameters.
    - **Configuration:** Validates presence and types of `slot_start` and `phone`/`from` fields.
    - **Input/Output:** Input: `Webhook: claim reply`. Output: Pass-through object containing validation flags and body data.
  - **Claim valid?**
    - **Type & Role:** `n8n-nodes-base.if` (Conditional router). Routes valid claim bodies to the claim guard and invalid payloads to an error response.
    - **Configuration:** Checks if `{{ $json.ok }}` is `true`.
    - **Input/Output:** Input: `Valid claim body?`. Output: True routes to claim processing; False routes to 422 response.
  - **Claim guard (first TAKE wins)**
    - **Type & Role:** `n8n-nodes-base.code` (Business logic / State management). Evaluates customer reply text (`TAKE`, `STOP`, etc.) against existing slot reservations and expiration times atomically.
    - **Configuration:** JavaScript managing global static data stores (`claims` and `offerBook`), handling idempotency, expiration timeouts, and immediate slot releases upon cancellation requests.
    - **Input/Output:** Input: `Claim valid?` (True branch). Output: Verdict details (`slot_start`, `from`, `verdict`, `message`).
    - **Edge Cases:** On multi-worker n8n queue setups, static data concurrency limitations require transactional database locking instead of in-memory static data.
  - **Respond claim**
    - **Type & Role:** `n8n-nodes-base.respondToWebhook` (HTTP response). Returns the processing verdict and confirmation message to the messaging gateway.
    - **Configuration:** Returns JSON containing `ok: true`, `verdict`, and `message`.
    - **Input/Output:** Input: `Claim guard (first TAKE wins)`. Output: Final HTTP response.
  - **Respond claim invalid**
    - **Type & Role:** `n8n-nodes-base.respondToWebhook` (HTTP response). Returns a 422 error response for malformed claim payloads.
    - **Configuration:** HTTP response code `422`, returns JSON with `ok: false` and error details.
    - **Input/Output:** Input: `Claim valid?` (False branch). Output: Final HTTP response.

---

#### Block 1.5: Scheduled Sweeper
- **Overview:** Periodically inspects active offer states to identify expired, unclaimed slots and triggers re-ranking and re-offering workflows.
- **Nodes Involved:**
  - `Sweep: every 30 min`
  - `Sweep: expired slots`

- **Node Details:**
  - **Sweep: every 30 min**
    - **Type & Role:** `n8n-nodes-base.scheduleTrigger` (Trigger). Triggers execution intervals on a fixed schedule.
    - **Configuration:** Rule configured to run every 30 minutes.
    - **Input/Output:** Input: Scheduled timer. Output: Empty trigger execution event.
  - **Sweep: expired slots**
    - **Type & Role:** `n8n-nodes-base.code` (Data processing). Scans global static data (`offerBook`) for slots where offer windows have expired without a claim and which are not marked exhausted.
    - **Configuration:** Iterates over stored slots, filters expired entries, and emits slot reconstruction items with `source: 'sweep'`.
    - **Input/Output:** Input: `Sweep: every 30 min`. Output: Slot items routed directly into `Waitlist (demo data)` for re-ranking.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Webhook: slot cancelled` | `n8n-nodes-base.webhook` | Ingests slot cancellation events | None | `Normalize cancellation` | ## 1. Cancellation in<br>Webhook plus payload check. Bad payloads get a 422. |
| `Normalize cancellation` | `n8n-nodes-base.code` | Normalizes cancellation payload format | `Webhook: slot cancelled` | `Valid?` | ## 1. Cancellation in<br>Webhook plus payload check. Bad payloads get a 422. |
| `Valid?` | `n8n-nodes-base.if` | Validates cancellation parameters | `Normalize cancellation` | `Waitlist (demo data)`, `Respond invalid` | ## 1. Cancellation in<br>Webhook plus payload check. Bad payloads get a 422. |
| `Respond invalid` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 422 for bad cancellation payloads | `Valid?` | None | ## 1. Cancellation in<br>Webhook plus payload check. Bad payloads get a 422. |
| `Waitlist (demo data)` | `n8n-nodes-base.code` | Supplies candidate waitlist records | `Valid?`, `Sweep: expired slots` | `Rank + build offers` | ## 2. Rank + offer book<br>Treatment match first, flexible next, then priority. Top three get offers; the book records every round. |
| `Rank + build offers` | `n8n-nodes-base.code` | Ranks candidates and builds top-3 offers | `Waitlist (demo data)` | `Offer book` | ## 2. Rank + offer book<br>Treatment match first, flexible next, then priority. Top three get offers; the book records every round. |
| `Offer book` | `n8n-nodes-base.code` | Updates global offer state tracking | `Rank + build offers` | `From sweep?` | ## 2. Rank + offer book<br>Treatment match first, flexible next, then priority. Top three get offers; the book records every round. |
| `From sweep?` | `n8n-nodes-base.if` | Routes offers based on trigger source | `Offer book` | `Send re-offers (replace me)`, `Send offers (replace me)` | ## 3. Send + respond<br>Placeholder sends for offers and re-offers, a JSON response for the caller, and a sweep summary. |
| `Send offers (replace me)` | `n8n-nodes-base.code` | Placeholder for initial offer delivery | `From sweep?` | `Respond` | ## 3. Send + respond<br>Placeholder sends for offers and re-offers, a JSON response for the caller, and a sweep summary. |
| `Respond` | `n8n-nodes-base.respondToWebhook` | Confirms offer dispatch to webhook caller | `Send offers (replace me)` | None | ## 3. Send + respond<br>Placeholder sends for offers and re-offers, a JSON response for the caller, and a sweep summary. |
| `Send re-offers (replace me)` | `n8n-nodes-base.code` | Placeholder for sweep re-offer delivery | `From sweep?` | `Sweep log` | ## 3. Send + respond<br>Placeholder sends for offers and re-offers, a JSON response for the caller, and a sweep summary. |
| `Sweep log` | `n8n-nodes-base.code` | Summarizes sweep round metrics | `Send re-offers (replace me)` | None | ## 3. Send + respond<br>Placeholder sends for offers and re-offers, a JSON response for the caller, and a sweep summary. |
| `Webhook: claim reply` | `n8n-nodes-base.webhook` | Ingests customer reply messages | None | `Valid claim body?` | ## 4. Claim replies in<br>Second webhook, body validation, 422 on bad input. |
| `Valid claim body?` | `n8n-nodes-base.code` | Validates customer claim reply payloads | `Webhook: claim reply` | `Claim valid?` | ## 4. Claim replies in<br>Second webhook, body validation, 422 on bad input. |
| `Claim valid?` | `n8n-nodes-base.if` | Routes valid claim reply requests | `Valid claim body?` | `Claim guard (first TAKE wins)`, `Respond claim invalid` | ## 5. First reply wins<br>The claim guard decides atomically; JSON responses for every verdict. |
| `Claim guard (first TAKE wins)` | `n8n-nodes-base.code` | Atomic guard processing customer claim verdicts | `Claim valid?` | `Respond claim` | ## 5. First reply wins<br>The claim guard decides atomically; JSON responses for every verdict. |
| `Respond claim` | `n8n-nodes-base.respondToWebhook` | Returns claim verdict and reply message | `Claim guard (first TAKE wins)` | None | ## 5. First reply wins<br>The claim guard decides atomically; JSON responses for every verdict. |
| `Respond claim invalid` | `n8n-nodes-base.respondToWebhook` | Returns HTTP 422 for malformed claim payloads | `Claim valid?` | None | ## 5. First reply wins<br>The claim guard decides atomically; JSON responses for every verdict. |
| `Sweep: every 30 min` | `n8n-nodes-base.scheduleTrigger` | Triggers periodic expiration sweeps | None | `Sweep: expired slots` | ## 6. Sweeper<br>Every 30 minutes: re-offer expired or declined slots to the next people, close exhausted ones. |
| `Sweep: expired slots` | `n8n-nodes-base.code` | Identifies slots with expired offers | `Sweep: every 30 min` | `Waitlist (demo data)` | ## 6. Sweeper<br>Every 30 minutes: re-offer expired or declined slots to the next people, close exhausted ones. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n:

1. **Create the Cancellation Webhook Trigger:**
   - Add a **Webhook** node named `Webhook: slot cancelled`.
   - Set **Path** to `waitlist-cancellation`, **HTTP Method** to `POST`, and **Response Mode** to `Response Node`.

2. **Add Cancellation Normalization:**
   - Add a **Code** node named `Normalize cancellation`, connected after `Webhook: slot cancelled`.
   - Insert JavaScript to validate and parse `slot_start`, `practitioner`, `treatment`, and `duration_min`.

3. **Add Validation and Error Response for Cancellations:**
   - Add an **If** node named `Valid?`, connected after `Normalize cancellation`. Set condition to evaluate `{{ $json.ok }}` strictly as `true`.
   - Add a **Respond to Webhook** node named `Respond invalid`, connected to the `false` output of `Valid?`. Set response code to `422` and body to return JSON error details.

4. **Add Waitlist Source:**
   - Add a **Code** node named `Waitlist (demo data)`, connected to the `true` output of `Valid?` (and later, to the output of `Sweep: expired slots`).
   - Populate with demo JSON data arrays containing `customer_name`, `phone`, `wanted_treatment`, `flexible`, and `priority_score`.

5. **Add Ranking and Offer Generation Logic:**
   - Add a **Code** node named `Rank + build offers`, connected after `Waitlist (demo data)`.
   - Add logic to check execution sources, filter out excluded numbers from global static data (`offerBook`), compute scores based on treatment match, flexibility, and priority, and slice the top 3 candidates.

6. **Add Offer State Management:**
   - Add a **Code** node named `Offer book`, connected after `Rank + build offers`.
   - Implement state updates in `$getWorkflowStaticData('global').offerBook` recording offers, expiration timestamps, and exhaustion flags.

7. **Add Source Routing and Messaging Nodes:**
   - Add an **If** node named `From sweep?`, connected after `Offer book`. Set condition to check if `{{ $json.source }}` equals `"sweep"`.
   - Add a **Code** node named `Send offers (replace me)`, connected to the `false` output of `From sweep?`.
   - Add a **Respond to Webhook** node named `Respond`, connected after `Send offers (replace me)`, returning success status and offer count.
   - Add a **Code** node named `Send re-offers (replace me)`, connected to the `true` output of `From sweep?`.
   - Add a **Code** node named `Sweep log`, connected after `Send re-offers (replace me)`, to output summary counts.

8. **Create the Claim Reply Webhook Trigger:**
   - Add a **Webhook** node named `Webhook: claim reply`.
   - Set **Path** to `waitlist-claim`, **HTTP Method** to `POST`, and **Response Mode** to `Response Node`.

9. **Add Claim Validation:**
   - Add a **Code** node named `Valid claim body?`, connected after `Webhook: claim reply`.
   - Add an **If** node named `Claim valid?`, connected after `Valid claim body?`, checking if `{{ $json.ok }}` is `true`.
   - Add a **Respond to Webhook** node named `Respond claim invalid`, connected to the `false` output of `Claim valid?`, configured with response code `422`.

10. **Add the Atomic Claim Guard:**
    - Add a **Code** node named `Claim guard (first TAKE wins)`, connected to the `true` output of `Claim valid?`.
    - Implement logic handling `TAKE`, `STOP`/`CANCEL`, duplicate claims, expired offers, and immediate reservation state changes.
    - Add a **Respond to Webhook** node named `Respond claim`, connected after `Claim guard (first TAKE wins)`, to return the verdict and message.

11. **Create the Scheduled Sweeper:**
    - Add a **Schedule Trigger** node named `Sweep: every 30 min`. Set interval to every 30 minutes.
    - Add a **Code** node named `Sweep: expired slots`, connected after `Sweep: every 30 min`.
    - Connect the output of `Sweep: expired slots` directly into the input of `Waitlist (demo data)` to feed expired slots back into the ranking pipeline.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Fill cancelled slots from a ranked waitlist workflow overview | Workflow design pattern leveraging webhook triggers, static data state management, and scheduled sweepers. |
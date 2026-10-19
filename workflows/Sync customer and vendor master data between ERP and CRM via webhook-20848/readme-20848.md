Sync customer and vendor master data between ERP and CRM via webhook

https://n8nworkflows.xyz/workflows/sync-customer-and-vendor-master-data-between-erp-and-crm-via-webhook-20848


# Sync customer and vendor master data between ERP and CRM via webhook

### 1. Workflow Overview

This workflow automates the ingestion, normalization, deduplication, conflict resolution, synchronization, and auditing of customer and vendor master-data across enterprise systems (ERP and CRM). It acts as an integration middleware endpoint exposed via a webhook.

The logical flow is grouped into the following functional blocks:
- **1.1 Input Reception & Normalization:** Receives incoming master-data payloads via HTTP POST, validates mandatory fields, and maps them into a canonical data structure.
- **1.2 Deduplication & Golden Record Matching:** Evaluates identifiers (Tax ID, External ID, Email) against existing records to assign a unique golden record identifier and compute match confidence.
- **1.3 Conflict Evaluation & Resolution:** Identifies field-level discrepancies between the incoming payload and the matched golden record, applying a “prefer incoming” resolution strategy.
- **1.4 Enterprise Synchronization:** Sequentially pushes the resolved master record to external ERP and CRM targets, capturing execution results and handling errors gracefully.
- **1.5 Verification & Audit Response:** Verifies the synchronization status across target endpoints, compiles a comprehensive audit report, and returns a structured JSON response to the caller.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** Ingests raw data payloads from source systems, validates core structural requirements (entity type, identifiers, entity name), and standardizes properties into a unified canonical model.
- **Nodes Involved:** 
  - `Webhook - Master Data Change`
  - `Code - Validate & Normalize`
- **Node Details:**
  - `Webhook - Master Data Change`
    - **Type & Role:** `n8n-nodes-base.webhook` (Version 2) – Trigger node that exposes a POST endpoint for incoming webhook requests.
    - **Configuration:** Path set to `master-data-sync`, HTTP method `POST`, response mode managed by the downstream response node.
    - **Expressions/Variables:** None.
    - **Connections:** Input: None (Trigger); Output: Connects to `Code - Validate & Normalize`.
    - **Edge Cases & Failures:** Incorrect HTTP method or invalid route yields a 404/405 from n8n.
  - `Code - Validate & Normalize`
    - **Type & Role:** `n8n-nodes-base.code` (Version 2) – JavaScript node that validates incoming fields and builds a standardized canonical payload.
    - **Configuration:** Custom script that checks for `entityType` (customer/vendor), verifies required name and identification keys (`externalId`, `taxId`, `email`), and initializes configuration parameters (such as `erpUrl`, `crmUrl`, `verifyUrl`, `timeoutMs`, and `allowSimulation`).
    - **Expressions/Variables:** Uses `$(name).item.json` / `$(name).first().json` helpers to safely extract data from the webhook trigger.
    - **Connections:** Input: `Webhook - Master Data Change`; Output: Connects to `Code - Match & Deduplicate`.
    - **Edge Cases & Failures:** Malformed JSON bodies result in empty extraction objects, triggering validation errors collected in the `errors` array.

#### 2.2 Deduplication & Golden Record Matching
- **Overview:** Matches the incoming canonical record against a simulated golden record repository using a priority hierarchy (`taxId` → `externalId` → `email` → `new`) to generate or resolve a unique `goldenId`.
- **Nodes Involved:** 
  - `Code - Match & Deduplicate`
  - `Wait - Pre Conflict Check`
- **Node Details:**
  - `Code - Match & Deduplicate`
    - **Type & Role:** `n8n-nodes-base.code` (Version 2) – JavaScript node that executes identity matching rules.
    - **Configuration:** Script evaluates matching criteria and assigns confidence scores alongside metadata indicating whether the record is new or existing.
    - **Expressions/Variables:** References upstream output via `ref('Code - Validate & Normalize')`.
    - **Connections:** Input: `Code - Validate & Normalize`; Output: Connects to `Wait - Pre Conflict Check`.
    - **Edge Cases & Failures:** Missing all unique keys forces fallback to a new record creation branch (`matchType = 'new'`).
  - `Wait - Pre Conflict Check`
    - **Type & Role:** `n8n-nodes-base.wait` (Version 1.1) – Introduces a controlled delay.
    - **Configuration:** Wait amount set to `1` second.
    - **Connections:** Input: `Code - Match & Deduplicate`; Output: Connects to `Code - Detect Conflicts`.

#### 2.3 Conflict Evaluation & Resolution
- **Overview:** Compares incoming field values against the existing golden record to detect discrepancies and constructs a resolved dataset using a defined conflict resolution rule.
- **Nodes Involved:** 
  - `Code - Detect Conflicts`
  - `Wait - Pre Sync`
- **Node Details:**
  - `Code - Detect Conflicts`
    - **Type & Role:** `n8n-nodes-base.code` (Version 2) – JavaScript node for field-level diff detection.
    - **Configuration:** Iterates through monitored fields (`name`, `email`, `taxId`, `phone`, `country`, `status`) to flag mismatches and apply the `prefer_incoming` resolution policy.
    - **Expressions/Variables:** Accesses upstream data using local reference helper functions.
    - **Connections:** Input: `Wait - Pre Conflict Check`; Output: Connects to `Wait - Pre Sync`.
  - `Wait - Pre Sync`
    - **Type & Role:** `n8n-nodes-base.wait` (Version 1.1) – Pause node before external system writes.
    - **Configuration:** Wait amount set to `1` second.
    - **Connections:** Input: `Code - Detect Conflicts`; Output: Connects to `HTTP Request - Sync to ERP`.

#### 2.4 Enterprise Synchronization
- **Overview:** Sequentially writes the resolved master record to external target systems (ERP and CRM via HTTP POST requests), capturing response statuses and processing execution outcomes.
- **Nodes Involved:** 
  - `HTTP Request - Sync to ERP`
  - `Code - Process ERP Result`
  - `Wait - Pre CRM Sync`
  - `HTTP Request - Sync to CRM`
  - `Code - Process CRM Result`
- **Node Details:**
  - `HTTP Request - Sync to ERP`
    - **Type & Role:** `n8n-nodes-base.httpRequest` (Version 4.2) – Outbound HTTP client.
    - **Configuration:** Method: `POST`; URL dynamic from `$json.urls.erp`; timeout mapped from configuration; response option set to never error (`neverError: true`, `fullResponse: true`) with JSON body payload containing the system type, `goldenId`, and resolved record. Custom headers include `content-type: application/json` and `X-Request-Id`.
    - **Expressions/Variables:** Uses `={{ $json.urls.erp }}`, `={{ JSON.stringify(...) }}`, and `={{ $json.requestId }}`.
    - **Connections:** Input: `Wait - Pre Sync`; Output: Connects to `Code - Process ERP Result`.
    - **Edge Cases & Failures:** Network timeouts or HTTP 4xx/5xx errors are caught gracefully due to `continueOnFail: true` and `neverError: true` configurations.
  - `Code - Process ERP Result`
    - **Type & Role:** `n8n-nodes-base.code` (Version 2) – JavaScript node to parse the ERP integration outcome.
    - **Configuration:** Evaluates the HTTP status code (success if between 200 and 299) and builds the initial `sync` tracking object.
    - **Connections:** Input: `HTTP Request - Sync to ERP`; Output: Connects to `Wait - Pre CRM Sync`.
  - `Wait - Pre CRM Sync`
    - **Type & Role:** `n8n-nodes-base.wait` (Version 1.1) – Brief pause between target writes.
    - **Configuration:** Wait amount set to `1` second.
    - **Connections:** Input: `Code - Process ERP Result`; Output: Connects to `HTTP Request - Sync to CRM`.
  - `HTTP Request - Sync to CRM`
    - **Type & Role:** `n8n-nodes-base.httpRequest` (Version 4.2) – Outbound HTTP client for CRM endpoint.
    - **Configuration:** Method: `POST`; URL dynamic from `$json.urls.crm`; timeout configuration applied; includes full response handling without throwing errors on non-2xx responses. Sends JSON body with system metadata and resolved data.
    - **Expressions/Variables:** Evaluates dynamic URL, request body serialization, and header injection.
    - **Connections:** Input: `Wait - Pre CRM Sync`; Output: Connects to `Code - Process CRM Result`.
  - `Code - Process CRM Result`
    - **Type & Role:** `n8n-nodes-base.code` (Version 2) – JavaScript node to aggregate CRM sync metrics.
    - **Configuration:** Appends CRM outcome status to the existing synchronization tracking object.
    - **Connections:** Input: `HTTP Request - Sync to CRM`; Output: Connects to `Wait - Pre Verify`.

#### 2.5 Verification & Audit Response
- **Overview:** Triggers a post-sync verification call against the target systems, compiles all execution stages into a unified audit report, determines overall synchronization health, and responds to the webhook caller.
- **Nodes Involved:** 
  - `Wait - Pre Verify`
  - `HTTP Request - Verify Sync`
  - `Code - Build Sync Report`
  - `Code - Finalize Response`
  - `Respond to Webhook`
- **Node Details:**
  - `Wait - Pre Verify`
    - **Type & Role:** `n8n-nodes-base.wait` (Version 1.1) – Pause node before verification.
    - **Configuration:** Wait amount set to `1` second.
    - **Connections:** Input: `Code - Process CRM Result`; Output: Connects to `HTTP Request - Verify Sync`.
  - `HTTP Request - Verify Sync`
    - **Type & Role:** `n8n-nodes-base.httpRequest` (Version 4.2) – Outbound HTTP client for verification.
    - **Configuration:** Method: `POST`; dynamic URL from `$json.urls.verify`; non-error response handling enabled. Submits verification request payload listing target systems (`erp`, `crm`).
    - **Connections:** Input: `Wait - Pre Verify`; Output: Connects to `Code - Build Sync Report`.
  - `Code - Build Sync Report`
    - **Type & Role:** `n8n-nodes-base.code` (Version 2) – JavaScript node that compiles the complete synchronization report.
    - **Configuration:** Aggregates match metadata, detected conflicts, sync outcomes, and verification results into a structured audit object, determining the overall status (`synced`, `synced_unverified`, or `partial_failure`).
    - **Connections:** Input: `HTTP Request - Verify Sync`; Output: Connects to `Code - Finalize Response`.
  - `Code - Finalize Response`
    - **Type & Role:** `n8n-nodes-base.code` (Version 2) – JavaScript node that maps internal status codes to HTTP response codes.
    - **Configuration:** Assigns appropriate HTTP status codes (200 for fully synced, 202 for synced unverified, 207 for partial failures) and structures the final response wrapper.
    - **Connections:** Input: `Code - Build Sync Report`; Output: Connects to `Respond to Webhook`.
  - `Respond to Webhook`
    - **Type & Role:** `n8n-nodes-base.respondToWebhook` (Version 1.1) – Response termination node.
    - **Configuration:** Responds with JSON data, dynamic response code mapped from `$json.httpStatus`, and custom response headers including `X-Request-Id`.
    - **Expressions/Variables:** Uses `={{ $json.httpStatus }}` and `={{ $json.body }}`.
    - **Connections:** Input: `Code - Finalize Response`; Output: None (Terminal Node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note - Overview | `n8n-nodes-base.stickyNote` | Overview documentation and summary list of workflow nodes. | None | None | ## Cross-Company Master-Data Synchronization<br><br>Receives customer or vendor master-data changes, normalizes and matches records, resolves conflicts, synchronizes to target systems, verifies the sync, and returns a full audit report.<br><br>### All nodes (1-line each)<br>1. Webhook - Master Data Change – Receives customer or vendor master-data payload via POST.<br>2. Code - Validate & Normalize – Validates the payload and normalizes fields into a canonical model.<br>3. Code - Match & Deduplicate – Matches the incoming record against existing golden records using keys and fuzzy rules.<br>4. Wait - Pre Conflict Check – Short pause before conflict evaluation.<br>5. Code - Detect Conflicts – Identifies field-level conflicts between incoming data and the golden record.<br>6. Wait - Pre Sync – Pause before writing to target systems.<br>7. HTTP Request - Sync to ERP – Pushes the resolved master record to the ERP system.<br>8. Code - Process ERP Result – Records ERP sync outcome.<br>9. Wait - Pre CRM Sync – Pause before CRM synchronization.<br>10. HTTP Request - Sync to CRM – Pushes the resolved master record to the CRM system.<br>11. Code - Process CRM Result – Records CRM sync outcome.<br>12. Wait - Pre Verify – Pause before verification.<br>13. HTTP Request - Verify Sync – Confirms the record exists and matches in target systems.<br>14. Code - Build Sync Report – Aggregates match, conflict, sync and verification results into a final report.<br>15. Code - Finalize Response – Assembles the response payload for the caller.<br>16. Respond to Webhook – Returns the complete master-data synchronization result. |
| Sticky Note - Intake & Match | `n8n-nodes-base.stickyNote` | Groups intake, normalization, and deduplication nodes. | None | None | ## 1. Intake, Normalize & Match<br><br>Webhook - Master Data Change – Receives customer or vendor master-data payload via POST.<br>Code - Validate & Normalize – Validates the payload and normalizes fields into a canonical model.<br>Code - Match & Deduplicate – Matches the incoming record against existing golden records using keys and fuzzy rules. |
| Sticky Note - Conflict & Sync | `n8n-nodes-base.stickyNote` | Groups conflict resolution and multi-system synchronization nodes. | None | None | ## 2. Conflict Resolution & Multi-System Sync<br><br>Wait - Pre Conflict Check – Short pause before conflict evaluation.<br>Code - Detect Conflicts – Identifies field-level conflicts between incoming data and the golden record.<br>Wait - Pre Sync – Pause before writing to target systems.<br>HTTP Request - Sync to ERP – Pushes the resolved master record to the ERP system.<br>Code - Process ERP Result – Records ERP sync outcome.<br>Wait - Pre CRM Sync – Pause before CRM synchronization.<br>HTTP Request - Sync to CRM – Pushes the resolved master record to the CRM system.<br>Code - Process CRM Result – Records CRM sync outcome. |
| Sticky Note - Verify & Respond | `n8n-nodes-base.stickyNote` | Groups verification and response-formatting nodes. | None | None | ## 3. Verify & Respond<br><br>Wait - Pre Verify – Pause before verification.<br>HTTP Request - Verify Sync – Confirms the record exists and matches in target systems.<br>Code - Build Sync Report – Aggregates match, conflict, sync and verification results into a final report.<br>Code - Finalize Response – Assembles the response payload for the caller.<br>Respond to Webhook – Returns the complete master-data synchronization result. |
| Webhook - Master Data Change | `n8n-nodes-base.webhook` | Receives customer or vendor master-data payload via POST. | None | Code - Validate & Normalize | Receives customer or vendor master-data payload via POST. |
| Code - Validate & Normalize | `n8n-nodes-base.code` | Validates the payload and normalizes fields into a canonical model. | Webhook - Master Data Change | Code - Match & Deduplicate | Validates the payload and normalizes fields into a canonical model. |
| Code - Match & Deduplicate | `n8n-nodes-base.code` | Matches the incoming record against existing golden records using keys and fuzzy rules. | Code - Validate & Normalize | Wait - Pre Conflict Check | Matches the incoming record against existing golden records using keys and fuzzy rules. |
| Wait - Pre Conflict Check | `n8n-nodes-base.wait` | Short pause before conflict evaluation. | Code - Match & Deduplicate | Code - Detect Conflicts | Short pause before conflict evaluation. |
| Code - Detect Conflicts | `n8n-nodes-base.code` | Identifies field-level conflicts between incoming data and the golden record. | Wait - Pre Conflict Check | Wait - Pre Sync | Identifies field-level conflicts between incoming data and the golden record. |
| Wait - Pre Sync | `n8n-nodes-base.wait` | Pause before writing to target systems. | Code - Detect Conflicts | HTTP Request - Sync to ERP | Pause before writing to target systems. |
| HTTP Request - Sync to ERP | `n8n-nodes-base.httpRequest` | Pushes the resolved master record to the ERP system. | Wait - Pre Sync | Code - Process ERP Result | Pushes the resolved master record to the ERP system. |
| Code - Process ERP Result | `n8n-nodes-base.code` | Records ERP sync outcome. | HTTP Request - Sync to ERP | Wait - Pre CRM Sync | Records ERP sync outcome. |
| Wait - Pre CRM Sync | `n8n-nodes-base.wait` | Pause before CRM synchronization. | Code - Process ERP Result | HTTP Request - Sync to CRM | Pause before CRM synchronization. |
| HTTP Request - Sync to CRM | `n8n-nodes-base.httpRequest` | Pushes the resolved master record to the CRM system. | Wait - Pre CRM Sync | Code - Process CRM Result | Pushes the resolved master record to the CRM system. |
| Code - Process CRM Result | `n8n-nodes-base.code` | Records CRM sync outcome. | HTTP Request - Sync to CRM | Wait - Pre Verify | Records CRM sync outcome. |
| Wait - Pre Verify | `n8n-nodes-base.wait` | Pause before verification. | Code - Process CRM Result | HTTP Request - Verify Sync | Pause before verification. |
| HTTP Request - Verify Sync | `n8n-nodes-base.httpRequest` | Confirms the record exists and matches in target systems. | Wait - Pre Verify | Code - Build Sync Report | Confirms the record exists and matches in target systems. |
| Code - Build Sync Report | `n8n-nodes-base.code` | Aggregates match, conflict, sync and verification results into a final report. | HTTP Request - Verify Sync | Code - Finalize Response | Aggregates match, conflict, sync and verification results into a final report. |
| Code - Finalize Response | `n8n-nodes-base.code` | Assembles the response payload for the caller. | Code - Build Sync Report | Respond to Webhook | Assembles the response payload for the caller. |
| Respond to Webhook | `n8n-nodes-base.respondToWebhook` | Returns the complete master-data synchronization result. | Code - Finalize Response | None | Returns the complete master-data synchronization result. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node (`n8n-nodes-base.webhook`).
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `master-data-sync`.
   - Set **Response Mode** to `Response Node`.

2. **Add Validation and Normalization Logic:**
   - Connect the Webhook to a **Code** node (`n8n-nodes-base.code`).
   - Name it `Code - Validate & Normalize`.
   - Insert JavaScript code that validates `entityType` ("customer" or "vendor"), checks mandatory identifier/name fields, establishes default configuration parameters (`erpUrl`, `crmUrl`, `verifyUrl`, `timeoutMs`, `allowSimulation`), and outputs a canonical data object.

3. **Add Deduplication and Golden Record Matching:**
   - Connect `Code - Validate & Normalize` to a second **Code** node named `Code - Match & Deduplicate`.
   - Insert JavaScript code that performs matching based on `taxId`, `externalId`, `email`, or falls back to a new record, generating a unique `goldenId` and confidence rating.

4. **Insert Pre-Conflict Pause:**
   - Connect `Code - Match & Deduplicate` to a **Wait** node named `Wait - Pre Conflict Check`.
   - Set the wait amount to `1` second.

5. **Add Conflict Detection Logic:**
   - Connect `Wait - Pre Conflict Check` to a **Code** node named `Code - Detect Conflicts`.
   - Insert JavaScript code that compares incoming fields against existing golden record attributes, logs differences in a `conflicts` array, and applies the resolution policy.

6. **Insert Pre-Sync Pause:**
   - Connect `Code - Detect Conflicts` to a **Wait** node named `Wait - Pre Sync` (amount: `1` second).

7. **Configure ERP Synchronization:**
   - Connect `Wait - Pre Sync` to an **HTTP Request** node named `HTTP Request - Sync to ERP`.
   - Set Method to `POST`, URL to `={{ $json.urls.erp }}`, and configure timeout using `={{ $json.config.timeoutMs }}`.
   - Enable options for never erroring and returning full responses.
   - Set body specification to **JSON**, and pass `={{ JSON.stringify({ system: 'erp', goldenId: $json.resolved.goldenId, record: $json.resolved }) }}`.
   - Add header parameters: `content-type` (`application/json`) and `X-Request-Id` (`={{ $json.requestId }}`).
   - Enable `continueOnFail`.

8. **Process ERP Results:**
   - Connect `HTTP Request - Sync to ERP` to a **Code** node named `Code - Process ERP Result`.
   - Insert script to evaluate response status code and build the `sync` object.

9. **Insert Pre-CRM Pause & Configure CRM Synchronization:**
   - Connect `Code - Process ERP Result` to a **Wait** node named `Wait - Pre CRM Sync` (amount: `1` second).
   - Connect to an **HTTP Request** node named `HTTP Request - Sync to CRM`.
   - Configure identically to the ERP HTTP request, replacing the system payload identifier with `'crm'` and pointing to `={{ $json.urls.crm }}`.
   - Enable `continueOnFail`.

10. **Process CRM Results:**
    - Connect `HTTP Request - Sync to CRM` to a **Code** node named `Code - Process CRM Result` to append CRM outcomes to the sync status tracking.

11. **Insert Pre-Verification Pause & Configure Verification:**
    - Connect `Code - Process CRM Result` to a **Wait** node named `Wait - Pre Verify` (amount: `1` second).
    - Connect to an **HTTP Request** node named `HTTP Request - Verify Sync`.
    - Configure Method as `POST`, URL as `={{ $json.urls.verify }}`, and pass a verification action JSON body requesting checks for ERP and CRM systems. Enable non-error response handling and `continueOnFail`.

12. **Build Sync Report:**
    - Connect `HTTP Request - Verify Sync` to a **Code** node named `Code - Build Sync Report`.
    - Insert JavaScript code to calculate overall synchronization status (`synced`, `synced_unverified`, or `partial_failure`) and construct the audit report object.

13. **Finalize and Respond:**
    - Connect `Code - Build Sync Report` to a **Code** node named `Code - Finalize Response` to compute the HTTP status code (200, 202, or 207).
    - Connect `Code - Finalize Response` to a **Respond to Webhook** node (`n8n-nodes-base.respondToWebhook`).
    - Set response body to `={{ $json.body }}` and response code to `={{ $json.httpStatus }}`, adding the `X-Request-Id` response header.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Cross-Company Master-Data Synchronization Architecture | Designed for enterprise data governance, integrating ERP and CRM endpoints through reliable webhook ingestion and audit-ready reporting. |
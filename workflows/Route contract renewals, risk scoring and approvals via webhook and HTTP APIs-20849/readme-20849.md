Route contract renewals, risk scoring and approvals via webhook and HTTP APIs

https://n8nworkflows.xyz/workflows/route-contract-renewals--risk-scoring-and-approvals-via-webhook-and-http-apis-20849


# Route contract renewals, risk scoring and approvals via webhook and HTTP APIs

### 1. Workflow Overview

This workflow automates the contract lifecycle management process by ingesting contract data via webhook, analyzing renewal urgency, evaluating risk through external APIs, applying internal multi-level approval policies, and returning a structured lifecycle decision. 

The execution logic is structured into three functional blocks:
- **1.1 Intake & Renewal Detection:** Ingests the incoming contract payload via an HTTP POST webhook, validates required fields, normalizes contract metadata, sets default configuration parameters, and calculates the renewal urgency based on contract expiry dates.
- **1.2 Risk Analysis & Policy Evaluation:** Introduces controlled pauses, queries an external risk analysis service via HTTP, processes and maps the returned risk scores to risk tiers, and evaluates required approval levels based on contract value, risk tier, and renewal status.
- **1.3 Approval, Reporting & Response:** Submits the contract data to an external approval system (unless auto-approved), interprets the approval outcome, builds a detailed audit report, establishes appropriate HTTP status codes, and responds to the original webhook caller.

---

### 2. Block-by-Block Analysis

#### Block 1: Intake & Renewal Detection
**Overview:** This block ingests raw contract submissions via a webhook, validates critical fields (such as title, vendor, and end date), normalizes metadata, and calculates days remaining until contract expiry to determine whether renewal action is required.

**Nodes Involved:**
- `Webhook - Contract Intake`
- `Code - Validate & Normalize`
- `Code - Renewal Detection`

**Node Details:**
- **Webhook - Contract Intake**
  - *Type & Role:* `n8n-nodes-base.webhook` (Trigger) - Exposes a public webhook endpoint to receive contract data via HTTP POST.
  - *Configuration:* Path set to `contract-lifecycle`, HTTP Method: `POST`, Response Mode set to use the response node.
  - *Key Expressions:* None (Webhook trigger).
  - *Connections:* Output connects to `Code - Validate & Normalize`.
  - *Edge Cases / Failures:* Network dropouts, invalid HTTP methods (GET instead of POST), or unparseable JSON bodies.

- **Code - Validate & Normalize**
  - *Type & Role:* `n8n-nodes-base.code` (Data Processing) - Validates mandatory fields and normalizes contract parameters, setting internal configuration rules (such as renewal windows and risk/approval endpoints).
  - *Configuration:* JavaScript execution block that extracts incoming payload properties, checks for missing required attributes (`title`, `vendor`, `endDate`, `value`), and populates default configuration variables.
  - *Key Expressions:* Uses helper function `ref('Webhook - Contract Intake')` to safely read upstream data. Supports optional simulation flags via `simulateStatus`.
  - *Connections:* Input from `Webhook - Contract Intake`; output to `Code - Renewal Detection`.
  - *Edge Cases / Failures:* Missing required fields populated in the webhook body will populate the `errors` array, marking `valid: false`.

- **Code - Renewal Detection**
  - *Type & Role:* `n8n-nodes-base.code` (Data Processing) - Computes the time remaining until contract expiry and classifies the renewal status.
  - *Configuration:* JavaScript block calculating the difference in days between the current date and the contract `endDate`.
  - *Key Expressions:* Uses `ref('Code - Validate & Normalize')` to retrieve normalized contract data and configuration thresholds.
  - *Connections:* Input from `Code - Validate & Normalize`; output to `Wait - Pre Risk`.
  - *Edge Cases / Failures:* Invalid date formats in `endDate` trigger a catch block setting status to `invalid_date`.

---

#### Block 2: Risk Analysis & Policy Evaluation
**Overview:** This block executes controlled pauses between steps, queries an external risk analysis engine, processes the returned risk score into categorical risk tiers, and evaluates internal policy criteria to determine required approval thresholds.

**Nodes Involved:**
- `Wait - Pre Risk`
- `HTTP Request - Risk Analysis Engine`
- `Code - Process Risk Score`
- `Wait - Pre Policy`
- `Code - Evaluate Approval Policy`

**Node Details:**
- **Wait - Pre Risk**
  - *Type & Role:* `n8n-nodes-base.wait` (Flow Control) - Pauses execution briefly before calling the risk service.
  - *Configuration:* Amount set to `1` second.
  - *Connections:* Input from `Code - Renewal Detection`; output to `HTTP Request - Risk Analysis Engine`.

- **HTTP Request - Risk Analysis Engine**
  - *Type & Role:* `n8n-nodes-base.httpRequest` (Integration) - Sends contract context data to an external risk scoring API.
  - *Configuration:* Method: `POST`, URL driven dynamically by `$json.urls.risk`, timeout configured from environment config. Configured with `neverError: true` and full response capture.
  - *Key Expressions:* URL: `={{ $json.urls.risk }}`, JSON body maps contract ID, vendor, value, type, end date, and renewal status. Header includes `X-Request-Id`.
  - *Connections:* Input from `Wait - Pre Risk`; output to `Code - Process Risk Score`.
  - *Edge Cases / Failures:* API timeouts, network unreachable, or HTTP 4xx/5xx errors (mitigated by `continueOnFail: true` and `neverError: true`).

- **Code - Process Risk Score**
  - *Type & Role:* `n8n-nodes-base.code` (Data Processing) - Parses the risk analysis API response and assigns a risk tier (low, medium, or high), falling back to a deterministic heuristic if the API fails.
  - *Configuration:* JavaScript block evaluating HTTP status codes, parsing JSON bodies, and computing numerical scores based on contract value and renewal status.
  - *Key Expressions:* References upstream data via `ref('Code - Renewal Detection')` and inspects `$input.first().json`.
  - *Connections:* Input from `HTTP Request - Risk Analysis Engine`; output to `Wait - Pre Policy`.

- **Wait - Pre Policy**
  - *Type & Role:* `n8n-nodes-base.wait` (Flow Control) - Pauses execution briefly before evaluating internal policy rules.
  - *Configuration:* Amount set to `1` second.
  - *Connections:* Input from `Code - Process Risk Score`; output to `Code - Evaluate Approval Policy`.

- **Code - Evaluate Approval Policy**
  - *Type & Role:* `n8n-nodes-base.code` (Data Processing) - Determines whether the contract qualifies for auto-approval or requires escalation through managerial, director, or executive approval levels.
  - *Configuration:* JavaScript evaluating contract value against thresholds (`autoApproveBelow`, `managerBelow`, `directorBelow`) combined with the risk tier.
  - *Key Expressions:* References `ref('Code - Process Risk Score')`.
  - *Connections:* Input from `Wait - Pre Policy`; output to `Wait - Pre Approval`.

---

#### Block 3: Approval, Reporting & Response
**Overview:** This block submits contracts requiring manual approval to an external approval system, processes the outcome, generates a complete audit and lifecycle report, determines the final HTTP response status, and responds to the webhook caller.

**Nodes Involved:**
- `Wait - Pre Approval`
- `HTTP Request - Approval System`
- `Code - Process Approval Result`
- `Code - Build Lifecycle Report`
- `Code - Finalize Response`
- `Respond to Webhook`

**Node Details:**
- **Wait - Pre Approval**
  - *Type & Role:* `n8n-nodes-base.wait` (Flow Control) - Pauses execution briefly before hitting the external approval system.
  - *Configuration:* Amount set to `1` second.
  - *Connections:* Input from `Code - Evaluate Approval Policy`; output to `HTTP Request - Approval System`.

- **HTTP Request - Approval System**
  - *Type & Role:* `n8n-nodes-base.httpRequest` (Integration) - Submits contract metadata and required approval levels to an external approval routing system.
  - *Configuration:* Method: `POST`, URL driven by `$json.urls.approval`, timeout configured via setup variables. `neverError: true` enabled.
  - *Key Expressions:* URL: `={{ $json.urls.approval }}`, JSON body passes contract ID, value, risk tier, required levels, and renewal status.
  - *Connections:* Input from `Wait - Pre Approval`; output to `Code - Process Approval Result`.
  - *Edge Cases / Failures:* External system downtime or invalid payload formatting.

- **Code - Process Approval Result**
  - *Type & Role:* `n8n-nodes-base.code` (Data Processing) - Interprets the response from the approval system and assigns the final lifecycle status (`approved`, `rejected`, `approved_pending_renewal`, etc.).
  - *Configuration:* JavaScript evaluating auto-approval status and HTTP response status codes.
  - *Key Expressions:* References `ref('Code - Evaluate Approval Policy')`.
  - *Connections:* Input from `HTTP Request - Approval System`; output to `Code - Build Lifecycle Report`.

- **Code - Build Lifecycle Report**
  - *Type & Role:* `n8n-nodes-base.code` (Data Processing) - Compiles a comprehensive audit report incorporating renewal metrics, risk scoring, policy evaluation results, and approval outcomes.
  - *Configuration:* JavaScript constructing a clean hierarchical report object with a UTC timestamp.
  - *Key Expressions:* References `ref('Code - Process Approval Result')`.
  - *Connections:* Input from `Code - Process Approval Result`; output to `Code - Finalize Response`.

- **Code - Finalize Response**
  - *Type & Role:* `n8n-nodes-base.code` (Data Processing) - Determines the appropriate HTTP status code based on the lifecycle status (returning `403` for rejections, `200` for successes) and packages the final payload.
  - *Configuration:* JavaScript mapping lifecycle results to HTTP status responses.
  - *Key Expressions:* References `ref('Code - Build Lifecycle Report')`.
  - *Connections:* Input from `Code - Build Lifecycle Report`; output to `Respond to Webhook`.

- **Respond to Webhook**
  - *Type & Role:* `n8n-nodes-base.respondToWebhook` (Output) - Returns the contract lifecycle decision payload and audit headers back to the original HTTP webhook requester.
  - *Configuration:* Response mode set to JSON, response code mapped dynamically from `$json.httpStatus`, custom header `X-Request-Id` included.
  - *Key Expressions:* Response Code: `={{ $json.httpStatus }}`, Body: `={{ $json.body }}`.
  - *Connections:* Input from `Code - Finalize Response`; output is the workflow termination point.
  - *Edge Cases / Failures:* Connection closed by client prematurely before response transmission.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note - Overview** | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## Contract Lifecycle Workflow<br><br>Ingests a contract, detects upcoming renewals, runs risk analysis, evaluates dynamic approval requirements, and returns a full lifecycle decision with audit trail.<br><br>### All nodes (1-line each)<br>1. Webhook - Contract Intake – Receives new or updated contract data via POST.<br>2. Code - Validate & Normalize – Validates required fields and normalizes contract metadata.<br>3. Code - Renewal Detection – Calculates days-to-expiry and flags contracts needing renewal action.<br>4. Wait - Pre Risk – Short pause before risk analysis.<br>5. HTTP Request - Risk Analysis Engine – Sends contract details to an external risk scoring service.<br>6. Code - Process Risk Score – Interprets risk score and maps it to risk tier (low / medium / high).<br>7. Wait - Pre Policy – Pause before policy evaluation.<br>8. Code - Evaluate Approval Policy – Determines required approval levels based on value, risk and renewal status.<br>9. Wait - Pre Approval – Pause before approval routing.<br>10. HTTP Request - Approval System – Submits the contract for the required multi-level approvals.<br>11. Code - Process Approval Result – Records approval decision and final lifecycle status.<br>12. Code - Build Lifecycle Report – Creates a complete report including renewal, risk and approval outcomes.<br>13. Code - Finalize Response – Assembles the final response payload for the caller.<br>14. Respond to Webhook – Returns the contract lifecycle decision and audit trail. |
| **Sticky Note - Intake & Renewal** | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## 1. Intake & Renewal Detection<br><br>Webhook - Contract Intake – Receives new or updated contract data via POST.<br>Code - Validate & Normalize – Validates required fields and normalizes contract metadata.<br>Code - Renewal Detection – Calculates days-to-expiry and flags contracts needing renewal action. |
| **Sticky Note - Risk & Policy** | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## 2. Risk Analysis & Policy<br><br>Wait - Pre Risk – Short pause before risk analysis.<br>HTTP Request - Risk Analysis Engine – Sends contract details to an external risk scoring service.<br>Code - Process Risk Score – Interprets risk score and maps it to risk tier (low / medium / high).<br>Wait - Pre Policy – Pause before policy evaluation.<br>Code - Evaluate Approval Policy – Determines required approval levels based on value, risk and renewal status. |
| **Sticky Note - Approval & Finish** | `n8n-nodes-base.stickyNote` | Documentation placeholder | None | None | ## 3. Approval & Finish<br><br>Wait - Pre Approval – Pause before approval routing.<br>HTTP Request - Approval System – Submits the contract for the required multi-level approvals.<br>Code - Process Approval Result – Records approval decision and final lifecycle status.<br>Code - Build Lifecycle Report – Creates a complete report including renewal, risk and approval outcomes.<br>Code - Finalize Response – Assembles the final response payload for the caller.<br>Respond to Webhook – Returns the contract lifecycle decision and audit trail. |
| **Webhook - Contract Intake** | `n8n-nodes-base.webhook` | Receives POST requests containing contract data | None | Code - Validate & Normalize | ## Contract Lifecycle Workflow<br>...<br>## 1. Intake & Renewal Detection<br>Webhook - Contract Intake – Receives new or updated contract data via POST. |
| **Code - Validate & Normalize** | `n8n-nodes-base.code` | Validates required fields & sets default configs | Webhook - Contract Intake | Code - Renewal Detection | ## Contract Lifecycle Workflow<br>...<br>## 1. Intake & Renewal Detection<br>Code - Validate & Normalize – Validates required fields and normalizes contract metadata. |
| **Code - Renewal Detection** | `n8n-nodes-base.code` | Calculates expiry countdown & renewal status | Code - Validate & Normalize | Wait - Pre Risk | ## Contract Lifecycle Workflow<br>...<br>## 1. Intake & Renewal Detection<br>Code - Renewal Detection – Calculates days-to-expiry and flags contracts needing renewal action. |
| **Wait - Pre Risk** | `n8n-nodes-base.wait` | Pause prior to external risk scoring call | Code - Renewal Detection | HTTP Request - Risk Analysis Engine | ## Contract Lifecycle Workflow<br>...<br>## 2. Risk Analysis & Policy<br>Wait - Pre Risk – Short pause before risk analysis. |
| **HTTP Request - Risk Analysis Engine** | `n8n-nodes-base.httpRequest` | Calls external risk assessment API | Wait - Pre Risk | Code - Process Risk Score | ## Contract Lifecycle Workflow<br>...<br>## 2. Risk Analysis & Policy<br>HTTP Request - Risk Analysis Engine – Sends contract details to an external risk scoring service. |
| **Code - Process Risk Score** | `n8n-nodes-base.code` | Maps risk score numbers to tiers (low/med/high) | HTTP Request - Risk Analysis Engine | Wait - Pre Policy | ## Contract Lifecycle Workflow<br>...<br>## 2. Risk Analysis & Policy<br>Code - Process Risk Score – Interprets risk score and maps it to risk tier (low / medium / high). |
| **Wait - Pre Policy** | `n8n-nodes-base.wait` | Pause prior to policy evaluation | Code - Process Risk Score | Code - Evaluate Approval Policy | ## Contract Lifecycle Workflow<br>...<br>## 2. Risk Analysis & Policy<br>Wait - Pre Policy – Pause before policy evaluation. |
| **Code - Evaluate Approval Policy** | `n8n-nodes-base.code` | Computes approval requirements based on value & risk | Wait - Pre Policy | Wait - Pre Approval | ## Contract Lifecycle Workflow<br>...<br>## 2. Risk Analysis & Policy<br>Code - Evaluate Approval Policy – Determines required approval levels based on value, risk and renewal status. |
| **Wait - Pre Approval** | `n8n-nodes-base.wait` | Pause prior to approval system routing | Code - Evaluate Approval Policy | HTTP Request - Approval System | ## Contract Lifecycle Workflow<br>...<br>## 3. Approval & Finish<br>Wait - Pre Approval – Pause before approval routing. |
| **HTTP Request - Approval System** | `n8n-nodes-base.httpRequest` | Submits contract for multi-level approval routing | Wait - Pre Approval | Code - Process Approval Result | ## Contract Lifecycle Workflow<br>...<br>## 3. Approval & Finish<br>HTTP Request - Approval System – Submits the contract for the required multi-level approvals. |
| **Code - Process Approval Result** | `n8n-nodes-base.code` | Records approval decision and lifecycle state | HTTP Request - Approval System | Code - Build Lifecycle Report | ## Contract Lifecycle Workflow<br>...<br>## 3. Approval & Finish<br>Code - Process Approval Result – Records approval decision and final lifecycle status. |
| **Code - Build Lifecycle Report** | `n8n-nodes-base.code` | Compiles comprehensive audit and status report | Code - Process Approval Result | Code - Finalize Response | ## Contract Lifecycle Workflow<br>...<br>## 3. Approval & Finish<br>Code - Build Lifecycle Report – Creates a complete report including renewal, risk and approval outcomes. |
| **Code - Finalize Response** | `n8n-nodes-base.code` | Sets HTTP response code and final payload structure | Code - Build Lifecycle Report | Respond to Webhook | ## Contract Lifecycle Workflow<br>...<br>## 3. Approval & Finish<br>Code - Finalize Response – Assembles the final response payload for the caller. |
| **Respond to Webhook** | `n8n-nodes-base.respondToWebhook` | Returns response payload and audit headers | Code - Finalize Response | None | ## Contract Lifecycle Workflow<br>...<br>## 3. Approval & Finish<br>Respond to Webhook – Returns the contract lifecycle decision and audit trail. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Webhook Trigger Node**
   - Add a **Webhook** node. Name it `Webhook - Contract Intake`.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `contract-lifecycle`.
   - Set **Response Mode** to `Response Node`.

2. **Add Validation and Normalization Code Node**
   - Add a **Code** node named `Code - Validate & Normalize`. Connect `Webhook - Contract Intake` output to it.
   - Insert JavaScript to extract body parameters (`title`, `vendor`, `value`, `endDate`, etc.), validate required fields, and initialize the configuration object (including endpoints like `https://httpbin.org/post` and policy thresholds).

3. **Add Renewal Detection Code Node**
   - Add a **Code** node named `Code - Renewal Detection`. Connect `Code - Validate & Normalize` to it.
   - Insert JavaScript to compute `daysToExpiry` relative to the current date and set renewal statuses (`expired`, `critical`, `upcoming`, `active`).

4. **Add Pre-Risk Wait Node**
   - Add a **Wait** node named `Wait - Pre Risk`. Connect `Code - Renewal Detection` to it.
   - Set duration amount to `1` second.

5. **Add Risk Analysis HTTP Request Node**
   - Add an **HTTP Request** node named `HTTP Request - Risk Analysis Engine`. Connect `Wait - Pre Risk` to it.
   - Set Method to `POST`, URL to `={{ $json.urls.risk }}`, and body content type to JSON.
   - Add header `content-type: application/json` and `X-Request-Id: ={{ $json.requestId }}`.
   - Enable **Continue On Fail** and configure options for never error and full response return.

6. **Add Risk Score Processing Code Node**
   - Add a **Code** node named `Code - Process Risk Score`. Connect `HTTP Request - Risk Analysis Engine` to it.
   - Insert JavaScript to parse API responses, fallback to heuristic scoring if needed, and assign a risk tier (`low`, `medium`, `high`).

7. **Add Pre-Policy Wait Node**
   - Add a **Wait** node named `Wait - Pre Policy`. Connect `Code - Process Risk Score` to it.
   - Set duration amount to `1` second.

8. **Add Approval Policy Evaluation Code Node**
   - Add a **Code** node named `Code - Evaluate Approval Policy`. Connect `Wait - Pre Policy` to it.
   - Insert JavaScript to determine auto-approval eligibility and required approval tiers (`manager`, `director`, `executive`).

9. **Add Pre-Approval Wait Node**
   - Add a **Wait** node named `Wait - Pre Approval`. Connect `Code - Evaluate Approval Policy` to it.
   - Set duration amount to `1` second.

10. **Add Approval System HTTP Request Node**
    - Add an **HTTP Request** node named `HTTP Request - Approval System`. Connect `Wait - Pre Approval` to it.
    - Set Method to `POST`, URL to `={{ $json.urls.approval }}`, and body content type to JSON passing contract metadata and required approval levels.
    - Enable **Continue On Fail** and configure options for never error.

11. **Add Approval Result Processing Code Node**
    - Add a **Code** node named `Code - Process Approval Result`. Connect `HTTP Request - Approval System` to it.
    - Insert JavaScript to record final decision outcomes and establish the `lifecycleStatus`.

12. **Add Lifecycle Report Builder Code Node**
    - Add a **Code** node named `Code - Build Lifecycle Report`. Connect `Code - Process Approval Result` to it.
    - Insert JavaScript to bundle audit trails, renewal metrics, risk assessments, and approval history into a unified report object.

13. **Add Response Finalizer Code Node**
    - Add a **Code** node named `Code - Finalize Response`. Connect `Code - Build Lifecycle Report` to it.
    - Insert JavaScript to evaluate HTTP status codes (e.g., `403` for rejected contracts, `200` otherwise).

14. **Add Respond to Webhook Node**
    - Add a **Respond to Webhook** node named `Respond to Webhook`. Connect `Code - Finalize Response` to it.
    - Configure response format as JSON, map response code to `={{ $json.httpStatus }}`, and include header `X-Request-Id: ={{ $json.requestId }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| External API endpoints default to httpbin.org for testing and demonstration purposes. | Replace `https://httpbin.org/post` with your production Risk Analysis and Approval System service URLs prior to deployment. |
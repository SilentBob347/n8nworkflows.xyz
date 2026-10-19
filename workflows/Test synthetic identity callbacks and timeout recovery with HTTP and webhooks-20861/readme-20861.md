Test synthetic identity callbacks and timeout recovery with HTTP and webhooks

https://n8nworkflows.xyz/workflows/test-synthetic-identity-callbacks-and-timeout-recovery-with-http-and-webhooks-20861


# Test synthetic identity callbacks and timeout recovery with HTTP and webhooks

### 1. Workflow Overview

The **GridZen synthetic identity delivery: duplicate callbacks and bounded recovery** workflow is a self-hosted integration testing suite designed to evaluate synthetic identity onboarding callbacks, duplicate event handling, account binding validation, and bounded timeout recovery. It interacts with a local SQLite-backed Python fixture service to simulate onboarding state transitions without calling paid providers or modifying production customer permissions.

The workflow logic is grouped into three independent functional execution blocks:

- **1.1 Session Initialization and Bounded Timeout Recovery:** Initiates a synthetic session via the local fixture service, waits for a predefined period (65 seconds), and triggers a timeout recovery evaluation against the persistent state.
- **1.2 Webhook Callback Processing:** Receives incoming synthetic events via webhook, posts them to the local fixture for validation and state transition, and responds to the webhook with the calculated outcome or error handling codes.
- **1.3 Fixture Status Inspection:** Exposes a GET webhook endpoint to inspect the current persisted state of a fixture session directly from the local SQLite storage.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Session Initialization and Bounded Timeout Recovery
- **Overview:** This block handles the manual lifecycle start of a synthetic testing session. It creates the session via the local HTTP fixture, pauses execution for a bounded deadline, and then queries the fixture service to recover or evaluate the expired or pending state.
- **Nodes Involved:** 
  - `Start fixture`
  - `Create fixture session`
  - `Wait bounded deadline`
  - `Recover existing state`
- **Node Details:**
  - **Start fixture**
    - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` (Entry Point) — Manually initiates the testing sequence.
    - *Configuration Choices:* Default manual trigger parameters.
    - *Input/Output Connections:* Output connects to `Create fixture session`.
    - *Edge Cases/Failures:* None (manual execution).
  - **Create fixture session**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request) — Sends a POST request to initialize a new session in the local fixture.
    - *Configuration Choices:* URL set to `http://127.0.0.1:8787/sessions`, method `POST`, 3000ms timeout, JSON body `{}`.
    - *Input/Output Connections:* Input from `Start fixture`; output connects to `Wait bounded deadline`.
    - *Edge Cases/Failures:* HTTP timeout or connection failure if the Python fixture server is not running in the same network namespace.
  - **Wait bounded deadline**
    - *Type & Technical Role:* `n8n-nodes-base.wait` (Flow Control) — Pauses workflow execution for a bounded duration before checking recovery states.
    - *Configuration Choices:* Wait amount set to `65` seconds.
    - *Input/Output Connections:* Input from `Create fixture session`; output connects to `Recover existing state`.
    - *Edge Cases/Failures:* n8n instance restarts during the wait period may disrupt resumption depending on persistence settings.
  - **Recover existing state**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request) — Sends a POST request to evaluate session recovery after the deadline expires.
    - *Configuration Choices:* URL set to `http://127.0.0.1:8787/recover`, method `POST`, 3000ms timeout, JSON body `{}`.
    - *Input/Output Connections:* Input from `Wait bounded deadline`; no downstream output.
    - *Edge Cases/Failures:* HTTP failure if the fixture service is unreachable.

#### Block 1.2: Webhook Callback Processing
- **Overview:** This block receives incoming synthetic callback events, forwards them to the local fixture for tenant, user, and session binding validation, and returns an appropriate HTTP status response based on the fixture's output.
- **Nodes Involved:** 
  - `Fixture callback`
  - `Apply bound fixture event`
  - `Respond with fixture outcome`
- **Node Details:**
  - **Fixture callback**
    - *Type & Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger) — Listens for incoming POST requests representing synthetic onboarding events.
    - *Configuration Choices:* Path set to `gridzen-fixture-callback`, HTTP method `POST`, response mode set to `responseNode`.
    - *Input/Output Connections:* Output connects to `Apply bound fixture event`.
    - *Edge Cases/Failures:* Malformed payloads or incorrect webhook paths.
  - **Apply bound fixture event**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request) — Forwards the incoming webhook payload to the local fixture service for validation and state application.
    - *Configuration Choices:* URL set to `http://127.0.0.1:8787/events`, method `POST`, 3000ms timeout, JSON body configured via expression: `={{ JSON.stringify($json.body) }}`. Error handling configured to "Continue Regular Output" (`onError: continueRegularOutput`).
    - *Input/Output Connections:* Input from `Fixture callback`; output connects to `Respond with fixture outcome`.
    - *Edge Cases/Failures:* Fixture validation errors (e.g., incorrect bindings or duplicate events) trigger regular output continuation to allow proper error response formatting.
  - **Respond with fixture outcome**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Webhook Response) — Returns the processed fixture result or a standardized error object to the webhook caller.
    - *Configuration Choices:* Respond with `json`. Dynamic response code expression: `={{ $json.error ? 422 : 200 }}`. Dynamic response body expression: `={{ $json.error ? {mode:'synthetic_fixture',status:'unknown',permission_granted:false,error:'fixture_service_or_binding_error'} : $json }}`.
    - *Input/Output Connections:* Input from `Apply bound fixture event`; no downstream output.
    - *Edge Cases/Failures:* Expression evaluation failures if the upstream JSON structure differs from expectations.

#### Block 1.3: Fixture Status Inspection
- **Overview:** Provides an inspection endpoint to read the current persisted state of a specific fixture session from the local SQLite store via HTTP GET.
- **Nodes Involved:** 
  - `Read fixture status`
  - `Get persisted state`
  - `Return persisted state`
- **Node Details:**
  - **Read fixture status**
    - *Type & Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger) — Listens for GET requests to inspect session status.
    - *Configuration Choices:* Path set to `gridzen-fixture-status`, HTTP method `GET`, response mode set to `responseNode`.
    - *Input/Output Connections:* Output connects to `Get persisted state`.
    - *Edge Cases/Failures:* Incorrect webhook path or method.
  - **Get persisted state**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request) — Fetches the session state from the local fixture service.
    - *Configuration Choices:* URL set to `http://127.0.0.1:8787/sessions/fixture-session`, method `GET`, 3000ms timeout.
    - *Input/Output Connections:* Input from `Read fixture status`; output connects to `Return persisted state`.
    - *Edge Cases/Failures:* Session not found or fixture service offline.
  - **Return persisted state**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Webhook Response) — Returns the retrieved session state JSON to the caller.
    - *Configuration Choices:* Respond with `json`. Response body expression: `={{ $json }}`.
    - *Input/Output Connections:* Input from `Get persisted state`; no downstream output.
    - *Edge Cases/Failures:* Empty or invalid JSON payloads from the HTTP request node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Start fixture | `n8n-nodes-base.manualTrigger` | Entry point to start the test session | None | Create fixture session | Synthetic fixture only<br>Read before running — synthetic local fixture |
| Create fixture session | `n8n-nodes-base.httpRequest` | Initializes session on local fixture | Start fixture | Wait bounded deadline | Synthetic fixture only<br>Read before running — synthetic local fixture |
| Wait bounded deadline | `n8n-nodes-base.wait` | Pauses execution for 65 seconds | Create fixture session | Recover existing state | Synthetic fixture only<br>Read before running — synthetic local fixture |
| Recover existing state | `n8n-nodes-base.httpRequest` | Triggers timeout recovery check | Wait bounded deadline | None | Synthetic fixture only<br>Read before running — synthetic local fixture |
| Fixture callback | `n8n-nodes-base.webhook` | Receives synthetic onboarding events | None | Apply bound fixture event | Synthetic fixture only<br>Read before running — synthetic local fixture |
| Apply bound fixture event | `n8n-nodes-base.httpRequest` | Sends event payload to fixture | Fixture callback | Respond with fixture outcome | Synthetic fixture only<br>Read before running — synthetic local fixture |
| Respond with fixture outcome | `n8n-nodes-base.respondToWebhook` | Returns fixture result or error code | Apply bound fixture event | None | Synthetic fixture only<br>Read before running — synthetic local fixture |
| Read fixture status | `n8n-nodes-base.webhook` | Endpoint to inspect session status | None | Get persisted state | Synthetic fixture only<br>Read before running — synthetic local fixture |
| Get persisted state | `n8n-nodes-base.httpRequest` | Fetches session state via HTTP GET | Read fixture status | Return persisted state | Synthetic fixture only<br>Read before running — synthetic local fixture |
| Return persisted state | `n8n-nodes-base.respondToWebhook` | Returns session state JSON | Get persisted state | None | Synthetic fixture only<br>Read before running — synthetic local fixture |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in a self-hosted n8n environment, follow these steps:

1. **Create Node 1 (`Start fixture`):**
   - Type: `n8n-nodes-base.manualTrigger`
   - Position: `[0, 0]`
2. **Create Node 2 (`Create fixture session`):**
   - Type: `n8n-nodes-base.httpRequest` (v4.2)
   - Position: `[220, 0]`
   - Parameters: Set method to `POST`, URL to `http://127.0.0.1:8787/sessions`, specify body as `JSON`, set JSON body to `{}`, and configure options timeout to `3000` ms.
3. **Create Node 3 (`Wait bounded deadline`):**
   - Type: `n8n-nodes-base.wait` (v1.1)
   - Position: `[440, 0]`
   - Parameters: Set unit to `seconds` and amount to `65`.
4. **Create Node 4 (`Recover existing state`):**
   - Type: `n8n-nodes-base.httpRequest` (v4.2)
   - Position: `[660, 0]`
   - Parameters: Set method to `POST`, URL to `http://127.0.0.1:8787/recover`, specify body as `JSON`, set JSON body to `{}`, and options timeout to `3000` ms.
5. **Create Node 5 (`Fixture callback`):**
   - Type: `n8n-nodes-base.webhook` (v2)
   - Position: `[0, 260]`
   - Parameters: Set HTTP method to `POST`, path to `gridzen-fixture-callback`, and response mode to `Response Node`.
6. **Create Node 6 (`Apply bound fixture event`):**
   - Type: `n8n-nodes-base.httpRequest` (v4.2)
   - Position: `[220, 260]`
   - Parameters: Set method to `POST`, URL to `http://127.0.0.1:8787/events`, specify body as `JSON`, set JSON body expression to `={{ JSON.stringify($json.body) }}`, options timeout to `3000` ms, and configure error handling to `Continue Regular Output`.
7. **Create Node 7 (`Respond with fixture outcome`):**
   - Type: `n8n-nodes-base.respondToWebhook` (v1.4)
   - Position: `[440, 260]`
   - Parameters: Respond with `json`, set response code expression to `={{ $json.error ? 422 : 200 }}`, and response body expression to `={{ $json.error ? {mode:'synthetic_fixture',status:'unknown',permission_granted:false,error:'fixture_service_or_binding_error'} : $json }}`.
8. **Create Node 8 (`Read fixture status`):**
   - Type: `n8n-nodes-base.webhook` (v2)
   - Position: `[0, 520]`
   - Parameters: Set HTTP method to `GET`, path to `gridzen-fixture-status`, and response mode to `Response Node`.
9. **Create Node 9 (`Get persisted state`):**
   - Type: `n8n-nodes-base.httpRequest` (v4.2)
   - Position: `[220, 520]`
   - Parameters: Set method to `GET`, URL to `http://127.0.0.1:8787/sessions/fixture-session`, and options timeout to `3000` ms.
10. **Create Node 10 (`Return persisted state`):**
    - Type: `n8n-nodes-base.respondToWebhook` (v1.4)
    - Position: `[440, 520]`
    - Parameters: Respond with `json`, and set response body expression to `={{ $json }}`.
11. **Establish Connections:**
    - `Start fixture` (main 0) $\rightarrow$ `Create fixture session` (main 0)
    - `Create fixture session` (main 0) $\rightarrow$ `Wait bounded deadline` (main 0)
    - `Wait bounded deadline` (main 0) $\rightarrow$ `Recover existing state` (main 0)
    - `Fixture callback` (main 0) $\rightarrow$ `Apply bound fixture event` (main 0)
    - `Apply bound fixture event` (main 0) $\rightarrow$ `Respond with fixture outcome` (main 0)
    - `Read fixture status` (main 0) $\rightarrow$ `Get persisted state` (main 0)
    - `Get persisted state` (main 0) $\rightarrow$ `Return persisted state` (main 0)

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Clone the repository and follow setup instructions for the Python fixture server. | [GitHub Repository: immurray/gridzen-developer-kit](https://github.com/immurray/gridzen-developer-kit) |
| Requirements & Network constraints | Requires self-hosted n8n and Python 3 running in the same network namespace to reach `127.0.0.1:8787`. n8n Cloud cannot reach this loopback service. |
| Scope & Limitations | Synthetic integration testing only. Does not perform real identity verification, signed supplier callbacks, or paid API calls. Approved status does not equal unconditional customer permission. |
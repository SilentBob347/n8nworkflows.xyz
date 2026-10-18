Monitor AI citation changes for your domain with TriRank and SMTP email

https://n8nworkflows.xyz/workflows/monitor-ai-citation-changes-for-your-domain-with-trirank-and-smtp-email-20942


# Monitor AI citation changes for your domain with TriRank and SMTP email

### 1. Workflow Overview

This workflow automates the weekly monitoring of a target domain's AI citation performance using the TriRank API. Executing every Monday morning, it fetches cached citation metrics (spanning ChatGPT, Claude, Gemini, and Perplexity), compares them against the previous week’s saved state via n8n workflow static data, and dispatches a plain-text email notification only if a change or baseline needs reporting. 

The logic is grouped into the following functional blocks:
- **1.1 Schedule & Configuration:** Triggers the weekly execution and injects the target domain configuration.
- **1.2 Data Retrieval:** Queries the TriRank API for cached citation signals without initiating costly AI generation tasks.
- **1.3 State Comparison & Evaluation:** Evaluates API responses, categorizes citation states, compares against static data, and generates change messages.
- **1.4 Notification Dispatch:** Determines if an alert is warranted and sends an email via SMTP.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Schedule & Configuration
- **Overview:** Initializes the workflow execution on a scheduled weekly cadence and injects the target domain settings and first-run notification preferences.
- **Nodes Involved:** 
  - `Every Monday 09:00` (`scheduleTrigger`)
  - `Configure domain` (`set`)

##### Node Details: `Every Monday 09:00`
- **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` (Trigger Node)
- **Configuration Choices:** Configured to fire execution every Monday at 09:00 based on the instance's timezone.
- **Key Expressions/Variables:** None.
- **Connections:** Output connects to `Configure domain`.
- **Version Requirements:** v1.2.
- **Edge Cases & Failure Types:** Timezone misconfigurations can cause unexpected execution times. Manual test executions will bypass this trigger and require manual inputs if static data dependencies aren't met.

##### Node Details: `Configure domain`
- **Type & Technical Role:** `n8n-nodes-base.set` (Data Transformation / Setup Node)
- **Configuration Choices:** Sets fixed string assignments for the domain to monitor (`example.com`) and a flag (`notifyOnFirstRun` set to `true`).
- **Key Expressions/Variables:** 
  - `domain`: `example.com`
  - `notifyOnFirstRun`: `true`
- **Connections:** Input from `Every Monday 09:00`; output connects to `Read TriRank signals`.
- **Version Requirements:** v3.4.
- **Edge Cases & Failure Types:** Invalid domain formats will result in 4xx or 5xx responses from the downstream API request.

---

#### Block 1.2: Data Retrieval
- **Overview:** Performs an HTTP GET request to fetch cached AI citation signals for the configured domain from the public TriRank API.
- **Nodes Involved:** 
  - `Read TriRank signals` (`httpRequest`)

##### Node Details: `Read TriRank signals`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Request Node)
- **Configuration Choices:** Executes a GET request with a 30-second timeout. Configured with error handling (`onError: continueRegularOutput`) and full response parsing enabled to capture HTTP status codes and headers without halting the execution on non-200 responses.
- **Key Expressions/Variables:** 
  - URL: `=https://trirankai.com/api/signals/{{ $json.domain }}`
- **Connections:** Input from `Configure domain`; output connects to `Compare with last run`.
- **Version Requirements:** v4.4.
- **Edge Cases & Failure Types:** Rate-limiting (HTTP 429) or upstream downtime (HTTP 5xx) are caught safely via `continueRegularOutput` and handled gracefully in the subsequent code node. Optional authentication via a Header Auth credential (`x-api-key`) can be added to increase rate limits.

---

#### Block 1.3: State Comparison & Evaluation
- **Overview:** Processes the raw HTTP response, categorizes citation states (scored, partial, or unmeasured), saves the current reading to workflow static data, and generates a notification message if changes or initial baselines occur.
- **Nodes Involved:** 
  - `Compare with last run` (`code`)
  - `Anything to report?` (`if`)

##### Node Details: `Compare with last run`
- **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node)
- **Configuration Choices:** Runs once for all items (`runOnceForAllItems`). Interprets HTTP status codes and JSON bodies, determines citation states (`scored`, `partial`, `not_measured`, or stale), interacts with workflow static data (`$getWorkflowStaticData('global')`), and determines if a notification is necessary.
- **Key Expressions/Variables:** 
  - Reads upstream domain configuration via `$('Configure domain').first().json`
  - Accesses workflow static data storage for persistence across scheduled runs.
- **Connections:** Input from `Read TriRank signals`; output connects to `Anything to report?`.
- **Version Requirements:** v2.
- **Edge Cases & Failure Types:** Static data does not persist across *manual test executions*, meaning manual runs will always behave as a "first run" baseline unless executed in an active schedule context.

##### Node Details: `Anything to report?`
- **Type & Technical Role:** `n8n-nodes-base.if` (Conditional Router Node)
- **Configuration Choices:** Evaluates whether the generated `message` property exists and is populated.
- **Key Expressions/Variables:** 
  - Left Value: `={{ $json.message }}`
  - Operator: Exists
- **Connections:** Input from `Compare with last run`; true output connects to `Send email`, false branch terminates execution.
- **Version Requirements:** v2.3.
- **Edge Cases & Failure Types:** If no changes occur week-over-week, the message evaluates to null, correctly routing execution to the inactive branch (quiet weeks send no email).

---

#### Block 1.4: Notification Dispatch
- **Overview:** Delivers the generated alert message via an SMTP email provider.
- **Nodes Involved:** 
  - `Send email` (`emailSend`)

##### Node Details: `Send email`
- **Type & Technical Role:** `n8n-nodes-base.emailSend` (Notification Node)
- **Configuration Choices:** Configured to send plain-text emails with up to 3 retry attempts on failure, using a 5-second backoff delay between tries (`retryOnFail: true`, `maxTries: 3`, `waitBetweenTries: 5000`). Handles errors gracefully using `continueRegularOutput`.
- **Key Expressions/Variables:** 
  - Subject: `={{ ($json.changed ? "AI citation change for " : "TriRank citation baseline for ") + $json.domain }}`
  - Body Text: `={{ $json.message }}`
  - To Email: `you@example.com`
  - From Email: `user@example.com`
- **Connections:** Input from the true branch of `Anything to report?`.
- **Version Requirements:** v2.1.
- **Edge Cases & Failure Types:** Requires valid SMTP credentials. Authentication failures or unreachable mail servers will trigger retry logic before completing or continuing execution.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Every Monday 09:00** | `scheduleTrigger` | Weekly workflow trigger | None | Configure domain | **Overview & Setup**<br>**1 · Schedule & config** |
| **Configure domain** | `set` | Defines target domain and run options | Every Monday 09:00 | Read TriRank signals | **Overview & Setup**<br>**1 · Schedule & config** |
| **Read TriRank signals** | `httpRequest` | Fetches cached citation signals via API | Configure domain | Compare with last run | **Overview & Setup**<br>**2 · Read TriRank** |
| **Compare with last run** | `code` | Evaluates states and computes changes | Read TriRank signals | Anything to report? | **Overview & Setup**<br>**3 · Compare with last run** |
| **Anything to report?** | `if` | Routes workflow based on message presence | Compare with last run | Send email, (none) | **Overview & Setup**<br>**4 · Notify** |
| **Send email** | `emailSend` | Sends alerts via SMTP | Anything to report? | None | **Overview & Setup**<br>**4 · Notify** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** (`scheduleTrigger`) node.
   - Name it `Every Monday 09:00`.
   - Set interval rules to trigger every week on Monday at 09:00.

2. **Create the Domain Configuration Node:**
   - Add a **Set** (`set`) node named `Configure domain`.
   - Connect the output of `Every Monday 09:00` to this node.
   - Add string assignments:
     - Name: `domain`, Value: `example.com`
     - Name: `notifyOnFirstRun`, Value: `true`

3. **Create the HTTP Request Node:**
   - Add an **HTTP Request** (`httpRequest`) node named `Read TriRank signals`.
   - Connect the output of `Configure domain` to this node.
   - Set Method to `GET`.
   - Set URL to `=https://trirankai.com/api/signals/{{ $json.domain }}`.
   - Under Options, configure a timeout of `30000` ms, set Response Format to `JSON`, enable `Never Error`, and set Response to return the Full Response.
   - Set Node Error Handling to **Continue Regular Output**.
   - *(Optional)* Add a Header Auth credential named `x-api-key` if utilizing a personal TriRank API key.

4. **Create the Comparison Code Node:**
   - Add a **Code** (`code`) node named `Compare with last run`.
   - Connect the output of `Read TriRank signals` to this node.
   - Set mode to `Run Once for All Items`.
   - Paste the JavaScript logic that reads upstream configuration, parses TriRank states (`scored`, `partial`, `not_measured`), interacts with workflow static data (`$getWorkflowStaticData('global')`), and outputs the change state and message.

5. **Create the Conditional Router Node:**
   - Add an **If** (`if`) node named `Anything to report?`.
   - Connect the output of `Compare with last run` to this node.
   - Configure a condition where the left value `={{ $json.message }}` exists.

6. **Create the Email Notification Node:**
   - Add a **Send Email** (`emailSend`) node named `Send email`.
   - Connect the true output branch of `Anything to report?` to this node.
   - Configure parameters:
     - **Email Format:** Text
     - **From Email:** `user@example.com`
     - **To Email:** `you@example.com`
     - **Subject:** `={{ ($json.changed ? "AI citation change for " : "TriRank citation baseline for ") + $json.domain }}`
     - **Text:** `={{ $json.message }}`
   - Configure Retry on Fail: enabled (`maxTries: 3`, `waitBetweenTries: 5000`).
   - Set Node Error Handling to **Continue Regular Output**.
   - Attach a valid **SMTP** credential.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| TriRank Official Website | https://trirankai.com |
| Free Audit and Measurement Setup | https://trirankai.com/audit |
| Site Report URL Structure | https://trirankai.com/site/{domain} |
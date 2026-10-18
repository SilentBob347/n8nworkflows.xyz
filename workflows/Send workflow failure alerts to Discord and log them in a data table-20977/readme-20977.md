Send workflow failure alerts to Discord and log them in a data table

https://n8nworkflows.xyz/workflows/send-workflow-failure-alerts-to-discord-and-log-them-in-a-data-table-20977


# Send workflow failure alerts to Discord and log them in a data table

### 1. Workflow Overview

This workflow catches production execution failures from other n8n workflows (or manual test executions), standardizes and formats the error information, distributes a readable alert to a Discord channel via webhook, logs structured metadata into an n8n Data Table for auditing and reporting, and optionally triggers a secondary asynchronous triage process.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception:** Triggers the workflow either automatically upon an error in a monitored production workflow or manually for testing purposes.
- **1.2 Configuration & Formatting:** Establishes operational parameters (such as timezones) and transforms raw error objects into clean, Discord-ready markdown messages and structured JSON payloads.
- **1.3 Delivery & Logging:** Transmits the formatted alert payload to Discord with built-in retry logic and independently records the failure details inside an n8n Data Table.
- **1.4 Optional Triage Hand-Off:** Dispatches the raw failure payload asynchronously to an external sub-workflow for AI-driven or advanced triage without blocking the main alert delivery.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** This block acts as the entry point for the workflow, handling both live production failures and manual verification tests.
- **Nodes Involved:**
  - `On Workflow Error`
  - `Test Alert (manual)`

- **Node Details:**
  - **On Workflow Error**
    - *Type and Technical Role:* `n8n-nodes-base.errorTrigger` (v1) - Automatically activates whenever a designated production workflow encounters an unhandled exception.
    - *Configuration Choices:* Standard error trigger configuration with no custom parameters required.
    - *Key Expressions/Variables:* Outputs execution context object containing workflow metadata, error details, and node execution states.
    - *Input/Output Connections:* Output connects to `Build Config` and `Hand Off to Failure Triage`.
    - *Edge Cases/Failure Types:* Does not fire on manual test executions of monitored workflows; only triggers during active production runs.
  
  - **Test Alert (manual)**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (v1) - Allows developers to manually initiate the workflow to test alert routing and formatting.
    - *Configuration Choices:* Default manual execution trigger.
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Output connects to `Build Config`.
    - *Edge Cases/Failure Types:* Generates mock payload properties; relies on `Format Alert` to simulate fallback messaging when workflow metadata is absent.

---

#### 2.2 Configuration & Formatting
- **Overview:** Standardizes the runtime environment variables and parses complex error objects into a human-readable, unified notification structure.
- **Nodes Involved:**
  - `Build Config`
  - `Format Alert`

- **Node Details:**
  - **Build Config**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) - Injects configuration parameters (specifically timezone formatting) into the execution data stream.
    - *Configuration Choices:* Manual assignment mode, passing along other input fields (`includeOtherFields: true`).
    - *Key Expressions/Variables:* Sets `timezone` string to `America/New_York`.
    - *Input/Output Connections:* Inputs from `On Workflow Error` and `Test Alert (manual)`. Output connects to `Format Alert`.
    - *Edge Cases/Failure Types:* Mismatched timezone strings may cause invalid date formatting outputs in subsequent nodes.

  - **Format Alert**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) - Executes JavaScript to process raw error contexts, truncate lengthy messages, construct markdown lines, and format payload keys for downstream services.
    - *Configuration Choices:* Runs once for all items using JavaScript.
    - *Key Expressions/Variables:* Accesses `{{ $input.first().json }}` to extract execution details (`$json.workflow`, `$json.execution`, `$json.trigger`, `$json.timezone`), truncates error descriptions, and constructs execution links based on the `N8N_EDITOR_BASE_URL` environment variable.
    - *Input/Output Connections:* Input from `Build Config`. Outputs connect to `Post Alert to Discord` and `Log Failure to Data Table`.
    - *Edge Cases/Failure Types:* JavaScript execution failures if incoming object structures deviate completely from expected schemas; handled safely via internal nullish coalescing.

---

#### 2.3 Delivery & Logging
- **Overview:** Persists error logs to long-term storage and broadcasts formatted alerts to communication channels with fault-tolerant execution protection.
- **Nodes Involved:**
  - `Post Alert to Discord`
  - `Log Failure to Data Table`

- **Node Details:**
  - **Post Alert to Discord**
    - *Type and Technical Role:* `n8n-nodes-base.discord` (v2) - Sends HTTP requests via webhook integration to a configured Discord channel.
    - *Configuration Choices:* Uses legacy webhook operation (`sendLegacy`), webhook authentication, and error-handling retries (`maxTries: 3`, `waitBetweenTries: 3000ms`, `retryOnFail: true`).
    - *Key Expressions/Variables:* `={{ $json.content }}`
    - *Input/Output Connections:* Input from `Format Alert`. Terminal node.
    - *Edge Cases/Failure Types:* Invalid or expired Discord webhooks, rate limiting, or network timeouts. Mitigated by built-in retry parameters.
    - *Credentials Required:* `discordWebhookApi` (Discord webhook).

  - **Log Failure to Data Table**
    - *Type and Technical Role:* `n8n-nodes-base.dataTable` (v1.1) - Inserts structured workflow failure records into an internal n8n Data Table.
    - *Configuration Choices:* Operation set to row insertion (`insert`), mapping mode configured manually (`defineBelow`), and error handling set to continue regular execution upon failure (`onError: continueRegularOutput`).
    - *Key Expressions/Variables:* Maps columns:
      - `workflow`: `={{ $json.workflow }}`
      - `failed_node`: `={{ $json.failed_node }}`
      - `error`: `={{ $json.error }}`
      - `mode`: `={{ $json.mode }}`
      - `execution_url`: `={{ $json.execution_url }}`
    - *Input/Output Connections:* Input from `Format Alert`. Terminal node.
    - *Edge Cases/Failure Types:* Missing target data table (`workflow_failures`) or schema mismatch. Configured with fail-soft error handling (`continueRegularOutput`) to ensure Discord notifications succeed even if table insertion fails.

---

#### 2.4 Optional Triage Hand-Off
- **Overview:** Offloads the raw failure payload asynchronously to an external sub-workflow for secondary processing or AI-driven diagnosis without impacting primary alert delivery.
- **Nodes Involved:**
  - `Hand Off to Failure Triage`

- **Node Details:**
  - **Hand Off to Failure Triage**
    - *Type and Technical Role:* `n8n-nodes-base.executeWorkflow` (v1.3) - Invokes a designated sub-workflow.
    - *Configuration Choices:* Source set to database, execution mode set to `once`, configured *not* to wait for sub-workflow completion (`waitForSubWorkflow: false`), and failure handling set to continue regular execution (`onError: continueRegularOutput`).
    - *Key Expressions/Variables:* Target workflow ID is dynamically selectable.
    - *Input/Output Connections:* Input from `On Workflow Error`. Terminal node.
    - *Edge Cases/Failure Types:* Unselected or deleted sub-workflow targets will throw execution warnings. Fail-soft behavior ensures primary alerting remains unaffected.
    - *Sub-Workflow Reference:* Intended to integrate with auxiliary triage workflows (e.g., AI-based error analysis models like local Ollama instances).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On Workflow Error | `errorTrigger` | Triggers on production workflow failures | None | Build Config, Hand Off to Failure Triage | ## Send n8n workflow failure alerts to Discord and log them to a data table<br><br>### Who's it for<br>Anyone running n8n who wants to know within seconds when a production workflow fails, with enough detail to act before opening the editor.<br><br>### How it works<br>1. Set this workflow as the **Error workflow** of your other workflows (Workflow Settings). It fires when one of them fails a production run. Manual test runs never trigger it.<br>2. **Build Config** holds the time zone used in alerts.<br>3. **Format Alert** builds one message: workflow, failing node, error, detail, mode, time and a link to the execution.<br>4. **Post Alert to Discord** sends it with 3 retries. **Log Failure to Data Table** records it for later reporting (fail-soft).<br>5. Optional: **Hand Off to Failure Triage** passes the failure to an AI triage workflow without waiting. It sits lowest so it runs last and is fail-soft, so the alert never depends on AI.<br><br>### Setup<br>- A Discord webhook credential (channel settings > Integrations > Webhooks)<br>- A data table named `workflow_failures` with text columns `workflow`, `failed_node`, `error`, `mode`, `execution_url`. Select it in **Log Failure to Data Table**.<br><br>- Set `N8N_EDITOR_BASE_URL` so execution links open your editor<br>- Run **Test Alert (manual)** to check delivery<br><br>### Customization<br>Swap the Discord node for Slack, Telegram or email, or change the time zone in **Build Config**. For AI diagnosis, import the companion template *Diagnose failed n8n workflows with a local Ollama model* and select it in the hand-off node. Otherwise delete that node.<br>## 1. Triggers<br>A failed production run, or a manual test alert. |
| Test Alert (manual) | `manualTrigger` | Manual execution trigger for testing | None | Build Config | ## Send n8n workflow failure alerts to Discord and log them to a data table<br><br>### Who's it for<br>Anyone running n8n who wants to know within seconds when a production workflow fails, with enough detail to act before opening the editor.<br><br>### How it works<br>1. Set this workflow as the **Error workflow** of your other workflows (Workflow Settings). It fires when one of them fails a production run. Manual test runs never trigger it.<br>2. **Build Config** holds the time zone used in alerts.<br>3. **Format Alert** builds one message: workflow, failing node, error, detail, mode, time and a link to the execution.<br>4. **Post Alert to Discord** sends it with 3 retries. **Log Failure to Data Table** records it for later reporting (fail-soft).<br>5. Optional: **Hand Off to Failure Triage** passes the failure to an AI triage workflow without waiting. It sits lowest so it runs last and is fail-soft, so the alert never depends on AI.<br><br>### Setup<br>- A Discord webhook credential (channel settings > Integrations > Webhooks)<br>- A data table named `workflow_failures` with text columns `workflow`, `failed_node`, `error`, `mode`, `execution_url`. Select it in **Log Failure to Data Table**.<br><br>- Set `N8N_EDITOR_BASE_URL` so execution links open your editor<br>- Run **Test Alert (manual)** to check delivery<br><br>### Customization<br>Swap the Discord node for Slack, Telegram or email, or change the time zone in **Build Config**. For AI diagnosis, import the companion template *Diagnose failed n8n workflows with a local Ollama model* and select it in the hand-off node. Otherwise delete that node.<br>## 1. Triggers<br>A failed production run, or a manual test alert. |
| Build Config | `set` | Injects configuration variables | On Workflow Error, Test Alert (manual) | Format Alert | ## Send n8n workflow failure alerts to Discord and log them to a data table<br><br>### Who's it for<br>Anyone running n8n who wants to know within seconds when a production workflow fails, with enough detail to act before opening the editor.<br><br>### How it works<br>1. Set this workflow as the **Error workflow** of your other workflows (Workflow Settings). It fires when one of them fails a production run. Manual test runs never trigger it.<br>2. **Build Config** holds the time zone used in alerts.<br>3. **Format Alert** builds one message: workflow, failing node, error, detail, mode, time and a link to the execution.<br>4. **Post Alert to Discord** sends it with 3 retries. **Log Failure to Data Table** records it for later reporting (fail-soft).<br>5. Optional: **Hand Off to Failure Triage** passes the failure to an AI triage workflow without waiting. It sits lowest so it runs last and is fail-soft, so the alert never depends on AI.<br><br>### Setup<br>- A Discord webhook credential (channel settings > Integrations > Webhooks)<br>- A data table named `workflow_failures` with text columns `workflow`, `failed_node`, `error`, `mode`, `execution_url`. Select it in **Log Failure to Data Table**.<br><br>- Set `N8N_EDITOR_BASE_URL` so execution links open your editor<br>- Run **Test Alert (manual)** to check delivery<br><br>### Customization<br>Swap the Discord node for Slack, Telegram or email, or change the time zone in **Build Config**. For AI diagnosis, import the companion template *Diagnose failed n8n workflows with a local Ollama model* and select it in the hand-off node. Otherwise delete that node.<br>## 2. Configure and format<br>Add your settings, then build one readable alert message. |
| Format Alert | `code` | Formats raw error properties into markdown and structured keys | Build Config | Post Alert to Discord, Log Failure to Data Table | ## Send n8n workflow failure alerts to Discord and log them to a data table<br><br>### Who's it for<br>Anyone running n8n who wants to know within seconds when a production workflow fails, with enough detail to act before opening the editor.<br><br>### How it works<br>1. Set this workflow as the **Error workflow** of your other workflows (Workflow Settings). It fires when one of them fails a production run. Manual test runs never trigger it.<br>2. **Build Config** holds the time zone used in alerts.<br>3. **Format Alert** builds one message: workflow, failing node, error, detail, mode, time and a link to the execution.<br>4. **Post Alert to Discord** sends it with 3 retries. **Log Failure to Data Table** records it for later reporting (fail-soft).<br>5. Optional: **Hand Off to Failure Triage** passes the failure to an AI triage workflow without waiting. It sits lowest so it runs last and is fail-soft, so the alert never depends on AI.<br><br>### Setup<br>- A Discord webhook credential (channel settings > Integrations > Webhooks)<br>- A data table named `workflow_failures` with text columns `workflow`, `failed_node`, `error`, `mode`, `execution_url`. Select it in **Log Failure to Data Table**.<br><br>- Set `N8N_EDITOR_BASE_URL` so execution links open your editor<br>- Run **Test Alert (manual)** to check delivery<br><br>### Customization<br>Swap the Discord node for Slack, Telegram or email, or change the time zone in **Build Config**. For AI diagnosis, import the companion template *Diagnose failed n8n workflows with a local Ollama model* and select it in the hand-off node. Otherwise delete that node.<br>## 2. Configure and format<br>Add your settings, then build one readable alert message. |
| Post Alert to Discord | `discord` | Posts formatted message to Discord via webhook | Format Alert | None | ## Send n8n workflow failure alerts to Discord and log them to a data table<br><br>### Who's it for<br>Anyone running n8n who wants to know within seconds when a production workflow fails, with enough detail to act before opening the editor.<br><br>### How it works<br>1. Set this workflow as the **Error workflow** of your other workflows (Workflow Settings). It fires when one of them fails a production run. Manual test runs never trigger it.<br>2. **Build Config** holds the time zone used in alerts.<br>3. **Format Alert** builds one message: workflow, failing node, error, detail, mode, time and a link to the execution.<br>4. **Post Alert to Discord** sends it with 3 retries. **Log Failure to Data Table** records it for later reporting (fail-soft).<br>5. Optional: **Hand Off to Failure Triage** passes the failure to an AI triage workflow without waiting. It sits lowest so it runs last and is fail-soft, so the alert never depends on AI.<br><br>### Setup<br>- A Discord webhook credential (channel settings > Integrations > Webhooks)<br>- A data table named `workflow_failures` with text columns `workflow`, `failed_node`, `error`, `mode`, `execution_url`. Select it in **Log Failure to Data Table**.<br><br>- Set `N8N_EDITOR_BASE_URL` so execution links open your editor<br>- Run **Test Alert (manual)** to check delivery<br><br>### Customization<br>Swap the Discord node for Slack, Telegram or email, or change the time zone in **Build Config**. For AI diagnosis, import the companion template *Diagnose failed n8n workflows with a local Ollama model* and select it in the hand-off node. Otherwise delete that node.<br>## 3. Alert and log<br>Post to Discord with retries, and record the failure in a data table. |
| Log Failure to Data Table | `dataTable` | Inserts failure records into an n8n Data Table | Format Alert | None | ## Send n8n workflow failure alerts to Discord and log them to a data table<br><br>### Who's it for<br>Anyone running n8n who wants to know within seconds when a production workflow fails, with enough detail to act before opening the editor.<br><br>### How it works<br>1. Set this workflow as the **Error workflow** of your other workflows (Workflow Settings). It fires when one of them fails a production run. Manual test runs never trigger it.<br>2. **Build Config** holds the time zone used in alerts.<br>3. **Format Alert** builds one message: workflow, failing node, error, detail, mode, time and a link to the execution.<br>4. **Post Alert to Discord** sends it with 3 retries. **Log Failure to Data Table** records it for later reporting (fail-soft).<br>5. Optional: **Hand Off to Failure Triage** passes the failure to an AI triage workflow without waiting. It sits lowest so it runs last and is fail-soft, so the alert never depends on AI.<br><br>### Setup<br>- A Discord webhook credential (channel settings > Integrations > Webhooks)<br>- A data table named `workflow_failures` with text columns `workflow`, `failed_node`, `error`, `mode`, `execution_url`. Select it in **Log Failure to Data Table**.<br><br>- Set `N8N_EDITOR_BASE_URL` so execution links open your editor<br>- Run **Test Alert (manual)** to check delivery<br><br>### Customization<br>Swap the Discord node for Slack, Telegram or email, or change the time zone in **Build Config**. For AI diagnosis, import the companion template *Diagnose failed n8n workflows with a local Ollama model* and select it in the hand-off node. Otherwise delete that node.<br>## 3. Alert and log<br>Post to Discord with retries, and record the failure in a data table. |
| Hand Off to Failure Triage | `executeWorkflow` | Asynchronously triggers a sub-workflow for secondary triage | On Workflow Error | None | ## Send n8n workflow failure alerts to Discord and log them to a data table<br><br>### Who's it for<br>Anyone running n8n who wants to know within seconds when a production workflow fails, with enough detail to act before opening the editor.<br><br>### How it works<br>1. Set this workflow as the **Error workflow** of your other workflows (Workflow Settings). It fires when one of them fails a production run. Manual test runs never trigger it.<br>2. **Build Config** holds the time zone used in alerts.<br>3. **Format Alert** builds one message: workflow, failing node, error, detail, mode, time and a link to the execution.<br>4. **Post Alert to Discord** sends it with 3 retries. **Log Failure to Data Table** records it for later reporting (fail-soft).<br>5. Optional: **Hand Off to Failure Triage** passes the failure to an AI triage workflow without waiting. It sits lowest so it runs last and is fail-soft, so the alert never depends on AI.<br><br>### Setup<br>- A Discord webhook credential (channel settings > Integrations > Webhooks)<br>- A data table named `workflow_failures` with text columns `workflow`, `failed_node`, `error`, `mode`, `execution_url`. Select it in **Log Failure to Data Table**.<br><br>- Set `N8N_EDITOR_BASE_URL` so execution links open your editor<br>- Run **Test Alert (manual)** to check delivery<br><br>### Customization<br>Swap the Discord node for Slack, Telegram or email, or change the time zone in **Build Config**. For AI diagnosis, import the companion template *Diagnose failed n8n workflows with a local Ollama model* and select it in the hand-off node. Otherwise delete that node.<br>## 4. Optional AI triage<br>Hands the raw failure to a triage workflow without waiting. Fail-soft: the alert never depends on it. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to manually recreate the workflow in your n8n instance:

1. **Create Triggers:**
   - Add an **Error Trigger** node (`n8n-nodes-base.errorTrigger`). Name it `On Workflow Error`.
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Test Alert (manual)`.

2. **Add Configuration Node:**
   - Create a **Set (Edit Fields)** node (`n8n-nodes-base.set`). Name it `Build Config`.
   - Configure parameters: Set mode to `manual`, keep other fields, and add an assignment with Name: `timezone`, Type: `string`, Value: `America/New_York`.
   - Connect both `On Workflow Error` and `Test Alert (manual)` outputs to `Build Config`.

3. **Add Formatting Code Node:**
   - Create a Code node (`n8n-nodes-base.code`). Name it `Format Alert`.
   - Set mode to `Run Once for All Items` and language to `JavaScript`.
   - Paste the following transformation logic into the JS code editor:
     ```javascript
     const d = $input.first().json || {};
     const tz = d.timezone || 'UTC';
     const wf = d.workflow || {};
     const ex = d.execution || {};
     const tr = d.trigger || {};
     const err = ex.error || tr.error || {};
     const isTest = !d.workflow;
     const name = wf.name || 'TEST - Error Handler';
     const node = ex.lastNodeExecuted || (tr.error && tr.error.node && tr.error.node.name) || 'n/a';
     const msg = String(err.message || (isTest ? 'This is a test alert. If you can read this, failure alerts are working.' : 'Unknown error')).slice(0, 500);
     const desc = err.description ? String(err.description).slice(0, 300) : '';
     const when = new Date().toLocaleString('en-US', { timeZone: tz });
     const execUrl = String(ex.url || '');
     const lines = [];
     lines.push('⚠️ **n8n workflow failed: ' + name + '**');
     lines.push('- Failed at node: ' + node);
     lines.push('- Error: ' + msg);
     if (desc) lines.push('- Detail: ' + desc);
     lines.push('- Mode: ' + (ex.mode || (tr.mode || 'manual')) + ' | ' + when);
     if (execUrl) lines.push('- Execution: <' + execUrl + '>');
     return [{ json: { content: lines.join('\n'), workflow: name, failed_node: String(node), error: msg, mode: String(ex.mode || tr.mode || 'manual'), execution_url: execUrl } }];
     ```
   - Connect `Build Config` output to `Format Alert`.

4. **Add Discord Notification Node:**
   - Create a Discord node (`n8n-nodes-base.discord`). Name it `Post Alert to Discord`.
   - Configure parameters: Resource: `webhook`, Operation: `sendLegacy`, Authentication: `webhook`.
   - Set Content expression to: `={{ $json.content }}`
   - Configure retry parameters: Enable retry on fail (`retryOnFail: true`), set max tries to `3`, and wait between tries to `3000` ms.
   - Configure Credentials: Link or create a `Discord webhook` credential.
   - Connect `Format Alert` output to `Post Alert to Discord`.

5. **Add Data Table Logging Node:**
   - Create a Data Table node (`n8n-nodes-base.dataTable`). Name it `Log Failure to Data Table`.
   - Configure parameters: Resource: `row`, Operation: `insert`, Mapping Mode: `Define Below`.
   - Select the target data table named `workflow_failures` (ensure table contains text columns: `workflow`, `failed_node`, `error`, `mode`, `execution_url`).
   - Map columns to JSON keys:
     - `workflow`: `={{ $json.workflow }}`
     - `failed_node`: `={{ $json.failed_node }}`
     - `error`: `={{ $json.error }}`
     - `mode`: `={{ $json.mode }}`
     - `execution_url`: `={{ $json.execution_url }}`
   - Set On Error behavior to `Continue Regular Output` (`onError: continueRegularOutput`).
   - Connect `Format Alert` output to `Log Failure to Data Table`.

6. **Add Optional Triage Sub-Workflow Execution Node:**
   - Create an Execute Workflow node (`n8n-nodes-base.executeWorkflow`). Name it `Hand Off to Failure Triage`.
   - Configure parameters: Source: `database`, Execution Mode: `Once`, Uncheck `Wait for Sub-Workflow` (`waitForSubWorkflow: false`).
   - Select your target triage sub-workflow from the workflow list (if applicable).
   - Set On Error behavior to `Continue Regular Output` (`onError: continueRegularOutput`).
   - Connect `On Workflow Error` output directly to `Hand Off to Failure Triage`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Production vs Manual Triggers | Error workflows only fire on production runs and never on manual test runs. Always use the manual test trigger (`Test Alert (manual)`) to verify alert routing and webhook delivery. |
| Environment Configuration | Ensure the environment variable `N8N_EDITOR_BASE_URL` is configured in your n8n host environment so execution links generated in alerts link back to your n8n editor. |
| Fail-Soft Architecture | Logging and triage nodes are configured with fail-soft execution parameters (`continueRegularOutput`) to ensure that delivery of critical alerts to Discord remains uninterrupted even if auxiliary database writes or sub-workflows encounter issues. |
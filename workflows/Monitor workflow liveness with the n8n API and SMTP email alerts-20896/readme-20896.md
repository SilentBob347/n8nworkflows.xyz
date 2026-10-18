Monitor workflow liveness with the n8n API and SMTP email alerts

https://n8nworkflows.xyz/workflows/monitor-workflow-liveness-with-the-n8n-api-and-smtp-email-alerts-20896


# Monitor workflow liveness with the n8n API and SMTP email alerts

### 1. Workflow Overview

This workflow functions as an automated daily watchdog for self-hosted or cloud-based n8n instances. Its primary purpose is to monitor active scheduled workflows, verify whether they have executed according to their expected schedule, and detect operational anomalies such as missed runs, high error rates, or unusual execution slowdowns. 

Rather than requiring a separately maintained catalog of monitored workflows, the system dynamically inspects all enabled `Schedule Trigger` nodes to calculate the maximum acceptable gap between runs (automatically accounting for non-working days like weekends). It evaluates execution history against predefined thresholds and dispatches a single consolidated HTML email alert only when actionable findings are discovered.

The workflow logic is categorized into four distinct functional blocks:

- **1.1 Input Reception & Configuration:** Handles entry points (scheduled execution and manual testing) and defines global operational parameters (such as recipient email addresses, error thresholds, and retention periods).
- **1.2 Watch List Construction:** Interfaces with the n8n API to fetch all active workflows, parses their scheduling logic, applies exclusion rules or overrides, and compiles a clean watch list.
- **1.3 Execution Analysis & Evaluation:** Queries historical execution data for each watched workflow via the n8n API, filters out manual test runs, and evaluates them against four specific liveness metrics (`OVERDUE`, `NO_SUCCESS`, `ERROR_RATE`, and `SLOW`).
- **1.4 Reporting & Notification Dispatch:** Consolidates all findings into a structured HTML report and conditionally triggers an SMTP email dispatch if any issues are identified.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Configuration
- **Overview:** Initializes the workflow execution either automatically via a daily schedule or manually on demand, and establishes the configuration variables required for filtering, thresholds, and notifications.
- **Nodes Involved:** 
  - `Daily Trigger 09:30`
  - `Manual Start Trigger`
  - `Set Configuration Values`

- **Node Details:**
  - **Daily Trigger 09:30**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (v1.3) — Serves as the automated primary entry point, firing once daily at 09:30.
    - *Configuration:* Rule interval set to 1 day, triggering at hour 9 and minute 30.
    - *Expressions / Variables:* None.
    - *Connections:* Output connects to `Set Configuration Values`.
    - *Version Requirements:* v1.3.
    - *Edge Cases / Failures:* Missed triggers if the n8n instance is offline at 09:30.

  - **Manual Start Trigger**
    - *Type & Role:* `n8n-nodes-base.manualTrigger` (v1) — Serves as the manual entry point for testing and debugging.
    - *Configuration:* Default manual execution settings.
    - *Expressions / Variables:* None.
    - *Connections:* Output connects to `Set Configuration Values`.
    - *Version Requirements:* v1.

  - **Set Configuration Values**
    - *Type & Role:* `n8n-nodes-base.set` (v3.4) — Defines global configuration parameters and operational thresholds as static key-value pairs.
    - *Configuration:* Manual mode with explicit assignments: `notifyTo` (string), `notifyFrom` (string), `overrides` (string/JSON), `ignoreNamePattern` (string), `defaultToleranceHours` (number, default `2`), `executionRetentionDays` (number, default `14`), `errorRateThreshold` (number, default `0.2`), and `minSlowSeconds` (number, default `60`).
    - *Expressions / Variables:* None.
    - *Connections:* Inputs from `Daily Trigger 09:30` and `Manual Start Trigger`; output connects to `Fetch Active Workflows`.
    - *Version Requirements:* v3.4.
    - *Edge Cases / Failures:* Invalid JSON strings supplied in `overrides` or malformed regex patterns in `ignoreNamePattern` will throw parsing errors in downstream code nodes.

---

#### Block 1.2: Watch List Construction
- **Overview:** Retrieves all active workflows from the n8n API, parses their trigger rhythms, applies filtering rules, and constructs an array of workflows slated for liveness evaluation.
- **Nodes Involved:**
  - `Fetch Active Workflows`
  - `Build Watch List from Schedules`

- **Node Details:**
  - **Fetch Active Workflows**
    - *Type & Role:* `n8n-nodes-base.n8n` (v1) — Communicates with the native n8n API to pull all active workflow definitions.
    - *Configuration:* Resource: `workflow`, Operation: `getAll`, Filters: `activeWorkflows: true`, `excludePinnedData: true`, `returnAll: true`. Requires valid n8n API credentials.
    - *Expressions / Variables:* None.
    - *Connections:* Input from `Set Configuration Values`; output connects to `Build Watch List from Schedules`.
    - *Version Requirements:* v1.
    - *Edge Cases / Failures:* Authentication errors or API connectivity failures if the n8n API key lacks permissions or has expired.

  - **Build Watch List from Schedules**
    - *Type & Role:* `n8n-nodes-base.code` (v2) — JavaScript execution node that processes active workflow definitions, evaluates cron expressions and interval triggers over a 400-day sliding window, calculates maximum gaps between runs, applies name-matching exclusions and configuration overrides, and outputs a refined watch list while storing un-assessed workflows in workflow static data.
    - *Configuration:* Mode set to `runOnceForAllItems` using custom JavaScript logic.
    - *Expressions / Variables:* References `$('Set Configuration Values').first().json` to read global configurations.
    - *Connections:* Input from `Fetch Active Workflows`; output connects to `Fetch Last 50 Executions`.
    - *Version Requirements:* v2.
    - *Edge Cases / Failures:* Complex or non-standard custom cron strings that fail parsing will cause the workflow to skip assessment for those specific items and log them to the skipped list.

---

#### Block 1.3: Execution Analysis & Evaluation
- **Overview:** Iterates through each watched workflow to query its recent execution history, filters out manual test runs, and runs algorithmic checks to identify liveness issues (`OVERDUE`, `NO_SUCCESS`, `ERROR_RATE`, and `SLOW`).
- **Nodes Involved:**
  - `Fetch Last 50 Executions`
  - `Check Liveness and Build Report`

- **Node Details:**
  - **Fetch Last 50 Executions**
    - *Type & Role:* `n8n-nodes-base.n8n` (v1) — Queries the n8n API to retrieve historical execution logs for each individual workflow passed from the watch list.
    - *Configuration:* Resource: `execution`, Operation: `getAll`, Limit: `50`, Return All: `false`, with execution filtering configured by workflow ID. Option `alwaysOutputData` enabled. Requires n8n API credentials.
    - *Expressions / Variables:* Uses `={{ $json.id }}` to dynamically target each workflow ID in the watch list.
    - *Connections:* Input from `Build Watch List from Schedules`; output connects to `Check Liveness and Build Report`.
    - *Version Requirements:* v1.
    - *Edge Cases / Failures:* Rate limiting or API timeout issues if the instance monitors a very large volume of active workflows.

  - **Check Liveness and Build Report**
    - *Type & Role:* `n8n-nodes-base.code` (v2) — JavaScript execution node that processes execution histories, computes error percentages over 7-day and 24-hour windows, compares run durations against rolling medians, formats an HTML report payload, and sets flags indicating whether findings exist.
    - *Configuration:* Mode set to `runOnceForAllItems` using custom JavaScript logic.
    - *Expressions / Variables:* Reads global configurations from `Set Configuration Values` and the watch list items from `Build Watch List from Schedules`.
    - *Connections:* Input from `Fetch Last 50 Executions`; output connects to `Check for Findings`.
    - *Version Requirements:* v2.
    - *Edge Cases / Failures:* Division by zero or date parsing exceptions if execution metadata contains unexpected null values.

---

#### Block 1.4: Reporting & Notification Dispatch
- **Overview:** Evaluates whether any operational issues were detected and conditionally dispatches a formatted HTML summary report via SMTP.
- **Nodes Involved:**
  - `Check for Findings`
  - `Dispatch Report Email`

- **Node Details:**
  - **Check for Findings**
    - *Type & Role:* `n8n-nodes-base.if` (v2.2) — Acts as a conditional gatekeeper, ensuring notifications are only sent when actionable anomalies are detected.
    - *Configuration:* Expression-based condition evaluating whether `{{ $json.hasFindings }}` evaluates to `true`.
    - *Expressions / Variables:* Evaluates `={{ $json.hasFindings }}`.
    - *Connections:* Input from `Check Liveness and Build Report`; output (`true` branch) connects to `Dispatch Report Email`.
    - *Version Requirements:* v2.2.
    - *Edge Cases / Failures:* None; standard boolean evaluation.

  - **Dispatch Report Email**
    - *Type & Role:* `n8n-nodes-base.emailSend` (v2.1) — Connects to an external SMTP server to deliver the HTML-formatted monitoring report.
    - *Configuration:* Operation: `send`, Email Format: `html`, Options: `appendAttribution: false`. Requires valid SMTP credentials.
    - *Expressions / Variables:* Uses `={{ $json.html }}` for the email body, `={{ $json.subject }}` for the subject line, `={{ $('Set Configuration Values').first().json.notifyTo }}` for recipients, and `={{ $('Set Configuration Values').first().json.notifyFrom }}` for the sender address.
    - *Connections:* Input from `Check for Findings` (`true` branch).
    - *Version Requirements:* v2.1.
    - *Edge Cases / Failures:* SMTP authentication failures, network timeouts, or invalid sender/recipient addresses will cause this node to fail.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Daily Trigger 09:30` | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow daily at 09:30 UTC. | None | `Set Configuration Values` | ## Trigger and configuration<br>Runs daily or manually. All settings live in one node. |
| `Set Configuration Values` | `n8n-nodes-base.set` | Assigns global variables, thresholds, and configuration options. | `Daily Trigger 09:30`, `Manual Start Trigger` | `Fetch Active Workflows` | ## Trigger and configuration<br>Runs daily or manually. All settings live in one node. |
| `Manual Start Trigger` | `n8n-nodes-base.manualTrigger` | Allows manual workflow execution for testing. | None | `Set Configuration Values` | ## Trigger and configuration<br>Runs daily or manually. All settings live in one node. |
| `Fetch Active Workflows` | `n8n-nodes-base.n8n` | Retrieves all active workflows from the n8n API. | `Set Configuration Values` | `Build Watch List from Schedules` | ## Watch list<br>Reads active workflows and derives the expected gap from each Schedule Trigger. |
| `Build Watch List from Schedules` | `n8n-nodes-base.code` | Parses schedules and builds the monitored workflow watch list. | `Fetch Active Workflows` | `Fetch Last 50 Executions` | ## Watch list<br>Reads active workflows and derives the expected gap from each Schedule Trigger. |
| `Fetch Last 50 Executions` | `n8n-nodes-base.n8n` | Fetches historical execution logs for each watched workflow. | `Build Watch List from Schedules` | `Check Liveness and Build Report` | ## Check executions<br>One API call per watched workflow, then OVERDUE, NO_SUCCESS, ERROR_RATE and SLOW are evaluated. |
| `Check Liveness and Build Report` | `n8n-nodes-base.code` | Evaluates executions against liveness rules and builds the HTML report. | `Fetch Last 50 Executions` | `Check for Findings` | ## Check executions<br>One API call per watched workflow, then OVERDUE, NO_SUCCESS, ERROR_RATE and SLOW are evaluated. |
| `Check for Findings` | `n8n-nodes-base.if` | Evaluates if findings exist to determine whether to send an alert. | `Check Liveness and Build Report` | `Dispatch Report Email` | ## Report<br>Email only when there is a finding. Workflows that could not be assessed are listed at the bottom. |
| `Dispatch Report Email` | `n8n-nodes-base.emailSend` | Sends the compiled HTML findings report via SMTP email. | `Check for Findings` | None | ## Report<br>Email only when there is a finding. Workflows that could not be assessed are listed at the bottom. |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | ## Monitor scheduled n8n workflows and email missed or failing runs<br>Daily check that your scheduled workflows actually ran. A failing workflow triggers your error workflow. A workflow that silently stops running triggers nothing - this one catches it.<br><br>### How it works<br>1. Loads all active workflows and reads the rhythm from each Schedule Trigger (interval, weekdays or cron). The largest gap between two runs becomes the threshold, so Mon-Fri schedules don't alarm on weekends.<br>2. Loads the last 50 executions of each watched workflow. Manual runs never count.<br>3. Reports OVERDUE, NO_SUCCESS, ERROR_RATE (more than 20% of at least 5 runs failed in 7 days, one of them in the last 24h) and SLOW (latest run more than 3x the median and over 60s).<br>4. Sends one email, only when there is something to report. Workflows that could not be assessed are listed in that email.<br><br>### Setup<br>1. Create an n8n API key (Settings > n8n API) and add it as an n8n API credential in *Fetch Active Workflows* and *Fetch Last 50 Executions*.<br>2. Add an SMTP credential to *Dispatch Report Email*.<br>3. Fill in notifyTo and notifyFrom in *Set Configuration Values*.<br>4. Set your error workflow in the workflow settings.<br>5. Run once manually, then activate.<br><br>### Customization<br>- overrides (JSON): watch webhook or sub-workflows too, or tune one workflow, for example `{"Order webhook": {"maxGapHours": 48}}`.<br>- ignoreNamePattern: skip workflows by name.<br>- executionRetentionDays: set it to your instance value. Longer rhythms are listed as not assessed.<br><br>Limits: workflows that don't save successful executions can't be assessed, and the watchdog can't report its own absence. | None | None | None |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Triggers:**
   - Add a **Schedule Trigger** node named `Daily Trigger 09:30`. Configure its rule interval to 1 day, with execution set for hour `9`, minute `30`.
   - Add a **Manual Trigger** node named `Manual Start Trigger` using default parameters.

2. **Configure Global Settings:**
   - Add a **Set** node named `Set Configuration Values`.
   - Set the mode to `Manual` and create the following string and number assignments:
     - `notifyTo` (string): Target email address (e.g., `you@example.com`)
     - `notifyFrom` (string): Sender address (e.g., `n8n <user@example.com>`)
     - `overrides` (string): `{}`
     - `ignoreNamePattern` (string): `""`
     - `defaultToleranceHours` (number): `2`
     - `executionRetentionDays` (number): `14`
     - `errorRateThreshold` (number): `0.2`
     - `minSlowSeconds` (number): `60`
   - Connect both `Daily Trigger 09:30` and `Manual Start Trigger` to `Set Configuration Values`.

3. **Fetch Active Workflows:**
   - Add an **n8n** node named `Fetch Active Workflows`.
   - Set Resource to `workflow`, Operation to `getAll`. Under filters, enable `activeWorkflows` and `excludePinnedData`, and set `returnAll` to `true`.
   - Configure n8n API credentials.
   - Connect `Set Configuration Values` to `Fetch Active Workflows`.

4. **Build the Watch List:**
   - Add a **Code** node named `Build Watch List from Schedules`.
   - Set mode to `runOnceForAllItems` and insert the JavaScript logic to parse schedules, compute maximum gaps, check overrides/patterns, and output the watch array while caching skipped names in global static data.
   - Connect `Fetch Active Workflows` to `Build Watch List from Schedules`.

5. **Retrieve Executions:**
   - Add an **n8n** node named `Fetch Last 50 Executions`.
   - Set Resource to `execution`, Operation to `getAll`, Limit to `50`, and Return All to `false`.
   - Bind workflow ID filtering to `={{ $json.id }}` and enable `alwaysOutputData`.
   - Configure n8n API credentials.
   - Connect `Build Watch List from Schedules` to `Fetch Last 50 Executions`.

6. **Analyze Liveness and Build the Report:**
   - Add a **Code** node named `Check Liveness and Build Report`.
   - Set mode to `runOnceForAllItems` and insert the JavaScript logic to filter out manual runs, evaluate `OVERDUE`, `NO_SUCCESS`, `ERROR_RATE`, and `SLOW` conditions, and construct the HTML findings payload and subject line.
   - Connect `Fetch Last 50 Executions` to `Check Liveness and Build Report`.

7. **Evaluate Findings:**
   - Add an **If** node named `Check for Findings`.
   - Configure condition to evaluate whether `={{ $json.hasFindings }}` is true.
   - Connect `Check Liveness and Build Report` to `Check for Findings`.

8. **Dispatch Notification Email:**
   - Add a **Send Email (SMTP)** node named `Dispatch Report Email`.
   - Configure operation to `send`, email format to `html`, and disable attribution appending.
   - Set `To Email` to `={{ $('Set Configuration Values').first().json.notifyTo }}`, `From Email` to `={{ $('Set Configuration Values').first().json.notifyFrom }}`, `Subject` to `={{ $json.subject }}`, and `Html` to `={{ $json.html }}`.
   - Configure SMTP credentials.
   - Connect the `true` output of `Check for Findings` to `Dispatch Report Email`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Daily check that scheduled workflows actually ran, catching silent failures that error workflows cannot detect. | Purpose & Liveness Watchdog Overview |
| Requires an n8n API key configured in the API settings of your self-hosted or cloud instance, and valid SMTP credentials for alert dispatch. | Authentication Prerequisites |
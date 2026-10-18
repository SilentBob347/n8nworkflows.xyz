Diagnose failed workflows with a local Ollama model and send triage reports to Discord

https://n8nworkflows.xyz/workflows/diagnose-failed-workflows-with-a-local-ollama-model-and-send-triage-reports-to-discord-20978


# Diagnose failed workflows with a local Ollama model and send triage reports to Discord

### 1. Workflow Overview

This workflow automates the triage of failed n8n executions using a local Ollama AI model and delivers a structured diagnostics report to Discord. It captures runtime errors, fetches execution and workflow contexts, redacts sensitive information, deduplicates alerts against a data table, generates an AI-powered root cause analysis with schema enforcement, applies deterministic safety guards, and logs the outcome.

The architecture is organized into four logical blocks:
- **1.1 Input Reception & Normalization:** Captures failures via error triggers, sub-workflow error handler handoffs, or manual sample tests, normalizing disparate input schemas into a unified payload format.
- **1.2 Configuration & Excluded Workflow Filtering:** Injects runtime operational parameters (Ollama URL, model name, retention windows) and prevents alert feedback loops by filtering out excluded workflow IDs.
- **1.3 Execution Context Retrieval, Data Scrubbing & Deduplication:** Queries the n8n API to gather node definitions and execution metadata, aggressively sanitizes secrets and PII, and cross-references an n8n Data Table to suppress repeat alerts within a configurable time window.
- **1.4 AI Diagnosis, Formatting, Reporting & Logging:** Submits the redacted context to Ollama for a schema-constrained JSON diagnosis, post-processes the output through deterministic safety checks, posts the message to Discord, and records the fingerprint in the triage history table.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Normalization
- **Overview:** Acts as the multi-source entry point for the workflow, accepting live production failures, propagated errors from companion handlers, or manual test cases, and unifies them into a standard schema.
- **Nodes Involved:** `On Workflow Error`, `Called by Error Handler`, `Test with Sample Failures`, `Sample Failures`, `Normalize Failure`.
- **Node Details:**
  - **On Workflow Error**
    - Type and technical role: `n8n-nodes-base.errorTrigger` (Trigger). Listens globally or per-workflow for execution failures.
    - Configuration choices: Default settings.
    - Input and output connections: Output connects to `Normalize Failure`.
    - Edge cases or potential failure types: Does not fire if an execution fails silently or inside a try-catch pattern without bubbling up.
  - **Called by Error Handler**
    - Type and technical role: `n8n-nodes-base.executeWorkflowTrigger` (Sub-workflow Trigger). Allows invocation from a parent error-handling workflow using passthrough input.
    - Configuration choices: Input source set to `passthrough`.
    - Input and output connections: Output connects to `Normalize Failure`.
  - **Test with Sample Failures**
    - Type and technical role: `n8n-nodes-base.manualTrigger` (Trigger). Enables interactive dry runs during development.
    - Configuration choices: Default parameters.
    - Input and output connections: Output connects to `Sample Failures`.
  - **Sample Failures**
    - Type and technical role: `n8n-nodes-base.code` (Data Transformation). Generates mocked failure objects containing placeholder execution and workflow IDs or simulated trigger failures.
    - Key expressions or variables: Outputs an array of objects mapped with `bypass_dedup: true`.
    - Input and output connections: Input from `Test with Sample Failures`; output to `Normalize Failure`.
  - **Normalize Failure**
    - Type and technical role: `n8n-nodes-base.code` (Data Transformation). Evaluates incoming payloads from any of the three triggers and extracts standard fields (`executionId`, `workflowId`, `workflowName`, `failedNode`, `errorMessage`, `bypass_dedup`).
    - Key expressions or variables: Uses JavaScript conditionals to parse `execution`, `workflow`, and `trigger` object paths dynamically.
    - Input and output connections: Inputs from `On Workflow Error`, `Called by Error Handler`, and `Sample Failures`; output to `Build Config`.

#### Block 1.2: Configuration & Excluded Workflow Filtering
- **Overview:** Enriches the failure item with environmental configuration constants and filters out workflows designated for exclusion to prevent infinite alert recursion.
- **Nodes Involved:** `Build Config`, `Skip Excluded Workflows`.
- **Node Details:**
  - **Build Config**
    - Type and technical role: `n8n-nodes-base.set` (Parameter Setter). Appends operational settings to the execution context.
    - Configuration choices: Assigns string and numeric fields: `ollama_url` (`http://localhost:11434`), `model` (`qwen3:8b`), `n8n_editor_url`, `excluded_workflow_ids`, `max_context_chars` (6000), and `dedup_hours` (6). Retains other input fields.
    - Input and output connections: Input from `Normalize Failure`; output to `Skip Excluded Workflows`.
  - **Skip Excluded Workflows**
    - Type and technical role: `n8n-nodes-base.filter` (Conditional Logic). Drops items if the failing workflow matches the error workflow's own ID or appears in the comma-separated exclusion list.
    - Key expressions or variables: `={{ $json.workflowId !== '' && $json.workflowId !== $workflow.id && !$json.excluded_workflow_ids.split(',').map(s => s.trim()).includes($json.workflowId) }}`
    - Input and output connections: Input from `Build Config`; output to `Get Failed Execution`.
    - Edge cases or potential failure types: Misconfigured comma-separated IDs can cause string matching failures.

#### Block 1.3: Execution Context Retrieval, Data Scrubbing & Deduplication
- **Overview:** Queries the n8n REST API for execution details and workflow schemas, sanitizes sensitive text, generates a unique failure fingerprint, and queries the data table to suppress duplicate alerts.
- **Nodes Involved:** `Get Failed Execution`, `Get Workflow Definition`, `Redact and Build Context`, `Get Recent Diagnoses`, `Skip Repeat Failures`.
- **Node Details:**
  - **Get Failed Execution**
    - Type and technical role: `n8n-nodes-base.n8n` (API Integration). Fetches complete execution logs from the n8n instance.
    - Configuration choices: Resource `execution`, operation `get`, using `executionId`. Configured with `retryOnFail: true`, max tries 2, wait time 2000ms, and `onError: continueRegularOutput`.
    - Credentials: `n8n API` credential.
    - Input and output connections: Input from `Skip Excluded Workflows`; output to `Get Workflow Definition`.
    - Edge cases or potential failure types: Authentication failures against the n8n API, or execution IDs that have already been pruned from instance retention history.
  - **Get Workflow Definition**
    - Type and technical role: `n8n-nodes-base.n8n` (API Integration). Retrieves the JSON structure of the failing workflow.
    - Configuration choices: Resource `workflow`, operation `get`, targeting `={{ $('Skip Excluded Workflows').item.json.workflowId }}`. Configured with retry logic and error continuation.
    - Credentials: `n8n API` credential.
    - Input and output connections: Input from `Get Failed Execution`; output to `Redact and Build Context`.
  - **Redact and Build Context**
    - Type and technical role: `n8n-nodes-base.code` (Data Transformation). Parses execution data, extracts node definitions, upstream input samples, sanitizes secrets/PII via regex, constructs the Ollama system prompt, and generates a deduplication fingerprint.
    - Key expressions or variables: References nodes using `$()`, including `Skip Excluded Workflows` and `Get Failed Execution`. Applies regex masks for Bearer tokens, JWTs, emails, keys, and webhooks.
    - Input and output connections: Input from `Get Workflow Definition`; output to `Get Recent Diagnoses`.
    - Edge cases or potential failure types: Regex patterns might miss non-standard secret key formats.
  - **Get Recent Diagnoses**
    - Type and technical role: `n8n-nodes-base.dataTable` (Database Operation). Queries historical records from the triage data table to check for past failures.
    - Configuration choices: Resource `row`, operation `get`, matching rows where `diagnosed_at` is greater than or equal to `={{ $now.minus({ hours: $json.dedup_hours }).toISO() }}`. Returns all matching rows, executed once, with error handling set to continue regular output.
    - Input and output connections: Input from `Redact and Build Context`; output to `Skip Repeat Failures`.
  - **Skip Repeat Failures**
    - Type and technical role: `n8n-nodes-base.code` (Data Transformation). Compares current failure fingerprints against recent database entries to filter out duplicates.
    - Key expressions or variables: References `Redact and Build Context` items via `$()`.
    - Input and output connections: Input from `Get Recent Diagnoses`; output to `Diagnose Failure`.

#### Block 1.4: AI Diagnosis, Formatting, Reporting & Logging
- **Overview:** Submits the structured prompt to Ollama, enforces deterministic code guards on the output, formats the triage message, sends it via Discord webhook, and writes an audit entry to the data table.
- **Nodes Involved:** `Diagnose Failure`, `Format Diagnosis`, `Post Diagnosis to Discord`, `Record Diagnosis`.
- **Node Details:**
  - **Diagnose Failure**
    - Type and technical role: `n8n-nodes-base.httpRequest` (API Request). Calls the local Ollama chat completion endpoint.
    - Configuration choices: URL set to `={{ $json.ollama_url }}/api/chat`, method `POST`, JSON body transmission, request timeout set to 240,000ms (4 minutes), retry on fail enabled (max 2 tries, 3000ms wait), error continuation enabled.
    - Input and output connections: Input from `Skip Repeat Failures`; output to `Format Diagnosis`.
    - Edge cases or potential failure types: Local Ollama service down, container network routing blocks, or model timeouts on heavy payloads.
  - **Format Diagnosis**
    - Type and technical role: `n8n-nodes-base.code` (Data Transformation). Parses JSON output from the model, validates control flags, downgrades unverified code rewrites to human review requirements, handles missing execution contexts, and formats Markdown strings for Discord.
    - Key expressions or variables: References `Skip Repeat Failures` via `$()`.
    - Input and output connections: Input from `Diagnose Failure`; output to `Post Diagnosis to Discord`.
  - **Post Diagnosis to Discord**
    - Type and technical role: `n8n-nodes-base.discord` (Messaging). Posts the formatted triage markdown message to a Discord channel.
    - Configuration choices: Resource `webhook`, operation `sendLegacy`, authentication via webhook. Configured with retry on fail (max 3 tries, 3000ms wait).
    - Credentials: `Discord webhook` credential.
    - Input and output connections: Input from `Format Diagnosis`; output to `Record Diagnosis`.
    - Edge cases or potential failure types: Discord rate limiting (HTTP 429) or invalid webhook URL/credentials.
  - **Record Diagnosis**
    - Type and technical role: `n8n-nodes-base.dataTable` (Database Operation). Inserts the triage execution metadata and fingerprint into the data table for future deduplication cycles.
    - Configuration choices: Resource `row`, operation `insert`, mapping fields (`fingerprint`, `workflow_id`, `workflow_name`, `failed_node`, `error_message`, `execution_id`, `fix_type`, `diagnosed_at`). Error handling set to continue regular output.
    - Input and output connections: Input from `Post Diagnosis to Discord`; output none (terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Overview` | n8n-nodes-base.stickyNote | Documentation container outlining target audience, mechanism, setup instructions, and limits. | None | None | Diagnose failed n8n workflows with a local Ollama model and post to Discord... |
| `Group 1 Notes` | n8n-nodes-base.stickyNote | Group comment describing the trigger phase. | None | None | ## 1. Trigger A real failure (Error Trigger), a hand-off from a shared error-handler workflow, or a test run over known past failures. |
| `Group 2 Notes` | n8n-nodes-base.stickyNote | Group comment describing the normalization and configuration phase. | None | None | ## 2. Prepare Normalize both input shapes, add settings, and skip this workflow and any excluded IDs so it can never loop. |
| `Group 3 Notes` | n8n-nodes-base.stickyNote | Group comment describing context gathering, redaction, and deduplication. | None | None | ## 3. Gather, redact, dedup Read the failed execution and workflow through the n8n API (trigger failures fall back to the error message alone), strip secrets and personal data, then skip any failure already diagnosed within `dedup_hours`. |
| `Group 4 Notes` | n8n-nodes-base.stickyNote | Group comment describing the diagnosis and reporting phase. | None | None | ## 4. Diagnose and report The model returns schema-constrained JSON. A Code node formats it and applies the fix-type guard, so the model never writes the delivered message. Each diagnosis is recorded for dedup. |
| `On Workflow Error` | n8n-nodes-base.errorTrigger | Triggers when any workflow encounters an unhandled runtime error. | None | `Normalize Failure` | |
| `Called by Error Handler` | n8n-nodes-base.executeWorkflowTrigger | Receives error payloads passed from a parent error handling workflow. | None | `Normalize Failure` | |
| `Test with Sample Failures` | n8n-nodes-base.manualTrigger | Manual entry point to trigger manual tests. | None | `Sample Failures` | |
| `Sample Failures` | n8n-nodes-base.code | Generates mock failure payloads for local testing. | `Test with Sample Failures` | `Normalize Failure` | |
| `Normalize Failure` | n8n-nodes-base.code | Unifies error trigger, sub-workflow, and test payloads into a standard schema. | `On Workflow Error`, `Called by Error Handler`, `Sample Failures` | `Build Config` | |
| `Build Config` | n8n-nodes-base.set | Appends operational environment variables and tuning parameters. | `Normalize Failure` | `Skip Excluded Workflows` | |
| `Skip Excluded Workflows` | n8n-nodes-base.filter | Excludes self-referencing runs or blacklisted workflow IDs to prevent loops. | `Build Config` | `Get Failed Execution` | |
| `Get Failed Execution` | n8n-nodes-base.n8n | Fetches raw execution logs from the n8n REST API. | `Skip Excluded Workflows` | `Get Workflow Definition` | |
| `Get Workflow Definition` | n8n-nodes-base.n8n | Retrieves the structural node configuration of the failed workflow via the n8n API. | `Get Failed Execution` | `Redact and Build Context` | |
| `Redact and Build Context` | n8n-nodes-base.code | Sanitizes sensitive secrets/PII, builds Ollama system prompt, and generates deduplication fingerprints. | `Get Workflow Definition` | `Get Recent Diagnoses` | |
| `Get Recent Diagnoses` | n8n-nodes-base.dataTable | Queries recent history rows from the triage data table based on the configured time window. | `Redact and Build Context` | `Skip Repeat Failures` | |
| `Skip Repeat Failures` | n8n-nodes-base.code | Filters out failures already processed within the deduplication window. | `Get Recent Diagnoses` | `Diagnose Failure` | |
| `Diagnose Failure` | n8n-nodes-base.httpRequest | Sends the redacted failure context to the local Ollama instance for structured JSON diagnosis. | `Skip Repeat Failures` | `Format Diagnosis` | |
| `Format Diagnosis` | n8n-nodes-base.code | Applies deterministic safety validation guards, formats Markdown templates, and prepares audit metadata. | `Diagnose Failure` | `Post Diagnosis to Discord` | |
| `Post Diagnosis to Discord` | n8n-nodes-base.discord | Sends the formatted markdown report to a Discord webhook. | `Format Diagnosis` | `Record Diagnosis` | |
| `Record Diagnosis` | n8n-nodes-base.dataTable | Inserts diagnosis metadata and audit fingerprints into the triage history data table. | `Post Diagnosis to Discord` | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these numbered steps to recreate the workflow manually in n8n:

1. **Create the Entry Triggers:**
   - Add an **Error Trigger** node named `On Workflow Error`.
   - Add an **Execute Workflow Trigger** node named `Called by Error Handler` with `inputSource` set to `passthrough`.
   - Add a **Manual Trigger** node named `Test with Sample Failures`.

2. **Add Sample Data Generation:**
   - Create a **Code** node named `Sample Failures`. Connect `Test with Sample Failures` to it. Set mode to *Run Once for All Items* and provide JavaScript code returning mock execution and workflow ID test cases with `bypass_dedup: true`.

3. **Normalize Incoming Payloads:**
   - Create a **Code** node named `Normalize Failure`. Connect `On Workflow Error`, `Called by Error Handler`, and `Sample Failures` to it. Set mode to *Run Once for Each Item* and implement normalization logic for execution, workflow, and trigger failure objects.

4. **Configure Operational Parameters:**
   - Create a **Set (Edit Fields)** node named `Build Config`. Connect `Normalize Failure` to it.
   - Add string assignments: `ollama_url` (`http://localhost:11434`), `model` (`qwen3:8b`), `n8n_editor_url` (your instance base URL), and `excluded_workflow_ids` (blank or comma-separated list).
   - Add numeric assignments: `max_context_chars` (`6000`) and `dedup_hours` (`6`). Enable *Include Other Fields*.

5. **Implement Alert Loop Protection:**
   - Create a **Filter** node named `Skip Excluded Workflows`. Connect `Build Config` to it.
   - Configure a strict boolean condition checking that `workflowId` is non-empty, does not match current workflow ID (`$workflow.id`), and is absent from the excluded workflows list.

6. **Fetch n8n API Data:**
   - Create an **n8n** node named `Get Failed Execution`. Connect `Skip Excluded Workflows` to it.
   - Set resource to `execution`, operation to `get`, and expression `={{ $json.executionId }}`. Enable `retryOnFail`, max tries `2`, wait time `2000`ms, and set error behavior to *Continue Regular Output*. Configure or select an **n8n API** credential.
   - Create a second **n8n** node named `Get Workflow Definition`. Connect `Get Failed Execution` to it.
   - Set resource to `workflow`, operation to `get`, and workflow ID expression referencing `$('Skip Excluded Workflows').item.json.workflowId`. Enable identical retry and error-continuation settings.

7. **Build Redacted Context and Fingerprints:**
   - Create a **Code** node named `Redact and Build Context`. Connect `Get Workflow Definition` to it.
   - Set mode to *Run Once for Each Item*. Implement context extraction, regex sanitization for secrets/tokens, prompt engineering rules, and fingerprint generation.

8. **Perform Deduplication Checks:**
   - Create a **Data Table** node named `Get Recent Diagnoses`. Connect `Redact and Build Context` to it.
   - Set resource to `row`, operation to `get`, return all rows, execute once, with error handling set to continue regular output. Filter rows where `diagnosed_at` is `>=` `={{ $now.minus({ hours: $json.dedup_hours }).toISO() }}` against your created data table (`triage_history`).
   - Create a **Code** node named `Skip Repeat Failures`. Connect `Get Recent Diagnoses` to it.
   - Set mode to *Run Once for All Items* to filter out items matching existing database fingerprints within the dedup window.

9. **Execute AI Diagnosis:**
   - Create an **HTTP Request** node named `Diagnose Failure`. Connect `Skip Repeat Failures` to it.
   - Set method to `POST`, URL to `={{ $json.ollama_url }}/api/chat`, body content type to `JSON`, body specify type to `JSON`, with JSON body expression passing payload body. Set timeout to `240000`ms, retry on fail (`2` tries, `3000`ms wait), and continue on error.

10. **Format Report and Apply Safety Guards:**
    - Create a **Code** node named `Format Diagnosis`. Connect `Diagnose Failure` to it.
    - Set mode to *Run Once for Each Item*. Parse JSON content, enforce parameter change rules, downgrade unsafe code rewrites, and build the Discord Markdown message array.

11. **Post to Discord:**
    - Create a **Discord** node named `Post Diagnosis to Discord`. Connect `Format Diagnosis` to it.
    - Set resource to `webhook`, operation to `Send Legacy`, authentication to `Webhook`. Map content to `={{ $json.content }}`. Configure retry on fail (`3` tries, `3000`ms wait). Configure a **Discord webhook** credential.

12. **Log Audit Trail:**
    - Create a **Data Table** node named `Record Diagnosis`. Connect `Post Diagnosis to Discord` to it.
    - Set resource to `row`, operation to `insert`, selecting the `triage_history` data table. Map columns (`fingerprint`, `workflow_id`, `workflow_name`, `failed_node`, `error_message`, `execution_id`, `fix_type`, `diagnosed_at`) to corresponding properties from the `Format Diagnosis` output.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Only sees runs that fail: missing fields evaluated in expressions return empty values and succeed silently; explicit data checks should be added where required. | Workflow operational limit note |
| Nodes referenced via `$()` expressions: `Skip Excluded Workflows`, `Get Failed Execution`, `Redact and Build Context`, `Skip Repeat Failures`, `Format Diagnosis`. Do not rename these nodes without updating references. | Dependency integrity note |
| Required n8n Data Table schema: `triage_history` with text columns (`fingerprint`, `workflow_id`, `workflow_name`, `failed_node`, `error_message`, `execution_id`, `fix_type`) and a date-time column (`diagnosed_at`). | Data Table requirement |
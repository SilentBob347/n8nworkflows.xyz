Orchestrate procurement risk workflows and approvals with OpenAI, Slack and ERP data

https://n8nworkflows.xyz/workflows/orchestrate-procurement-risk-workflows-and-approvals-with-openai--slack-and-erp-data-20852


# Orchestrate procurement risk workflows and approvals with OpenAI, Slack and ERP data

### 1. Workflow Overview

This workflow automates the end-to-end management of corporate procurement operations. It ingests procurement goals via a secure webhook, parses and validates requests, leverages OpenAI models to plan execution and assess supplier risk, normalizes enterprise data (spend, contracts, risk metrics), routes processes to specialized sub-workflows, manages human-in-the-loop approvals via Slack, executes verified actions, generates a comprehensive audit trail, and responds to the initiating system.

The application logic is partitioned into the following functional blocks:
- **1.1 Input Reception & Request Initialization:** Captures incoming POST requests, validates the core payload parameters (`goal`, `priority`, `requestedBy`), and generates a unique execution tracking ID (`requestId`).
- **1.2 AI Goal Interpretation & Planning:** Translates the raw text business goal into a structured operational plan via OpenAI, validating the designated procurement domain and required data sources.
- **1.3 Enterprise Procurement Data Collection & Normalization:** Gathers supplier profiles, ERP spend amounts, contracts, and risk indicators (currently via mock nodes) and consolidates them into a standardized schema per supplier.
- **1.4 AI Supplier Risk Analysis & Deterministic Routing:** Evaluates individual supplier records using OpenAI to compute exposure scores, combines the assessment with normalized metrics, applies deterministic business rules, and routes execution paths using a switch node.
- **1.5 Sub-Workflow Execution & Aggregation:** Invokes specialized sub-workflows (`Supplier Risk Assessment`, `Contract Review`, `Supplier Monitoring`), merges their outputs, and runs a secondary OpenAI evaluation to generate an overarching recommendation and summary.
- **1.6 Human Approval Gate:** Prepares an approval request payload, posts an interactive message to Slack, halts execution via a wait node, and parses incoming responses into an explicit approval decision (`APPROVED` or `REJECTED`).
- **1.7 Execution Routing & Verification:** Splits composite approved decisions, routes tasks to corresponding operational sub-workflows, checks execution outcomes, and captures rejections when approvals are denied.
- **1.8 Audit, Response & Sub-Workflow Modules:** Consolidates successful or rejected states, registers an audit log entry, returns a JSON response to the webhook caller, and contains independent sub-workflows for dedicated risk, contract, and monitoring duties.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception & Request Initialization
- **Overview:** Receives external procurement requests, authenticates the connection via header verification, validates mandatory structural parameters, and assigns a unique traceability identifier.
- **Nodes Involved:** `Webhook`, `Validate Procurement Request`, `Generate Request ID`
- **Node Details:**
  - **Webhook**
    - *Type & Role:* `n8n-nodes-base.webhook` (Trigger). Exposes an HTTP POST endpoint (`/procurement/orchestrate`) secured with header authentication.
    - *Configuration:* HTTP method `POST`, response mode set to use the explicit response node, authentication via `headerAuth`.
    - *Credentials:* `ODT Site WordPress Auth` (`httpHeaderAuth`).
    - *Connections:* Input: None (Trigger); Output: `Validate Procurement Request`.
    - *Potential Failures:* Invalid authentication headers or missing content types from callers.
  - **Validate Procurement Request**
    - *Type & Role:* `n8n-nodes-base.code` (Data transformation and validation).
    - *Configuration:* JavaScript evaluation verifying the existence of the `goal` property and ensuring a minimum length of 5 characters. Sets a default priority of `medium` if omitted.
    - *Expressions:* Reads `$json.body` properties.
    - *Connections:* Input: `Webhook`; Output: `Generate Request ID`.
    - *Potential Failures:* JavaScript exceptions thrown if the `goal` field is missing or too short.
  - **Generate Request ID**
    - *Type & Role:* `n8n-nodes-base.code` (Enrichment).
    - *Configuration:* Generates a unique tracking string using a timestamp and randomized alphanumeric substring.
    - *Expressions:* `PROC-${Date.now()}-${Math.random().toString(36)...}`.
    - *Connections:* Input: `Validate Procurement Request`; Output: `AI - Goal Interpreter`.
    - *Potential Failures:* None.

---

#### 1.2 AI Goal Interpretation & Planning
- **Overview:** Passes the validated request to an AI model to interpret the underlying procurement intent and output a structured JSON execution plan.
- **Nodes Involved:** `AI - Goal Interpreter`, `Parse Goal`, `Validate AI Plan`
- **Node Details:**
  - **AI - Goal Interpreter**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` (AI processing).
    - *Configuration:* Uses model `gpt-4o-mini` with system instructions restricting output strictly to valid JSON matching an enterprise procurement orchestration schema.
    - *Expressions:* References `{{ $json.goal }}`, `{{ $json.priority }}`, `{{ JSON.stringify($json.scope) }}`, and `{{ $json.requestId }}`.
    - *Credentials:* `Ollama` (`openAiApi`).
    - *Connections:* Input: `Generate Request ID`; Output: `Parse Goal`.
    - *Potential Failures:* OpenAI rate limits, token exhaustion, or connection timeouts.
  - **Parse Goal**
    - *Type & Role:* `n8n-nodes-base.code` (Data parsing).
    - *Configuration:* Strips markdown code blocks (e.g., ```json ... ```) from the AI response and parses the text into a structured JSON object.
    - *Connections:* Input: `AI - Goal Interpreter`; Output: `Validate AI Plan`.
    - *Potential Failures:* Malformed JSON outputs from the LLM triggering syntax parsing errors.
  - **Validate AI Plan**
    - *Type & Role:* `n8n-nodes-base.code` (Validation).
    - *Configuration:* Verifies that the returned domain is within an allowed list (`supplier_risk`, `spend_management`, `contract_management`, `sourcing`, `compliance`, `procurement_performance`) and that required data sources are listed.
    - *Connections:* Input: `Parse Goal`; Output: `Mock Supplier Data`.
    - *Potential Failures:* Domain validation failures throwing explicit Error exceptions.

---

#### 1.3 Enterprise Procurement Data Collection & Normalization
- **Overview:** Gathers foundational procurement records from mock data systems and merges them into uniform, supplier-centric data structures.
- **Nodes Involved:** `Mock Supplier Data`, `Mock ERP Spend Data`, `Mock Contract Data`, `Mock Supplier Risk Data`, `Normalize Procurement Data`
- **Node Details:**
  - **Mock Supplier Data**
    - *Type & Role:* `n8n-nodes-base.code` (Data provider).
    - *Configuration:* Returns a hardcoded list of five supplier entity records containing IDs, names, categories, criticality ratings, countries, and statuses.
    - *Connections:* Input: `Validate AI Plan`; Output: `Mock ERP Spend Data`.
  - **Mock ERP Spend Data**
    - *Type & Role:* `n8n-nodes-base.code` (Data provider).
    - *Configuration:* Returns a hardcoded list of annual spend totals, currencies, and purchase order counts indexed by supplier ID.
    - *Connections:* Input: `Mock Supplier Data`; Output: `Mock Contract Data`.
  - **Mock Contract Data**
    - *Type & Role:* `n8n-nodes-base.code` (Data provider).
    - *Configuration:* Returns active contract metadata, expiration dates, contract values, and auto-renewal flags per supplier ID.
    - *Connections:* Input: `Mock ERP Spend Data`; Output: `Mock Supplier Risk Data`.
  - **Mock Supplier Risk Data**
    - *Type & Role:* `n8n-nodes-base.code` (Data provider).
    - *Configuration:* Returns financial, operational, compliance, geopolitical, and overall risk scores per supplier ID.
    - *Connections:* Input: `Mock Contract Data`; Output: `Normalize Procurement Data`.
  - **Normalize Procurement Data**
    - *Type & Role:* `n8n-nodes-base.code` (Data merging).
    - *Configuration:* Combines datasets from all four mock source nodes using `Map` lookups based on `supplierId`, establishing a unified nested schema (`contract`, `risk`).
    - *Connections:* Input: `Mock Supplier Risk Data`; Output: `Supplier Risk Analyst`.

---

#### 1.4 AI Supplier Risk Analysis & Deterministic Routing
- **Overview:** Evaluates supplier records using AI to determine risk factors and score exposures, then passes the enriched data through a deterministic rule engine and switch router.
- **Nodes Involved:** `Supplier Risk Analyst`, `Parse AI Risk Analysis`, `Combine Supplier + AI Risk`, `Deterministic Decision Engine`, `Route based on risk`
- **Node Details:**
  - **Supplier Risk Analyst**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` (AI analysis).
    - *Configuration:* Evaluates individual supplier records against risk metrics using `gpt-4o-mini`, requesting strict JSON outputs for financial exposure, operational exposure, risk drivers, and recommendations.
    - *Expressions:* `{{ JSON.stringify($json) }}`.
    - *Credentials:* `Ollama` (`openAiApi`).
    - *Connections:* Input: `Normalize Procurement Data`; Output: `Parse AI Risk Analysis`.
  - **Parse AI Risk Analysis**
    - *Type & Role:* `n8n-nodes-base.code` (Parsing).
    - *Configuration:* Extracts text payloads from the OpenAI response structure, cleans markdown fences, and parses results into JSON objects.
    - *Connections:* Input: `Supplier Risk Analyst`; Output: `Combine Supplier + AI Risk`.
  - **Combine Supplier + AI Risk**
    - *Type & Role:* `n8n-nodes-base.code` (Data consolidation).
    - *Configuration:* Merges normalized supplier records with AI analysis outputs, falling back to existing data if specific fields are missing.
    - *Connections:* Input: `Parse AI Risk Analysis`; Output: `Deterministic Decision Engine`.
  - **Deterministic Decision Engine**
    - *Type & Role:* `n8n-nodes-base.code` (Business rules).
    - *Configuration:* Evaluates numerical risk scores, annual spend thresholds, and contract expiration dates to set deterministic `decision` flags (`MONITOR`, `RISK_REVIEW`, `RISK_REVIEW_AND_CONTRACT_REVIEW`) and requirement flags.
    - *Connections:* Input: `Combine Supplier + AI Risk`; Output: `Route based on risk`.
  - **Route based on risk**
    - *Type & Role:* `n8n-nodes-base.switch` (Routing).
    - *Configuration:* Evaluates `={{ $json.decision }}` to route items to `Supplier Risk Assessment`, `Contract Review`, or `Call 'Supplier Monitoring'`.
    - *Connections:* Input: `Deterministic Decision Engine`; Outputs: `Supplier Risk Assessment`, `Contract Review`, `Call 'Supplier Monitoring'`.

---

#### 1.5 Sub-Workflow Execution & Aggregation
- **Overview:** Executes specialized downstream workflows for selected risk or contract tracks, aggregates all results, and requests a final synthesis from OpenAI.
- **Nodes Involved:** `Supplier Risk Assessment`, `Contract Review` (Main route), `Call 'Supplier Monitoring'`, `Merge`, `Aggregate Results`, `AI Recommendation`
- **Node Details:**
  - **Supplier Risk Assessment**
    - *Type & Role:* `n8n-nodes-base.executeWorkflow` (Sub-workflow execution).
    - *Configuration:* Executes sub-workflow ID `psbzlwJEUqkvih5v`.
    - *Connections:* Input: `Route based on risk`; Output: `Merge`.
  - **Contract Review**
    - *Type & Role:* `n8n-nodes-base.executeWorkflow` (Sub-workflow execution).
    - *Configuration:* Executes sub-workflow ID `yQoFszgvahoj9AYC`, passing mapped contract and spend attributes.
    - *Connections:* Input: `Route based on risk`; Output: `Merge`.
  - **Call 'Supplier Monitoring'**
    - *Type & Role:* `n8n-nodes-base.executeWorkflow` (Sub-workflow execution).
    - *Configuration:* Executes sub-workflow ID `eY21FxuyeNglO47w`, passing risk metrics and contract details.
    - *Connections:* Input: `Route based on risk`; Output: `Merge`.
  - **Merge**
    - *Type & Role:* `n8n-nodes-base.merge` (Data combination).
    - *Configuration:* 3-input merge node consolidating branches back into a single stream.
    - *Connections:* Inputs: All three sub-workflow execution nodes; Output: `Aggregate Results`.
  - **Aggregate Results**
    - *Type & Role:* `n8n-nodes-base.code` (Aggregation).
    - *Configuration:* Wraps processed supplier workflow outputs into an overarching array with status metadata and a completion timestamp.
    - *Connections:* Input: `Merge`; Output: `AI Recommendation`.
  - **AI Recommendation**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` (AI synthesis).
    - *Configuration:* Analyzes aggregated sub-workflow results using `gpt-4o-mini` to summarize findings and determine whether human approval is required.
    - *Expressions:* `{{ JSON.stringify($json) }}`.
    - *Credentials:* `Ollama` (`openAiApi`).
    - *Connections:* Input: `Aggregate Results`; Output: `Prepare Approval Request`.

---

#### 1.6 Human Approval Gate
- **Overview:** Formats the approval payload, publishes a notification message to Slack, pauses execution until a response is received, and parses the decision.
- **Nodes Involved:** `Prepare Approval Request`, `Send Approval Request`, `Wait`, `Process Approval Response`, `Approval Decision`
- **Node Details:**
  - **Prepare Approval Request**
    - *Type & Role:* `n8n-nodes-base.code` (Payload formatting).
    - *Configuration:* Generates a unique `approvalRequestId` (e.g., `APR-1758...`), establishes initial status as `PENDING`, and constructs a multi-line human-readable `approvalMessage`.
    - *Connections:* Input: `AI Recommendation`; Output: `Send Approval Request`.
  - **Send Approval Request**
    - *Type & Role:* `n8n-nodes-base.slack` (Messaging integration).
    - *Configuration:* Sends a message to the specified Slack channel containing summary risk data and approval identifiers.
    - *Credentials:* Slack API account connection.
    - *Connections:* Input: `Prepare Approval Request`; Output: `Wait`.
    - *Potential Failures:* Slack authentication failures or missing channel permissions.
  - **Wait**
    - *Type & Role:* `n8n-nodes-base.wait` (Execution pause).
    - *Configuration:* Pauses execution indefinitely until an incoming webhook or callback associated with the sent message is received.
    - *Connections:* Input: `Send Approval Request`; Output: `Process Approval Response`.
  - **Process Approval Response**
    - *Type & Role:* `n8n-nodes-base.code` (Response parsing).
    - *Configuration:* Extracts text strings from Slack replies or interactive buttons, evaluates regular expressions to locate fields, and standardizes decision values into `APPROVED` or `REJECTED`.
    - *Connections:* Input: `Wait`; Output: `Approval Decision`.
  - **Approval Decision**
    - *Type & Role:* `n8n-nodes-base.if` (Conditional branching).
    - *Configuration:* Evaluates `={{ $json.approvalDecision }}` matching `APPROVED`.
    - *Connections:* Input: `Process Approval Response`; Outputs: `Prepare Approved Action` (true path), `Prepare Rejection` (false path).

---

#### 1.7 Execution Routing & Verification
- **Overview:** Splits composite actions into individual task items, routes them to designated sub-workflows, and verifies execution success or logs rejections.
- **Nodes Involved:** `Prepare Approved Action`, `Split Actions`, `Execute Approved Action`, `Call 'Supplier Risk Assessment'` (Execution route), `Call 'Contract Review'` (Execution route), `Call 'Supplier Monitoring'1`, `Verify Execution`, `Prepare Approved Result`, `Prepare Rejection`, `Prepare Rejected Result`, `Merge1`
- **Node Details:**
  - **Prepare Approved Action**
    - *Type & Role:* `n8n-nodes-base.code` (Validation).
    - *Configuration:* Confirms approval status equals `APPROVED` and checks for the presence of recommended actions.
    - *Connections:* Input: `Approval Decision`; Output: `Split Actions`.
  - **Split Actions**
    - *Type & Role:* `n8n-nodes-base.code` (Array splitting).
    - *Configuration:* Splits combined actions separated by commas or `AND` connectors into discrete items assigned to `singleAction`.
    - *Connections:* Input: `Prepare Approved Action`; Output: `Execute Approved Action`.
  - **Execute Approved Action**
    - *Type & Role:* `n8n-nodes-base.switch` (Routing).
    - *Configuration:* Evaluates `={{ $json.singleAction }}` against substrings (`RISK_REVIEW`, `CONTRACT_REVIEW`, `MONITORING`). Configured to allow all matching outputs.
    - *Connections:* Input: `Split Actions`; Outputs: `Call 'Supplier Risk Assessment'`, `Call 'Contract Review'`, `Call 'Supplier Monitoring'1`.
  - **Call 'Supplier Risk Assessment' (Execution route)**
    - *Type & Role:* `n8n-nodes-base.executeWorkflow` (Sub-workflow execution).
    - *Configuration:* Executes sub-workflow ID `psbzlwJEUqkvih5v` with mapped action parameters.
    - *Connections:* Input: `Execute Approved Action`; Output: `Verify Execution`.
  - **Call 'Contract Review' (Execution route)**
    - *Type & Role:* `n8n-nodes-base.executeWorkflow` (Sub-workflow execution).
    - *Configuration:* Executes sub-workflow ID `yQoFszgvahoj9AYC`.
    - *Connections:* Input: `Execute Approved Action`; Output: `Verify Execution`.
  - **Call 'Supplier Monitoring'1**
    - *Type & Role:* `n8n-nodes-base.executeWorkflow` (Sub-workflow execution).
    - *Configuration:* Executes sub-workflow ID `eY21FxuyeNglO47w`.
    - *Connections:* Input: `Execute Approved Action`; Output: `Verify Execution`.
  - **Verify Execution**
    - *Type & Role:* `n8n-nodes-base.code` (Verification).
    - *Configuration:* Confirms whether sub-workflow execution statuses indicate success and sets final statuses (`COMPLETED` or `EXECUTION_FAILED`).
    - *Connections:* Input: Sub-workflow execution nodes; Output: `Prepare Approved Result`.
  - **Prepare Approved Result**
    - *Type & Role:* `n8n-nodes-base.code` (Formatting).
    - *Configuration:* Constructs standardized JSON payloads representing successfully executed approved requests.
    - *Connections:* Input: `Verify Execution`; Output: `Merge1`.
  - **Prepare Rejection**
    - *Type & Role:* `n8n-nodes-base.code` (Formatting).
    - *Configuration:* Packages rejection reasons and timestamps when approval is denied.
    - *Connections:* Input: `Approval Decision`; Output: `Prepare Rejected Result`.
  - **Prepare Rejected Result**
    - *Type & Role:* `n8n-nodes-base.code` (Formatting).
    - *Configuration:* Structures payload parameters for rejected requests.
    - *Connections:* Input: `Prepare Rejection`; Output: `Merge1`.
  - **Merge1**
    - *Type & Role:* `n8n-nodes-base.merge` (Combination).
    - *Configuration:* Combines approved and rejected branches into a single stream.
    - *Connections:* Inputs: `Prepare Approved Result`, `Prepare Rejected Result`; Output: `Audit Log`.

---

#### 1.8 Audit, Response & Sub-Workflow Modules
- **Overview:** Writes a unified audit record to downstream systems, returns an HTTP JSON response to the webhook client, and encapsulates isolated helper sub-workflows.
- **Nodes Involved:** `Audit Log`, `Prepare Webhook Response`, `Respond to Webhook`, plus all independent sub-workflow trigger nodes and logic branches.
- **Node Details:**
  - **Audit Log**
    - *Type & Role:* `n8n-nodes-base.code` (Audit generation).
    - *Configuration:* Generates a unique audit event ID (`AUD-${Date.now()}`), compiles metadata across IDs, risk scores, execution states, and approver comments, and categorizes audit events.
    - *Connections:* Input: `Merge1`; Output: `Prepare Webhook Response`.
  - **Prepare Webhook Response**
    - *Type & Role:* `n8n-nodes-base.code` (Formatting).
    - *Configuration:* Prepares human-readable success or failure messages and nests attributes into standard blocks (`supplier`, `risk`, `action`, `approval`, `execution`).
    - *Connections:* Input: `Audit Log`; Output: `Respond to Webhook`.
  - **Respond to Webhook**
    - *Type & Role:* `n8n-nodes-base.respondToWebhook` (HTTP response).
    - *Configuration:* Responds with structured JSON payload confirming completion status back to the original caller.
    - *Connections:* Input: `Prepare Webhook Response`; Output: None (Terminal).
  - **Supplier Risk Assessment Sub-Workflow Nodes** (`When Executed by Another Workflow`, `Validate Supplier Data`, `Calculate Risk Indicators`, `Risk Classification`, `Prepare Risk Assessment Result`, `Respond to Webhook1`)
    - *Type & Role:* Sub-workflow trigger and data processing nodes.
    - *Configuration:* Validates input supplier metrics, classifies risk levels based on thresholds, and responds with structured risk assessment data.
  - **Contract Review Sub-Workflow Nodes** (`Validate Contract Data`, `Check Contract Expiration`, `Calculate Contract Exposure`, `AI Contract Analysis`, `Parse Contract Analysis`, `Prepare Contract Result`)
    - *Type & Role:* Sub-workflow processing nodes.
    - *Configuration:* Validates contract data, computes expiration windows and exposure levels, runs AI analysis using `gpt-4o-mini`, and prepares structured contract review findings.
  - **Supplier Monitoring Sub-Workflow Nodes** (`Validate Supplier Data1`, `Check Monitoring Conditions`, `Prepare Monitoring Result`, `HTTP Request`)
    - *Type & Role:* Sub-workflow processing nodes.
    - *Configuration:* Validates incoming supplier data, evaluates monitoring tiers (`NORMAL`, `ENHANCED_MONITORING`, `REVIEW_REQUIRED`), and issues outbound HTTP requests to external supplier APIs (`http://localhost:5678/suppliers/SUP-3001`).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook | `n8n-nodes-base.webhook` | Receives incoming POST HTTP requests | None | Validate Procurement Request | Request Intake & Validation |
| Validate Procurement Request | `n8n-nodes-base.code` | Validates mandatory request parameters | Webhook | Generate Request ID | Request Intake & Validation |
| Generate Request ID | `n8n-nodes-base.code` | Generates unique tracking identifier | Validate Procurement Request | AI - Goal Interpreter | Request Intake & Validation |
| AI - Goal Interpreter | `@n8n/n8n-nodes-langchain.openAi` | Interprets procurement goals via LLM | Generate Request ID | Parse Goal | AI Goal Interpretation & Planning |
| Parse Goal | `n8n-nodes-base.code` | Parses AI output into JSON plan | AI - Goal Interpreter | Validate AI Plan | AI Goal Interpretation & Planning |
| Validate AI Plan | `n8n-nodes-base.code` | Validates domains and required data | Parse Goal | Mock Supplier Data | AI Goal Interpretation & Planning |
| Mock Supplier Data | `n8n-nodes-base.code` | Provides mock supplier entity records | Validate AI Plan | Mock ERP Spend Data | Procurement Data Collection |
| Mock ERP Spend Data | `n8n-nodes-base.code` | Provides mock ERP spend records | Mock Supplier Data | Mock Contract Data | Procurement Data Collection |
| Mock Contract Data | `n8n-nodes-base.code` | Provides mock contract data | Mock ERP Spend Data | Mock Supplier Risk Data | Procurement Data Collection |
| Mock Supplier Risk Data | `n8n-nodes-base.code` | Provides mock risk score records | Mock Contract Data | Normalize Procurement Data | Procurement Data Collection |
| Normalize Procurement Data | `n8n-nodes-base.code` | Combines mock datasets into unified schema | Mock Supplier Risk Data | Supplier Risk Analyst | Data Normalization |
| Supplier Risk Analyst | `@n8n/n8n-nodes-langchain.openAi` | AI supplier risk evaluation | Normalize Procurement Data | Parse AI Risk Analysis | AI Supplier Risk Analysis |
| Parse AI Risk Analysis | `n8n-nodes-base.code` | Parses AI risk evaluation text | Supplier Risk Analyst | Combine Supplier + AI Risk | AI Supplier Risk Analysis |
| Combine Supplier + AI Risk | `n8n-nodes-base.code` | Merges supplier data with AI risk metrics | Parse AI Risk Analysis | Deterministic Decision Engine | AI Supplier Risk Analysis |
| Deterministic Decision Engine | `n8n-nodes-base.code` | Applies business rules for routing | Combine Supplier + AI Risk | Route based on risk | Deterministic Risk Decision & Routing |
| Route based on risk | `n8n-nodes-base.switch` | Switches path based on risk decision | Deterministic Decision Engine | Supplier Risk Assessment, Contract Review, Call 'Supplier Monitoring' | Deterministic Risk Decision & Routing |
| Supplier Risk Assessment | `n8n-nodes-base.executeWorkflow` | Executes risk assessment sub-workflow | Route based on risk | Merge | Procurement Analysis & Monitoring |
| Contract Review | `n8n-nodes-base.executeWorkflow` | Executes contract review sub-workflow | Route based on risk | Merge | Procurement Analysis & Monitoring |
| Call 'Supplier Monitoring' | `n8n-nodes-base.executeWorkflow` | Executes supplier monitoring sub-workflow | Route based on risk | Merge | Procurement Analysis & Monitoring |
| Merge | `n8n-nodes-base.merge` | Merges sub-workflow results | Supplier Risk Assessment, Contract Review, Call 'Supplier Monitoring' | Aggregate Results | None |
| Aggregate Results | `n8n-nodes-base.code` | Aggregates sub-workflow outputs | Merge | AI Recommendation | Procurement Recommendation |
| AI Recommendation | `@n8n/n8n-nodes-langchain.openAi` | AI recommendation and summary generation | Aggregate Results | Prepare Approval Request | Procurement Recommendation |
| Prepare Approval Request | `n8n-nodes-base.code` | Formats approval message payload | AI Recommendation | Send Approval Request | Human Approval Gate |
| Send Approval Request | `n8n-nodes-base.slack` | Posts approval request to Slack | Prepare Approval Request | Wait | Human Approval Gate |
| Wait | `n8n-nodes-base.wait` | Pauses for human approval response | Send Approval Request | Process Approval Response | Human Approval Gate |
| Process Approval Response | `n8n-nodes-base.code` | Parses approval response from Slack | Wait | Approval Decision | Human Approval Gate |
| Approval Decision | `n8n-nodes-base.if` | Evaluates approval status | Process Approval Response | Prepare Approved Action, Prepare Rejection | Human Approval Gate |
| Prepare Approved Action | `n8n-nodes-base.code` | Validates approved actions | Approval Decision | Split Actions | Approved Action Routing |
| Split Actions | `n8n-nodes-base.code` | Splits composite actions into items | Prepare Approved Action | Execute Approved Action | Approved Action Routing |
| Execute Approved Action | `n8n-nodes-base.switch` | Routes individual approved actions | Split Actions | Call 'Supplier Risk Assessment', Call 'Contract Review', Call 'Supplier Monitoring'1 | Approved Action Routing |
| Call 'Supplier Risk Assessment' | `n8n-nodes-base.executeWorkflow` | Executes risk sub-workflow for approved action | Execute Approved Action | Verify Execution | Approved Action Routing |
| Call 'Contract Review' | `n8n-nodes-base.executeWorkflow` | Executes contract sub-workflow for approved action | Execute Approved Action | Verify Execution | Approved Action Routing |
| Call 'Supplier Monitoring'1 | `n8n-nodes-base.executeWorkflow` | Executes monitoring sub-workflow for approved action | Execute Approved Action | Verify Execution | Approved Action Routing |
| Verify Execution | `n8n-nodes-base.code` | Verifies execution success status | Call 'Supplier Risk Assessment', Call 'Contract Review', Call 'Supplier Monitoring'1 | Prepare Approved Result | Execution Verification |
| Prepare Approved Result | `n8n-nodes-base.code` | Prepares successful execution result | Verify Execution | Merge1 | Execution Verification |
| Prepare Rejection | `n8n-nodes-base.code` | Formats rejection record | Approval Decision | Prepare Rejected Result | Approval Rejection Handling |
| Prepare Rejected Result | `n8n-nodes-base.code` | Prepares rejected execution result | Prepare Rejection | Merge1 | Approval Rejection Handling |
| Merge1 | `n8n-nodes-base.merge` | Merges approved and rejected outcomes | Prepare Approved Result, Prepare Rejected Result | Audit Log | None |
| Audit Log | `n8n-nodes-base.code` | Generates audit log entry | Merge1 | Prepare Webhook Response | Audit & Final Response |
| Prepare Webhook Response | `n8n-nodes-base.code` | Formats final response payload | Audit Log | Respond to Webhook | Audit & Final Response |
| Respond to Webhook | `n8n-nodes-base.respondToWebhook` | Returns JSON response to webhook caller | Prepare Webhook Response | None | Audit & Final Response |
| When Executed by Another Workflow | `n8n-nodes-base.executeWorkflowTrigger` | Trigger for Supplier Risk Assessment sub-workflow | None | Validate Supplier Data | Recieves Suplier Data and Validate |
| Validate Supplier Data | `n8n-nodes-base.code` | Validates incoming supplier data | When Executed by Another Workflow | Calculate Risk Indicators | Recieves Suplier Data and Validate |
| Calculate Risk Indicators | `n8n-nodes-base.code` | Calculates risk scores and levels | Validate Supplier Data | Risk Classification | Recieves Suplier Data and Validate |
| Risk Classification | `n8n-nodes-base.code` | Classifies recommended actions | Calculate Risk Indicators | Prepare Risk Assessment Result | Risk Classification & Risk Assessment Result |
| Prepare Risk Assessment Result | `n8n-nodes-base.code` | Formats risk assessment sub-workflow output | Risk Classification | Respond to Webhook1 | Risk Classification & Risk Assessment Result |
| Respond to Webhook1 | `n8n-nodes-base.respondToWebhook` | Responds to sub-workflow caller | Prepare Risk Assessment Result | None | Risk Classification & Risk Assessment Result |
| Validate Contract Data | `n8n-nodes-base.code` | Validates contract IDs and suppliers | Sub-workflow Trigger | Check Contract Expiration | Recieves, Validate & Contract Data & Check status |
| Check Contract Expiration | `n8n-nodes-base.code` | Calculates days to contract expiry | Validate Contract Data | Calculate Contract Exposure | Recieves, Validate & Contract Data & Check status |
| Calculate Contract Exposure | `n8n-nodes-base.code` | Calculates contract spend exposure | Check Contract Expiration | AI Contract Analysis | Recieves, Validate & Contract Data & Check status |
| AI Contract Analysis | `@n8n/n8n-nodes-langchain.openAi` | AI contract risk evaluation | Calculate Contract Exposure | Parse Contract Analysis | AI Contract Risk Analysis & Prepare Result |
| Parse Contract Analysis | `n8n-nodes-base.code` | Parses AI contract analysis JSON | AI Contract Analysis | Prepare Contract Result | AI Contract Risk Analysis & Prepare Result |
| Prepare Contract Result | `n8n-nodes-base.code` | Formats contract review output | Parse Contract Analysis | None | AI Contract Risk Analysis & Prepare Result |
| Validate Supplier Data1 | `n8n-nodes-base.code` | Validates monitoring supplier inputs | Sub-workflow Trigger | Check Monitoring Conditions | Recieves & Validate Supplier Data |
| Check Monitoring Conditions | `n8n-nodes-base.code` | Determines monitoring status levels | Validate Supplier Data1 | Prepare Monitoring Result | Recieves & Validate Supplier Data |
| Prepare Monitoring Result | `n8n-nodes-base.code` | Formats monitoring result payload | Check Monitoring Conditions | HTTP Request | Prepare Monitoring Result |
| HTTP Request | `n8n-nodes-base.httpRequest` | Calls external supplier monitoring API | Prepare Monitoring Result | None | Prepare Monitoring Result |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Main Webhook and Intake
1. Create a **Webhook** node (`n8n-nodes-base.webhook`). Set method to `POST`, path to `procurement/orchestrate`, response mode to `responseNode`, and authentication to `headerAuth`. Configure credentials using `ODT Site WordPress Auth`.
2. Create a **Code** node named `Validate Procurement Request` (`n8n-nodes-base.code`). Connect Webhook output to this node. Add logic to check that `body.goal` exists and has a minimum length of 5 characters.
3. Create a **Code** node named `Generate Request ID` (`n8n-nodes-base.code`). Connect `Validate Procurement Request` output here. Add code to generate a unique `requestId` string.

#### Step 2: Configure AI Goal Interpretation
4. Create an **OpenAI** node named `AI - Goal Interpreter` (`@n8n/n8n-nodes-langchain.openAi`). Connect `Generate Request ID` to this node. Set model ID to `gpt-4o-mini`, add system instructions defining procurement orchestration domains, and bind OpenAI credentials.
5. Create a **Code** node named `Parse Goal` (`n8n-nodes-base.code`). Connect `AI - Goal Interpreter` output here. Add code to clean markdown code fences and parse the JSON plan.
6. Create a **Code** node named `Validate AI Plan` (`n8n-nodes-base.code`). Connect `Parse Goal` output here. Validate allowed domains and required data arrays.

#### Step 3: Setup Procurement Data Sources and Normalization
7. Create four consecutive **Code** nodes named `Mock Supplier Data`, `Mock ERP Spend Data`, `Mock Contract Data`, and `Mock Supplier Risk Data`. Chain them sequentially, connecting `Validate AI Plan` into `Mock Supplier Data`.
8. Create a **Code** node named `Normalize Procurement Data` (`n8n-nodes-base.code`). Connect `Mock Supplier Risk Data` output here. Write mapping logic utilizing JavaScript `Map` lookups on `supplierId` to merge spend, contract, and risk datasets into a unified supplier object.

#### Step 4: Configure AI Risk Analysis and Decision Routing
9. Create an **OpenAI** node named `Supplier Risk Analyst` (`@n8n/n8n-nodes-langchain.openAi`). Connect `Normalize Procurement Data` here. Configure model `gpt-4o-mini` with a system prompt instructing single-supplier risk evaluation returning exact JSON structures.
10. Create a **Code** node named `Parse AI Risk Analysis` (`n8n-nodes-base.code`). Connect `Supplier Risk Analyst` output here.
11. Create a **Code** node named `Combine Supplier + AI Risk` (`n8n-nodes-base.code`). Connect `Parse AI Risk Analysis` output here to merge base attributes with AI risk metrics.
12. Create a **Code** node named `Deterministic Decision Engine` (`n8n-nodes-base.code`). Connect `Combine Supplier + AI Risk` here. Set rules for high spend, risk scores, and decisions.
13. Create a **Switch** node named `Route based on risk` (`n8n-nodes-base.switch`). Connect `Deterministic Decision Engine` output here. Add three rules matching `RISK_REVIEW`, `RISK_REVIEW_AND_CONTRACT_REVIEW`, and `MONITOR`.

#### Step 5: Implement Sub-Workflow Execution and Recommendation
14. Create three **Execute Workflow** nodes: `Supplier Risk Assessment`, `Contract Review`, and `Call 'Supplier Monitoring'`. Connect them respectively to the three outputs of `Route based on risk`. Select target sub-workflow IDs (`psbzlwJEUqkvih5v`, `yQoFszgvahoj9AYC`, `eY21FxuyeNglO47w`).
15. Create a **Merge** node (`n8n-nodes-base.merge`) configured with 3 input ports. Connect all three sub-workflow nodes into this merge node.
16. Create a **Code** node named `Aggregate Results` (`n8n-nodes-base.code`). Connect the Merge output here.
17. Create an **OpenAI** node named `AI Recommendation` (`@n8n/n8n-nodes-langchain.openAi`). Connect `Aggregate Results` here using model `gpt-4o-mini` to synthesize a final recommendation JSON.

#### Step 6: Configure Human Approval Gate (Slack & Wait)
18. Create a **Code** node named `Prepare Approval Request` (`n8n-nodes-base.code`). Connect `AI Recommendation` output here. Set `approvalRequestId`, status, and markdown message strings.
19. Create a **Slack** node named `Send Approval Request` (`n8n-nodes-base.slack`). Connect `Prepare Approval Request` here. Configure channel selection and connect Slack credentials.
20. Create a **Wait** node (`n8n-nodes-base.wait`). Connect `Send Approval Request` here.
21. Create a **Code** node named `Process Approval Response` (`n8n-nodes-base.code`). Connect `Wait` output here. Add regex extractors to parse Slack response text and determine decision states (`APPROVED` or `REJECTED`).
22. Create an **If** node named `Approval Decision` (`n8n-nodes-base.if`). Connect `Process Approval Response` here. Add a condition checking `={{ $json.approvalDecision }}` equals `APPROVED`.

#### Step 7: Build Approved Action Execution and Verification Path
23. Create a **Code** node named `Prepare Approved Action` (`n8n-nodes-base.code`). Connect the true output of `Approval Decision` here.
24. Create a **Code** node named `Split Actions` (`n8n-nodes-base.code`). Connect `Prepare Approved Action` here. Split combined decision strings by `AND` or commas.
25. Create a **Switch** node named `Execute Approved Action` (`n8n-nodes-base.switch`). Connect `Split Actions` here. Enable all matching outputs and fallback handling, matching `RISK_REVIEW`, `CONTRACT_REVIEW`, and `MONITORING`.
26. Create three **Execute Workflow** nodes (`Call 'Supplier Risk Assessment'`, `Call 'Contract Review'`, `Call 'Supplier Monitoring'1`) connected to the respective outputs of `Execute Approved Action`, linking to the correct sub-workflow IDs with mapped parameters.
27. Create a **Code** node named `Verify Execution` (`n8n-nodes-base.code`). Connect all three sub-workflow execution nodes to this node.
28. Create a **Code** node named `Prepare Approved Result` (`n8n-nodes-base.code`). Connect `Verify Execution` output here.

#### Step 8: Build Rejection Handling Path
29. Create a **Code** node named `Prepare Rejection` (`n8n-nodes-base.code`). Connect the false output of `Approval Decision` here.
30. Create a **Code** node named `Prepare Rejected Result` (`n8n-nodes-base.code`). Connect `Prepare Rejection` output here.

#### Step 9: Finalize Audit Log, Webhook Response, and Sub-Workflows
31. Create a **Merge** node named `Merge1` (`n8n-nodes-base.merge`). Connect both `Prepare Approved Result` and `Prepare Rejected Result` into this node.
32. Create a **Code** node named `Audit Log` (`n8n-nodes-base.code`). Connect `Merge1` here to generate audit event records.
33. Create a **Code** node named `Prepare Webhook Response` (`n8n-nodes-base.code`). Connect `Audit Log` here to format nested status payloads.
34. Create a **Respond to Webhook** node (`n8n-nodes-base.respondToWebhook`). Connect `Prepare Webhook Response` here, setting response with JSON.
35. Recreate the three auxiliary sub-workflows (`Supplier Risk Assessment`, `Contract Review`, `Supplier Monitoring`) with their respective trigger nodes (`When Executed by Another Workflow`), validation code nodes, analytical steps, and responses as specified in the architecture.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Cross-System Procurement Orchestration Agent Architecture | Orchestrates cross-system procurement intake, evaluates supplier risk using AI, routes tasks to specialized sub-workflows, requires human approval, and audits all executed actions across ERP and compliance systems. |
| Sub-Workflow ID References | Referenced sub-workflows include `Supplier Risk Assessment` (`psbzlwJEUqkvih5v`), `Contract Review` (`yQoFszgvahoj9AYC`), and `Supplier Monitoring` (`eY21FxuyeNglO47w`). Ensure these workflows are published and active in your n8n instance before testing. |
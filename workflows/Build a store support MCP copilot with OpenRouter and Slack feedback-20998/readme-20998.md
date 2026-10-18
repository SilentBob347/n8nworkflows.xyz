Build a store support MCP copilot with OpenRouter and Slack feedback

https://n8nworkflows.xyz/workflows/build-a-store-support-mcp-copilot-with-openrouter-and-slack-feedback-20998


# Build a store support MCP copilot with OpenRouter and Slack feedback

### 1. Workflow Overview

This workflow establishes a complete store-support Model Context Protocol (MCP) server environment integrated with n8n Data Tables and an AI-driven support copilot. The primary goal is to empower internal customer support staff to efficiently query order histories, process compliant product returns, and dynamically capture tooling gaps or friction points ("tool-gap feedback") to route directly to a designated Slack channel.

The workflow logic is divided into four distinct operational blocks:
- **1.1 Demo Data Setup:** Manually triggered initialization and seeding of required n8n Data Tables with sample customer, order, and return records.
- **1.2 MCP Server & Tools Exposure:** Exposes a secure, bearer-authenticated MCP server endpoint hosting five discrete tools (`find_orders`, `get_order`, `get_returns`, `create_return`, and `send_feedback`).
- **1.3 Tool Execution Backend (Sub-Workflow):** Intercepts dynamic tool invocation payloads via an execution trigger, routes them by target tool name, executes complex business logic (e.g., validation, 30-day return policy enforcement, data scrubbing, rate limiting), and records data tables or dispatches Slack notifications.
- **1.4 AI Support Copilot:** Combines a chat interface trigger, an OpenRouter-backed large language model, buffer memory, and an MCP client tool pointing to the local MCP server endpoint to autonomously assist support staff.

---

### 2. Block-by-Block Analysis

#### 1.1 Demo Data Setup
- **Overview:** Initializes the database schema by provisioning three distinct n8n Data Tables, clears existing state, and seeds realistic demo order payloads relative to the current date.
- **Nodes Involved:** `Set Up Demo Data`, `Create Orders Table`, `Create Returns Table`, `Create Feedback Table`, `Clear Orders`, `Clear Returns`, `Seed Orders`, `One Row per Line`, `Insert Orders`.
- **Node Details:**
  - **Set Up Demo Data**: Type `n8n-nodes-base.manualTrigger`. Acts as the manual entry point to build or reset the environment.
  - **Create Orders Table / Returns Table / Feedback Table**: Type `n8n-nodes-base.dataTable`. Programmatically generates tables named `Store Support Orders`, `Store Support Returns`, and `Store Support Feedback` if they do not already exist.
  - **Clear Orders / Clear Returns**: Type `n8n-nodes-base.dataTable`. Truncates existing records to maintain a clean testing environment.
  - **Seed Orders**: Type `n8n-nodes-base.set`. Assigns an array of mock order objects utilizing dynamic Luxon expressions (e.g., `$now.minus(...)`) for relative date calculations.
  - **One Row per Line**: Type `n8n-nodes-base.splitOut`. Explodes the array of seed rows into individual items for bulk insertion.
  - **Insert Orders**: Type `n8n-nodes-base.dataTable`. Appends each processed line into the `Store Support Orders` table.

---

#### 1.2 MCP Server & Tools Exposure
- **Overview:** Exposes a secure Model Context Protocol server endpoint via HTTP Bearer Authentication, mapping native Data Table lookups and sub-workflow actions into standardized agent tools.
- **Nodes Involved:** `Store Support MCP`, `find_orders`, `get_order`, `get_returns`, `create_return`, `send_feedback`.
- **Node Details:**
  - **Store Support MCP**: Type `@n8n/n8n-nodes-langchain.mcpTrigger`. Exposes endpoints at `/mcp/store-support-template` using `httpBearerAuth` credentials.
  - **find_orders**: Type `n8n-nodes-base.dataTableTool`. Queries orders based on exact lowercase customer email matches, ordered descending by date.
  - **get_order**: Type `n8n-nodes-base.dataTableTool`. Retrieves all line items associated with a specific order number.
  - **get_returns**: Type `n8n-nodes-base.dataTableTool`. Pulls existing return records associated with an order number.
  - **create_return**: Type `@n8n/n8n-nodes-langchain.toolWorkflow`. Invokes the current workflow (`$workflow.id`) as a tool, passing parameters for SKU, quantity, reason, and order number.
  - **send_feedback**: Type `@n8n/n8n-nodes-langchain.toolWorkflow`. Invokes the current workflow as a tool to log agent friction points, capturing situation, observed problem, and expected solution.

---

#### 1.3 Tool Execution Backend
- **Overview:** Receives execution payloads from tool-based workflow triggers, routes them by requested action name, validates return eligibility or feedback constraints, updates data tables, and integrates with Slack.
- **Nodes Involved:** `MCP Tool Call`, `Route by Tool`, `Find Order Line`, `Existing Returns`, `Check Eligibility`, `Eligible?`, `Generate Return ID`, `Build Return`, `Store Return`, `Return Created`, `Return Refused`, `Normalize Feedback`, `Feedback Valid?`, `Feedback Last Hour`, `Over 30 per Hour?`, `Feedback Rate Limited`, `Generate Feedback ID`, `Build Feedback Row`, `Store Feedback`, `Notify Slack`, `Save Slack ts`, `Feedback Recorded`, `Feedback Invalid`, `Unknown Tool`.
- **Node Details:**
  - **MCP Tool Call**: Type `n8n-nodes-base.executeWorkflowTrigger`. Entry point for sub-workflow executions called by the MCP tools.
  - **Route by Tool**: Type `n8n-nodes-base.switch`. Evaluates `$json.tool` to branch execution between `create_return`, `send_feedback`, or an `Unknown Tool` fallback.
  - **Find Order Line / Existing Returns**: Type `n8n-nodes-base.dataTable`. Fetches targeted order line items and previously filed returns for validation.
  - **Check Eligibility**: Type `n8n-nodes-base.set`. Executes custom JavaScript rules checking order existence, delivery status, final sale flags, 30-day return windows, and quantity thresholds.
  - **Eligible?**: Type `n8n-nodes-base.if`. Branches based on whether the eligibility check returned an error code.
  - **Generate Return ID / Generate Feedback ID**: Type `n8n-nodes-base.crypto`. Generates unique identifiers using cryptographic UUID functions.
  - **Build Return / Store Return / Return Created / Return Refused**: Type `n8n-nodes-base.set` and `n8n-nodes-base.dataTable`. Formats RMA identifiers, writes return records to data tables, and returns structured success or failure payloads.
  - **Normalize Feedback**: Type `n8n-nodes-base.set`. Sanitizes input strings and validates character length constraints.
  - **Feedback Valid? / Feedback Last Hour / Over 30 per Hour?**: Type `n8n-nodes-base.if` and `n8n-nodes-base.dataTable`. Enforces a rate limit threshold of 30 feedback submissions per hour.
  - **Build Feedback Row**: Type `n8n-nodes-base.set`. Strips potential secrets, API keys, tokens, and email addresses from feedback payloads using regular expressions.
  - **Store Feedback / Notify Slack / Save Slack ts / Feedback Recorded**: Type `n8n-nodes-base.dataTable` and `n8n-nodes-base.slack`. Persists feedback data, publishes formatted Block Kit payloads to `#store-mcp-feedback` (configured with `continueErrorOutput` error handling), and stores the resulting message timestamp (`slack_ts`).

---

#### 1.4 AI Support Copilot
- **Overview:** Provides an interactive chat interface backed by an OpenRouter LLM, chat window memory, and an MCP client tool configured to communicate with the local store support MCP server.
- **Nodes Involved:** `Support Chat`, `Support Copilot`, `Chat Memory`, `Store Support`, `Copilot Model`.
- **Node Details:**
  - **Support Chat**: Type `@n8n/n8n-nodes-langchain.chatTrigger`. Exposes a streaming web chat interface.
  - **Support Copilot**: Type `@n8n/n8n-nodes-langchain.agent`. Orchestrates tool usage, prompt instructions, and agent reasoning.
  - **Chat Memory**: Type `@n8n/n8n-nodes-langchain.memoryBufferWindow`. Maintains conversation context restricted to a 10-message window.
  - **Store Support**: Type `@n8n/n8n-nodes-langchain.mcpClientTool`. Connects to the local MCP server endpoint (`http://localhost:5678/mcp/store-support-template`) using bearer token authentication.
  - **Copilot Model**: Type `@n8n/n8n-nodes-langchain.lmChatOpenRouter`. Integrates the OpenRouter language model (`anthropic/claude-haiku-5.5`).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Set Up Demo Data | manualTrigger | Manual trigger to seed demo data | None | Create Orders Table | 0 · Demo data |
| Template overview | stickyNote | Documentation and template overview | None | None | Build a store support MCP server and AI copilot that report tool gaps to Slack |
| Self-hosted note | stickyNote | Deployment and networking notes | None | None | Self-hosted note |
| Clear Orders | dataTable | Clears existing orders table | Create Feedback Table | Clear Returns | Set up demo data |
| Clear Returns | dataTable | Clears existing returns table | Clear Orders | Seed Orders | Set up demo data |
| Seed Orders | set | Defines array of mock order records | Clear Returns | One Row per Line | Set up demo data |
| One Row per Line | splitOut | Splits seed array into individual items | Seed Orders | Insert Orders | Set up demo data |
| Insert Orders | dataTable | Inserts order records into data table | One Row per Line | None | Set up demo data |
| Store Support MCP | mcpTrigger | Exposes MCP server endpoint | None | find_orders, get_order, get_returns, create_return, send_feedback | 1 · Store Support MCP |
| find_orders | dataTableTool | MCP tool for querying customer orders | Store Support MCP | Store Support MCP | 1 · Store Support MCP |
| get_order | dataTableTool | MCP tool for fetching order lines | Store Support MCP | Store Support MCP | 1 · Store Support MCP |
| get_returns | dataTableTool | MCP tool for listing existing returns | Store Support MCP | Store Support MCP | 1 · Store Support MCP |
| create_return | toolWorkflow | MCP tool invoking workflow sub-route | Store Support MCP | Store Support MCP | 1 · Store Support MCP |
| send_feedback | toolWorkflow | MCP tool invoking feedback sub-route | Store Support MCP | Store Support MCP | 1 · Store Support MCP |
| MCP Tool Call | executeWorkflowTrigger | Sub-workflow execution entry point | None | Route by Tool | 2 · Tool backend (sub-workflow) |
| Route by Tool | switch | Routes execution by tool name | MCP Tool Call | Find Order Line, Normalize Feedback, Unknown Tool | 2 · Tool backend (sub-workflow) |
| Find Order Line | dataTable | Queries target order line for return | Route by Tool | Existing Returns | create_return |
| Existing Returns | dataTable | Queries existing returns for order line | Find Order Line | Check Eligibility | create_return |
| Check Eligibility | set | Validates return rules and calculates refund | Existing Returns | Eligible? | create_return |
| Eligible? | if | Checks if return check passed | Check Eligibility | Generate Return ID, Return Refused | create_return |
| Generate Return ID | crypto | Generates cryptographic UUID for RMA | Eligible? | Build Return | create_return |
| Build Return | set | Formats RMA return payload | Generate Return ID | Store Return | create_return |
| Store Return | dataTable | Inserts return record into data table | Build Return | Return Created | create_return |
| Return Created | set | Formats success response for tool | Store Return | None | create_return |
| Return Refused | set | Formats policy refusal response | Eligible? | None | create_return |
| Normalize Feedback | set | Sanitizes feedback input fields | Route by Tool | Feedback Valid? | send_feedback |
| Feedback Valid? | if | Validates feedback character limits | Normalize Feedback | Feedback Last Hour, Feedback Invalid | send_feedback |
| Feedback Last Hour | dataTable | Queries feedback submitted in past hour | Feedback Valid? | Over 30 per Hour? | send_feedback |
| Over 30 per Hour? | if | Enforces hourly rate limit threshold | Feedback Last Hour | Feedback Rate Limited, Generate Feedback ID | send_feedback |
| Feedback Rate Limited | set | Formats rate limit error response | Over 30 per Hour? | None | send_feedback |
| Generate Feedback ID | crypto | Generates unique feedback identifier | Over 30 per Hour? | Build Feedback Row | send_feedback |
| Build Feedback Row | set | Scrubs secrets and personal data | Generate Feedback ID | Store Feedback | send_feedback |
| Store Feedback | dataTable | Inserts feedback record into table | Build Feedback Row | Notify Slack | send_feedback |
| Notify Slack | slack | Posts feedback report to Slack channel | Store Feedback | Save Slack ts, Feedback Recorded | send_feedback |
| Save Slack ts | dataTable | Saves Slack message timestamp to record | Notify Slack | Feedback Recorded | send_feedback |
| Feedback Recorded | set | Formats success response for feedback tool | Notify Slack, Save Slack ts | None | send_feedback |
| Feedback Invalid | set | Formats validation error response | Feedback Valid? | None | send_feedback |
| Unknown Tool | set | Formats unknown tool error response | Route by Tool | None | 2 · Tool backend (sub-workflow) |
| Support Chat | chatTrigger | Web chat trigger for support copilot | None | Support Copilot | 3 · Support Copilot Agent |
| Support Copilot | agent | AI Agent orchestrating support actions | Support Chat, Chat Memory, Store Support, Copilot Model | None | 3 · Support Copilot Agent |
| Chat Memory | memoryBufferWindow | Maintains conversation window memory | None | Support Copilot | 3 · Support Copilot Agent |
| Store Support | mcpClientTool | MCP client tool connecting to local server | None | Support Copilot | 3 · Support Copilot Agent |
| Lane 0 · Demo data | stickyNote | Grouping note for demo data setup | None | None | ## 0 · Demo data... |
| Lane 1 · Store Support MCP | stickyNote | Grouping note for MCP server tools | None | None | ## 1 · Store Support MCP... |
| Lane 2 · Tool backend | stickyNote | Grouping note for tool sub-workflow | None | None | ## 2 · Tool backend (sub-workflow)... |
| Lane 3 · Support Copilot | stickyNote | Grouping note for AI support copilot | None | None | ## 3 · Support Copilot Agent... |
| Copilot Model | lmChatOpenRouter | OpenRouter language model integration | None | Support Copilot | 3 · Support Copilot Agent |
| Create Orders Table | dataTable | Creates Store Support Orders data table | Set Up Demo Data | Create Returns Table | Set up demo data |
| Create Returns Table | dataTable | Creates Store Support Returns data table | Create Orders Table | Create Feedback Table | Set up demo data |
| Create Feedback Table | dataTable | Creates Store Support Feedback data table | Create Returns Table | Clear Orders | Set up demo data |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Data Tables**:
   - Create a Data Table named `Store Support Orders` with columns: `order_number` (string), `customer_email` (string), `ordered_at` (string), `delivered_at` (string), `status` (string), `sku` (string), `product_name` (string), `size` (string), `quantity` (number), `unit_price` (number), `currency` (string), `final_sale` (boolean).
   - Create a Data Table named `Store Support Returns` with columns: `return_id` (string), `order_number` (string), `sku` (string), `quantity` (number), `reason` (string), `status` (string), `refund_amount` (number), `currency` (string).
   - Create a Data Table named `Store Support Feedback` with columns: `feedback_id` (string), `status` (string), `situation` (string), `observed_problem` (string), `expected_solution` (string), `related_tool` (string), `ai_model` (string), `agent_harness` (string), `slack_ts` (string).

2. **Setup Credentials**:
   - Configure an HTTP Bearer Auth credential (`Store Support MCP Template`) with a secure token.
   - Configure an OpenRouter API credential.
   - Configure a Slack OAuth2 API credential.

3. **Build Block 1: Demo Data Initialization**:
   - Add a `Manual Trigger` node (`Set Up Demo Data`).
   - Add three `Data Table` nodes configured to create `Store Support Orders`, `Store Support Returns`, and `Store Support Feedback` if they do not exist. Connect them sequentially from the manual trigger.
   - Add two `Data Table` nodes configured to `clear` the `Store Support Orders` and `Store Support Returns` tables.
   - Add a `Set` node (`Seed Orders`) assigning the array of mock orders using dynamic Luxon date functions.
   - Add a `Split Out` node (`One Row per Line`) targeting the `rows` field.
   - Add a `Data Table` node (`Insert Orders`) inserting rows into `Store Support Orders`.

4. **Build Block 2: MCP Server & Tools**:
   - Add an `MCP Server Trigger` node (`Store Support MCP`) with path `store-support-template`, authentication set to `bearerAuth`, and select your Bearer credential.
   - Add three `Data Table Tool` nodes (`find_orders`, `get_order`, `get_returns`) linked as AI tools to the MCP trigger, configuring their filters and descriptions for Data Table access.
   - Add two `Workflow Tool` nodes (`create_return`, `send_feedback`) linked as AI tools to the MCP trigger, pointing to the current workflow ID (`{{ $workflow.id }}`) and defining the required input schemas.

5. **Build Block 3: Tool Execution Backend (Sub-Workflow)**:
   - Add an `Execute Workflow Trigger` node (`MCP Tool Call`) defining workflow inputs (`tool`, `order_number`, `sku`, `quantity`, `reason`, `situation`, `observed_problem`, `expected_solution`, `related_tool`, `ai_model`, `agent_harness`).
   - Add a `Switch` node (`Route by Tool`) evaluating `{{ $json.tool }}` with rules for `create_return` and `send_feedback`, plus a fallback output for unknown tools.
   - **For the `create_return` branch**:
     - Add Data Table nodes to fetch matching order lines (`Find Order Line`) and existing returns (`Existing Returns`).
     - Add a `Set` node (`Check Eligibility`) implementing the business logic script (verifying delivery status, 30-day window, final sale status, and remaining returnable quantities).
     - Add an `If` node (`Eligible?`) checking if the error code is empty.
     - On the true branch, add a `Crypto` node (`Generate Return ID`) to generate a UUID, a `Set` node (`Build Return`) formatting the RMA payload, a Data Table node (`Store Return`) inserting the record, and a `Set` node (`Return Created`) returning success details.
     - On the false branch, add a `Set` node (`Return Refused`) returning policy codes and failure details.
   - **For the `send_feedback` branch**:
     - Add a `Set` node (`Normalize Feedback`) to trim and validate string inputs.
     - Add an `If` node (`Feedback Valid?`) checking for validation errors.
     - Add a Data Table node (`Feedback Last Hour`) querying submissions where `createdAt` is greater than or equal to 1 hour ago.
     - Add an `If` node (`Over 30 per Hour?`) checking if submissions count is `>= 30`.
     - On the true rate-limited branch, add a `Set` node (`Feedback Rate Limited`).
     - On the false branch, add a `Crypto` node (`Generate Feedback ID`), a `Set` node (`Build Feedback Row`) using regex replacements to scrub secrets and emails, a Data Table node (`Store Feedback`), a `Slack` node (`Notify Slack`) targeting `#store-mcp-feedback` with `continueErrorOutput` enabled, a Data Table node (`Save Slack ts`) updating the record with the message timestamp, and a `Set` node (`Feedback Recorded`) returning success.
   - Add a `Set` node (`Unknown Tool`) for unhandled tool routing branches.

6. **Build Block 4: AI Support Copilot**:
   - Add a `Chat Trigger` node (`Support Chat`) with response mode set to streaming.
   - Add an `AI Agent` node (`Support Copilot`) with system instructions defining agent behavior, feedback criteria, and strict privacy handling.
   - Connect a `Buffer Window Memory` node (`Chat Memory`) and an OpenRouter Language Model node (`Copilot Model`) to the AI Agent.
   - Add an `MCP Client Tool` node (`Store Support`) configured with endpoint URL `http://localhost:5678/mcp/store-support-template` and the same `httpBearerAuth` credential, connecting it as an AI tool to the AI Agent.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Built by Cloud Labs | [Cloud Labs Website](https://low-code-consultant.com/brand/n8n-strip.png#full-width) |
| Self-hosted deployment note | Uses `http://localhost:5678/mcp/store-support-template` for local Docker container accessibility. |
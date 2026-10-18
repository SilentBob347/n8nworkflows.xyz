Qualify chat leads and create HubSpot companies with DeepSeek and Google Calendar

https://n8nworkflows.xyz/workflows/qualify-chat-leads-and-create-hubspot-companies-with-deepseek-and-google-calendar-21005


# Qualify chat leads and create HubSpot companies with DeepSeek and Google Calendar

### 1. Workflow Overview

This workflow is designed to automate lead qualification and CRM record creation through an interactive chat interface. It leverages an AI-powered agent backed by DeepSeek to evaluate inbound chat messages for intent regarding automation workflows. Depending on the conversation content, the workflow either terminates with a default response or automatically provisions a new company record in HubSpot and schedules a corresponding follow-up event in Google Calendar.

The architecture is divided into two primary logical blocks:
- **1.1 Input Reception & AI Qualification:** Receives chat inputs, manages conversation memory, uses the DeepSeek model for intent analysis, and conditionally executes HubSpot record creation via tool calling.
- **1.2 Conditional Verification, Data Extraction & Calendar Scheduling:** Evaluates whether a HubSpot company was successfully created, parses structured JSON data from the AI's markdown response using JavaScript, sets a success status flag, and creates a follow-up event in Google Calendar.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & AI Qualification
- **Overview:** This block acts as the entry point for chat interactions, maintains conversation history, and coordinates an AI agent powered by DeepSeek to analyze user intent and perform tool-based CRM actions.
- **Nodes Involved:** 
  - When chat message received
  - AI Agent
  - DeepSeek Chat Model
  - Simple Memory
  - Create a company in HubSpot

- **Node Details:**
  - **When chat message received**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chatTrigger` (Chat Trigger) - Listens for incoming public webhook messages from chat clients.
    - *Configuration Choices:* Public access enabled (`true`), file uploads allowed, and CORS origins configured to accept all (`*`).
    - *Key Expressions or Variables:* None (triggers on inbound payload).
    - *Input/Output Connections:* Output connects to the `AI Agent` main input.
    - *Potential Failure Types:* Webhook timeout, unreachable public URL, or unauthorized origin requests.
  
  - **AI Agent**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent) - Orchestrates the reasoning loop using system instructions, language models, memory, and tools.
    - *Configuration Choices:* System message instructs the agent to act as a helpful assistant that evaluates whether incoming messages concern n8n workflows, creates HubSpot companies when relevant, or replies with "sorry, I don't know" otherwise.
    - *Key Expressions or Variables:* Uses system prompt text defining operational steps and output formatting rules (markdown with embedded JSON).
    - *Input/Output Connections:* Inputs from Chat Trigger, DeepSeek Chat Model, Simple Memory, and HubSpot Tool. Output connects to the `If` node.
    - *Potential Failure Types:* LLM rate limits, hallucinated tool calls, or failure to match expected markdown output structures.

  - **DeepSeek Chat Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatDeepSeek` (DeepSeek Chat Model) - Provides the underlying Large Language Model engine for the AI Agent.
    - *Configuration Choices:* Uses default options linked to a DeepSeek API credential.
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Connects to the `AI Agent` via the language model (`ai_languageModel`) interface.
    - *Potential Failure Types:* API authentication failures, insufficient account credits, or API downtime.

  - **Simple Memory**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.memoryBufferWindow` (Window Buffer Memory) - Maintains short-term conversational context across message turns.
    - *Configuration Choices:* Standard window buffer implementation (Version 1.4).
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Connects to the `AI Agent` via the memory (`ai_memory`) interface.
    - *Potential Failure Types:* Memory state loss if workflow resets or concurrency issues under heavy load.

  - **Create a company in HubSpot**
    - *Type and Technical Role:* `n8n-nodes-base.hubspotTool` (HubSpot Tool) - Exposes HubSpot company creation capabilities as a callable tool for the AI Agent.
    - *Configuration Choices:* Resource set to `company`, authentication via OAuth2. Maps properties (`Name`, `City`, `Postal_Code`, `Website_URL`, `Description`, `Phone_Number`, `State_Region`, `Annual_Revenue`, `Country_Region`, `Street_Address`, `Street_Address_2`) dynamically via AI extraction expressions.
    - *Key Expressions or Variables:* Uses `$fromAI()` binding expressions (e.g., `={{ /*n8n-auto-generated-fromAI-override*/ $fromAI('Name', '', 'string') }}`).
    - *Input/Output Connections:* Connects to the `AI Agent` via the tool (`ai_tool`) interface.
    - *Potential Failure Types:* OAuth2 token expiration, missing scope permissions to create companies, or invalid data types returned by the LLM.

---

#### Block 1.2: Conditional Verification, Data Extraction & Calendar Scheduling
- **Overview:** This block filters successful qualification events, extracts structured JSON payloads from markdown responses using JavaScript, flags the operation as successful, and schedules a follow-up calendar event.
- **Nodes Involved:**
  - If
  - Code in JavaScript
  - Edit Fields
  - Create an event

- **Node Details:**
  - **If**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router) - Evaluates whether the AI agent's output confirms successful record creation.
    - *Configuration Choices:* Strict type validation enabled with an expression check.
    - *Key Expressions or Variables:* Evaluates if `{{ $json.output }}` contains the string `"Company Created in HubSpot"`.
    - *Input/Output Connections:* Input from `AI Agent`. True branch connects to `Code in JavaScript`.
    - *Potential Failure Types:* Expression evaluation errors if the output property is missing.

  - **Code in JavaScript**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Node) - Parses embedded JSON blocks within the LLM's markdown response.
    - *Configuration Choices:* Custom script loops through items, uses regular expressions (`/```json\s*([\s\S]*?)\s*```/`) to locate JSON blocks, parses the content, and assigns the resulting company object to `item.json.company`. Handles parsing errors gracefully.
    - *Key Expressions or Variables:* Accesses `$input.all()` and evaluates `item.json.output`.
    - *Input/Output Connections:* Input from `If` (True branch). Output connects to `Edit Fields`.
    - *Potential Failure Types:* Malformed JSON syntax in LLM output throwing parsing errors, resulting in `null` company payloads.

  - **Edit Fields**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Set / Edit Fields) - Modifies item properties to append execution metadata.
    - *Configuration Choices:* Includes other fields and assigns a boolean flag.
    - *Key Expressions or Variables:* Assigns `is_susccess` (boolean) set to `=true`.
    - *Input/Output Connections:* Input from `Code in JavaScript`. Output connects to `Create an event`.
    - *Potential Failure Types:* Property assignment conflicts.

  - **Create an event**
    - *Type and Technical Role:* `n8n-nodes-base.googleCalendar` (Google Calendar) - Creates a follow-up event in the specified user calendar.
    - *Configuration Choices:* Uses OAuth2 authentication. Configures start time to 1 day from now (`$now.plus(1, 'day')`) and end time to 26 hours from now (`$now.plus(26, 'hour')`).
    - *Key Expressions or Variables:* Event summary uses `={{"Follow up: " + $json.company.Name}}`. Calendar list targets `user@example.com`.
    - *Input/Output Connections:* Input from `Edit Fields`.
    - *Potential Failure Types:* OAuth2 token expiry, invalid calendar ID, or missing `.Name` property if JSON parsing failed upstream.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When chat message received | `@n8n/n8n-nodes-langchain.chatTrigger` | Chat Trigger | None | AI Agent | n8n2Hubspot / Chat-driven HubSpot agent |
| AI Agent | `@n8n/n8n-nodes-langchain.agent` | AI Agent | When chat message received, DeepSeek Chat Model, Simple Memory, Create a company in HubSpot | If | n8n2Hubspot / Chat-driven HubSpot agent |
| DeepSeek Chat Model | `@n8n/n8n-nodes-langchain.lmChatDeepSeek` | LLM Model | None | AI Agent | n8n2Hubspot / Chat-driven HubSpot agent |
| Simple Memory | `@n8n/n8n-nodes-langchain.memoryBufferWindow` | Conversation Memory | None | AI Agent | n8n2Hubspot / Chat-driven HubSpot agent |
| Create a company in HubSpot | `n8n-nodes-base.hubspotTool` | AI Tool (HubSpot) | None | AI Agent | n8n2Hubspot / Chat-driven HubSpot agent |
| If | `n8n-nodes-base.if` | Conditional Router | AI Agent | Code in JavaScript | n8n2Hubspot |
| Code in JavaScript | `n8n-nodes-base.code` | JSON Data Extraction | If | Edit Fields | n8n2Hubspot / Create Google Calendar Event |
| Edit Fields | `n8n-nodes-base.set` | Status Flag Assignment | Code in JavaScript | Create an event | n8n2Hubspot / Create Google Calendar Event |
| Create an event | `n8n-nodes-base.googleCalendar` | Calendar Scheduling | Edit Fields | None | n8n2Hubspot / Create Google Calendar Event |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Point:** Add a **When chat message received** node (`@n8n/n8n-nodes-langchain.chatTrigger`). Set public access to `true`, allow file uploads, and configure allowed origins as `*`.
2. **Add the Core AI Agent:** Add an **AI Agent** node (`@n8n/n8n-nodes-langchain.agent`). Configure its system message to instruct the agent to evaluate incoming messages regarding automation workflows, create a HubSpot company when interested, respond with "sorry, I don't know" otherwise, and format outputs in markdown including a JSON block named `company` prefixed by the phrase `'Company Created in HubSpot'`.
3. **Configure the Language Model:** Add a **DeepSeek Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatDeepSeek`). Connect its output to the `ai_languageModel` input of the **AI Agent**. Configure credentials with a valid DeepSeek API account.
4. **Configure Conversation Memory:** Add a **Simple Memory** node (`@n8n/n8n-nodes-langchain.memoryBufferWindow`). Connect its output to the `ai_memory` input of the **AI Agent**.
5. **Configure the HubSpot Tool:** Add a **Create a company in HubSpot** node (`n8n-nodes-base.hubspotTool`). Set resource to `company` and authentication to OAuth2 using HubSpot account credentials. Map standard parameters (`Name`, `City`, `Postal_Code`, `Website_URL`, `Description`, `Phone_Number`, `State_Region`, `Annual_Revenue`, `Country_Region`, `Street_Address`, `Street_Address_2`) using `$fromAI()` expression bindings. Connect its output to the `ai_tool` input of the **AI Agent**.
6. **Connect Trigger to Agent:** Connect the **When chat message received** output to the main input of the **AI Agent**.
7. **Add Conditional Filtering:** Add an **If** node (`n8n-nodes-base.if`). Connect the main output of the **AI Agent** to its input. Set the condition to check if `{{ $json.output }}` contains the string `Company Created in HubSpot`.
8. **Add JSON Parsing Script:** Add a **Code in JavaScript** node (`n8n-nodes-base.code`). Connect the True branch of the **If** node to its input. Insert the script to match `/```json\s*([\s\S]*?)\s*```/`, parse the JSON payload, and attach the resulting object to `item.json.company`.
9. **Add Status Assignment:** Add an **Edit Fields** node (`n8n-nodes-base.set`). Connect the **Code in JavaScript** output to its input. Add an assignment field named `is_susccess` of type boolean with value `=true`, and retain other fields.
10. **Add Calendar Scheduling:** Add a **Create an event** node (`n8n-nodes-base.googleCalendar`). Connect the **Edit Fields** output to its input. Configure Google Calendar OAuth2 credentials, select the target calendar (e.g., `user@example.com`), set start time to `={{ $now.plus(1, 'day') }}`, set end time to `={{ $now.plus(26, 'hour') }}`, and set the summary additional field to `={{"Follow up: " + $json.company.Name}}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| n8n2Hubspot Workflow Overview and Customization Guide | Refer to sticky note annotations within the workflow canvas for architecture setup guidelines and prompt modification instructions. |
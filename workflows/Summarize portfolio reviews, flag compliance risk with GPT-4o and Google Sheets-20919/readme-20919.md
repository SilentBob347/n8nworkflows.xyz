Summarize portfolio reviews, flag compliance risk with GPT-4o and Google Sheets

https://n8nworkflows.xyz/workflows/summarize-portfolio-reviews--flag-compliance-risk-with-gpt-4o-and-google-sheets-20919


# Summarize portfolio reviews, flag compliance risk with GPT-4o and Google Sheets

### 1. Workflow Overview

This workflow automates the processing of wealth advisory portfolio review meeting notes. Its primary purpose is to transform raw unstructured meeting notes into an objective compliance audit trail by verifying discussed topics against a client's risk tolerance on file using OpenAI GPT-4o. The system identifies potential suitability mismatches (such as concentration risk or risk-tolerance drift), records a structured review into Google Sheets, and executes conditional escalations based on the determined risk rating.

The workflow logic is divided into four functional blocks:
- **1.1 Input Reception & Normalization:** Receives incoming HTTP POST requests containing meeting data and normalizes payload fields with default values.
- **1.2 AI Processing & Structuring:** Utilizes OpenAI GPT-4o alongside a structured output parser to evaluate compliance risk and generate summaries, flagged issues, and recommended follow-ups.
- **1.3 Data Logging & Standardization:** Reshapes the AI output into a standardized schema and records the review permanently into a Google Sheets compliance log.
- **1.4 Conditional Routing & Notification:** Evaluates the risk level (high, medium, or low) to route alerts appropriately via Gmail and Telegram.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Normalization

- **Overview:** This block captures raw advisor meeting notes via a webhook and standardizes all input fields to guarantee data consistency for downstream processes.
- **Nodes Involved:** 
  - `Webhook - Advisor Meeting Notes Form`
  - `Normalize Input`

- **Node Details:**
  - **Webhook - Advisor Meeting Notes Form**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger). Acts as the entry point for external CRMs or intake forms.
    - *Configuration Choices:* Configured to listen for incoming HTTP `POST` requests on the relative path `portfolio-review-intake`.
    - *Key Expressions/Variables:* Uses standard webhook body extraction.
    - *Input/Output:* No incoming connections (triggers execution). Outputs data payload to `Normalize Input`.
    - *Edge Cases/Failures:* Invalid payload structures or missing headers will result in unparsed body data; mitigated by subsequent normalization.

  - **Normalize Input**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data transformation / Edit Fields). Ensures missing optional fields receive predictable default values.
    - *Configuration Choices:* Maps incoming properties (`clientName`, `advisorName`, `advisorEmail`, `meetingNotes`, `riskToleranceOnFile`) using nullish coalescing operators and adds a standardized ISO timestamp (`submittedAt`).
    - *Key Expressions/Variables:* 
      - `clientName`: `={{ $json.body?.clientName ?? 'Client' }}`
      - `advisorName`: `={{ $json.body?.advisorName ?? 'Advisor' }}`
      - `advisorEmail`: `={{ $json.body?.advisorEmail ?? '' }}`
      - `meetingNotes`: `={{ $json.body?.meetingNotes ?? '' }}`
      - `riskToleranceOnFile`: `={{ $json.body?.riskToleranceOnFile ?? 'Not on file' }}`
      - `submittedAt`: `={{ $now.toISO() }}`
    - *Input/Output:* Connected from `Webhook - Advisor Meeting Notes Form`. Outputs normalized variables to `AI Agent - Summarize & Flag Compliance Risk`.
    - *Edge Cases/Failures:* Expression execution failures if upstream body is entirely missing (handled by optional chaining `?.`).

---

#### Block 1.2: AI Processing & Structuring

- **Overview:** This block connects the normalized meeting data to an OpenAI LLM using a rigid JSON schema parser to extract structured compliance assessments.
- **Nodes Involved:** 
  - `OpenAI Chat Model`
  - `Structured Output Parser`
  - `AI Agent - Summarize & Flag Compliance Risk`

- **Node Details:**
  - **OpenAI Chat Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model Sub-node). Provides the GPT-4o engine for the AI Agent.
    - *Configuration Choices:* Uses model `gpt-4o` with a low temperature (`0.2`) to maintain objective, deterministic compliance analysis.
    - *Credentials:* Requires an active OpenAI API credential.
    - *Input/Output:* Connects exclusively to the `ai_languageModel` input of the AI Agent node.
    - *Edge Cases/Failures:* API rate limits, insufficient account balance, or upstream OpenAI outages.

  - **Structured Output Parser**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Parser Sub-node). Enforces a strict JSON response schema from the LLM.
    - *Configuration Choices:* Configured manually via JSON schema requiring four properties: `complianceRisk` (enum: high, medium, low), `summary` (string), `flaggedIssues` (string), and `recommendedFollowUp` (string).
    - *Input/Output:* Connects to the `ai_outputParser` input of the AI Agent node.
    - *Edge Cases/Failures:* Model hallucinations failing schema validation are handled by n8n’s LangChain error structures.

  - **AI Agent - Summarize & Flag Compliance Risk**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent orchestration node). Coordinates the prompt evaluation using the connected model and parser.
    - *Configuration Choices:* Configured with a compliance-aware system message instructing the model to act as a review assistant that avoids investment advice and flags factual mismatches only. Error handling is set to `continueRegularOutput`.
    - *Key Expressions/Variables:* 
      - Prompt text interpolates client name, advisor name, risk tolerance, and raw notes: `=Client: {{ $json.clientName }} \n Advisor: {{ $json.advisorName }} ...`
    - *Input/Output:* Receives input from `Normalize Input`. Outputs AI response structures to `Shape Output`.
    - *Edge Cases/Failures:* Parsing failures or execution timeouts. Configured with `onError: continueRegularOutput`.

---

#### Block 1.3: Data Logging & Standardization

- **Overview:** This block standardizes the parsed AI output and logs the resulting review row into a designated Google Sheets audit spreadsheet.
- **Nodes Involved:** 
  - `Shape Output`
  - `Log Review to Compliance Tracker`

- **Node Details:**
  - **Shape Output**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation). Consolidates original client identifiers with the nested AI outputs into a flat data structure.
    - *Configuration Choices:* Maps top-level JSON properties from the previous node and preserves original inputs using item-reference expressions.
    - *Key Expressions/Variables:*
      - `clientName`: `={{ $('Normalize Input').item.json.clientName }}`
      - `complianceRisk`: `={{ $json.output.complianceRisk }}`
      - `summary`: `={{ $json.output.summary }}`
      - `loggedAt`: `={{ $now.toISO() }}`
    - *Input/Output:* Receives input from `AI Agent - Summarize & Flag Compliance Risk`. Outputs flat objects to `Log Review to Compliance Tracker`.

  - **Log Review to Compliance Tracker**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets Integration). Appends the compliance review record to an audit spreadsheet tab.
    - *Configuration Choices:* Operation set to `append`. Document ID uses a placeholder (`REPLACE_WITH_YOUR_SHEET_ID`), and sheet name targets `Portfolio Reviews`. Configured with `onError: continueRegularOutput`.
    - *Credentials:* Requires valid Google Sheets OAuth2 credentials.
    - *Input/Output:* Receives input from `Shape Output`. Outputs appended row confirmation to `Route by Compliance Risk`.
    - *Edge Cases/Failures:* Authentication revocation, missing columns in the target sheet, or API timeouts. Mitigated by `continueRegularOutput`.

---

#### Block 1.4: Conditional Routing & Notification

- **Overview:** This block inspects the compliance risk rating and routes notifications via email to advisors/officers and instant alerts via Telegram for high-risk flags.
- **Nodes Involved:** 
  - `Route by Compliance Risk`
  - `Alert Compliance Officer`
  - `Send Advisor Summary Email`
  - `Alert Compliance Team (High Risk)`

- **Node Details:**
  - **Route by Compliance Risk**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Flow Control). Splits execution paths based on the `complianceRisk` string value.
    - *Configuration Choices:* Defines three rule outputs: `high`, `medium`, and `low`, evaluating `={{ $json.complianceRisk }}`. Fallback output is set to `none`.
    - *Input/Output:* Receives input from `Log Review to Compliance Tracker`. Routes execution to `Alert Compliance Officer` (high risk) or `Send Advisor Summary Email` (medium/low risk).

  - **Alert Compliance Officer**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Integration). Sends an immediate notification email regarding high-risk items.
    - *Configuration Choices:* Recipient set to `={{ $json.advisorEmail }}` (or routing target), subject prefixed with an warning emoji, and message containing flagged issues and summaries. `onError: continueRegularOutput`.
    - *Credentials:* Requires Gmail OAuth2 credentials.
    - *Input/Output:* Receives execution from the `high` branch of the Switch node. Triggers `Alert Compliance Team (High Risk)`.

  - **Send Advisor Summary Email**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Integration). Sends a standard review summary for medium- and low-risk reviews.
    - *Configuration Choices:* Sends summary and recommended follow-ups to `={{ $json.advisorEmail }}`. `onError: continueRegularOutput`.
    - *Credentials:* Requires Gmail OAuth2 credentials.
    - *Input/Output:* Receives execution from the `medium` or `low` branches of the Switch node. Terminal node for those branches.

  - **Alert Compliance Team (High Risk)**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Telegram Integration). Broadcasts an instant high-risk alert to a Telegram channel.
    - *Configuration Choices:* Chat ID uses a placeholder (`REPLACE_WITH_YOUR_CHAT_ID`), message formatted using Markdown (`parse_mode: Markdown`), referencing client name, advisor, and flagged issues. `onError: continueRegularOutput`.
    - *Credentials:* Requires Telegram Bot API credentials.
    - *Input/Output:* Receives execution exclusively from `Alert Compliance Officer`. Terminal node for the high-risk escalation path.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Webhook - Advisor Meeting Notes Form** | `n8n-nodes-base.webhook` | Receives raw advisor meeting notes via HTTP POST. | None (Trigger) | Normalize Input | # 📘 WORKFLOW DETAILS<br><br>**AI Portfolio Review Summarizer & Compliance Risk Flagger**<br>Value prop: turns an advisor's raw post-meeting notes into an objective summary plus a suitability/compliance policy check against the client's risk tolerance on file — not just a meeting summary.<br><br>Who it's for: RIAs and wealth-management firms needing a documented, timestamped compliance trail on every client review meeting.<br>Problem solved: suitability mismatches (concentration, risk-tolerance drift) go unnoticed until an audit or complaint.<br><br>How it works:<br>1. Webhook - Advisor Meeting Notes Form receives the raw notes + risk profile.<br>2. Normalize Input defaults every optional field.<br>3. OpenAI Chat Model + Structured Output Parser feed AI Agent - Summarize & Flag Compliance Risk, which checks the notes against the client's stated risk tolerance.<br>4. Shape Output is the single source of truth downstream.<br>5. Log Review to Compliance Tracker writes the permanent audit row.<br>6. Route by Compliance Risk sends high-risk cases to an instant Telegram + email alert to the compliance officer, and medium/low to a routine advisor summary email.<br><br>Expected input (POST /portfolio-review-intake): `{"clientName":"J. Thompson","advisorName":"M. Lee","advisorEmail":"...","meetingNotes":"Discussed moving 60% into a single tech stock...","riskToleranceOnFile":"Conservative"}`<br><br>Expected output: one row in the compliance audit sheet, plus a routed email (and Telegram alert for high risk). |
| **Normalize Input** | `n8n-nodes-base.set` | Normalizes fields and injects default values and timestamps. | Webhook - Advisor Meeting Notes Form | AI Agent - Summarize & Flag Compliance Risk | |
| **OpenAI Chat Model** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Connects GPT-4o language model to the AI agent. | None (Sub-node) | AI Agent - Summarize & Flag Compliance Risk | |
| **Structured Output Parser** | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces JSON output schema for compliance assessments. | None (Sub-node) | AI Agent - Summarize & Flag Compliance Risk | |
| **AI Agent - Summarize & Flag Compliance Risk** | `@n8n/n8n-nodes-langchain.agent` | Evaluates compliance risk against client risk tolerance. | Normalize Input, OpenAI Chat Model, Structured Output Parser | Shape Output | |
| **Shape Output** | `n8n-nodes-base.set` | Flattens and standardizes AI and client data. | AI Agent - Summarize & Flag Compliance Risk | Log Review to Compliance Tracker | |
| **Log Review to Compliance Tracker** | `n8n-nodes-base.googleSheets` | Appends review records to a Google Sheets audit log. | Shape Output | Route by Compliance Risk | |
| **Route by Compliance Risk** | `n8n-nodes-base.switch` | Branches execution paths based on risk classification. | Log Review to Compliance Tracker | Alert Compliance Officer, Send Advisor Summary Email | |
| **Alert Compliance Officer** | `n8n-nodes-base.gmail` | Emails the compliance officer on high-risk findings. | Route by Compliance Risk | Alert Compliance Team (High Risk) | |
| **Send Advisor Summary Email** | `n8n-nodes-base.gmail` | Sends routine review summary to the advisor. | Route by Compliance Risk | None (Terminal) | |
| **Alert Compliance Team (High Risk)** | `n8n-nodes-base.telegram` | Broadcasts high-risk alerts to a Telegram channel. | Alert Compliance Officer | None (Terminal) | |
| **Doc Panel 1 - Workflow Details** | `n8n-nodes-base.stickyNote` | Provides high-level documentation and expected inputs/outputs. | None | None | # 📘 WORKFLOW DETAILS<br><br>**AI Portfolio Review Summarizer & Compliance Risk Flagger**<br>Value prop: turns an advisor's raw post-meeting notes into an objective summary plus a suitability/compliance policy check against the client's risk tolerance on file — not just a meeting summary.<br><br>Who it's for: RIAs and wealth-management firms needing a documented, timestamped compliance trail on every client review meeting.<br>Problem solved: suitability mismatches (concentration, risk-tolerance drift) go unnoticed until an audit or complaint.<br><br>How it works:<br>1. Webhook - Advisor Meeting Notes Form receives the raw notes + risk profile.<br>2. Normalize Input defaults every optional field.<br>3. OpenAI Chat Model + Structured Output Parser feed AI Agent - Summarize & Flag Compliance Risk, which checks the notes against the client's stated risk tolerance.<br>4. Shape Output is the single source of truth downstream.<br>5. Log Review to Compliance Tracker writes the permanent audit row.<br>6. Route by Compliance Risk sends high-risk cases to an instant Telegram + email alert to the compliance officer, and medium/low to a routine advisor summary email.<br><br>Expected input (POST /portfolio-review-intake): `{"clientName":"J. Thompson","advisorName":"M. Lee","advisorEmail":"...","meetingNotes":"Discussed moving 60% into a single tech stock...","riskToleranceOnFile":"Conservative"}`<br><br>Expected output: one row in the compliance audit sheet, plus a routed email (and Telegram alert for high risk). |
| **Doc Panel 2 - Setup Guide** | `n8n-nodes-base.stickyNote` | Documents required credentials, setup steps, and testing instructions. | None | None | # ⚙️ SETUP GUIDE<br><br>Required credentials — attach at these exact nodes:<br>1. OpenAI -> OpenAI Chat Model node.<br>2. Google Sheets -> Log Review to Compliance Tracker node.<br>3. Gmail -> Alert Compliance Officer and Send Advisor Summary Email nodes.<br>4. Telegram -> Alert Compliance Team (High Risk) node.<br><br>Setup steps:<br>1. Import and assign your firm's credentials on each flagged node.<br>2. Replace the Google Sheet ID in Log Review to Compliance Tracker with your firm's audit-log spreadsheet (tab: Portfolio Reviews).<br>3. Replace the Telegram chat ID in Alert Compliance Team (High Risk) with your compliance team's channel.<br>4. Point your internal meeting-notes form / CRM webhook at the Webhook production URL.<br>5. Review the compliance system message before customizing — this workflow explicitly does not give investment advice.<br><br>How to test: POST a mock high-risk note and a mock clean note; confirm both branches route correctly.<br><br>Estimated setup time: 20-25 minutes. |
| **Doc Panel 3 - Customisation** | `n8n-nodes-base.stickyNote` | Outlines potential extensions and customizations. | None | None | # 🎨 CUSTOMISATION<br><br>1. Add a PDF-generation node to produce a signed, dated compliance memo automatically — edit after Log Review to Compliance Tracker.<br>2. Connect to a CRM (Salesforce/Redtail/Wealthbox) via HTTP Request fed from Shape Output.<br>3. Add a quarterly digest sub-workflow rolling up flaggedIssues into one CCO report.<br>4. Add a second AI pass on Shape Output drafting the client-facing meeting recap.<br>5. Swap notification channel: replace Alert Compliance Team (High Risk) (Telegram) with Slack. |
| **Doc Panel 4 - Additional Information** | `n8n-nodes-base.stickyNote` | Covers error handling, data privacy, limitations, and creator credits. | None | None | # ℹ️ ADDITIONAL INFORMATION<br><br>Error handling: Sheets, Gmail and Telegram nodes use onError: continueRegularOutput so one failed integration never blocks the audit log or the other notification.<br><br>API cost & rate limits: 1 OpenAI call per submission; all integrations run within free-tier quotas for typical review volumes.<br><br>Data privacy: only client name, advisor name, risk tolerance on file, and raw meeting notes are sent to the model. Stored data is limited to client/advisor names, complianceRisk, summary, flaggedIssues, recommendedFollowUp, and timestamp.<br><br>Known limitations & prerequisites: flags POTENTIAL mismatches for human compliance review — does not make compliance determinations or give investment advice; requires a Google Sheet already shared and a Telegram bot in your compliance channel.<br><br>Creator: Swapnil AI Labs — swapnil.mandloi7@gmail.com — swapnilailabs.netlify.app |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow manually in n8n:

1. **Create Webhook Trigger Node**
   - Add a **Webhook** node named `Webhook - Advisor Meeting Notes Form`.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `portfolio-review-intake`.

2. **Add Input Normalization Node**
   - Add an **Edit Fields (Set)** node named `Normalize Input`.
   - Connect `Webhook - Advisor Meeting Notes Form` output to `Normalize Input`.
   - Add the following string assignments:
     - `clientName`: `={{ $json.body?.clientName ?? 'Client' }}`
     - `advisorName`: `={{ $json.body?.advisorName ?? 'Advisor' }}`
     - `advisorEmail`: `={{ $json.body?.advisorEmail ?? '' }}`
     - `meetingNotes`: `={{ $json.body?.meetingNotes ?? '' }}`
     - `riskToleranceOnFile`: `={{ $json.body?.riskToleranceOnFile ?? 'Not on file' }}`
     - `submittedAt`: `={{ $now.toISO() }}`

3. **Configure AI Agent and Sub-nodes**
   - Add an **AI Agent** node named `AI Agent - Summarize & Flag Compliance Risk`.
   - Set **Prompt Type** to `Define` and enter text:
     ```text
     =Client: {{ $json.clientName }}
     Advisor: {{ $json.advisorName }}
     Risk tolerance on file: {{ $json.riskToleranceOnFile }}
     Raw meeting notes:
     {{ $json.meetingNotes }}

     Summarize and flag compliance risk per the schema.
     ```
   - Set **System Message**:
     ```text
     You are a compliance-aware review assistant for a registered investment advisory firm. You do NOT give investment advice or recommendations. Objectively summarize the advisor's raw meeting notes, then flag ONLY factual mismatches worth a human compliance officer's attention: risk-tolerance-on-file vs. discussed changes, concentration in a single position/sector, or any language suggesting an unsuitable recommendation. Classify complianceRisk as high, medium, or low. If nothing is flagged, say so plainly — do not invent risk.
     ```
   - Enable **Has Output Parser** and set **Error Handling** to `Continue Regular Output`.
   - Connect `Normalize Input` main output to the `AI Agent` main input.
   - **OpenAI Chat Model Sub-node:**
     - Add an **OpenAI Chat Model** node.
     - Select model `gpt-4o`, set temperature to `0.2`.
     - Configure an OpenAI API credential (`openAiApi`).
     - Connect its output to the `ai_languageModel` input of the AI Agent.
   - **Structured Output Parser Sub-node:**
     - Add a **Structured Output Parser** node.
     - Set schema type to manual and configure the JSON schema with properties `complianceRisk` (enum: high, medium, low), `summary`, `flaggedIssues`, and `recommendedFollowUp` (all required strings).
     - Connect its output to the `ai_outputParser` input of the AI Agent.

4. **Add Output Shaping Node**
   - Add an **Edit Fields (Set)** node named `Shape Output`.
   - Connect `AI Agent - Summarize & Flag Compliance Risk` to `Shape Output`.
   - Add string assignments:
     - `clientName`: `={{ $('Normalize Input').item.json.clientName }}`
     - `advisorName`: `={{ $('Normalize Input').item.json.advisorName }}`
     - `advisorEmail`: `={{ $('Normalize Input').item.json.advisorEmail }}`
     - `complianceRisk`: `={{ $json.output.complianceRisk }}`
     - `summary`: `={{ $json.output.summary }}`
     - `flaggedIssues`: `={{ $json.output.flaggedIssues }}`
     - `recommendedFollowUp`: `={{ $json.output.recommendedFollowUp }}`
     - `loggedAt`: `={{ $now.toISO() }}`

5. **Configure Google Sheets Logging Node**
   - Add a **Google Sheets** node named `Log Review to Compliance Tracker`.
   - Set **Operation** to `Append`.
   - Configure **Document ID** to your Google Sheet ID (`REPLACE_WITH_YOUR_SHEET_ID`) and **Sheet Name** to `Portfolio Reviews`.
   - Map columns (`summary`, `loggedAt`, `clientName`, `advisorName`, `flaggedIssues`, `complianceRisk`, `recommendedFollowUp`) to their respective JSON variables.
   - Set **Error Handling** to `Continue Regular Output`.
   - Connect `Shape Output` to `Log Review to Compliance Tracker`.
   - Configure Google Sheets OAuth2 credentials.

6. **Configure Risk Routing Switch Node**
   - Add a **Switch** node named `Route by Compliance Risk`.
   - Set rules based on `={{ $json.complianceRisk }}`:
     - Output 0 (`high`): equals `high`
     - Output 1 (`medium`): equals `medium`
     - Output 2 (`low`): equals `low`
   - Set **Fallback Output** to `none`.
   - Connect `Log Review to Compliance Tracker` to `Route by Compliance Risk`.

7. **Configure Notification Nodes**
   - **Alert Compliance Officer (Gmail):**
     - Add a **Gmail** node named `Alert Compliance Officer`.
     - Set **Send To** to `={{ $json.advisorEmail }}`.
     - Set **Subject** to `=⚠️ Compliance review required: {{ $json.clientName }}`.
     - Set **Message** to `=Flagged issues: {{ $json.flaggedIssues }}<br><br>Summary: {{ $json.summary }}<br><br>Recommended follow-up: {{ $json.recommendedFollowUp }}`.
     - Configure Gmail OAuth2 credentials and set error handling to `Continue Regular Output`.
     - Connect output 0 (`high`) of `Route by Compliance Risk` to this node.
   - **Send Advisor Summary Email (Gmail):**
     - Add a **Gmail** node named `Send Advisor Summary Email`.
     - Set **Send To** to `={{ $json.advisorEmail }}`.
     - Set **Subject** to `=Review summary logged: {{ $json.clientName }}`.
     - Set **Message** to `=Summary: {{ $json.summary }}<br><br>Recommended follow-up: {{ $json.recommendedFollowUp }}`.
     - Configure Gmail OAuth2 credentials and set error handling to `Continue Regular Output`.
     - Connect outputs 1 and 2 (`medium` and `low`) of `Route by Compliance Risk` to this node.
   - **Alert Compliance Team (High Risk) (Telegram):**
     - Add a **Telegram** node named `Alert Compliance Team (High Risk)`.
     - Set **Chat ID** to your Telegram chat/channel ID (`REPLACE_WITH_YOUR_CHAT_ID`).
     - Set **Text** to `=⚠️ *High compliance risk*: {{ $json.clientName }} (advisor: {{ $json.advisorName }})\nFlags: {{ $json.flaggedIssues }}\nFollow-up: {{ $json.recommendedFollowUp }}`.
     - Set **Parse Mode** to `Markdown`.
     - Configure Telegram credentials and set error handling to `Continue Regular Output`.
     - Connect `Alert Compliance Officer` main output to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Creator & Credits | Swapnil AI Labs — swapnil.mandloi7@gmail.com — [swapnilailabs.netlify.app](https://swapnilailabs.netlify.app) |
| Workflow Purpose | Designed for RIAs and wealth-management firms to automate compliance auditing and risk flagging on portfolio review meeting notes. |
Triage and draft PII-safe support replies with OpenAI and Enterprise PII Guardrails

https://n8nworkflows.xyz/workflows/triage-and-draft-pii-safe-support-replies-with-openai-and-enterprise-pii-guardrails-20870


# Triage and draft PII-safe support replies with OpenAI and Enterprise PII Guardrails

### 1. Workflow Overview

This workflow automates customer support ticket intake, triage, and response drafting while strictly preventing the exposure of Personally Identifiable Information (PII) to external AI models. It accepts incoming support inquiries via a webhook or manual trigger, masks sensitive data into surrogate tokens using Enterprise PII Guardrails, classifies and drafts a response via OpenAI (`gpt-4o-mini`) using exact token preservation rules, unmasks the final text to restore original customer details, and packages a structured payload for CRM dispatch and compliance auditing.

The workflow logic is grouped into the following functional blocks:
- **1.1 Input Reception & Normalization:** Ingests raw ticket payloads from live webhook calls or manual testing and standardizes fields (ID, channel, customer details, subject, and body).
- **1.2 PII Masking:** Detects and redacts sensitive customer data from the ticket body, generating token-to-value mapping dictionaries.
- **1.3 AI Triage & Drafting:** Processes the sanitized text with OpenAI to evaluate priority, category, and sentiment while maintaining strict surrogate token structures.
- **1.4 PII Restoration & CRM Packaging:** Unmasks the AI-generated draft using the original mapping data and formats a final audit-ready payload for CRM routing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** Captures inbound ticket events from external systems or manual triggers and normalizes diverse data structures into a unified internal schema with default fallback values.
- **Nodes Involved:** 
  - `When Manually Executed`
  - `Receive Support Ticket Webhook`
  - `Prepare Ticket Data`
- **Node Details:**
  - **When Manually Executed**
    - *Type & Role:* `n8n-nodes-base.manualTrigger` — Initiates testing executions manually within the n8n editor.
    - *Configuration:* Default settings.
    - *Expressions:* None.
    - *Connections:* Output connects to `Prepare Ticket Data`.
    - *Failure Modes:* None.
  - **Receive Support Ticket Webhook**
    - *Type & Role:* `n8n-nodes-base.webhook` — Listens for external POST requests carrying customer support tickets.
    - *Configuration:* HTTP Method set to `POST`, path configured as `support-ticket-inbound`.
    - *Expressions:* None.
    - *Connections:* Output connects to `Prepare Ticket Data`.
    - *Failure Modes:* Webhook endpoint routing errors, invalid payload formatting, or missing authentication/headers if secured externally.
  - **Prepare Ticket Data**
    - *Type & Role:* `n8n-nodes-base.set` — Normalizes incoming fields from either the manual trigger or webhook body into structured properties (`ticket_id`, `channel`, `customer_name`, `customer_email`, `ticket_subject`, `ticket_body`).
    - *Configuration:* Assigns string values using ternary evaluation.
    - *Key Expressions:* `={{ $json.body?.ticket_id || $json.ticket_id || 'TICK-94281' }}` (and similar fallbacks for all ticket metadata fields).
    - *Connections:* Inputs from both triggers; output connects to `Mask PII Data`.
    - *Failure Modes:* Expression evaluation errors if nested webhook body properties are missing entirely without fallback defaults.

#### 2.2 PII Masking
- **Overview:** Evaluates the standardized ticket text to detect and replace sensitive information (such as names, emails, financial accounts, and medical records) with encrypted surrogate tokens.
- **Nodes Involved:**
  - `Mask PII Data`
- **Node Details:**
  - **Mask PII Data**
    - *Type & Role:* `n8n-nodes-piiguardrails.piiGuardrails` — Interacts with Enterprise PII Guardrails to tokenize sensitive data.
    - *Configuration:* Uses Enterprise PII Guardrails account credentials.
    - *Key Expressions:* `={{ $json.ticket_body }}` passed to the text parameter.
    - *Connections:* Input from `Prepare Ticket Data`; output connects to `OpenAI Triage & Response Drafting`.
    - *Failure Modes:* API authentication failures, network timeouts communicating with the PII Guardrails service, or malformed input text.

#### 2.3 AI Triage & Drafting
- **Overview:** Sends the sanitized, tokenized ticket text to an LLM to evaluate support metrics and draft an empathetic response while strictly preserving surrogate tokens.
- **Nodes Involved:**
  - `OpenAI Triage & Response Drafting`
- **Node Details:**
  - **OpenAI Triage & Response Drafting**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.openAi` — Generates text completions utilizing the OpenAI chat model.
    - *Configuration:* Model set to `gpt-4o-mini`. System instructions enforce classification (Priority, Category, Sentiment), drafting guidelines, and strict verbatim preservation rules for surrogate tokens enclosed in angle brackets.
    - *Key Expressions:* `={{ $json.masked_text }}` assigned as the user message content.
    - *Connections:* Input from `Mask PII Data`; output connects to `Restore PII Data`.
    - *Failure Modes:* OpenAI API rate limits, invalid API credentials, quota exhaustion, or model refusal due to prompt constraints.

#### 2.4 PII Restoration & CRM Packaging
- **Overview:** Replaces surrogate tokens in the AI-generated draft with original sensitive values using the mapping dictionary, then aggregates all ticket metadata, protection metrics, and unmasked text into a final CRM-ready payload.
- **Nodes Involved:**
  - `Restore PII Data`
  - `Build CRM Dispatch Data`
- **Node Details:**
  - **Restore PII Data**
    - *Type & Role:* `n8n-nodes-piiguardrails.piiGuardrails` — Reverts surrogate tokens back to original plaintext data.
    - *Configuration:* Operation set to `unmask`. Uses Enterprise PII Guardrails account credentials.
    - *Key Expressions:* 
      - Mapping: `={{ $('Mask PII Data').item.json.mapping }}`
      - Unmask Text: `={{ $json.message?.content || $json.output || $json.text || $json.response }}`
    - *Connections:* Input from `OpenAI Triage & Response Drafting`; output connects to `Build CRM Dispatch Data`.
    - *Failure Modes:* Missing mapping references from earlier nodes or corrupted token formats in the AI output.
  - **Build CRM Dispatch Data**
    - *Type & Role:* `n8n-nodes-base.set` — Structures final outputs for downstream CRM routing and audit logging.
    - *Configuration:* Assigns properties for ticket identifiers, customer details, subject, privacy verification metadata, calculated entity counts, and the final resolution draft.
    - *Key Expressions:* 
      - `pii_entities_protected_count`: `={{ Object.keys($('Mask PII Data').item.json.mapping || {}).length }}`
      - `final_resolution_draft`: `={{ $json.unmasked_text || $json.text }}`
      - References source data via `$('Prepare Ticket Data').item.json`.
    - *Connections:* Input from `Restore PII Data`; terminal workflow node.
    - *Failure Modes:* Broken node reference paths if preceding node names are modified.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow overview, setup instructions, and customization guidelines. | None | None | ## 🛡️ Zero-Leak AI Customer Support: Auto-Triage & Reply with PII Guardrails... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Describes the inbound ticket intake process. | None | None | ## Inbound ticket intake<br><br>Receives test or live customer support tickets and standardizes the incoming data into a customer ticket payload. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Describes the PII masking security layer. | None | None | ## Mask sensitive data<br><br>Applies PII guardrails to redact or tokenize sensitive customer information before the ticket content is sent to the AI model. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Describes the AI triage and drafting phase. | None | None | ## AI triage drafting<br><br>Uses OpenAI to classify or triage the sanitized support request and draft an appropriate customer support response. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Describes PII restoration and final CRM payload formatting. | None | None | ## Restore and format output<br><br>Restores protected PII after AI processing and formats the final ticket response, CRM dispatch fields, and audit trail payload. |
| When Manually Executed | n8n-nodes-base.manualTrigger | Initiates manual test runs. | None | Prepare Ticket Data | Inbound ticket intake |
| Receive Support Ticket Webhook | n8n-nodes-base.webhook | Receives inbound HTTP POST webhook requests with support tickets. | None | Prepare Ticket Data | Inbound ticket intake |
| Prepare Ticket Data | n8n-nodes-base.set | Normalizes incoming ticket fields into a consistent schema with fallback values. | When Manually Executed, Receive Support Ticket Webhook | Mask PII Data | Inbound ticket intake |
| Mask PII Data | n8n-nodes-piiguardrails.piiGuardrails | Tokenizes sensitive customer PII using guardrails. | Prepare Ticket Data | OpenAI Triage & Response Drafting | Mask sensitive data |
| OpenAI Triage & Response Drafting | @n8n/n8n-nodes-langchain.openAi | Classifies ticket attributes and drafts responses using `gpt-4o-mini`. | Mask PII Data | Restore PII Data | AI triage drafting |
| Restore PII Data | n8n-nodes-piiguardrails.piiGuardrails | Unmasks surrogate tokens in the AI-generated draft using mapping data. | OpenAI Triage & Response Drafting | Build CRM Dispatch Data | Restore and format output |
| Build CRM Dispatch Data | n8n-nodes-base.set | Packages unmasked resolution text and privacy audit metadata for CRM integration. | Restore PII Data | None | Restore and format output |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`).
   - Add a **Webhook** node (`n8n-nodes-base.webhook`), set HTTP Method to `POST`, and configure the path to `support-ticket-inbound`.
2. **Add Data Preparation Node:**
   - Add a **Set** node named `Prepare Ticket Data`.
   - Connect both `When Manually Executed` and `Receive Support Ticket Webhook` to this node.
   - Configure string assignments with fallbacks for `ticket_id`, `channel`, `customer_name`, `customer_email`, `ticket_subject`, and `ticket_body` (referencing sample ticket data containing mock PII such as names, emails, phone numbers, and financial details).
3. **Configure PII Masking:**
   - Add an **Enterprise PII Guardrails** node (`n8n-nodes-piiguardrails.piiGuardrails`) named `Mask PII Data`.
   - Connect `Prepare Ticket Data` to `Mask PII Data`.
   - Set the parameter `text` to `={{ $json.ticket_body }}`.
   - Configure the Enterprise PII Guardrails API credential.
4. **Configure AI Processing:**
   - Add an **OpenAI Chat Model** node (`@n8n/n8n-nodes-langchain.openAi`) named `OpenAI Triage & Response Drafting`.
   - Connect `Mask PII Data` to this node.
   - Select model `gpt-4o-mini`.
   - Set the system prompt to instruct the model on triage classification (Priority, Category, Sentiment), empathetic drafting, and strict verbatim preservation rules for surrogate tokens enclosed in angle brackets.
   - Set the user message content to `={{ $json.masked_text }}`.
   - Configure your OpenAI API credential.
5. **Configure PII Restoration:**
   - Add a second **Enterprise PII Guardrails** node (`n8n-nodes-piiguardrails.piiGuardrails`) named `Restore PII Data`.
   - Connect `OpenAI Triage & Response Drafting` to this node.
   - Set operation to `unmask`.
   - Set parameter `mapping` to `={{ $('Mask PII Data').item.json.mapping }}`.
   - Set parameter `unmaskText` to `={{ $json.message?.content || $json.output || $json.text || $json.response }}`.
   - Reuse your Enterprise PII Guardrails API credential.
6. **Configure CRM Payload Formatting:**
   - Add a final **Set** node (`n8n-nodes-base.set`) named `Build CRM Dispatch Data`.
   - Connect `Restore PII Data` to this node.
   - Configure assignments to map `ticket_id`, `customer_name`, `customer_email`, `subject`, `pii_entities_protected_count` (using `={{ Object.keys($('Mask PII Data').item.json.mapping || {}).length }}`), `privacy_status`, and `final_resolution_draft` (using `={{ $json.unmasked_text || $json.text }}`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Zero-Leak AI Customer Support: Auto-Triage & Reply with PII Guardrails | Core workflow template focusing on secure AI customer interactions and enterprise privacy compliance. |
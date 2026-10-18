Generate branded proposal PDFs from briefs with Claude, ShipPDF and Gmail

https://n8nworkflows.xyz/workflows/generate-branded-proposal-pdfs-from-briefs-with-claude--shippdf-and-gmail-21000


# Generate branded proposal PDFs from briefs with Claude, ShipPDF and Gmail

### 1. Workflow Overview

This workflow automates the creation, generation, delivery, and archiving of branded business proposals. It accepts raw project briefs submitted via an interactive n8n form, uses an AI model to structure the input data, requests a generated PDF from the ShipPDF API via polling, and handles successful delivery or error routing accordingly.

The logic is organized into the following functional blocks:
- **1.1 Input Reception:** Captures client details, recipient email, and project brief text through an n8n form trigger.
- **1.2 AI Processing:** Leverages Anthropic Claude to translate unstructured text briefs into structured proposal metadata (titles, scope, pricing, etc.).
- **1.3 Generation and Polling:** Submits the structured data to ShipPDF, waits briefly, and polls the generation status until the PDF is fully rendered.
- **1.4 Result Verification:** Checks whether the poll returned a completed status or requires error branching.
- **1.5 Delivery and Archiving:** Downloads the generated PDF, emails it directly to the client via Gmail, and archives a copy to Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Initializes the execution when a user submits a proposal request via a web form.
- **Nodes Involved:** `When Proposal Brief Submitted`
- **Node Details:**
  - **When Proposal Brief Submitted**
    - Type: `n8n-nodes-base.formTrigger` (v2.2)
    - Technical Role: Webhook-powered form entry point collecting user inputs.
    - Configuration: Configured fields include "Client or company", "Recipient email" (type: email), and "Project brief" (type: textarea).
    - Key Expressions / Variables: N/A (Triggers execution).
    - Input / Output: Input: None (Trigger) | Output: JSON payload containing submitted form field values.
    - Edge Cases / Failure Types: Form submission failures due to invalid email formats or missing required fields.

#### 2.2 AI Processing
- **Overview:** Connects an AI language model to extract and organize free-form text from the brief into structured schema fields.
- **Nodes Involved:** `Claude Sonnet Model`, `Extract Information from Brief`
- **Node Details:**
  - **Claude Sonnet Model**
    - Type: `@n8n/n8n-nodes-langchain.lmChatAnthropic` (v1.3)
    - Technical Role: Provides LLM chat capabilities utilizing Anthropic Claude.
    - Configuration: Model set to `claude-sonnet-4-6`.
    - Key Expressions / Variables: API credentials linked via Anthropic API.
    - Input / Output: Input: None | Output: AI model reference passed to the Information Extractor.
  - **Extract Information from Brief**
    - Type: `@n8n/n8n-nodes-langchain.informationExtractor` (v1.1)
    - Technical Role: Parses text input into defined JSON schema attributes.
    - Configuration: Extracts fields: `proposal_title`, `client_name`, `executive_summary`, `scope_of_work`, `timeline`, and `price`.
    - Key Expressions / Variables: `={{ $json['Project brief'] }}`
    - Input / Output: Input: Form submission JSON | Output: Structured JSON payload under the `output` property.
    - Edge Cases / Failure Types: API rate limits, model output hallucinations, or failure to parse required schema variables.

#### 2.3 Generation and Polling
- **Overview:** Submits the structured proposal attributes to the ShipPDF API and loops via a wait-and-poll mechanism until the document compilation finishes.
- **Nodes Involved:** `Post Document to ShipPDF`, `Wait 3s Before Polling`, `Poll ShipPDF for Result`
- **Node Details:**
  - **Post Document to ShipPDF**
    - Type: `n8n-nodes-base.httpRequest` (v4.2)
    - Technical Role: Initiates document generation via HTTP POST.
    - Configuration: POST request to `https://api.shippdf.com/v1/documents` using Bearer token authentication. Sends a JSON body mapping extracted attributes to the ShipPDF template.
    - Key Expressions / Variables: Uses `{{ $json.output.proposal_title }}` etc., and sets an idempotency header `={{ $execution.id }}`.
    - Input / Output: Input: Extracted AI fields | Output: ShipPDF response containing document ID and initial status.
    - Edge Cases / Failure Types: HTTP 422 errors if data fails template schema validation; authentication issues with the Bearer token.
  - **Wait 3s Before Polling**
    - Type: `n8n-nodes-base.wait` (v1.1)
    - Technical Role: Introduces a delay to allow the rendering service to process the document before polling.
    - Configuration: Amount: 3 seconds.
    - Input / Output: Input: Document generation response or pending status loop | Output: Passes execution forward after the timer.
  - **Poll ShipPDF for Result**
    - Type: `n8n-nodes-base.httpRequest` (v4.2)
    - Technical Role: Retrieves the current generation status of the document via HTTP GET.
    - Configuration: GET request to `https://api.shippdf.com/v1/documents/{{ $json.id }}` with Bearer auth.
    - Input / Output: Input: Wait node | Output: JSON payload containing the current document status (`completed`, `failed`, or `processing`).
    - Edge Cases / Failure Types: Network timeouts or invalid document IDs.

#### 2.4 Result Verification
- **Overview:** Evaluates the current status of the document generation to route successful documents to delivery or failed runs to alerting.
- **Nodes Involved:** `If PDF Generation Success`, `If Generation Failed`, `Alert on PDF Generation Failure`
- **Node Details:**
  - **If PDF Generation Success**
    - Type: `n8n-nodes-base.if` (v2)
    - Technical Role: Branches execution based on whether the generation status equals `completed`.
    - Configuration: Condition checks if `{{ $json.status }}` equals `completed`.
    - Input / Output: Input: Poll result | Output: True branch (to download) / False branch (to failure check).
  - **If Generation Failed**
    - Type: `n8n-nodes-base.if` (v2)
    - Technical Role: Checks if the non-completed generation status is explicitly marked as `failed`, looping back to wait if still processing.
    - Configuration: Condition checks if `{{ $json.status }}` equals `failed`.
    - Input / Output: Input: False branch of previous IF | Output: True branch (Alert placeholder) / False branch (Loops back to `Wait 3s Before Polling`).
  - **Alert on PDF Generation Failure**
    - Type: `n8n-nodes-base.noOp` (v1)
    - Technical Role: Placeholder node for routing failed document statuses to alerting integrations.
    - Input / Output: Input: Failed status verification | Output: Terminates branch.

#### 2.5 Delivery and Archiving
- **Overview:** Downloads the rendered PDF file, dispatches it via email to the client, and uploads a copy to cloud storage.
- **Nodes Involved:** `Fetch Generated PDF`, `Send Proposal via Email`, `Save Proposal to Drive`
- **Node Details:**
  - **Fetch Generated PDF**
    - Type: `n8n-nodes-base.httpRequest` (v4.2)
    - Technical Role: Downloads the binary PDF file from the remote URL.
    - Configuration: GET request to `={{ $json.pdf_url }}` with response format configured to save as a binary file in the `data` property.
    - Input / Output: Input: Completed status JSON | Output: Binary file data.
    - Edge Cases / Failure Types: Expired URLs or storage retrieval errors.
  - **Send Proposal via Email**
    - Type: `n8n-nodes-base.gmail` (v2.1)
    - Technical Role: Sends an email containing the downloaded PDF as an attachment.
    - Configuration: Resource: `message`, Operation: `send`. Uses recipient email from the trigger form and attaches binary property `data`.
    - Key Expressions / Variables: `={{ $('When Proposal Brief Submitted').item.json['Recipient email'] }}`, subject references proposal title.
    - Input / Output: Input: Binary PDF file | Output: Sent email confirmation.
    - Edge Cases / Failure Types: Gmail OAuth token expiration, attachment size limits, or blocked recipient addresses.
  - **Save Proposal to Drive**
    - Type: `n8n-nodes-base.googleDrive` (v3)
    - Technical Role: Archives the generated proposal PDF in Google Drive.
    - Configuration: Resource: `file`, Operation: `upload`. Uses input data field name `data` targeting Google Drive root.
    - Key Expressions / Variables: File naming expression: `=Proposal - {{ $('Extract Information from Brief').item.json.output.client_name }}.pdf`
    - Input / Output: Input: Binary PDF file | Output: Drive file metadata confirmation.
    - Edge Cases / Failure Types: Insufficient storage space, permission errors on the target folder.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | `stickyNote` | Documentation note describing the ShipPDF workflow. | *None* | *None* | # 🚢 ShipPDF — AI proposals on autopilot\n\n![ShipPDF](https://shippdf.com/assets/boat-mascot.png)\n\nGenerate branded proposal PDFs from a brief with AI + ShipPDF.\n\n### How it works\n\nA proposal brief submitted through the form is structured by an Anthropic-powered Information Extractor, then sent to ShipPDF to render a branded PDF from your own template. The workflow **polls** ShipPDF every few seconds until generation finishes, then branches on the result: a completed PDF is downloaded, emailed via Gmail, and archived to Google Drive; a failed one is routed to a placeholder alert. Polling means it runs on any n8n — Cloud or self-hosted — with no public callback URL.\n\n### Setup steps\n\n- Add a **ShipPDF API** credential (HTTP Bearer Auth, your key with `documents:create` + `documents:read`) and select it on both **Post Document to ShipPDF** and **Poll ShipPDF for Result**.\n- Add **Anthropic** credentials for the chat model and verify the extraction attributes match your ShipPDF template's data schema.\n- In **Post Document to ShipPDF**, set your `template_id` and map the `data` fields.\n- Connect **Gmail** and **Google Drive** credentials and set the recipient, subject, body, and archive folder.\n- Replace the failure placeholder with a real alert (Slack, email, ticket).\n\n### Notes\n\n- **Credits:** one ShipPDF credit per generated PDF. The `Idempotency-Key` **header** makes retries safe — a replay returns the original document, no double charge.\n- **Any model:** ships with Claude Sonnet 4.6; swap the chat model for ChatGPT (OpenAI), Gemini, Mistral, or local Ollama — the Information Extractor accepts any.\n- **Failures:** invalid data is rejected at the generate step with HTTP 422 before any credit; render/billing failures arrive as `status: failed` on the poll.\n\n### Customization\n\nCustomize the extraction schema, ShipPDF template/branding, email copy, Drive folder, and failure handling to fit different proposal types or sales processes.\n\nFree ShipPDF API key → https://shippdf.com |
| Section: capture | `stickyNote` | Visual grouping note for the capture block. | *None* | *None* | ## Capture proposal brief\n\nStarts the workflow when a user submits a new proposal brief through the form. |
| Section: structure | `stickyNote` | Visual grouping note for the structure block. | *None* | *None* | ## Structure brief with AI\n\nUses the Information Extractor and the nearby Anthropic chat model to convert the submitted brief into structured proposal data. |
| Section: generate-poll | `stickyNote` | Visual grouping note for generation and polling. | *None* | *None* | ## Generate and poll for PDF\n\nSends the structured data to ShipPDF, then polls `GET /v1/documents/{id}` every 3s until the PDF is ready. Anything still processing loops back to the wait. No public callback URL needed. |
| Section: check | `stickyNote` | Visual grouping note for result checking. | *None* | *None* | ## Check generation result\n\nRoutes a completed document to delivery and a failed one to the alert placeholder. |
| Section: deliver | `stickyNote` | Visual grouping note for delivery and archiving. | *None* | *None* | ## Deliver and archive proposal\n\nDownloads the completed PDF, emails it to the recipient, and stores a copy in Google Drive. |
| When Proposal Brief Submitted | `formTrigger` | Triggers workflow on form submit. | *None* | Extract Information from Brief | |
| Claude Sonnet Model | `lmChatAnthropic` | Provides Claude LLM backend. | *None* | Extract Information from Brief | |
| Extract Information from Brief | `informationExtractor` | Extracts structured fields from the brief. | When Proposal Brief Submitted, Claude Sonnet Model | Post Document to ShipPDF | |
| Post Document to ShipPDF | `httpRequest` | Submits proposal payload to ShipPDF API. | Extract Information from Brief | Wait 3s Before Polling | |
| Wait 3s Before Polling | `wait` | Delays execution before polling. | Post Document to ShipPDF, If Generation Failed | Poll ShipPDF for Result | |
| Poll ShipPDF for Result | `httpRequest` | Queries ShipPDF API for document status. | Wait 3s Before Polling | If PDF Generation Success | |
| If PDF Generation Success | `if` | Routes execution if document is completed. | Poll ShipPDF for Result | Fetch Generated PDF, If Generation Failed | |
| If Generation Failed | `if` | Routes execution if generation failed. | If PDF Generation Success | Alert on PDF Generation Failure, Wait 3s Before Polling | |
| Fetch Generated PDF | `httpRequest` | Downloads binary PDF file. | If PDF Generation Success | Send Proposal via Email | |
| Send Proposal via Email | `gmail` | Emails PDF attachment to recipient. | Fetch Generated PDF | Save Proposal to Drive | |
| Save Proposal to Drive | `googleDrive` | Uploads PDF copy to Google Drive. | Send Proposal via Email | *None* | |
| Alert on PDF Generation Failure | `noOp` | Placeholder for error/failure handling. | If Generation Failed | *None* | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**
   - Add a **Form Trigger** (`n8n-nodes-base.formTrigger`) named `When Proposal Brief Submitted`.
   - Set the form title to `Request a proposal` and description.
   - Add form fields: `Client or company` (Text, required), `Recipient email` (Email, required), and `Project brief` (Textarea, required).

2. **Add AI Model & Extractor Nodes:**
   - Add an **Anthropic Chat Model** (`@n8n/n8n-nodes-langchain.lmChatAnthropic`) named `Claude Sonnet Model`. Set model to `claude-sonnet-4-6`. Configure your Anthropic API credentials.
   - Add an **Information Extractor** (`@n8n/n8n-nodes-langchain.informationExtractor`) named `Extract Information from Brief`.
   - Set the text parameter to `={{ $json['Project brief'] }}` and link its AI model input to `Claude Sonnet Model`.
   - Configure extraction attributes: `proposal_title`, `client_name`, `executive_summary`, `scope_of_work`, `timeline`, and `price`.
   - Connect `When Proposal Brief Submitted` main output to `Extract Information from Brief`.

3. **Configure ShipPDF Generation and Polling:**
   - Add an **HTTP Request** node named `Post Document to ShipPDF`.
   - Set method to `POST`, URL to `https://api.shippdf.com/v1/documents`, authentication to `Generic Credential Type` -> `HTTP Bearer Auth` (with your ShipPDF API key).
   - Set body content type to `JSON` using the mapping expression for `template_id` and `data` objects (mapping `proposal_title`, `client_name`, `executive_summary`, `scope_of_work`, `timeline`, and `price` from `$json.output`).
   - Add a header `Idempotency-Key` with value `={{ $execution.id }}`.
   - Connect `Extract Information from Brief` to `Post Document to ShipPDF`.
   - Add a **Wait** node named `Wait 3s Before Polling` set to 3 seconds (`timeInterval` resume). Connect `Post Document to ShipPDF` to this node.
   - Add an **HTTP Request** node named `Poll ShipPDF for Result`. Set method to `GET`, URL to `=https://api.shippdf.com/v1/documents/{{ $json.id }}`, and configure Bearer Auth. Connect `Wait 3s Before Polling` to this node.

4. **Setup Validation Branching:**
   - Add an **If** node named `If PDF Generation Success`. Set condition to check if `={{ $json.status }}` equals `completed`. Connect `Poll ShipPDF for Result` to it.
   - Add a second **If** node named `If Generation Failed`. Set condition to check if `={{ $json.status }}` equals `failed`. Connect the false output of `If PDF Generation Success` to this node.
   - Connect the true output of `If Generation Failed` to a **No-Op** node named `Alert on PDF Generation Failure`.
   - Connect the false output of `If Generation Failed` back to `Wait 3s Before Polling` to continue the poll loop.

5. **Configure Delivery and Archiving:**
   - Add an **HTTP Request** node named `Fetch Generated PDF`. Set method to `GET`, URL to `={{ $json.pdf_url }}`, and response format to `File` (output property: `data`). Connect the true output of `If PDF Generation Success` to this node.
   - Add a **Gmail** node named `Send Proposal via Email`. Set resource to `message`, operation to `send`. Map recipient to `={{ $('When Proposal Brief Submitted').item.json['Recipient email'] }}`, subject to the proposal title expression, and include binary attachment property `data`. Configure your Gmail credentials. Connect `Fetch Generated PDF` to this node.
   - Add a **Google Drive** node named `Save Proposal to Drive`. Set resource to `file`, operation to `upload`, input data field name to `data`, target Drive to `My Drive`, and folder to `root` (or target folder). Configure Google Drive credentials. Set file name expression to `=Proposal - {{ $('Extract Information from Brief').item.json.output.client_name }}.pdf`. Connect `Send Proposal via Email` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| ShipPDF Platform & Free API Key | https://shippdf.com |
| ShipPDF Mascot Asset | https://shippdf.com/assets/boat-mascot.png |
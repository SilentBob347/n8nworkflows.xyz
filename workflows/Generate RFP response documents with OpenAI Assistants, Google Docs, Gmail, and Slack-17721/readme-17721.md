Generate RFP response documents with OpenAI Assistants, Google Docs, Gmail, and Slack

https://n8nworkflows.xyz/workflows/generate-rfp-response-documents-with-openai-assistants--google-docs--gmail--and-slack-17721


# Generate RFP response documents with OpenAI Assistants, Google Docs, Gmail, and Slack

### 1. Workflow Overview

This workflow automates the handling of incoming Requests for Proposal (RFPs). It accepts an RFP PDF via an HTTP webhook, extracts the document's raw text, identifies supplier-directed questions using OpenAI, answers each question through a custom OpenAI Assistant, logs the complete Q&A into a new Google Document, and notifies stakeholders via Gmail and Slack.

The logic is organized into four functional blocks:
- **1.1 Intake & Document Setup:** Receives the webhook payload containing the PDF, extracts the raw text, normalizes variables, creates a destination Google Doc, and writes an introductory metadata block.
- **1.2 Question Extraction:** Passes the raw RFP text to an OpenAI language model chained with an Item List Output Parser to isolate and structure all supplier-directed questions.
- **1.3 Answer Loop:** Iterates through the extracted questions sequentially, queries a specialized OpenAI Assistant for answers, and appends each Q&A pair into the Google Doc.
- **1.4 Completion & Notifications:** Sends a confirmation email to the requester via Gmail and broadcasts a completion update to a designated Slack channel once all processing finishes.

---

### 2. Block-by-Block Analysis

#### 2.1 Intake & Document Setup
This block ingests the incoming request, parses the binary PDF data, prepares internal variables, initializes a Google Doc, and populates its initial metadata header.

*   **Wait for Request**
    *   **Type & Role:** `n8n-nodes-base.webhook` (Trigger) — Listens for incoming HTTP POST requests containing the RFP binary data and metadata.
    *   **Configuration:** Configured to accept HTTP method `POST` on a fixed webhook path.
    *   **Key Expressions:** Expects a JSON body containing `title`, `reply_to`, and binary PDF attachment data.
    *   **Connections:** Input: None (Trigger). Output: `Get RFP Data`.
    *   **Edge Cases:** Timeout or payload size limits exceeded if the PDF is exceptionally large.

*   **Get RFP Data**
    *   **Type & Role:** `n8n-nodes-base.extractFromFile` (Data Transformation) — Extracts plain text from the binary PDF file provided by the webhook.
    *   **Configuration:** Operation set to `pdf`.
    *   **Key Expressions:** Operates on default binary data input from the preceding webhook.
    *   **Connections:** Input: `Wait for Request`. Output: `Set Variables`.
    *   **Edge Cases:** Failure if the PDF is password-protected, corrupted, or consists solely of un-OCR'd scanned images.

*   **Set Variables**
    *   **Type & Role:** `n8n-nodes-base.set` (Data Transformation) — Normalizes and structures payload metadata into reusable variables.
    *   **Configuration:** Mode set to `raw` JSON output.
    *   **Key Expressions:** 
        *   `doc_title`: `{{ $('Wait for Request').item.json.body.title }}`
        *   `doc_filename`: `{{ $('Wait for Request').item.json.body.id }} | {{ $('Wait for Request').item.json.body.title }} | {{ $now.format('yyyyMMddhhmmss') }}| RFP Response`
        *   `reply_to`: `{{ $('Wait for Request').item.json.body.reply_to }}`
    *   **Connections:** Input: `Get RFP Data`. Output: `Create new RFP Response Document`.
    *   **Edge Cases:** Missing body properties in the webhook payload will result in undefined expression evaluations.

*   **Create new RFP Response Document**
    *   **Type & Role:** `n8n-nodes-base.googleDocs` (Integration) — Creates a blank Google Document within a designated Google Drive folder.
    *   **Configuration:** Title driven by expression; destination folder ID specified (`1y0I8MH32maIWCJh767mRE_NMHC6A3bUu`).
    *   **Key Expressions:** `title: "={{ $json.doc_filename }}"`
    *   **Connections:** Input: `Set Variables`. Output: `Add Metadata to Response Doc`.
    *   **Credentials:** Requires Google Docs OAuth2 / Service Account credentials.
    *   **Edge Cases:** Insufficient Google Drive permissions or invalid folder IDs.

*   **Add Metadata to Response Doc**
    *   **Type & Role:** `n8n-nodes-base.googleDocs` (Integration) — Inserts an execution metadata header into the newly created Google Doc.
    *   **Configuration:** Operation set to `update` using actions UI, appending text to the document body.
    *   **Key Expressions:** Inserts document title, generation timestamp, requester email, and the n8n execution URL.
    *   **Connections:** Input: `Create new RFP Response Document`. Output: `Extract Questions From RFP`.
    *   **Credentials:** Requires Google Docs OAuth2 / Service Account credentials.

---

#### 2.2 Question Extraction
This block analyzes the full text of the RFP via OpenAI to extract precise questions intended for the supplier.

*   **OpenAI Chat Model**
    *   **Type & Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model Sub-node) — Provides the base LLM for the question-extraction chain.
    *   **Configuration:** Default options.
    *   **Connections:** Output (`ai_languageModel`): Connected to `Extract Questions From RFP`.
    *   **Credentials:** Requires OpenAI API credentials.

*   **Item List Output Parser**
    *   **Type & Role:** `@n8n/n8n-nodes-langchain.outputParserItemList` (AI Parser Sub-node) — Enforces a structured list output from the LLM.
    *   **Configuration:** Default options.
    *   **Connections:** Output (`ai_outputParser`): Connected to `Extract Questions From RFP`.

*   **Extract Questions From RFP**
    *   **Type & Role:** `@n8n/n8n-nodes-langchain.chainLlm` (AI / Advanced AI) — Prompts the LLM to parse out supplier-directed questions verbatim from the RFP text.
    *   **Configuration:** Prompt type defined inline with strict extraction instructions.
    *   **Key Expressions:** `<RFP>{{ $('Get RFP Data').item.json.text }}<RFP>`
    *   **Connections:** Input: `Add Metadata to Response Doc` (main), plus AI Model and Output Parser sub-nodes. Output: `For Each Question...`.
    *   **Edge Cases:** LLM returning malformed output or failing to parse questions if the document format is highly non-standard.

---

#### 2.3 Answer Loop
This block processes each extracted question individually, queries an OpenAI Assistant, and logs results sequentially.

*   **For Each Question...**
    *   **Type & Role:** `n8n-nodes-base.splitInBatches` (Flow Control) — Iterates through the list of extracted questions one item at a time.
    *   **Configuration:** Default batch iteration settings.
    *   **Connections:** Input: `Extract Questions From RFP`. Output 0 (Loop Complete): `Send Email Notification`. Output 1 (Loop Active): `Answer Question with Context`.
    *   **Edge Cases:** Infinite loops if reset parameters are misconfigured.

*   **Answer Question with Context**
    *   **Type & Role:** `@n8n/n8n-nodes-langchain.openAi` (AI / Integration) — Interacts with an OpenAI Assistant pre-trained on company knowledge to answer specific RFP questions.
    *   **Configuration:** Resource set to `assistant`. Assistant ID explicitly set to `asst_QBI5lLKOsjktr3DRB4MwrgZd` ("Nexus Digital Solutions Bot").
    *   **Key Expressions:** `text: "={{ $json.response.text }}"`
    *   **Connections:** Input: `For Each Question...`. Output: `Record Question & Answer in Response Doc`.
    *   **Credentials:** Requires OpenAI API credentials.
    *   **Edge Cases:** Assistant timeout, rate-limiting, or missing Assistant permissions/availability.

*   **Record Question & Answer in Response Doc**
    *   **Type & Role:** `n8n-nodes-base.googleDocs` (Integration) — Appends the numbered question and its corresponding AI-generated answer to the Google Document.
    *   **Configuration:** Operation set to `update` using actions UI, referencing the dynamic document ID from the initialization step.
    *   **Key Expressions:** `text: "={{ $runIndex+1 }}. {{ $json.content }}\n{{ $json.output }}\n\n"`
    *   **Connections:** Input: `Answer Question with Context`. Output: Loops back to `For Each Question...`.
    *   **Credentials:** Requires Google Docs OAuth2 / Service Account credentials.

---

#### 2.4 Completion & Notifications
This block dispatches confirmation alerts via email and chat once the document compilation finishes.

*   **Send Email Notification**
    *   **Type & Role:** `n8n-nodes-base.gmail` (Integration) — Emails the request submitter confirming that the RFP response document is ready.
    *   **Configuration:** Email type set to `text`. Subject and message body reference the document title and requester variables.
    *   **Key Expressions:** 
        *   `sendTo`: `={{ $('Set Variables').item.json.reply_to }}`
        *   `subject`: `=RFP Questionnaire "{{ $('Set Variables').item.json.title }}" Completed!`
        *   `message`: `=Your RFP document "{{ $('Set Variables').item.json.title }}" is now complete!`
    *   **Connections:** Input: `For Each Question...` (Loop Complete branch). Output: `Send Chat Notification`.
    *   **Credentials:** Requires Gmail OAuth2 credentials.
    *   **Edge Cases:** Invalid reply-to email address or SMTP/API rate limits.

*   **Send Chat Notification**
    *   **Type & Role:** `n8n-nodes-base.slack` (Integration) — Posts a completion alert to a designated Slack channel.
    *   **Configuration:** Select mode set to `channel`, target channel name set to `RFP-channel`.
    *   **Key Expressions:** `text: "=RFP document \"{{ $('Set Variables').item.json.title }}\" completed!"`
    *   **Connections:** Input: `Send Email Notification`. Output: None (Terminal node).
    *   **Credentials:** Requires Slack OAuth2 API credentials.
    *   **Edge Cases:** Bot lacking permissions to post in the target channel or invalid channel names.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Wait for Request** | `n8n-nodes-base.webhook` | Receives RFP payload via webhook | None | Get RFP Data | ## Automate RFP Answers with OpenAI Assistants, Google Docs, Gmail and Slack<br><br>Automatically turns an incoming RFP PDF into a complete response document. Every extracted question is answered by a trained OpenAI Assistant, recorded into a Google Doc, and the finished document is delivered by email and Slack.<br><br>**Perfect for:** Sales engineers, bid managers, and proposal teams who respond to RFPs and RFIs on a recurring basis.<br><br>---<br><br>## How it works<br><br>1. **Wait for Request**: Receives the RFP submission (PDF, title, and reply to address) via webhook.<br>2. **Get RFP Data**: Extracts the raw text from the uploaded RFP PDF.<br>3. **Set Variables**: Builds the document title, filename, and reply to address from the webhook payload.<br>4. **Create new RFP Response Document**: Creates a new Google Doc in the target Drive folder to hold the response.<br>5. **Add Metadata to Response Doc**: Inserts a header block with title, generation date, requester, and execution link.<br>6. **Extract Questions From RFP**: Reads the full RFP text and pulls out every question addressed to the supplier, using the *OpenAI Chat Model* for reasoning and the *Item List Output Parser* to return a clean list.<br>7. **For Each Question...**: Loops through the extracted questions one at a time.<br>8. **Answer Question with Context**: Sends each question to a pretrained OpenAI Assistant ("Nexus Digital Solutions Bot") that answers using the company's own knowledge.<br>9. **Record Question & Answer in Response Doc**: Appends the numbered question and answer into the Google Doc, then loops back for the next question.<br>10. **Send Email Notification**: Once every question is answered, emails the requester that the response document is ready. *(runs on loop completion)*<br>11. **Send Chat Notification**: Posts a matching completion message to the RFP Slack channel.<br><br>---<br><br>## Setup (~10 minutes)<br><br>1. **Webhook**: Note the generated URL on the *Wait for Request* node and point your RFP intake form or system to it.<br>2. **Google Docs**: Connect your account in *Create new RFP Response Document* and *Add Metadata to Response Doc*, and set the target folder ID.<br>3. **OpenAI**: Add your API key in *OpenAI Chat Model*, and set your own Assistant ID in *Answer Question with Context*.<br>4. **Gmail**: Connect your account in *Send Email Notification*.<br>5. **Slack**: Connect your account and set the target channel in *Send Chat Notification*.<br>> The OpenAI Assistant used in *Answer Question with Context* must already be trained on your company's RFP knowledge; this workflow does not build that Assistant for you.<br><br>## 1️⃣ Intake & Document Setup<br><br>The **Wait for Request** webhook receives the incoming RFP along with its title and reply to address, and **Get RFP Data** pulls the raw text out of the attached PDF. **Set Variables** assembles the document title and filename, then **Create new RFP Response Document** and **Add Metadata to Response Doc** spin up a fresh Google Doc with a header block ready to receive answers. |
| **Get RFP Data** | `n8n-nodes-base.extractFromFile` | Extracts text from PDF binary | Wait for Request | Set Variables | (Same as above) |
| **Set Variables** | `n8n-nodes-base.set` | Formats filenames and variables | Get RFP Data | Create new RFP Response Document | (Same as above) |
| **Create new RFP Response Document** | `n8n-nodes-base.googleDocs` | Creates destination Google Doc | Set Variables | Add Metadata to Response Doc | (Same as above) |
| **Add Metadata to Response Doc** | `n8n-nodes-base.googleDocs` | Inserts execution header into Doc | Create new RFP Response Document | Extract Questions From RFP | (Same as above) |
| **OpenAI Chat Model** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM model provider | None | Extract Questions From RFP | ## 2️⃣ Question Extraction<br><br>**Extract Questions From RFP** reads the entire RFP text and identifies every question meant for the supplier, exactly as written. It relies on the **OpenAI Chat Model** for the reasoning and the **Item List Output Parser** to return the questions as a clean, structured list ready for looping. |
| **Item List Output Parser** | `@n8n/n8n-nodes-langchain.outputParserItemList` | Parses LLM output into structured list | None | Extract Questions From RFP | (Same as above) |
| **Extract Questions From RFP** | `@n8n/n8n-nodes-langchain.chainLlm` | Extracts questions from raw RFP text | Add Metadata to Response Doc, OpenAI Chat Model, Item List Output Parser | For Each Question... | (Same as above) |
| **For Each Question...** | `n8n-nodes-base.splitInBatches` | Iterates over questions sequentially | Extract Questions From RFP | Send Email Notification, Answer Question with Context | ## 3️⃣ Answer Loop<br><br>**For Each Question...** walks through the extracted list one item at a time. **Answer Question with Context** hands each question to a pretrained OpenAI Assistant that answers from the company's own knowledge, and **Record Question & Answer in Response Doc** writes the numbered question and answer straight into the Google Doc before the loop moves on. |
| **Answer Question with Context** | `@n8n/n8n-nodes-langchain.openAi` | Queries OpenAI Assistant for answers | For Each Question... | Record Question & Answer in Response Doc | (Same as above) |
| **Record Question & Answer in Response Doc** | `n8n-nodes-base.googleDocs` | Appends Q&A pair to Google Doc | Answer Question with Context | For Each Question... | (Same as above) |
| **Send Email Notification** | `n8n-nodes-base.gmail` | Emails requester upon completion | For Each Question... | Send Chat Notification | ## 4️⃣ Completion & Notifications<br><br>Once the loop finishes, **Send Email Notification** emails the original requester that their RFP response is ready, and **Send Chat Notification** posts the same completion update to the team's RFP Slack channel. |
| **Send Chat Notification** | `n8n-nodes-base.slack` | Posts completion message to Slack | Send Email Notification | None | (Same as above) |

---

### 4. Reproducing the Workflow from Scratch

Follow these numbered steps to manually rebuild the workflow in an n8n instance:

1. **Create Webhook Trigger:**
   - Add a **Webhook** node named `Wait for Request`.
   - Set HTTP Method to `POST`.
   - Note the generated production/test URL for external triggers.

2. **Add PDF Extraction:**
   - Add an **Extract From File** node named `Get RFP Data`.
   - Set operation to `pdf`.
   - Connect `Wait for Request` (main) to `Get RFP Data` (main).

3. **Configure Variables:**
   - Add a **Set** node named `Set Variables`.
   - Set mode to `raw` JSON output and configure the expression payload for `doc_title`, `doc_filename`, and `reply_to` using webhook body properties.
   - Connect `Get RFP Data` (main) to `Set Variables` (main).

4. **Initialize Google Document:**
   - Add a **Google Docs** node named `Create new RFP Response Document`.
   - Set operation to `create`, configure title via expression `={{ $json.doc_filename }}`, and input your target Google Drive `folderId`.
   - Configure **Google Docs OAuth2** credentials.
   - Connect `Set Variables` (main) to `Create new RFP Response Document` (main).

5. **Add Document Metadata:**
   - Add a second **Google Docs** node named `Add Metadata to Response Doc`.
   - Set operation to `update`, document URL to `={{ $json.id }}`.
   - Add an action field (`insert`) with metadata text containing title, timestamp, requester, and execution link.
   - Connect `Create new RFP Response Document` (main) to `Add Metadata to Response Doc` (main).

6. **Configure Advanced AI (Question Extraction):**
   - Add an **OpenAI Chat Model** sub-node and connect it to an **OpenAI** credentials set.
   - Add an **Item List Output Parser** sub-node.
   - Add a **Basic LLM Chain** node named `Extract Questions From RFP`.
   - Set the prompt text to extract supplier questions verbatim using `={{ $('Get RFP Data').item.json.text }}`.
   - Connect `Add Metadata to Response Doc` (main) to `Extract Questions From RFP` (main).
   - Connect `OpenAI Chat Model` (`ai_languageModel`) and `Item List Output Parser` (`ai_outputParser`) to `Extract Questions From RFP`.

7. **Setup Iteration Loop:**
   - Add a **Split In Batches** node named `For Each Question...`.
   - Connect `Extract Questions From RFP` (main) to `For Each Question...` (main).

8. **Configure OpenAI Assistant Question Answering:**
   - Add an **OpenAI** (Assistant) node named `Answer Question with Context`.
   - Set resource to `assistant`, select your trained Assistant ID (e.g., `asst_QBI5lLKOsjktr3DRB4MwrgZd`), and set prompt text to `={{ $json.response.text }}`.
   - Configure **OpenAI** credentials.
   - Connect `For Each Question...` output index 1 (loop active) to `Answer Question with Context` (main).

9. **Record Answers in Google Docs:**
   - Add a third **Google Docs** node named `Record Question & Answer in Response Doc`.
   - Set operation to `update`, document URL to `={{ $('Create new RFP Response Document').item.json.id }}`, and insert action text formatting `={{ $runIndex+1 }}. {{ $json.content }}\n{{ $json.output }}\n\n`.
   - Connect `Answer Question with Context` (main) to `Record Question & Answer in Response Doc` (main).
   - Connect `Record Question & Answer in Response Doc` (main) back to `For Each Question...` (main) to close the loop.

10. **Configure Notifications:**
    - Add a **Gmail** node named `Send Email Notification`.
    - Set message options: recipient to `={{ $('Set Variables').item.json.reply_to }}`, subject and text to reference `title`. Configure **Gmail OAuth2** credentials.
    - Connect `For Each Question...` output index 0 (loop complete) to `Send Email Notification` (main).
    - Add a **Slack** node named `Send Chat Notification`.
    - Set select mode to `channel`, channel ID/name to `RFP-channel`, and message text referencing the document title. Configure **Slack OAuth2** credentials.
    - Connect `Send Email Notification` (main) to `Send Chat Notification` (main).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Target Audience | Designed for sales engineers, bid managers, and proposal teams handling recurring RFPs/RFIs. |
| Assistant Dependency | The OpenAI Assistant used in the answer node must be pre-trained on internal company documentation; this workflow does not initialize or train the assistant knowledge base. |
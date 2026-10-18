Capture website form leads in Google Sheets and send Gmail auto-replies

https://n8nworkflows.xyz/workflows/capture-website-form-leads-in-google-sheets-and-send-gmail-auto-replies-21013


# Capture website form leads in Google Sheets and send Gmail auto-replies

### 1. Workflow Overview

This workflow is designed for freelancers and small businesses to capture, validate, store, and respond to website form submissions automatically. It functions as an automated lead intake pipeline that processes incoming HTTP POST requests, validates the data for completeness and correctness, records valid leads in a Google Sheets document, sends a personalized confirmation email to the submitter via Gmail, and responds to the originating web form with the appropriate status code.

The workflow logic is organized into the following functional blocks:
- **1.1 Input Reception & Normalization:** Captures incoming web form submissions via webhook and normalizes/validates the data fields.
- **1.2 Conditional Routing:** Evaluates the validation results to separate valid leads from invalid submissions.
- **1.3 Success Processing Path:** Handles successful submissions by returning an HTTP 200 response, persisting the lead data to Google Sheets, and sending a confirmation email.
- **1.4 Error Handling Path:** Handles rejected submissions by returning an HTTP 400 error response with specific validation feedback.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** This block receives the raw HTTP POST request from the website form, extracts the payload, normalizes text inputs (trimming whitespace and lowercasing emails), adds a UTC timestamp, and runs structural/format validation rules.
- **Nodes Involved:** `Form Webhook`, `Clean & Validate`

##### Node Details:
- **Form Webhook**
  - **Type & Technical Role:** `n8n-nodes-base.webhook` (v2). Acts as the primary entry point for HTTP POST requests from the website form.
  - **Configuration Choices:** Configured to listen on path `lead-intake` for `POST` requests. Response mode is set to use a dedicated response node (`responseNode`).
  - **Key Expressions / Variables:** Uses webhook ID `00000000-0000-4000-8000-000000000001`.
  - **Connections:** Output connects to `Clean & Validate`.
  - **Version Requirements:** Version 2.
  - **Edge Cases / Failures:** Requests sent via incorrect HTTP methods (e.g., GET) or missing the correct path will fail to trigger the workflow.

- **Clean & Validate**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2). JavaScript execution node that sanitizes inputs and performs programmatic data validation.
  - **Configuration Choices:** Executes custom JavaScript to parse JSON or form-encoded payloads, trim strings, lowercase email addresses, and evaluate validation constraints (missing name, invalid email format, message length exceeding 5,000 characters).
  - **Key Expressions / Variables:** Iterates over `$input.all()`, sets `received_at` to an ISO timestamp, determines `valid` boolean status, and concatenates error messages into an `errors` string.
  - **Connections:** Input from `Form Webhook`; output connects to `Valid?`.
  - **Version Requirements:** Version 2.
  - **Edge Cases / Failures:** Malformed JSON structures in the incoming request body could trigger JavaScript runtime exceptions if payload properties are undefined.

---

#### 2.2 Conditional Routing
- **Overview:** Evaluates the boolean validation flag produced by the sanitization step to direct the workflow down either the success or error processing path.
- **Nodes Involved:** `Valid?`

##### Node Details:
- **Valid?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (v2). Conditional router based on expression evaluation.
  - **Configuration Choices:** Evaluates whether the incoming `valid` property equals `true` using strict type validation.
  - **Key Expressions / Variables:** Left value: `={{ $json.valid }}`, Right value: `true`, Operator: Boolean True.
  - **Connections:** Input from `Clean & Validate`. True branch connects to `Respond OK`; False branch connects to `Respond Error`.
  - **Version Requirements:** Version 2.
  - **Edge Cases / Failures:** Expression failure if the preceding node does not output a valid boolean property.

---

#### 2.3 Success Processing Path
- **Overview:** Processes valid lead submissions by acknowledging the webhook request with a 200 OK status, appending the structured data row into Google Sheets, and dispatching a confirmation email to the user.
- **Nodes Involved:** `Respond OK`, `Save to Google Sheet`, `Auto-reply (Gmail)`

##### Node Details:
- **Respond OK**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (v1.1). Sends an HTTP response back to the form submission client.
  - **Configuration Choices:** Response code set to `200`, response format set to JSON.
  - **Key Expressions / Variables:** `={{ JSON.stringify({ ok: true, message: 'Thanks ' + $json.name + ', we got your message.' }) }}`
  - **Connections:** Input from `Valid?` (True branch); output connects to `Save to Google Sheet`.
  - **Version Requirements:** Version 1.1.
  - **Edge Cases / Failures:** Timeout or disconnection from the submitting web client before the response is fully received.

- **Save to Google Sheet**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (v4.5). Appends a new row of data to a target Google Spreadsheet.
  - **Configuration Choices:** Operation set to `append`. Sheet name is specified by name (`Leads`). Document ID uses a placeholder URL. Mapping mode uses automatic input data mapping (`autoMapInputData`).
  - **Key Expressions / Variables:** Target Document ID: `PASTE_YOUR_GOOGLE_SHEET_URL_HERE`.
  - **Connections:** Input from `Respond OK`; output connects to `Auto-reply (Gmail)`.
  - **Version Requirements:** Version 4.5.
  - **Edge Cases / Failures:** Authentication failures, incorrect Sheet name, or missing header columns in the destination spreadsheet will cause the node to error.

- **Auto-reply (Gmail)**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` (v2.1). Sends an automated email message via Gmail.
  - **Configuration Choices:** Email type set to text (`text`), attribution append disabled.
  - **Key Expressions / Variables:** 
    - Recipient (`sendTo`): `={{ $('Clean & Validate').item.json.email }}`
    - Subject: `Thanks for getting in touch`
    - Message body references name and message fields from the `Clean & Validate` node alongside placeholder text `YOUR BUSINESS NAME`.
  - **Connections:** Input from `Save to Google Sheet`.
  - **Version Requirements:** Version 2.1.
  - **Edge Cases / Failures:** Gmail API rate limits, revoked OAuth2 tokens, or invalid recipient email formats.

---

#### 2.4 Error Handling Path
- **Overview:** Handles invalid submissions by returning an HTTP 400 error response containing the specific validation error messages.
- **Nodes Involved:** `Respond Error`

##### Node Details:
- **Respond Error**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (v1.1). Returns an HTTP error response to the form submission client.
  - **Configuration Choices:** Response code set to `400`, response format set to JSON.
  - **Key Expressions / Variables:** `={{ JSON.stringify({ ok: false, errors: $json.errors }) }}`
  - **Connections:** Input from `Valid?` (False branch).
  - **Version Requirements:** Version 1.1.
  - **Edge Cases / Failures:** Client disconnection prior to receiving the error payload.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Form Webhook** | `n8n-nodes-base.webhook` | Receives incoming POST requests from the website form | None | Clean & Validate | ## How it works<br>1. **Form Webhook** receives a POST from your website form (JSON or form-encoded: name, email, phone, message, source).<br>2. **Clean & Validate** trims fields, lowercases the email and checks required fields.<br>3. **Valid?** routes the lead:<br>   - valid -> **Respond OK** (200) to the form, **Save to Google Sheet** appends the lead, **Auto-reply (Gmail)** sends an instant thank-you email.<br>   - invalid -> **Respond Error** (400) with the validation errors. |
| **Clean & Validate** | `n8n-nodes-base.code` | Normalizes and validates form data fields | Form Webhook | Valid? | ## How it works<br>1. **Form Webhook** receives a POST from your website form (JSON or form-encoded: name, email, phone, message, source).<br>2. **Clean & Validate** trims fields, lowercases the email and checks required fields.<br>3. **Valid?** routes the lead:<br>   - valid -> **Respond OK** (200) to the form, **Save to Google Sheet** appends the lead, **Auto-reply (Gmail)** sends an instant thank-you email.<br>   - invalid -> **Respond Error** (400) with the validation errors. |
| **Valid?** | `n8n-nodes-base.if` | Routes workflow based on validation success | Clean & Validate | Respond OK, Respond Error | ## How it works<br>1. **Form Webhook** receives a POST from your website form (JSON or form-encoded: name, email, phone, message, source).<br>2. **Clean & Validate** trims fields, lowercases the email and checks required fields.<br>3. **Valid?** routes the lead:<br>   - valid -> **Respond OK** (200) to the form, **Save to Google Sheet** appends the lead, **Auto-reply (Gmail)** sends an instant thank-you email.<br>   - invalid -> **Respond Error** (400) with the validation errors. |
| **Respond OK** | `n8n-nodes-base.respondToWebhook` | Returns HTTP 200 success response to webhook caller | Valid? | Save to Google Sheet | ## How it works<br>1. **Form Webhook** receives a POST from your website form (JSON or form-encoded: name, email, phone, message, source).<br>2. **Clean & Validate** trims fields, lowercases the email and checks required fields.<br>3. **Valid?** routes the lead:<br>   - valid -> **Respond OK** (200) to the form, **Save to Google Sheet** appends the lead, **Auto-reply (Gmail)** sends an instant thank-you email.<br>   - invalid -> **Respond Error** (400) with the validation errors. |
| **Save to Google Sheet** | `n8n-nodes-base.googleSheets` | Appends valid lead data to a Google Sheet | Respond OK | Auto-reply (Gmail) | ## Set up steps<br>1. Connect your **Google Sheets** and **Gmail** credentials in n8n (none are included in this template).<br>2. Create a Google Sheet with a tab named **Leads** and header row: received_at, name, email, phone, message, source, valid, errors.<br>3. Open **Save to Google Sheet** and replace `PASTE_YOUR_GOOGLE_SHEET_URL_HERE` with your sheet URL.<br>4. In **Auto-reply (Gmail)**, replace `YOUR BUSINESS NAME` and adjust the text.<br>5. Activate the workflow and point your form's POST action to the production URL of **Form Webhook** (path `/lead-intake`). |
| **Auto-reply (Gmail)** | `n8n-nodes-base.gmail` | Sends a confirmation email to the lead via Gmail | Save to Google Sheet | None | ## Set up steps<br>1. Connect your **Google Sheets** and **Gmail** credentials in n8n (none are included in this template).<br>2. Create a Google Sheet with a tab named **Leads** and header row: received_at, name, email, phone, message, source, valid, errors.<br>3. Open **Save to Google Sheet** and replace `PASTE_YOUR_GOOGLE_SHEET_URL_HERE` with your sheet URL.<br>4. In **Auto-reply (Gmail)**, replace `YOUR BUSINESS NAME` and adjust the text.<br>5. Activate the workflow and point your form's POST action to the production URL of **Form Webhook** (path `/lead-intake`). |
| **Respond Error** | `n8n-nodes-base.respondToWebhook` | Returns HTTP 400 error response with validation reasons | Valid? | None | ## How it works<br>1. **Form Webhook** receives a POST from your website form (JSON or form-encoded: name, email, phone, message, source).<br>2. **Clean & Validate** trims fields, lowercases the email and checks required fields.<br>3. **Valid?** routes the lead:<br>   - valid -> **Respond OK** (200) to the form, **Save to Google Sheet** appends the lead, **Auto-reply (Gmail)** sends an instant thank-you email.<br>   - invalid -> **Respond Error** (400) with the validation errors. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Webhook Node**
   - Add a **Webhook** node named `Form Webhook`.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `lead-intake`.
   - Set **Response Mode** to `Using 'Respond to Webhook' Node`.

2. **Create Clean & Validate Node**
   - Add a **Code** node named `Clean & Validate`.
   - Connect the output of `Form Webhook` to this node.
   - Set **Mode** to `Run Once for All Items`.
   - Insert the sanitization and validation JavaScript code parsing incoming payload properties (`name`, `email`, `phone`, `message`, `source`), appending timestamps, and calculating the `valid` status and `errors` string.

3. **Create Valid? Node**
   - Add an **If** node named `Valid?`.
   - Connect the output of `Clean & Validate` to this node.
   - Set condition to evaluate whether `={{ $json.valid }}` equals boolean `true`.

4. **Create Success Path Nodes**
   - **Respond OK:**
     - Add a **Respond to Webhook** node named `Respond OK`.
     - Connect the **True** output branch of `Valid?` to this node.
     - Set **Response Code** to `200`.
     - Set body content to return a JSON string including the submitter's name.
   - **Save to Google Sheet:**
     - Add a **Google Sheet** node named `Save to Google Sheet`.
     - Connect the output of `Respond OK` to this node.
     - Configure **Credentials** with valid Google Sheets OAuth2/Service Account access.
     - Set **Operation** to `Append`.
     - Set **Document ID** to your target Google Sheet URL (`PASTE_YOUR_GOOGLE_SHEET_URL_HERE`).
     - Set **Sheet Name** to `Leads`.
     - Ensure the destination spreadsheet contains the header row: `received_at, name, email, phone, message, source, valid, errors`.
   - **Auto-reply (Gmail):**
     - Add a **Gmail** node named `Auto-reply (Gmail)`.
     - Connect the output of `Save to Google Sheet` to this node.
     - Configure **Credentials** with valid Gmail OAuth2 access.
     - Set **Send To** to `={{ $('Clean & Validate').item.json.email }}`.
     - Set **Subject** to `Thanks for getting in touch`.
     - Set **Email Type** to `text`.
     - Customize the message body referencing `{{ $('Clean & Validate').item.json.name }}` and `{{ $('Clean & Validate').item.json.message }}`.

5. **Create Error Path Node**
   - **Respond Error:**
     - Add a **Respond to Webhook** node named `Respond Error`.
     - Connect the **False** output branch of `Valid?` to this node.
     - Set **Response Code** to `400`.
     - Set body content to return a JSON string containing `={{ JSON.stringify({ ok: false, errors: $json.errors }) }}`.

6. **Finalization**
   - Save the workflow.
   - Activate the workflow to generate the production webhook URL for external testing.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Tested on n8n version 1.97. | Environment compatibility |
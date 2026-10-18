Collect and validate quote requests with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/collect-and-validate-quote-requests-with-google-sheets-and-gmail-20976


# Collect and validate quote requests with Google Sheets and Gmail

### 1. Workflow Overview

This workflow automates the collection, validation, deduplication, and notification process for incoming customer quote requests. Designed primarily for service-based businesses (e.g., movers, cleaners, contractors), it captures form submissions, verifies completeness, records data in Google Sheets, notifies the business owner, and replies to the customer automatically based on whether critical details are missing.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Configuration:** Captures quote requests via an n8n Form trigger and loads global business settings.
- **1.2 Data Validation & Deduplication Check:** Normalizes form fields, evaluates required information, generates a unique deterministic request reference ID (`request_id`), and checks Google Sheets to prevent duplicate submissions on the same day.
- **1.3 Storage & Business Notification:** Appends new requests to Google Sheets (status-tagged as `ready_to_quote` or `missing_info`) and sends an internal notification email to the business owner via Gmail.
- **1.4 Customer Response & Completion:** Branches based on completeness to send either a confirmation email or a request for missing details to the customer, followed by displaying a thank-you or duplicate notice page.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Initiates the workflow when a customer submits the quote request form and establishes the runtime parameters for business branding and validation rules.
- **Nodes Involved:** 
  - `Quote request form`
  - `Settings`
- **Node Details:**
  - `Quote request form`:
    - *Type and technical role:* `n8n-nodes-base.formTrigger` — Acts as the webhook-based entry point hosting an interactive customer intake form.
    - *Configuration choices:* Configured with fields: Your name (required), Email (required), Phone, Service needed (dropdown: Service A, Service B, Other), Job address, Preferred date, and Job details. Uses `responseMode: lastNode`.
    - *Key expressions or variables:* None.
    - *Input/Output connections:* Input: None (Trigger). Output: Connects to `Settings`.
    - *Version-specific requirements:* Type version 2.2.
    - *Edge cases:* Incomplete submissions are blocked natively by form validation rules for required fields.
  - `Settings`:
    - `n8n-nodes-base.set` — Defines global configuration variables consumed downstream.
    - *Configuration choices:* Manual assignment mode containing string values for `business_name`, `owner_email`, `reply_within`, and `required_fields` (`service, address, preferred_date, details`).
    - *Key expressions:* Static string parameters.
    - *Input/Output connections:* Input: `Quote request form`. Output: `Check what is missing`.
    - *Edge cases:* Typo in `required_fields` key names will cause validation checks to evaluate incorrect fields.

#### 2.2 Data Validation & Deduplication Check
- **Overview:** Normalizes form inputs, checks against required fields defined in settings, generates a deterministic daily request ID hash, and queries Google Sheets to verify if the request has already been logged.
- **Nodes Involved:**
  - `Check what is missing`
  - `Already received?`
  - `New request?`
- **Node Details:**
  - `Check what is missing`:
    - *Type and technical role:* `n8n-nodes-base.code` — Executes custom JavaScript to parse form input, cross-reference required settings, generate missing field labels, and calculate a unique `request_id`.
    - *Configuration choices:* JavaScript ES6 code block referencing `$json` from both the form trigger and the Settings node via `$('Settings').first().json`.
    - *Key expressions or variables:* Uses `$now.toFormat('yyyy-LL-dd')`, hashing logic over email, service, address, and day to generate `request_id`. Sets status to `missing_info` or `ready_to_quote`.
    - *Input/Output connections:* Input: `Settings`. Output: `Already received?`.
    - *Edge cases:* Null or undefined properties are safely cleaned via helper functions, but missing exact string keys in `required_fields` may lead to unhandled lookup anomalies.
  - `Already received?`:
    - *Type and technical role:* `n8n-nodes-base.googleSheets` — Queries the target Google Sheets spreadsheet to find existing rows matching the generated `request_id`.
    - *Configuration choices:* Lookup operation using filter criteria where `lookupColumn` equals `request_id` with value `={{ $json.request_id }}`. `alwaysOutputData` is enabled.
    - *Key expressions:* `={{ $json.request_id }}`.
    - *Input/Output connections:* Input: `Check what is missing`. Output: `New request?`.
    - *Edge cases:* API rate limits or invalid Google Sheets credentials will fail execution.
  - `New request?`:
    - *Type and technical role:* `n8n-nodes-base.if` — Evaluates whether the Google Sheets lookup returned an empty result indicating a new request.
    - *Configuration choices:* Condition checking if `={{ $json.request_id }}` is empty.
    - *Key expressions:* `={{ $json.request_id }}`.
    - *Input/Output connections:* Input: `Already received?`. Output (True): `Save request`. Output (False): `Duplicate page`.

#### 2.3 Storage & Business Notification
- **Overview:** Appends new, validated quote requests to the Google Sheets tracking sheet and dispatches a summary notification email to the business owner via Gmail.
- **Nodes Involved:**
  - `Save request`
  - `Email the owner`
  - `Anything missing?`
- **Node Details:**
  - `Save request`:
    - *Type and technical role:* `n8n-nodes-base.googleSheets` — Appends a new row of data to the spreadsheet.
    - *Configuration choices:* Operation set to `append` on sheet name `Requests` using explicit column mapping (`defineBelow`). Maps fields such as `name`, `email`, `phone`, `status`, `address`, `details`, `service`, `request_id`, `received_at`, and `missing_fields`.
    - *Key expressions:* References evaluated against items from `$('Check what is missing')`.
    - *Input/Output connections:* Input: `New request?` (True branch). Output: `Email the owner`.
    - *Edge cases:* Mismatched column headers in the spreadsheet or missing mapped properties will throw schema validation errors.
  - `Email the owner`:
    - *Type and technical role:* `n8n-nodes-base.gmail` — Sends an alert email to the business owner containing full request details and status.
    - *Configuration choices:* Plain text email (`emailType: text`) with `onError` set to `continueRegularOutput`.
    - *Key expressions:* Sender target `={{ $('Settings').first().json.owner_email }}` with dynamic message body and subject line referencing parsed item properties.
    - *Input/Output connections:* Input: `Save request`. Output: `Anything missing?`.
    - *Edge cases:* Gmail authentication expiration or API quota limits.
  - `Anything missing?`:
    - *Type and technical role:* `n8n-nodes-base.if` — Determines whether the request status is flagged as needing more information.
    - *Configuration choices:* Evaluates if `={{ $('Check what is missing').item.json.status }}` equals `missing_info`.
    - *Key expressions:* `={{ $('Check what is missing').item.json.status }}`.
    - *Input/Output connections:* Input: `Email the owner`. Output (True): `Ask for missing details`. Output (False): `Confirm request`.

#### 2.4 Customer Response & Completion
- **Overview:** Dispatches tailored follow-up emails to the customer based on request completeness and renders completion screens.
- **Nodes Involved:**
  - `Ask for missing details`
  - `Confirm request`
  - `Thank-you page`
  - `Duplicate page`
- **Node Details:**
  - `Ask for missing details`:
    - *Type and technical role:* `n8n-nodes-base.gmail` — Emails the customer requesting specific missing information.
    - *Configuration choices:* Plain text email sent to customer's email address containing formatted bullet points of missing items.
    - *Key expressions:* `={{ $('Check what is missing').item.json.email }}` and dynamic message construction splitting `missing_fields`.
    - *Input/Output connections:* Input: `Anything missing?` (True branch). Output: `Thank-you page`.
  - `Confirm request`:
    - *Type and technical role:* `n8n-nodes-base.gmail` — Emails the customer confirming receipt and stating the expected turnaround time.
    - *Configuration choices:* Plain text email sent to customer confirming request receipt.
    - *Key expressions:* References `reply_within`, `business_name`, and `request_id`.
    - *Input/Output connections:* Input: `Anything missing?` (False branch). Output: `Thank-you page`.
  - `Thank-you page`:
    - *Type and technical role:* `n8n-nodes-base.form` — Displays an HTML completion message confirming submission processing to the end user.
    - *Configuration choices:* Operation set to `completion` with custom title ("Thanks, we got it") and message.
    - *Input/Output connections:* Input: `Ask for missing details` or `Confirm request`. Output: None.
  - `Duplicate page`:
    - *Type and technical role:* `n8n-nodes-base.form` — Displays a notice page if a duplicate request submission is detected on the same day.
    - *Configuration choices:* Operation set to `completion` with title ("Already received") and duplicate notice message.
    - *Input/Output connections:* Input: `New request?` (False branch). Output: None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Quote request form | n8n-nodes-base.formTrigger | Customer intake form trigger | None | Settings | ## 1. Form and settings<br>The customer fills the form. **Check what is missing** compares the answers with `required_fields` in **Settings**.<br>Do not rename the form questions: the workflow reads them by name. |
| Settings | n8n-nodes-base.set | Defines configuration variables | Quote request form | Check what is missing | ## 1. Form and settings<br>The customer fills the form. **Check what is missing** compares the answers with `required_fields` in **Settings**.<br>Do not rename the form questions: the workflow reads them by name. |
| Check what is missing | n8n-nodes-base.code | Normalizes data, checks missing fields, creates ID | Settings | Already received? | ## 1. Form and settings<br>The customer fills the form. **Check what is missing** compares the answers with `required_fields` in **Settings**.<br>Do not rename the form questions: the workflow reads them by name. |
| Already received? | n8n-nodes-base.googleSheets | Checks spreadsheet for existing request ID | Check what is missing | New request? | ## 2. Check, save and tell you<br>Each request gets a reference from email, service, address and day. A repeat that day is usually not saved again; two submits in the same second can both get through. New requests are emailed to `owner_email`. |
| New request? | n8n-nodes-base.if | Routes new vs. duplicate submissions | Already received? | Save request, Duplicate page | ## 2. Check, save and tell you<br>Each request gets a reference from email, service, address and day. A repeat that day is usually not saved again; two submits in the same second can both get through. New requests are emailed to `owner_email`. |
| Save request | n8n-nodes-base.googleSheets | Appends new request row to Google Sheets | New request? | Email the owner | ## 2. Check, save and tell you<br>Each request gets a reference from email, service, address and day. A repeat that day is usually not saved again; two submits in the same second can both get through. New requests are emailed to `owner_email`. |
| Email the owner | n8n-nodes-base.gmail | Sends internal notification email to business owner | Save request | Anything missing? | ## 2. Check, save and tell you<br>Each request gets a reference from email, service, address and day. A repeat that day is usually not saved again; two submits in the same second can both get through. New requests are emailed to `owner_email`. |
| Anything missing? | n8n-nodes-base.if | Checks if request is missing required details | Email the owner | Ask for missing details, Confirm request | ## 3. Reply to the customer<br>Missing details: the customer gets a list of what to send. Complete: a confirmation.<br>Replies land in your Gmail inbox. The workflow does not read them: update the row yourself. |
| Ask for missing details | n8n-nodes-base.gmail | Emails customer for missing requirements | Anything missing? | Thank-you page | ## 3. Reply to the customer<br>Missing details: the customer gets a list of what to send. Complete: a confirmation.<br>Replies land in your Gmail inbox. The workflow does not read them: update the row yourself. |
| Confirm request | n8n-nodes-base.gmail | Emails confirmation to customer with timeframe | Anything missing? | Thank-you page | ## 3. Reply to the customer<br>Missing details: the customer gets a list of what to send. Complete: a confirmation.<br>Replies land in your Gmail inbox. The workflow does not read them: update the row yourself. |
| Thank-you page | n8n-nodes-base.form | Renders completion screen for successful submissions | Ask for missing details, Confirm request | None | ## 3. Reply to the customer<br>Missing details: the customer gets a list of what to send. Complete: a confirmation.<br>Replies land in your Gmail inbox. The workflow does not read them: update the row yourself. |
| Duplicate page | n8n-nodes-base.form | Renders notice screen for duplicate submissions | New request? | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually:

1. **Create the Google Sheet:**
   - Create a Google Sheet containing a tab named `Requests`.
   - In cell `A1`, paste the following header string: `request_id,received_at,status,name,email,phone,service,address,preferred_date,details,missing_fields,quote_id,quote_amount,thread_id,quote_sent_at,follow_ups_sent,last_follow_up_at,reply_label,job_start,event_id,closed_at`.
   - Use Google Sheets' **Data > Split text to columns** utility to separate headers across row 1.

2. **Node 1: Form Trigger (`Quote request form`)**
   - Add a **Form Trigger** node.
   - Set form title to `Request a quote` and description to `Tell us about the job and we will send you a quote.`
   - Add fields: 
     - Text: `Your name` (Required)
     - Email: `Email` (Required)
     - Text: `Phone`
     - Dropdown: `Service needed` with options `Service A`, `Service B`, `Other`
     - Text: `Job address`
     - Date: `Preferred date`
     - Textarea: `Job details (size, rooms, items...)`
   - Set Response Mode to `lastNode`.

3. **Node 2: Settings (`Settings`)**
   - Add a **Set** node and connect it to `Quote request form`.
   - Create manual assignments:
     - `business_name` (String): `Your Business Name`
     - `owner_email` (String): `user@example.com`
     - `reply_within` (String): `1 business day`
     - `required_fields` (String): `service, address, preferred_date, details`

4. **Node 3: Validation Logic (`Check what is missing`)**
   - Add a **Code** node and connect it to `Settings`.
   - Set Mode to JavaScript and insert normalization, required-field evaluation, and SHA-like hash generation code to compute `request_id`, `missing_fields`, and `status`.

5. **Node 4: Duplicate Lookup (`Already received?`)**
   - Add a **Google Sheets** node connected to `Check what is missing`.
   - Set operation to **Get Many** or configure filters UI with lookup column `request_id` matching expression `={{ $json.request_id }}`.
   - Configure a Google Sheets credential linked to your document URL. Enable **Always Output Data**.

6. **Node 5: Branch Evaluation (`New request?`)**
   - Add an **If** node connected to `Already received?`.
   - Configure condition: Left value `={{ $json.request_id }}` operator **Empty**.

7. **Node 6: Duplicate Completion (`Duplicate page`)**
   - Add a **Form (Completion)** node connected to the `false` branch of `New request?`.
   - Set title to `Already received` and message to `We already have this request from today. No need to send it again.`

8. **Node 7: Row Storage (`Save request`)**
   - Add a **Google Sheets** node connected to the `true` branch of `New request?`.
   - Set operation to **Append**, select your spreadsheet, and map schema values matching each incoming JSON property (e.g., `name`, `email`, `phone`, `status`, `address`, `details`, `service`, `request_id`, `received_at`, `missing_fields`).

9. **Node 8: Owner Notification (`Email the owner`)**
   - Add a **Gmail** node connected to `Save request`. Configure Gmail credentials.
   - Set Send To to `={{ $('Settings').first().json.owner_email }}`.
   - Set Subject and Text Message dynamically using expressions referencing node outputs. Enable `onError: continueRegularOutput`.

10. **Node 9: Status Check (`Anything missing?`)**
    - Add an **If** node connected to `Email the owner`.
    - Configure condition checking if `={{ $('Check what is missing').item.json.status }}` equals `missing_info`.

11. **Node 10: Customer Outreach (`Ask for missing details` & `Confirm request`)**
    - Add two **Gmail** nodes connected to the `true` and `false` branches of `Anything missing?` respectively.
    - Configure recipient to `={{ $('Check what is missing').item.json.email }}`.
    - Set appropriate plain text subject lines and messaging templates for requesting missing details or confirming receipt.

12. **Node 11: Final Completion (`Thank-you-page`)**
    - Add a **Form (Completion)** node connected as the convergence point for both customer-facing Gmail nodes.
    - Set title to `Thanks, we got it` and message to `Check your inbox: we just sent you an email about your request.`

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Cloud & n8n Credential Requirements | [n8n Google Credentials Documentation](https://docs.n8n.io/integrations/builtin/credentials/google/)<br>*(Note: If your Google Cloud app is in Testing mode, credentials expire every 7 days; publish the application to prevent disconnections).* |
| Project Credits | Built and tested by Small Hours, a team of AI agents. |
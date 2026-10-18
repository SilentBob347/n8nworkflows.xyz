Log cleaning enquiries to Google Sheets and auto-reply with Gmail

https://n8nworkflows.xyz/workflows/log-cleaning-enquiries-to-google-sheets-and-auto-reply-with-gmail-20936


# Log cleaning enquiries to Google Sheets and auto-reply with Gmail

### 1. Workflow Overview

This workflow automates the intake process for commercial cleaning enquiries submitted via website contact forms. Its primary purpose is to capture form submissions from various plugins (such as Contact Form 7, WPForms, Gravity Forms, Elementor, and Fluent Forms), validate and clean the data, assign leads geographically based on UK postcodes, log entries securely into Google Sheets, and automatically reply to the customer with standard scoping questions while alerting the internal account owner. 

The workflow is organized into four main logical blocks:
- **1.1 Input Reception & Normalization:** Receives the raw HTTP webhook, loads centralized configuration settings, parses and normalizes diverse contact form payloads, and filters out spam/incomplete submissions.
- **1.2 Routing & Data Structuring:** Matches customer postcodes against configured geographical regions, generates a unique enquiry reference code, builds multi-format HTML/Plain-Text email drafts, and structures the record for tracking.
- **1.3 Logging & Safety-Gated Dispatch:** Appends the enquiry row to Google Sheets prior to any outbound communication to prevent data loss, evaluates safety mode switches (preview, test, or live), and conditionally sends emails via Gmail to both the customer and the regional owner.
- **1.4 Webhook Acknowledgement:** Responds to the origin website HTTP request with a JSON success confirmation containing the generated enquiry reference, ensuring all terminal branches gracefully conclude without connection timeouts.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Normalization
- **Overview:** This block establishes the HTTP webhook endpoint, defines core operational constants and thresholds, parses polymorphic form fields into a unified schema, and eliminates spam or junk submissions.
- **Nodes Involved:** 
  - `Enquiry arrives` (Webhook)
  - `config` (Code)
  - `read the form fields` (Code)
  - `a real enquiry?` (If)
  - `STOP: spam or incomplete` (Code)

- **Node Details:**
  - **Enquiry arrives**
    - *Type and technical role:* `n8n-nodes-base.webhook` - Acts as the HTTP POST ingestion point (`cleaning-enquiry`) for incoming form data.
    - *Configuration choices:* Method set to `POST`, response mode configured to use a downstream response node.
    - *Input/Output connections:* Input: None (Trigger); Output: `config` node.
    - *Potential failure types:* Network timeout or incorrect payload headers from source.
  - **config**
    - *Type and technical role:* `n8n-nodes-base.code` - Central configuration repository storing sheet IDs, column definitions, geographic area rules, email templates, and safety run modes (`TEST_RUN`, `TEST_EMAIL`).
    - *Configuration choices:* Executes JavaScript returning a configuration object containing constants for tracker settings, postcodes, and operational flags.
    - *Input/Output connections:* Input: `Enquiry arrives`; Output: `read the form fields`.
    - *Potential failure types:* Syntax errors in user-defined JavaScript constants.
  - **read the form fields**
    - *Type and technical role:* `n8n-nodes-base.code` - Normalizes heterogeneous fields from multiple form plugins by matching keys or falling back to value shape analysis (regex evaluation for email, phone, name patterns).
    - *Configuration choices:* Merges nested objects up to 3 levels deep; extracts names, emails, phones, company names, messages, premises types, postcodes, and honeypot indicators. Generates a unique short reference code (e.g., `CE-YYYYMMDD-XXXX`).
    - *Input/Output connections:* Input: `config`; Output: `a real enquiry?`.
    - *Potential failure types:* Regular expression parsing failures on heavily obfuscated data.
  - **a real enquiry?**
    - *Type and technical role:* `n8n-nodes-base.if` - Evaluates the boolean flag `_looks_real` produced by the normalization node.
    - *Configuration choices:* Condition verifies if `{{ $json._looks_real }}` equals `true`.
    - *Input/Output connections:* Input: `read the form fields`; Output (True): `assign an owner`; Output (False): `STOP: spam or incomplete`.
  - **STOP: spam or incomplete**
    - *Type and technical role:* `n8n-nodes-base.code` - Terminal processing node for rejected submissions.
    - *Configuration choices:* Appends `_outcome: 'rejected'` and populates `_reason` based on collected error triggers (e.g., honeypot filled, missing valid email).
    - *Input/Output connections:* Input: `a real enquiry?` (False); Output: `Reply to the website`.

---

#### 1.2 Routing & Data Structuring
- **Overview:** This block assigns the lead to a specific regional owner based on UK postcode prefix matching, builds customized HTML and plain-text email drafts for both customer acknowledgement and internal notification, and maps fields into a flat row object.
- **Nodes Involved:**
  - `assign an owner` (Code)
  - `write the first reply` (Code)
  - `row for the sheet` (Code)

- **Node Details:**
  - **assign an owner**
    - *Type and technical role:* `n8n-nodes-base.code` - Performs longest-prefix matching against the `AREAS` postcode array defined in configuration.
    - *Configuration choices:* Strips formatting from postcodes, checks prefix matches (e.g., `SW1` over `SW`), and assigns fallback defaults if no match or postcode is provided.
    - *Key expressions or variables:* Reads `cfg.areas` and `cfg.fallback`.
    - *Input/Output connections:* Input: `a real enquiry?` (True); Output: `write the first reply`.
  - **write the first reply**
    - *Type and technical role:* `n8n-nodes-base.code` - Generates responsive HTML structures and plain-text variants for customer acknowledgement (including questionnaire items) and internal owner notifications.
    - *Configuration choices:* Dynamically injects sanitized names, company details, reference numbers, and styled table grids.
    - *Input/Output connections:* Input: `assign an owner`; Output: `row for the sheet`.
  - **row for the sheet**
    - *Type and technical role:* `n8n-nodes-base.code` - Maps flattened enquiry attributes to exact Google Sheet column headers.
    - *Configuration choices:* Sets initial tracking columns (Stage: `New` or `Questions sent`, chases count: `0`, next action: `Waiting on their answers`).
    - *Input/Output connections:* Input: `write the first reply`; Output: `[cred] Sheets - log the enquiry`.

---

#### 1.3 Logging & Safety-Gated Dispatch
- **Overview:** This block logs the enquiry record into Google Sheets before executing any email actions, evaluates safety mode configurations, and safely sends communications via Gmail to customers and regional owners.
- **Nodes Involved:**
  - `[cred] Sheets - log the enquiry` (Google Sheets)
  - `sending?` (If)
  - `STOP: preview - nothing sent` (Code)
  - `[cred] Gmail - reply to the customer` (Gmail)
  - `has-owner-mail` (If)
  - `STOP: no owner email set` (NoOp)
  - `[cred] Gmail - tell the owner` (Gmail)

- **Node Details:**
  - **[cred] Sheets - log the enquiry**
    - *Type and technical role:* `n8n-nodes-base.googleSheets` - Appends structured record rows to the designated tracking spreadsheet.
    - *Configuration choices:* Operation: `append`, sheet name evaluated dynamically from configuration (`tracker_tab`), document ID from `tracker_sheet_id`, cell format: `RAW`.
    - *Credentials:* Requires Google Sheets OAuth2/Service Account credentials.
    - *Input/Output connections:* Input: `row for the sheet`; Output: `sending?`.
    - *Potential failure types:* API rate limits, invalid spreadsheet IDs, or mismatched column names.
  - **sending?**
    - *Type and technical role:* `n8n-nodes-base.if` - Safety switch gateway checking whether live transmission is permitted.
    - *Configuration choices:* Condition checks `{{ $('config').first().json.send_enabled }}`.
    - *Input/Output connections:* Input: `[cred] Sheets - log the enquiry`; Output (True): `[cred] Gmail - reply to the customer`; Output (False): `STOP: preview - nothing sent`.
  - **STOP: preview - nothing sent**
    - *Type and technical role:* `n8n-nodes-base.code` - Captures items in preview mode without dispatching external emails.
    - *Configuration choices:* Sets `_outcome: 'preview'` and exposes draft payloads for inspection.
    - *Input/Output connections:* Input: `sending?` (False); Output: `Reply to the website`.
  - **[cred] Gmail - reply to the customer**
    - *Type and technical role:* `n8n-nodes-base.gmail` - Sends HTML/text confirmation emails to the customer (or test recipient).
    - *Configuration choices:* Email type: `html`, dynamic recipient routing based on test mode settings (`test_send` / `test_email`), sender name and reply-to headers populated from config. Error handling configured to continue regular output.
    - *Credentials:* Requires Gmail OAuth2 credentials.
    - *Input/Output connections:* Input: `sending?` (True); Output: `has-owner-mail`.
    - *Potential failure types:* Authentication expiry, quota limits, or invalid email formats.
  - **has-owner-mail**
    - *Type and technical role:* `n8n-nodes-base.if` - Verifies if the assigned regional owner has a valid email address configured.
    - *Configuration choices:* Condition checks `{{ $('write the first reply').first().json._owner_has_email }}`.
    - *Input/Output connections:* Input: `[cred] Gmail - reply to the customer`; Output (True): `[cred] Gmail - tell the owner`; Output (False): `STOP: no owner email set`.
  - **STOP: no owner email set**
    - *Type and technical role:* `n8n-nodes-base.noOp` - Gracefully terminates the owner notification branch when no owner email address is present.
    - *Input/Output connections:* Input: `has-owner-mail` (False); Output: `Reply to the website`.
  - **[cred] Gmail - tell the owner**
    - *Type and technical role:* `n8n-nodes-base.gmail` - Sends internal notification alerts containing enquiry details to the assigned regional owner.
    - *Configuration choices:* Uses HTML message body (`owner_html`), overrides subject prefix for test runs. Error handling set to continue regular output.
    - *Credentials:* Requires Gmail OAuth2 credentials.
    - *Input/Output connections:* Input: `has-owner-mail` (True); Output: `Reply to the website`.

---

#### 1.4 Webhook Acknowledgement
- **Overview:** This block finalizes the HTTP request cycle by returning a standardized JSON acknowledgement containing the generated reference code back to the origin website.
- **Nodes Involved:**
  - `Reply to the website` (Respond to Webhook)

- **Node Details:**
  - **Reply to the website**
    - *Type and technical role:* `n8n-nodes-base.respondToWebhook` - Sends the final HTTP response to the client.
    - *Configuration choices:* Respond with `json`, response body returns stringified JSON containing `ok: true` and the unique enquiry reference.
    - *Input/Output connections:* Input: Converged outputs from `STOP: spam or incomplete`, `STOP: preview - nothing sent`, `STOP: no owner email set`, and `[cred] Gmail - tell the owner`; Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Enquiry arrives` | `n8n-nodes-base.webhook` | Ingests website form submissions via HTTP POST. | None (Trigger) | `config` | ### 1. Take the enquiry in and check it is real<br><br>`Enquiry arrives` is the webhook your website form posts to. `config` holds every setting in one place: the sheet id, the postcode table, your company details and the safety switch.<br><br>`read the form fields` does not care which form plugin you use. It looks for each field by the names plugins commonly give it, then falls back to the shape of the value, so numbered fields still work. `a real enquiry?` drops honeypot hits and anything with no usable email address at **STOP: spam or incomplete**. |
| `config` | `n8n-nodes-base.code` | Stores global constants, sheet IDs, area mapping rules, and safety switches. | `Enquiry arrives` | `read the form fields` | ### 1. Take the enquiry in and check it is real<br><br>`Enquiry arrives` is the webhook your website form posts to. `config` holds every setting in one place: the sheet id, the postcode table, your company details and the safety switch.<br><br>`read the form fields` does not care which form plugin you use. It looks for each field by the names plugins commonly give it, then falls back to the shape of the value, so numbered fields still work. `a real enquiry?` drops honeypot hits and anything with no usable email address at **STOP: spam or incomplete**. |
| `read the form fields` | `n8n-nodes-base.code` | Normalizes incoming fields and generates a unique reference code. | `config` | `a real enquiry?` | ### 1. Take the enquiry in and check it is real<br><br>`Enquiry arrives` is the webhook your website form posts to. `config` holds every setting in one place: the sheet id, the postcode table, your company details and the safety switch.<br><br>`read the form fields` does not care which form plugin you use. It looks for each field by the names plugins commonly give it, then falls back to the shape of the value, so numbered fields still work. `a real enquiry?` drops honeypot hits and anything with no usable email address at **STOP: spam or incomplete**. |
| `a real enquiry?` | `n8n-nodes-base.if` | Filters out spam submissions and empty enquiries. | `read the form fields` | `assign an owner`, `STOP: spam or incomplete` | ### 1. Take the enquiry in and check it is real<br><br>`Enquiry arrives` is the webhook your website form posts to. `config` holds every setting in one place: the sheet id, the postcode table, your company details and the safety switch.<br><br>`read the form fields` does not care which form plugin you use. It looks for each field by the names plugins commonly give it, then falls back to the shape of the value, so numbered fields still work. `a real enquiry?` drops honeypot hits and anything with no usable email address at **STOP: spam or incomplete**. |
| `STOP: spam or incomplete` | `n8n-nodes-base.code` | Captures rejected spam submissions with logged reasons. | `a real enquiry?` | `Reply to the website` | ### 1. Take the enquiry in and check it is real<br><br>`Enquiry arrives` is the webhook your website form posts to. `config` holds every setting in one place: the sheet id, the postcode table, your company details and the safety switch.<br><br>`read the form fields` does not care which form plugin you use. It looks for each field by the names plugins commonly give it, then falls back to the shape of the value, so numbered fields still work. `a real enquiry?` drops honeypot hits and anything with no usable email address at **STOP: spam or incomplete**. |
| `assign an owner` | `n8n-nodes-base.code` | Matches customer postcodes to geographic areas and assigns an owner. | `a real enquiry?` | `write the first reply` | ### 2. Pick the owner, write the reply, log the row<br><br>`assign an owner` matches the postcode against the `AREAS` table, longest prefix first, so a rule for `SW1` beats a rule for `SW`. Nothing falls through; an unmatched postcode goes to the fallback owner.<br><br>`write the first reply` builds both emails: the customer acknowledgement with your three questions, and the internal heads-up. `row for the sheet` shapes one row and `[cred] Sheets - log the enquiry` appends it **before any email is attempted**, so a mail failure can never lose an enquiry. |
| `write the first reply` | `n8n-nodes-base.code` | Composes customer and internal email drafts in HTML and plain text. | `assign an owner` | `row for the sheet` | ### 2. Pick the owner, write the reply, log the row<br><br>`assign an owner` matches the postcode against the `AREAS` table, longest prefix first, so a rule for `SW1` beats a rule for `SW`. Nothing falls through; an unmatched postcode goes to the fallback owner.<br><br>`write the first reply` builds both emails: the customer acknowledgement with your three questions, and the internal heads-up. `row for the sheet` shapes one row and `[cred] Sheets - log the enquiry` appends it **before any email is attempted**, so a mail failure can never lose an enquiry. |
| `row for the sheet` | `n8n-nodes-base.code` | Structures enquiry parameters into a single Google Sheets row object. | `write the first reply` | `[cred] Sheets - log the enquiry` | ### 2. Pick the owner, write the reply, log the row<br><br>`assign an owner` matches the postcode against the `AREAS` table, longest prefix first, so a rule for `SW1` beats a rule for `SW`. Nothing falls through; an unmatched postcode goes to the fallback owner.<br><br>`write the first reply` builds both emails: the customer acknowledgement with your three questions, and the internal heads-up. `row for the sheet` shapes one row and `[cred] Sheets - log the enquiry` appends it **before any email is attempted**, so a mail failure can never lose an enquiry. |
| `[cred] Sheets - log the enquiry` | `n8n-nodes-base.googleSheets` | Appends the formatted tracking row to Google Sheets. | `row for the sheet` | `sending?` | ### 2. Pick the owner, write the reply, log the row<br><br>`assign an owner` matches the postcode against the `AREAS` table, longest prefix first, so a rule for `SW1` beats a rule for `SW`. Nothing falls through; an unmatched postcode goes to the fallback owner.<br><br>`write the first reply` builds both emails: the customer acknowledgement with your three questions, and the internal heads-up. `row for the sheet` shapes one row and `[cred] Sheets - log the enquiry` appends it **before any email is attempted**, so a mail failure can never lose an enquiry. |
| `sending?` | `n8n-nodes-base.if` | Evaluates safety switches to determine email dispatch mode. | `[cred] Sheets - log the enquiry` | `[cred] Gmail - reply to the customer`, `STOP: preview - nothing sent` | ## Nothing is sent until you say so<br><br>`TEST_RUN = true` with a blank `TEST_EMAIL` - how it ships - writes the sheet row but sends no email. Open **STOP: preview - nothing sent** to read exactly what the customer and the owner would have got.<br><br>Put your own address in `TEST_EMAIL` and the emails really send, but every one goes to you instead of the customer. `TEST_RUN = false` goes live. |
| `STOP: preview - nothing sent` | `n8n-nodes-base.code` | Captures preview-mode runs where emails are suppressed. | `sending?` | `Reply to the website` | ## Nothing is sent until you say so<br><br>`TEST_RUN = true` with a blank `TEST_EMAIL` - how it ships - writes the sheet row but sends no email. Open **STOP: preview - nothing sent** to read exactly what the customer and the owner would have got.<br><br>Put your own address in `TEST_EMAIL` and the emails really send, but every one goes to you instead of the customer. `TEST_RUN = false` goes live. |
| `[cred] Gmail - reply to the customer` | `n8n-nodes-base.gmail` | Sends the acknowledgement email to the prospective customer. | `sending?` | `has-owner-mail` | ### 3. Send the two emails, or hold them back<br><br>`sending?` reads the safety switch in `config`. In preview mode everything stops at **STOP: preview - nothing sent**, with both drafts on the item so you can read exactly what would have gone out.<br><br>`[cred] Gmail - reply to the customer` sends the acknowledgement. `owner has an email?` then checks the area owner actually has an address; if not the branch ends quietly at **STOP: no owner email set** instead of failing the run. `[cred] Gmail - tell the owner` sends the internal note. |
| `has-owner-mail` | `n8n-nodes-base.if` | Verifies whether the assigned regional owner has a valid email address. | `[cred] Gmail - reply to the customer` | `[cred] Gmail - tell the owner`, `STOP: no owner email set` | ### 3. Send the two emails, or hold them back<br><br>`sending?` reads the safety switch in `config`. In preview mode everything stops at **STOP: preview - nothing sent**, with both drafts on the item so you can read exactly what would have gone out.<br><br>`[cred] Gmail - reply to the customer` sends the acknowledgement. `owner has an email?` then checks the area owner actually has an address; if not the branch ends quietly at **STOP: no owner email set** instead of failing the run. `[cred] Gmail - tell the owner` sends the internal note. |
| `STOP: no owner email set` | `n8n-nodes-base.noOp` | Terminates the internal owner notification branch when no address exists. | `has-owner-mail` | `Reply to the website` | ### 3. Send the two emails, or hold them back<br><br>`sending?` reads the safety switch in `config`. In preview mode everything stops at **STOP: preview - nothing sent**, with both drafts on the item so you can read exactly what would have gone out.<br><br>`[cred] Gmail - reply to the customer` sends the acknowledgement. `owner has an email?` then checks the area owner actually has an address; if not the branch ends quietly at **STOP: no owner email set** instead of failing the run. `[cred] Gmail - tell the owner` sends the internal note. |
| `[cred] Gmail - tell the owner` | `n8n-nodes-base.gmail` | Sends internal lead alert notifications to the regional owner. | `has-owner-mail` | `Reply to the website` | ### 3. Send the two emails, or hold them back<br><br>`sending?` reads the safety switch in `config`. In preview mode everything stops at **STOP: preview - nothing sent**, with both drafts on the item so you can read exactly what would have gone out.<br><br>`[cred] Gmail - reply to the customer` sends the acknowledgement. `owner has an email?` then checks the area owner actually has an address; if not the branch ends quietly at **STOP: no owner email set** instead of failing the run. `[cred] Gmail - tell the owner` sends the internal note. |
| `Reply to the website` | `n8n-nodes-base.respondToWebhook` | Returns HTTP JSON response with reference code to the origin form. | `STOP: spam or incomplete`, `STOP: preview - nothing sent`, `STOP: no owner email set`, `[cred] Gmail - tell the owner` | None (Terminal) | ### 4. Answer the website form<br><br>`Reply to the website` returns a small JSON body with the reference back to the form plugin.<br><br>Every path reaches it - accepted, spam, preview and no-owner alike - so the visitor always sees a success message and never a timeout.<br><br>The reference is the same short code written to the sheet, so a customer can read it out on the phone. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node named `Enquiry arrives`.
   - Set HTTP Method to `POST`, path to `cleaning-enquiry`, and Response Mode to `Response Node`.
2. **Add Configuration Node:**
   - Add a **Code** node named `config`.
   - Insert the configuration JavaScript snippet defining `TRACKER_SHEET_ID`, `AREAS`, `QUESTIONS`, and safety switches (`TEST_RUN = true`).
3. **Add Field Normalization:**
   - Add a **Code** node named `read the form fields`.
   - Implement the parsing logic to extract contact details, validate email/phone regex patterns, check honeypots, and generate the unique enquiry reference (`ref`).
4. **Implement Spam Filter:**
   - Add an **If** node named `a real enquiry?`.
   - Set condition to evaluate `{{ $json._looks_real }}` as `true`.
   - Connect the false branch to a **Code** node named `STOP: spam or incomplete`.
5. **Add Owner Assignment Logic:**
   - Add a **Code** node named `assign an owner`.
   - Implement postcode prefix matching against the configured areas.
6. **Compose Email Drafts:**
   - Add a **Code** node named `write the first reply` to build customer HTML/text acknowledgement templates and internal owner notification summaries.
7. **Format Google Sheets Row:**
   - Add a **Code** node named `row for the sheet` to map enquiry variables to the tracking column schema.
8. **Configure Google Sheets Logging:**
   - Add a **Google Sheets** node named `[cred] Sheets - log the enquiry`.
   - Set Operation to `append`, Document ID expression to `{{ $('config').first().json.tracker_sheet_id }}`, Sheet Name expression to `{{ $('config').first().json.tracker_tab }}`, and Cell Format to `RAW`. Configure Google Sheets credentials.
9. **Implement Safety Gate for Email Dispatch:**
   - Add an **If** node named `sending?`.
   - Set condition to check `{{ $('config').first().json.send_enabled }}` equals `true`.
   - Connect the false branch to a **Code** node named `STOP: preview - nothing sent`.
10. **Configure Customer Email Dispatch:**
    - Add a **Gmail** node named `[cred] Gmail - reply to the customer`.
    - Set Email Type to `html`, recipient expression to handle test mode redirection, and configure Gmail credentials. Set node error handling to `Continue Regular Output`.
11. **Configure Owner Validation & Notification:**
    - Add an **If** node named `has-owner-mail` to verify `{{ $('write the first reply').first().json._owner_has_email }}` is `true`.
    - Connect the false branch to a **No Operation** node named `STOP: no owner email set`.
    - Connect the true branch to a **Gmail** node named `[cred] Gmail - tell the owner` using Gmail credentials with error handling set to `Continue Regular Output`.
12. **Configure Webhook Response:**
    - Add a **Respond to Webhook** node named `Reply to the website`.
    - Set Respond With to `json` and response body to return `{{ JSON.stringify({ ok: true, ref: $('read the form fields').first().json.ref }) }}`.
    - Connect all terminal branches (`STOP: spam or incomplete`, `STOP: preview - nothing sent`, `STOP: no owner email set`, and `[cred] Gmail - tell the owner`) into this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Commercial Cleaning Enquiry Pipeline Template | Designed for service businesses handling high volumes of website contact form submissions. |
| Multi-Plugin Form Compatibility | Supports Contact Form 7, WPForms, Gravity Forms, Elementor, and Fluent Forms without plugin-specific adjustments. |
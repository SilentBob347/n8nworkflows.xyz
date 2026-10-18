Detect duplicate payment charges from Gmail using Gemini, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/detect-duplicate-payment-charges-from-gmail-using-gemini--google-sheets-and-slack-20859


# Detect duplicate payment charges from Gmail using Gemini, Google Sheets and Slack

### 1. Workflow Overview

This workflow is designed to automate the detection of duplicate or double-charge payment emails. It monitors a Gmail inbox for receipt emails, leverages Google Gemini AI to extract transaction metadata, validates and standardizes the data, checks a Google Sheets ledger for prior transactions within a 48-hour window, logs the transaction, alerts the team via Slack if a duplicate is found, and marks the processed email in Gmail.

The workflow logic is divided into six functional blocks:
- **1.1 Email Intake & Preparation:** Triggers upon new receipt emails, retrieves full message details, and parses the email body from MIME formats.
- **1.2 AI Transaction Extraction:** Sends structured email metadata and body text to Google Gemini to extract financial details and parses the returned JSON output.
- **1.3 Validation & Transaction Preparation:** Validates the payment confirmation status, normalizes transaction attributes, and generates a deterministic hash-based transaction ID.
- **1.4 Duplicate Detection:** Fetches recent transactions from Google Sheets and compares them against the current transaction to flag potential duplicates.
- **1.5 Transaction Logging:** Evaluates whether a duplicate was detected and branches execution to log either a normal or duplicate transaction entry into Google Sheets.
- **1.6 Alert & Completion:** Dispatches a Slack notification for verified potential duplicates and applies a processed label to the original Gmail message.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Intake & Preparation
**Overview:** Monitors the inbox for incoming receipts, retrieves the full message payload via the Gmail API, and extracts clean plain-text or HTML content.
- **Nodes Involved:** `Payment Emails`, `Get Payment Email`, `Get Full Message`, `Extract Email Body`, `Prepare Email`
- **Node Details:**
  - **Payment Emails** (`n8n-nodes-base.gmailTrigger`)
    - *Role:* Polls Gmail every minute using the query filter `receipt`.
    - *Key Expressions:* Filter query: `receipt`
    - *Connections:* Input: None | Output: `Get Payment Email`
    - *Edge Cases/Failures:* API authentication failure, rate limiting during high email volumes.
  - **Get Payment Email** (`n8n-nodes-base.gmail`)
    - *Role:* Retrieves basic message metadata for the triggered email ID.
    - *Key Expressions:* Message ID: `{{$json.id}}`
    - *Connections:* Input: `Payment Emails` | Output: `Get Full Message`
  - **Get Full Message** (`n8n-nodes-base.httpRequest`)
    - *Role:* Calls the Gmail REST API directly to fetch the full message payload in MIME format.
    - *Key Expressions:* URL: `https://gmail.googleapis.com/gmail/v1/users/me/messages/{{ $('Payment Emails').item.json.id }}` (Query: `format=full`)
    - *Connections:* Input: `Get Payment Email` | Output: `Extract Email Body`
  - **Extract Email Body** (`n8n-nodes-base.code`)
    - *Role:* Decodes base64url-encoded MIME parts, handles nested multi-part messages, and extracts plain-text bodies or fallbacks.
    - *Key Expressions:* Processes `message.payload` using custom base64url decoding functions.
    - *Connections:* Input: `Get Full Message` | Output: `Prepare Email`
  - **Prepare Email** (`n8n-nodes-base.set`)
    - *Role:* Structures and maps email properties into standard properties (`message_id`, `thread_id`, `subject`, `sender`, `email_body`, `Received timestamp`).
    - *Key Expressions:* `{{ $('Payment Emails').item.json.id }}`, `{{ $json.email_body }}`
    - *Connections:* Input: `Extract Email Body` | Output: `Extract Transaction`

#### 2.2 AI Transaction Extraction
**Overview:** Sends email content and metadata to Google Gemini to extract transaction details and safely parses the response back into structured JSON data.
- **Nodes Involved:** `Extract Transaction`, `Parse Gemini Transaction`
- **Node Details:**
  - **Extract Transaction** (`@n8n/n8n-nodes-langchain.googleGemini`)
    - *Role:* Interacts with Google Gemini to extract structured financial metrics (merchant, amount, currency, timestamps, references) following strict prompt rules.
    - *Configuration:* Model: `models/gemini-3.5-flash-lite`
    - *Key Expressions:* Injects `sender`, `subject`, `Received timestamp`, and `email_body` into the prompt.
    - *Connections:* Input: `Prepare Email` | Output: `Parse Gemini Transaction`
    - *Edge Cases/Failures:* Model unavailability, rate limits, or hallucinations returning invalid JSON.
  - **Parse Gemini Transaction** (`n8n-nodes-base.code`)
    - *Role:* Cleans markdown code fences from the AI's response, parses the text into a JSON object, and merges it with original email fields.
    - *Key Expressions:* `const geminiText = $json.content?.parts?.[0]?.text;`
    - *Connections:* Input: `Extract Transaction` | Output: `Transaction Valid?`
    - *Edge Cases/Failures:* Throws an error if the Gemini response is empty or contains malformed JSON.

#### 2.3 Validation & Transaction Preparation
**Overview:** Validates extracted payment details, normalizes fields (amounts, currencies, merchants), and generates a deterministic transaction ID hash.
- **Nodes Involved:** `Transaction Valid?`, `Normalize Transaction`, `Generate Transaction ID`, `Update row in sheet`
- **Node Details:**
  - **Transaction Valid?** (`n8n-nodes-base.if`)
    - *Role:* Splits execution based on whether the extracted data represents a verified payment confirmation with all required fields.
    - *Key Expressions:* Checks if `is_payment_confirmation === true` and required string/date/amount fields exist.
    - *Connections:* Input: `Parse Gemini Transaction` | Outputs: True branch to `Normalize Transaction`, False branch to `Update row in sheet`.
  - **Normalize Transaction** (`n8n-nodes-base.code`)
    - *Role:* Cleans, trims, and standardizes strings, currencies, amounts (to 2 decimal places), and ISO dates/timestamps.
    - *Connections:* Input: `Transaction Valid?` (True) | Output: `Generate Transaction ID`
  - **Generate Transaction ID** (`n8n-nodes-base.code`)
    - *Role:* Combines message identifiers, normalized merchant names, amounts, currencies, and timestamps to build a unique cryptographic hash ID (`TXN-[hash]`).
    - *Connections:* Input: `Normalize Transaction` | Output: `Get Recent Transactions`
  - **Update row in sheet** (`n8n-nodes-base.googleSheets`)
    - *Role:* Logs invalid or non-payment confirmation extractions back to Google Sheets with failure reasons.
    - *Configuration:* Operation: `update`, Matching Column: `email_message_id`
    - *Connections:* Input: `Transaction Valid?` (False) | Output: None

#### 2.4 Duplicate Detection
**Overview:** Pulls recent ledger records from Google Sheets and performs programmatic duplicate checking against the current transaction.
- **Nodes Involved:** `Get Recent Transactions`, `Detect Duplicate`
- **Node Details:**
  - **Get Recent Transactions** (`n8n-nodes-base.googleSheets`)
    - *Role:* Retrieves existing rows from the target spreadsheet to supply historical data for comparison.
    - *Configuration:* Operation: `get` (or default read list), Sheet: `Transactions`
    - *Connections:* Input: `Generate Transaction ID` | Output: `Detect Duplicate`
  - **Detect Duplicate** (`n8n-nodes-base.code`)
    - *Role:* Compares the current transaction against all retrieved records, checking for matching merchants, currencies, exact amounts, and a transaction window within 48 hours.
    - *Connections:* Input: `Get Recent Transactions` | Output: `Potential Duplicate?`

#### 2.5 Transaction Logging
**Overview:** Evaluates the duplicate detection outcome and directs execution to log either a normal or duplicate transaction entry into Google Sheets.
- **Nodes Involved:** `Potential Duplicate?`, `Log Duplicate Transaction`, `Log Normal Transaction`
- **Node Details:**
  - **Potential Duplicate?** (`n8n-nodes-base.if`)
    - *Role:* Determines if the duplicate status is marked as a potential duplicate.
    - *Key Expressions:* `{{ $json.duplicate_status }}` equals `POTENTIAL_DUPLICATE`
    - *Connections:* Input: `Detect Duplicate` | Outputs: True branch to `Log Duplicate Transaction`, False branch to `Log Normal Transaction`.
  - **Log Duplicate Transaction** (`n8n-nodes-base.googleSheets`)
    - *Role:* Appends the transaction details and duplicate confidence metadata to the Google Sheet.
    - *Configuration:* Operation: `append`, Sheet: `Transactions`
    - *Connections:* Input: `Potential Duplicate?` (True) | Output: `Duplicate Charge Alert`
  - **Log Normal Transaction** (`n8n-nodes-base.googleSheets`)
    - *Role:* Appends standard transaction details with a `NORMAL` duplicate status to the Google Sheet.
    - *Configuration:* Operation: `append`, Sheet: `Transactions`
    - *Connections:* Input: `Potential Duplicate?` (False) | Output: `Mark Processed`

#### 2.6 Alert & Completion
**Overview:** Sends a notification message via Slack when a duplicate charge is flagged and updates the email status in Gmail.
- **Nodes Involved:** `Duplicate Charge Alert`, `Mark Processed`
- **Node Details:**
  - **Duplicate Charge Alert** (`n8n-nodes-base.slack`)
    - *Role:* Posts an alert message to a designated Slack channel containing transaction details, confidence scores, and reasoning.
    - *Configuration:* Select: `channel`, Channel ID: `n8n-general`
    - *Connections:* Input: `Log Duplicate Transaction` | Output: `Mark Processed`
    - *Edge Cases/Failures:* Slack authentication errors, missing channel permissions.
  - **Mark Processed** (`n8n-nodes-base.gmail`)
    - *Role:* Adds a specific processed label to the original email thread to prevent reprocessing.
    - *Configuration:* Operation: `addLabels`, Label IDs: `Label_7577227223451520237`, Message ID: `{{ $('Payment Emails').item.json.id }}`
    - *Connections:* Inputs: `Duplicate Charge Alert`, `Log Normal Transaction` | Output: None

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Payment Emails** | `gmailTrigger` | Polls inbox for receipt emails | None | Get Payment Email | Email Intake & Preparation |
| **Get Payment Email** | `gmail` | Fetches base message metadata | Payment Emails | Get Full Message | Email Intake & Preparation |
| **Get Full Message** | `httpRequest` | Fetches full MIME message payload | Get Payment Email | Extract Email Body | Email Intake & Preparation |
| **Extract Email Body** | `code` | Decodes base64url text/HTML bodies | Get Full Message | Prepare Email | Email Intake & Preparation |
| **Prepare Email** | `set` | Structures core email parameters | Extract Email Body | Extract Transaction | Email Intake & Preparation |
| **Extract Transaction** | `googleGemini` | Extracts financial metadata using AI | Prepare Email | Parse Gemini Transaction | AI Transaction Extraction |
| **Parse Gemini Transaction** | `code` | Parses Gemini JSON response | Extract Transaction | Transaction Valid? | AI Transaction Extraction |
| **Transaction Valid?** | `if` | Validates required payment fields | Parse Gemini Transaction | Normalize Transaction, Update row in sheet | Validation & Transaction Preparation |
| **Normalize Transaction** | `code` | Normalizes extracted data values | Transaction Valid? | Generate Transaction ID | Validation & Transaction Preparation |
| **Generate Transaction ID** | `code` | Generates a deterministic hash ID | Normalize Transaction | Get Recent Transactions | Validation & Transaction Preparation |
| **Update row in sheet** | `googleSheets` | Logs invalid/non-payment extractions | Transaction Valid? | None | Validation & Transaction Preparation |
| **Get Recent Transactions** | `googleSheets` | Retrieves ledger history | Generate Transaction ID | Detect Duplicate | Duplicate Detection |
| **Detect Duplicate** | `code` | Compares current transaction against history | Get Recent Transactions | Potential Duplicate? | Duplicate Detection |
| **Potential Duplicate?** | `if` | Branches based on duplicate status | Detect Duplicate | Log Duplicate Transaction, Log Normal Transaction | Transaction Logging |
| **Log Duplicate Transaction** | `googleSheets` | Appends flagged duplicate row | Potential Duplicate? | Duplicate Charge Alert | Transaction Logging |
| **Log Normal Transaction** | `googleSheets` | Appends normal transaction row | Potential Duplicate? | Mark Processed | Transaction Logging |
| **Duplicate Charge Alert** | `slack` | Sends Slack notification | Log Duplicate Transaction | Mark Processed | Alert & Completion |
| **Mark Processed** | `gmail` | Adds processed label to email | Duplicate Charge Alert, Log Normal Transaction | None | Alert & Completion |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:** Add a **Gmail Trigger** (`Payment Emails`) set to poll every minute with the filter query `receipt`.
2. **Retrieve Message Metadata:** Add a **Gmail** node (`Get Payment Email`) configured with the `get` operation using Message ID `{{$json.id}}`.
3. **Fetch Full Payload:** Add an **HTTP Request** node (`Get Full Message`) pointing to `https://gmail.googleapis.com/gmail/v1/users/me/messages/{{ $('Payment Emails').item.json.id }}` using Predefined Gmail OAuth2 authentication with query parameter `format=full`.
4. **Extract Body Content:** Add a **Code** node (`Extract Email Body`) implementing base64url decoding logic to extract plain-text or HTML email bodies.
5. **Prepare Email Payload:** Add a **Set** (`Prepare Email`) node to map parameters: `message_id`, `thread_id`, `subject`, `sender`, `email_body`, and `Received timestamp`.
6. **AI Extraction:** Add a **Google Gemini Chat Model** / AI node (`Extract Transaction`) configured with model `models/gemini-3.5-flash-lite` and the transaction extraction prompt template referencing email metadata and body.
7. **Parse AI Response:** Add a **Code** node (`Parse Gemini Transaction`) to sanitize markdown code blocks and parse the JSON response.
8. **Validate Extraction:** Add an **If** node (`Transaction Valid?`) to check if `is_payment_confirmation === true` alongside populated mandatory fields.
9. **Handle Invalid Transactions:** Connect the false branch to a **Google Sheets** node (`Update row in sheet`) configured to update records matched by `email_message_id`.
10. **Normalize Data:** Connect the true branch to a **Code** node (`Normalize Transaction`) to clean and standardize amounts, currencies, merchants, and dates.
11. **Generate Transaction ID:** Add a **Code** node (`Generate Transaction ID`) to produce a deterministic hash string (`TXN-[hash]`) from transaction attributes.
12. **Fetch Ledger History:** Add a **Google Sheets** node (`Get Recent Transactions`) connected to the target spreadsheet and sheet named `Transactions`.
13. **Run Duplicate Detection:** Add a **Code** node (`Detect Duplicate`) to compare the current transaction against fetched sheet records within a 48-hour window.
14. **Evaluate Duplicate Status:** Add an **If** node (`Potential Duplicate?`) evaluating whether `duplicate_status` equals `POTENTIAL_DUPLICATE`.
15. **Log Transactions:**
    - Connect the true branch to a **Google Sheets** node (`Log Duplicate Transaction`) set to `append`.
    - Connect the false branch to a **Google Sheets** node (`Log Normal Transaction`) set to `append`.
16. **Send Alert & Cleanup:**
    - Connect the duplicate logging branch to a **Slack** node (`Duplicate Charge Alert`) configured to post messages to your target channel.
    - Connect both the Slack alert output and the normal logging output to a final **Gmail** node (`Mark Processed`) configured with `addLabels` to tag the email with your processed label ID.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ensure all connected OAuth2 accounts (Gmail, Google Sheets, Slack) and API credentials (Google Gemini) are active and granted correct scopes before activating the workflow. | Credential Setup |
| Customize the duplicate detection time window (set to 48 hours by default) inside the `Detect Duplicate` code node if your business logic requires a broader or narrower review window. | Configuration Note |
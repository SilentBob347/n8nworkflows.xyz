Track and approve expense receipts with Gmail, Google Gemini, Sheets and Slack

https://n8nworkflows.xyz/workflows/track-and-approve-expense-receipts-with-gmail--google-gemini--sheets-and-slack-20785


# Track and approve expense receipts with Gmail, Google Gemini, Sheets and Slack

### 1. Workflow Overview

The **AI Receipt and Expense Tracker** automates the end-to-end lifecycle of corporate expense management. It monitors incoming emails for receipts, extracts structured data using a multimodal AI model, validates the extracted information, logs or updates entries in Google Sheets, routes approvals based on spending limits, and generates scheduled monthly summaries for the finance department.

The logical execution splits into two distinct processing flows:
- **1.1 Real-Time Expense Ingestion & Processing:** Triggers on new Gmail messages with attachments, parses receipt images using Google Gemini, validates fields via JavaScript, records data in Google Sheets, and triggers notifications via Slack (for high-value items) or direct email confirmations.
- **1.2 Scheduled Monthly Financial Reporting:** Triggers on a monthly schedule, aggregates all expense rows from Google Sheets, computes departmental/category totals and highest expenditures, and emails a formatted HTML report to the finance team.

---

### 2. Block-by-Block Analysis

#### Block 1: Real-Time Expense Ingestion & Processing

##### Overview
This block intercepts incoming expense receipts via email, passes them to a vision-capable AI model for data extraction, sanitizes and validates the raw JSON response, assigns a unique expense ID, and persists the record to a tracking spreadsheet.

##### Nodes Involved
- `Gmail Trigger`
- `Analyze an image`
- `Code in JavaScript`
- `Code in JavaScript1`
- `Append or update row in sheet`

##### Node Details

###### `Gmail Trigger`
- **Type and Technical Role:** `n8n-nodes-base.gmailTrigger` (Trigger Node)
- **Configuration Choices:** Polls Gmail every minute using the search query `has:attachment (receipt OR invoice OR expense)`. Configured to automatically download attachments with the prefix `attachment_`.
- **Key Expressions or Variables:** None (Trigger node).
- **Input and Output Connections:** Input: None; Output: Connects to `Analyze an image`.
- **Version-specific Requirements:** Version 1.4+. Requires active Gmail OAuth2 credentials.
- **Edge Cases / Potential Failure Types:** Authentication token expiry, API rate limits, or missing binary data if the attachment is embedded inline rather than attached.

###### `Analyze an image`
- **Type and Technical Role:** `@n8n/n8n-nodes-langchain.googleGemini` (AI / Action Node)
- **Configuration Choices:** Uses model `models/gemini-2.5-flash`. Operation is set to `analyze` using input type `binary` and property `attachment_0`. Employs a strict prompt demanding a valid JSON response containing `vendor`, `date`, `amount`, `tax`, `currency`, `category`, `employee`, `payment_method`, `receipt_number`, `description`, and `confidence`.
- **Key Expressions or Variables:** Reads binary property `{{ $binary.attachment_0 }}` implicitly via node configuration.
- **Input and Output Connections:** Input: `Gmail Trigger`; Output: Connects to `Code in JavaScript`.
- **Version-specific Requirements:** Version 1.2+. Requires Google Gemini (PaLM) API credentials.
- **Edge Cases / Potential Failure Types:** AI output containing markdown code blocks (e.g., ```json ... ```) requiring downstream parsing, or low confidence scores leading to ambiguous fields.

###### `Code in JavaScript`
- **Type and Technical Role:** `n8n-nodes-base.code` (Data Transformation Node)
- **Configuration Choices:** Processes input items to extract raw text from `item.json.content.parts[0].text`. Strips markdown code blocks and parses the text into a clean JSON object.
- **Key Expressions or Variables:** `const text = item.json.content.parts[0].text;`
- **Input and Output Connections:** Input: `Analyze an image`; Output: Connects to `Code in JavaScript1`.
- **Version-specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Throws runtime errors if the AI returns non-string content, malformed JSON, or unexpected array structures.

###### `Code in JavaScript1`
- **Type and Technical Role:** `n8n-nodes-base.code` (Data Transformation Node)
- **Configuration Choices:** Runs once per item. Sanitizes textual attributes, converts financial amounts and tax values to clean numbers (stripping currency symbols and commas), validates date formats (`YYYY-MM-DD`), enforces allowed category/payment method enums, and computes business flags.
- **Key Expressions or Variables:** 
  - `const senderEmail = asText(gmail.from.value[0].address);`
  - `const expenseId = 'EXP-' + $execution.id + '-' + $runIndex + '-' + $itemIndex;`
- **Input and Output Connections:** Input: `Code in JavaScript`; Output: Connects to `Append or update row in sheet`.
- **Version-specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Missing sender address in Gmail trigger payload or invalid date formats causing validation errors to flag.

###### `Append or update row in sheet`
- **Type and Technical Role:** `n8n-nodes-base.googleSheets` (Action Node)
- **Configuration Choices:** Operation set to `appendOrUpdate`. Matches rows based on the `Expense ID` column and writes mapped values for `Amount`, `Tax`, `Vendor`, `Category`, `Currency`, `Employee`, `Over Limit`, `Description`, `Processed At`, `Receipt Date`, `AI Confidence`, `Expense Limit`, `Payment Method`, `Receipt Number`, `Approval Status`, and `Validation Status`.
- **Key Expressions or Variables:** `={{ $json.expense_id }}`, `={{ $json.amount }}`
- **Input and Output Connections:** Input: `Code in JavaScript1`; Output: Connects to `If`.
- **Version-specific Requirements:** Version 4.7+. Requires Google Sheets OAuth2 credentials.
- **Edge Cases / Potential Failure Types:** Schema mismatch between the spreadsheet columns and the mapped node parameters, or Google API rate limits.

---

#### Block 2: Approval Routing & Notifications

##### Overview
This block evaluates whether a processed expense exceeds the configured spending limit. If the limit is exceeded, it notifies the finance team via Slack and sends a confirmation email to the employee. If within the limit, it skips the Slack alert and emails the employee directly.

##### Nodes Involved
- `If`
- `Send a message1` (Slack Notification)
- `Send a message2` (Post-Approval Employee Email)
- `Send a message` (Within-Limit Employee Email)

##### Node Details

###### `If`
- **Type and Technical Role:** `n8n-nodes-base.if` (Flow Control Node)
- **Configuration Choices:** Evaluates a single condition: checks if `{{ $json.Amount }}` is greater than `{{ $json['Expense Limit'] }}`.
- **Key Expressions or Variables:** `={{ $json.Amount }} > {{ $json['Expense Limit'] }}`
- **Input and Output Connections:** Input: `Append or update row in sheet`; Output True: Connects to `Send a message1` (Slack). Output False: Connects to `Send a message` (Email).
- **Version-specific Requirements:** Version 2.3.
- **Edge Cases / Potential Failure Types:** Type coercion mismatches if spreadsheet values are read as strings rather than numbers.

###### `Send a message1` (Slack)
- **Type and Technical Role:** `n8n-nodes-base.slack` (Action Node)
- **Configuration Choices:** Posts an alert message to a designated Slack channel (`n8n-finance`) detailing expense attributes, employee name, vendor, and approval status.
- **KeyExpressions or Variables:** Uses template strings referencing `$json['Expense ID']`, `$json.Employee`, `$json.Amount`, etc.
- **Input and Output Connections:** Input: `If` (True branch); Output: Connects to `Send a message2`.
- **Version-specific Requirements:** Version 2.5+. Requires Slack Bot OAuth2 credentials.
- **Edge Cases / Potential Failure Types:** Invalid channel ID or missing bot scopes (`chat:write`).

###### `Send a message2` (Gmail - Post-Approval)
- **Type and Technical Role:** `n8n-nodes-base.gmail` (Action Node)
- **Configuration Choices:** Sends an HTML-formatted confirmation email to the employee stating that the expense has been logged and flagged for finance review.
- **Key Expressions or Variables:** Uses fixed referencing expressions tied to the upstream sheet node: `{{ $('Append or update row in sheet').item.json.Employee }}`.
- **Input and Output Connections:** Input: `Send a message1`; Output: None (Terminal node for this branch).
- **Version-specific Requirements:** Version 2.2+. Requires Gmail OAuth2 credentials.
- **Edge Cases / Potential Failure Types:** Invalid recipient email address or malformed HTML table structures.

###### `Send a message` (Gmail - Within Limit)
- **Type and Technical Role:** `n8n-nodes-base.gmail` (Action Node)
- **Configuration Choices:** Sends an HTML-formatted confirmation email to the employee confirming that the expense is within limits and successfully recorded.
- **Key Expressions or Variables:** `={{ $json.Employee }}`, `={{ $json['Expense ID'] }}`
- **Input and Output Connections:** Input: `If` (False branch); Output: None (Terminal node).
- **Version-specific Requirements:** Version 2.2+. Requires Gmail OAuth2 credentials.
- **Edge Cases / Potential Failure Types:** Undefined employee email property.

---

#### Block 3: Scheduled Monthly Financial Reporting

##### Overview
This block triggers on a monthly schedule, retrieves all expense ledger entries from Google Sheets, aggregates totals and category/employee breakdowns via JavaScript, and emails a comprehensive report to the finance team.

##### Nodes Involved
- `Schedule Trigger`
- `Get row(s) in sheet`
- `Code in JavaScript2`
- `Send a message3` (Monthly Finance Report Email)

##### Node Details

###### `Schedule Trigger`
- **Type and Technical Role:** `n8n-nodes-base.scheduleTrigger` (Trigger Node)
- **Configuration Choices:** Configured to execute monthly at 09:00 AM.
- **Key Expressions or Variables:** None (Trigger node).
- **Input and Output Connections:** Input: None; Output: Connects to `Get row(s) in sheet`.
- **Version-specific Requirements:** Version 1.3+.
- **Edge Cases / Potential Failure Types:** Workflow must be active for scheduled triggers to fire.

###### `Get row(s) in sheet`
- **Type and Technical Role:** `n8n-nodes-base.googleSheets` (Action Node)
- **Configuration Choices:** Operation set to retrieve all rows from the specified spreadsheet document and sheet.
- **Key Expressions or Variables:** Uses document ID and sheet name references.
- **Input and Output Connections:** Input: `Schedule Trigger`; Output: Connects to `Code in JavaScript2`.
- **Version-specific Requirements:** Version 4.7+. Requires Google Sheets OAuth2 credentials.
- **Edge Cases / Potential Failure Types:** Large datasets may approach execution memory limits or Google API pagination limits.

###### `Code in JavaScript2`
- **Type and Technical Role:** `n8n-nodes-base.code` (Data Transformation Node)
- **Configuration Choices:** Iterates through all expense rows, parses financial figures safely, calculates total expenses, total amount, total tax, average expense, over-limit counts, category totals, employee totals, and identifies the single highest expense item.
- **Key Expressions or Variables:** Uses iterative aggregation loops over `$input.all()`.
- **Input and Output Connections:** Input: `Get row(s) in sheet`; Output: Connects to `Send a message3`.
- **Version-specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Empty spreadsheet datasets resulting in division-by-zero or missing property errors.

###### `Send a message3` (Gmail - Monthly Report)
- **Type and Technical Role:** `n8n-nodes-base.gmail` (Action Node)
- **Configuration Choices:** Sends an HTML report to a hardcoded finance email address (`user@example.com`) containing summary metrics, category breakdowns, employee breakdowns, and highest expense details.
- **Key Expressions or Variables:** Uses template literals embedded in HTML for table generation: `{{ $json.report_month }}`, `{{ $json.total_expenses }}`, etc.
- **Input and Output Connections:** Input: `Code in JavaScript2`; Output: None (Terminal node).
- **Version-specific Requirements:** Version 2.2+. Requires Gmail OAuth2 credentials.
- **Edge Cases / Potential Failure Types:** Placeholder recipient email (`user@example.com`) must be updated prior to production deployment.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Gmail Trigger` | `n8n-nodes-base.gmailTrigger` | Monitors inbox for receipt emails | None | `Analyze an image` | ## Gmail Trigger, AI modal<br>1. Trigger gmail every minutes and fetch latest one email with attachment<br>2. Google gemini pass attachment with image analysis modal and extract data from that |
| `Analyze an image` | `@n8n/n8n-nodes-langchain.googleGemini` | Extracts expense data from receipt images | `Gmail Trigger` | `Code in JavaScript` | ## Gmail Trigger, AI modal<br>1. Trigger gmail every minutes and fetch latest one email with attachment<br>2. Google gemini pass attachment with image analysis modal and extract data from that |
| `Code in JavaScript` | `n8n-nodes-base.code` | Parses and cleans AI text output | `Analyze an image` | `Code in JavaScript1` | ## Code , Google Sheet<br>1. Code node extract gemini data proper formate with required filed expense id, category, vendor, amount, etc<br>2. Google sheet update or append row based on Expense Id |
| `Code in JavaScript1` | `n8n-nodes-base.code` | Validates, normalizes, and assigns expense metadata | `Code in JavaScript` | `Append or update row in sheet` | ## Code , Google Sheet<br>1. Code node extract gemini data proper formate with required filed expense id, category, vendor, amount, etc<br>2. Google sheet update or append row based on Expense Id |
| `Append or update row in sheet` | `n8n-nodes-base.googleSheets` | Logs expense record into Google Sheets | `Code in JavaScript1` | `If` | ## Code , Google Sheet<br>1. Code node extract gemini data proper formate with required filed expense id, category, vendor, amount, etc<br>2. Google sheet update or append row based on Expense Id |
| `If` | `n8n-nodes-base.if` | Evaluates if expense exceeds approval limit | `Append or update row in sheet` | `Send a message1`, `Send a message` | ## IF, Slack and Gmail<br>1. If Receipt amount is grater then limit amount then send message in slack and gmail with basic information<br>2. If Receipt amount is less then limit amount then direct email for approval |
| `Send a message1` | `n8n-nodes-base.slack` | Alerts finance channel on high-value expenses | `If` (True) | `Send a message2` | ## IF, Slack and Gmail<br>1. If Receipt amount is grater then limit amount then send message in slack and gmail with basic information<br>2. If Receipt amount is less then limit amount then direct email for approval |
| `Send a message2` | `n8n-nodes-base.gmail` | Sends confirmation email to employee (over limit) | `Send a message1` | None | ## IF, Slack and Gmail<br>1. If Receipt amount is grater then limit amount then send message in slack and gmail with basic information<br>2. If Receipt amount is less then limit amount then direct email for approval |
| `Send a message` | `n8n-nodes-base.gmail` | Sends confirmation email to employee (within limit) | `If` (False) | None | ## IF, Slack and Gmail<br>1. If Receipt amount is grater then limit amount then send message in slack and gmail with basic information<br>2. If Receipt amount is less then limit amount then direct email for approval |
| `Schedule Trigger` | `n8n-nodes-base.scheduleTrigger` | Triggers monthly reporting execution | None | `Get row(s) in sheet` | ##  Trigger, Sheet, Code, Send Email<br>1. Trigger schedule every month 9 AM and fetch data from sheet<br>2. Code node prepare monthly report with highest amount and highest category name and vender information and send to email finance department |
| `Get row(s) in sheet` | `n8n-nodes-base.googleSheets` | Retrieves all expense rows from sheet | `Schedule Trigger` | `Code in JavaScript2` | ##  Trigger, Sheet, Code, Send Email<br>1. Trigger schedule every month 9 AM and fetch data from sheet<br>2. Code node prepare monthly report with highest amount and highest category name and vender information and send to email finance department |
| `Code in JavaScript2` | `n8n-nodes-base.code` | Aggregates monthly totals and metrics | `Get row(s) in sheet` | `Send a message3` | ##  Trigger, Sheet, Code, Send Email<br>1. Trigger schedule every month 9 AM and fetch data from sheet<br>2. Code node prepare monthly report with highest amount and highest category name and vender information and send to email finance department |
| `Send a message3` | `n8n-nodes-base.gmail` | Emails monthly report summary to finance | `Code in JavaScript2` | None | ##  Trigger, Sheet, Code, Send Email<br>1. Trigger schedule every month 9 AM and fetch data from sheet<br>2. Code node prepare monthly report with highest amount and highest category name and vender information and send to email finance department |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Initial Trigger:**
   - Add a **Gmail Trigger** node (`n8n-nodes-base.gmailTrigger`). Configure polling to every minute, set search query to `has:attachment (receipt OR invoice OR expense)`, and enable attachment downloading with prefix `attachment_`. Connect Gmail OAuth2 credentials.
2. **Add Vision AI Processing:**
   - Create a **Google Gemini** node (`@n8n/n8n-nodes-langchain.googleGemini`). Set resource to `image`, operation to `analyze`, model to `models/gemini-2.5-flash`, and binary property name to `attachment_0`. Provide the extraction prompt requesting strict JSON format for vendor, date, amount, tax, currency, category, employee, payment method, receipt number, description, and confidence. Connect Google Gemini API credentials.
3. **Parse AI Output:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Code in JavaScript`. Insert the parsing logic to strip markdown code fences and parse the text content into structured JSON items.
4. **Normalize and Validate Data:**
   - Add a second **Code** node named `Code in JavaScript1`. Insert the validation logic that normalizes currencies, validates `YYYY-MM-DD` date structures, applies fallback employee mapping to the Gmail sender, computes unique expense IDs (`EXP-<execution_id>-<run_index>-<item_index>`), and flags expenses exceeding the 500 limit.
5. **Persist Record to Spreadsheet:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) configured for `appendOrUpdate`. Set matching column to `Expense ID`. Map all extracted schema fields (`Expense ID`, `Employee`, `Vendor`, `Receipt Date`, `Amount`, `Tax`, `Currency`, `Category`, `Payment Method`, `Receipt Number`, `Description`, `AI Confidence`, `Expense Limit`, `Over Limit`, `Approval Status`, `Validation Status`, `Validation Errors`, `Processed At`). Connect Google Sheets OAuth2 credentials.
6. **Implement Conditional Approval Routing:**
   - Add an **If** node (`n8n-nodes-base.if`). Configure a numeric condition checking if `{{ $json.Amount }}` is greater than `{{ $json['Expense Limit'] }}`.
7. **Configure High-Value Alerting (True Branch):**
   - Connect the True branch to a **Slack** node (`n8n-nodes-base.slack`) configured to post a formatted approval-required message to the finance channel (e.g., `n8n-finance`). Connect Slack Bot credentials.
   - Connect the Slack node output to a **Gmail** node (`n8n-nodes-base.gmail`) configured to send an HTML-formatted receipt processing notification to the employee using upstream sheet references.
8. **Configure Standard Notification (False Branch):**
   - Connect the False branch of the **If** node to a **Gmail** node configured to send an HTML-formatted receipt confirmation email directly to the employee.
9. **Configure Scheduled Monthly Reporting:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`) set to execute monthly at 9:00 AM.
   - Connect it to a **Google Sheets** node configured to get all rows from the expense tracker sheet.
   - Connect the sheet output to a **Code** node that aggregates totals, taxes, averages, category breakdowns, employee breakdowns, and highest expense items.
   - Connect the aggregation code node to a final **Gmail** node configured to send the HTML monthly expense report to the finance team recipient address.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| AI Receipt & Expense Tracker Solution Overview | Automated end-to-end receipt extraction, validation, approval routing, and monthly reporting workflow. |
| Target Spreadsheet Setup | Requires a Google Sheet configured with headers matching the mapped parameter keys (Expense ID, Employee, Vendor, Receipt Date, Amount, etc.). |
| Placeholder Recipient Notice | The monthly finance summary email defaults to `user@example.com` and must be updated to an active internal finance distribution list prior to activation. |
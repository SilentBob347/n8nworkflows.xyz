Send a daily cashwerk invoice summary report using the cashwerk API and SMTP email

https://n8nworkflows.xyz/workflows/send-a-daily-cashwerk-invoice-summary-report-using-the-cashwerk-api-and-smtp-email-20997


# Send a daily cashwerk invoice summary report using the cashwerk API and SMTP email

### 1. Workflow Overview

This workflow is an automated reporting solution designed to fetch daily invoice metrics from the Cashwerk platform, compile them into a formatted HTML summary, and email the report to a finance inbox. By executing daily, it eliminates the need for manual platform checks and provides stakeholders with an immediate overview of daily invoicing volumes, gross revenues, and status breakdowns.

The execution logic is divided into three consecutive functional blocks:
- **1.1 Schedule & Retrieval:** Triggers the automation daily at 08:00 and retrieves up to 100 recent invoices from Cashwerk.
- **1.2 Data Processing & HTML Generation:** Parses the incoming invoice items, aggregates gross amounts (ignoring non-financial documents), counts invoices by status, and constructs a structured HTML summary table.
- **1.3 Email Delivery:** Formats the final email subject with the current date and sends the HTML payload via an SMTP server.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Schedule & Retrieval
- **Overview:** Initiates the workflow on a fixed daily schedule and requests the latest invoice dataset from the Cashwerk API using API key authentication.
- **Nodes Involved:** 
  - `Every day 08:00`
  - `Get Invoices`

##### Node Details: `Every day 08:00`
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger` (Trigger)
- **Configuration choices:** Configured to fire once daily at 08:00 in the instance's local timezone.
- **Key expressions or variables used:** None.
- **Input and output connections:** 
  - Input: None (Entry point)
  - Output: Connects to `Get Invoices`
- **Version-specific requirements:** Version 1.4.
- **Edge cases or potential failure types:** Instance downtime or timezone misconfigurations can delay execution.

##### Node Details: `Get Invoices`
- **Type and technical role:** `n8n-nodes-cashwerk.cashwerk` (Action / API Integration)
- **Configuration choices:** Resource set to `invoice`, operation set to `getAll`, limit capped at `100`, `returnAll` disabled, and authentication configured via API Key (`apiKey`).
- **Key expressions or variables used:** None.
- **Input and output connections:** 
  - Input: `Every day 08:00`
  - Output: Connects to `Build Report`
- **Version-specific requirements:** Requires the community node package `n8n-nodes-cashwerk`.
- **Edge cases or potential failure types:** API rate limits, invalid API credentials, or network timeouts when communicating with Cashwerk.

---

#### Block 1.2: Data Processing & HTML Generation
- **Overview:** Receives raw invoice objects, calculates gross totals while safely handling null values for non-financial documents, computes status distributions, and outputs a ready-to-render HTML string.
- **Nodes Involved:** 
  - `Build Report`

##### Node Details: `Build Report`
- **Type and technical role:** `n8n-nodes-base.code` (Data Transformation / JavaScript execution)
- **Configuration choices:** Executes custom JavaScript to map items, sum gross amounts using `de-DE` localized currency formatting, categorize statuses, slice the array to the top 25 records for the table display, and generate a styled HTML string.
- **Key expressions or variables used:** `$input.all()`
- **Input and output connections:** 
  - Input: `Get Invoices`
  - Output: Connects to `Send Email`
- **Version-specific requirements:** Version 2.
- **Edge cases or potential failure types:** Script failures if the API response schema changes unexpectedly (e.g., missing expected fields like `totalGross` or `status`).

---

#### Block 1.3: Email Delivery
- **Overview:** Takes the processed HTML payload and delivers the summary report to the designated recipient via SMTP with a dynamically generated date stamp in the subject line.
- **Nodes Involved:** 
  - `Send Email`

##### Node Details: `Send Email`
- **Type and technical role:** `n8n-nodes-base.emailSend` (Notification / SMTP Integration)
- **Configuration choices:** Configured for HTML email format (`emailFormat: html`), static recipient and sender addresses, and dynamic subject line rendering.
- **Key expressions or variables used:** 
  - HTML body: `={{ $json.html }}`
  - Subject: `=cashwerk – daily invoice summary {{ $now.format('yyyy-MM-dd') }}`
- **Input and output connections:** 
  - Input: `Build Report`
  - Output: None (Terminal node)
- **Version-specific requirements:** Version 2.1. Requires valid SMTP credentials.
- **Edge cases or potential failure types:** SMTP connection refusal, authentication failures, rejected recipient addresses, or exceeding mail server rate limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Every day 08:00` | `n8n-nodes-base.scheduleTrigger` | Triggers workflow daily at 08:00 | None | `Get Invoices` | ## Schedule<br>Fires once a day at 08:00 in the instance timezone. Change the hour here. |
| `Get Invoices` | `n8n-nodes-cashwerk.cashwerk` | Fetches up to 100 invoices from Cashwerk | `Every day 08:00` | `Build Report` | ## Read and summarise<br>Reads the newest invoices and builds the HTML table. Gross totals arrive computed from cashwerk; the Code node only adds them up and counts the statuses. |
| `Build Report` | `n8n-nodes-base.code` | Aggregates totals and generates the HTML summary table | `Get Invoices` | `Send Email` | ## Read and summarise<br>Reads the newest invoices and builds the HTML table. Gross totals arrive computed from cashwerk; the Code node only adds them up and counts the statuses. |
| `Send Email` | `n8n-nodes-base.emailSend` | Sends the HTML summary report via SMTP | `Build Report` | None | ## Deliver<br>Sends the HTML with the date in the subject. Set sender, recipient and SMTP here. |

*(Note: The overview instructions provided in the original workflow's primary sticky note apply globally to setup, operational limits, and architecture, and are detailed in sections 1 and 5.)*

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in your n8n instance and name it `Send a daily cashwerk invoice summary report via email`.
2. **Add the Schedule Trigger node:**
   - Type: `Schedule Trigger` (Node name: `Every day 08:00`)
   - Configuration: Set the interval trigger to fire at hour `8`.
3. **Add the Cashwerk node:**
   - Type: `Cashwerk` (Node name: `Get Invoices`)
   - Configuration: Set Resource to `invoice`, Operation to `getAll`, Limit to `100`, and `Return All` to `false`.
   - Credentials: Create or select a `cashwerkApi` credential configured with your API key from Cashwerk (`Settings → API Platform`).
   - Connect `Every day 08:00` to `Get Invoices`.
4. **Add the Code node:**
   - Type: `Code` (Node name: `Build Report`)
   - Configuration: Set Mode to `Run Once for All Items` (or default JavaScript mode) and paste the data aggregation and HTML generation script handling gross calculations, status parsing, and table formatting.
   - Connect `Get Invoices` to `Build Report`.
5. **Add the Send Email node:**
   - Type: `Send Email` (Node name: `Send Email`)
   - Configuration: 
     - Set `Email Format` to `HTML`.
     - Set `To Email` to your target finance inbox (e.g., `user@example.com`).
     - Set `From Email` to your sender address (e.g., `noreply@example.com`).
     - Set `Subject` to `=cashwerk – daily invoice summary {{ $now.format('yyyy-MM-dd') }}`.
     - Set `HTML` body to `={{ $json.html }}`.
   - Credentials: Configure standard SMTP credentials for your mail server.
   - Connect `Build Report` to `Send Email`.
6. **Test and Validate:** Run the workflow manually once, verify the output email content in your inbox, and then activate the workflow.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Install community node `n8n-nodes-cashwerk` via Settings → Community nodes. | Required for fetching data from Cashwerk. |
| Generate an API key in Cashwerk under Settings → API Platform and scope it to reading invoices. | Required for `Get Invoices` authentication. |
| The workflow processes up to 100 invoices per run; adjust limits or enable "Return All" if daily invoice volume exceeds this threshold. | Operational limit consideration. |
| Emails are sent regardless of whether invoices were generated (an empty table indicates a quiet day rather than a broken execution). | Behavioral design note. |
Verify Google Sheets lead emails with Apify Email Verifier

https://n8nworkflows.xyz/workflows/verify-google-sheets-lead-emails-with-apify-email-verifier-21192


# Verify Google Sheets lead emails with Apify Email Verifier

### 1. Workflow Overview

This workflow automates the verification of email addresses stored within a Google Sheets document before launching outbound campaigns. It identifies unverified contacts, deduplicates and batches them, performs deep validation via the Apify Email Verifier API, and writes the structured results back to the spreadsheet. 

The logic is organized into three sequential functional blocks:
- **1.1 Input Reception & Preprocessing:** Initiates the workflow manually, reads unverified lead rows from Google Sheets, cleans whitespace, removes duplicate addresses, and splits the contacts into manageable batches.
- **1.2 API Processing & Result Mapping:** Sends the batched email addresses to the Apify API synchronously, handles the response, and normalizes/maps the returned verification verdicts back to the original sheet data structure.
- **1.3 Data Persistence:** Writes the updated statuses, scores, and verification flags back to the corresponding rows in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Preprocessing
- **Overview:** This block triggers the workflow, pulls raw lead data from a Google Spreadsheet, filters out already verified entries, deduplicates the remaining addresses to optimize API costs, and divides them into chunks.
- **Nodes Involved:** `Manual Execution Trigger`, `Read Unverified Rows in Sheets`, `Filter and Batch Rows`

- **Node Details:**
  - **Manual Execution Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` — Acts as the manual starting point for the workflow.
    - *Configuration Choices:* Default configuration.
    - *Input/Output Connections:* Output connects to `Read Unverified Rows in Sheets`.
    - *Edge Cases / Potential Failures:* None (manual trigger).
  - **Read Unverified Rows in Sheets**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.5) — Retrieves all rows from the specified target document and sheet.
    - *Configuration Choices:* Operation set to read/get rows (implicit via input schema). Requires user-configured Document ID and Sheet Name.
    - *Input/Output Connections:* Input from `Manual Execution Trigger`; output connects to `Filter and Batch Rows`.
    - *Edge Cases / Potential Failures:* Google API rate limits, invalid credentials, or missing sheet headers (`email`, `status`, `sub_status`, `score`, `catch_all`, `disposable`, `verified_at`).
  - **Filter and Batch Rows**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript code node that filters unverified, non-empty emails, drops duplicates using a `Set`, and chunks items into arrays of 100.
    - *Configuration Choices:* Defines `BATCH_SIZE = 100`, `EMAIL_COLUMN = 'email'`, and `STATUS_COLUMN = 'status'`.
    - *Key Expressions or Variables:* Uses `$input.all()` to loop over all spreadsheet rows.
    - *Input/Output Connections:* Input from `Read Unverified Rows in Sheets`; output connects to `Post to Apify Email Verifier`.
    - *Edge Cases / Potential Failures:* Memory limits if processing excessively large datasets in a single execution; timeout risks if `BATCH_SIZE` exceeds Apify synchronous run limits.

---

#### 2.2 API Processing & Result Mapping
- **Overview:** Sends the batched email lists to the Apify email verification service and reconstructs the data alignment by mapping the asynchronous domain-grouped responses back to the original row order.
- **Nodes Involved:** `Post to Apify Email Verifier`, `Map Verdicts to Rows`

- **Node Details:**
  - **Post to Apify Email Verifier**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) — Executes a POST request to the Apify synchronous actor run endpoint.
    - *Configuration Choices:* URL set to `https://api.apify.com/v2/acts/sumitr_mardy~email-verifier/run-sync-get-dataset-items`. Method: `POST`. Timeout configured to 300,000ms (5 minutes). Body format: JSON stringified `$json.emails`. Authentication via Header Auth.
    - *Key Expressions/Variables:* `={{ JSON.stringify({ emails: $json.emails }) }}`
    - *Input/Output Connections:* Input from `Filter and Batch Rows`; output connects to `Map Verdicts to Rows`.
    - *Credentials:* Uses `Apify API` (Header Auth: `Authorization: Bearer <TOKEN>`).
    - *Edge Cases / Potential Failures:* API timeouts, invalid authentication tokens, exhausted Apify monthly credits, or payload size limits exceeded.
  - **Map Verdicts to Rows**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript code node that constructs an in-memory lookup map from API responses and correlates them with the original spreadsheet rows.
    - *Configuration Choices:* Normalizes email addresses (lowercasing) to handle case discrepancies and matches responses back to rows retrieved from the `Read Unverified Rows in Sheets` node.
    - *Key Expressions/Variables:* Uses `$('Read Unverified Rows in Sheets').all()` to retrieve initial row states.
    - *Input/Output Connections:* Input from `Post to Apify Email Verifier`; output connects to `Update Sheets with Verdicts`.
    - *Edge Cases / Potential Failures:* Unmatched responses if an email format is altered during transit.

---

#### 2.3 Data Persistence
- **Overview:** Writes the mapped verification metrics and flags back to the designated Google Sheet rows.
- **Nodes Involved:** `Update Sheets with Verdicts`

- **Node Details:**
  - **Update Sheets with Verdicts**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.5) — Updates existing rows in Google Sheets based on matching column values.
    - *Configuration Choices:* Operation set to `update`. Mapping mode set to `autoMapInputData`. Matching column configured as `email`.
    - *Input/Output Connections:* Input from `Map Verdicts to Rows`; no downstream output.
    - *Edge Cases / Potential Failures:* Google Sheets API write limits, permission issues, or mismatching identifiers preventing row updates.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Manual Execution Trigger | n8n-nodes-base.manualTrigger | Initiates workflow execution manually | None | Read Unverified Rows in Sheets | Verify a Google Sheets lead list (Email Verifier)<br><br>How it works<br>This workflow manually processes a Google Sheets lead list to verify email addresses. It reads rows that still need verification, filters and batches eligible email addresses, sends them to an Apify email verification actor, then maps the returned verdicts back to the original sheet rows. Finally, it writes the verification results back into Google Sheets.<br><br>Setup steps<br>- Configure Google Sheets credentials for both the read and write nodes.<br>- Make sure your sheet has these columns: `email`, `status`, `sub_status`, `score`, `catch_all`, `disposable`, `verified_at`.<br>- Create an n8n Header Auth credential named **Apify API** (`Authorization: Bearer <YOUR_APIFY_TOKEN>`).<br>- Confirm the code nodes reference the correct email, verdict, and row identifier fields from your sheet.<br><br>Customization<br>Adjust the batching logic (`BATCH_SIZE = 100`), filtering rules, verdict column names, or Apify actor parameters to match your lead list structure and verification requirements. |
| Read Unverified Rows in Sheets | n8n-nodes-base.googleSheets | Retrieves lead data rows from Google Sheets | Manual Execution Trigger | Filter and Batch Rows | Start and prepare rows<br><br>Manual start, Google Sheets read, and preprocessing code that selects rows with email addresses that do not yet have verification verdicts, then chunks them into batches. |
| Filter and Batch Rows | n8n-nodes-base.code | Cleans, deduplicates, and batches email addresses | Read Unverified Rows in Sheets | Post to Apify Email Verifier | Start and prepare rows<br><br>Manual start, Google Sheets read, and preprocessing code that selects rows with email addresses that do not yet have verification verdicts, then chunks them into batches. |
| Post to Apify Email Verifier | n8n-nodes-base.httpRequest | Sends email batches to Apify API for verification | Filter and Batch Rows | Map Verdicts to Rows | Verify and align results<br><br>Sends batched emails to the Apify verification API and reshapes the response so verdicts are matched back to the correct original rows. |
| Map Verdicts to Rows | n8n-nodes-base.code | Aligns API verification results with original rows | Post to Apify Email Verifier | Update Sheets with Verdicts | Verify and align results<br><br>Sends batched emails to the Apify verification API and reshapes the response so verdicts are matched back to the correct original rows. |
| Update Sheets with Verdicts | n8n-nodes-base.googleSheets | Writes verification verdicts back to Google Sheets | Map Verdicts to Rows | None | Write sheet verdicts<br><br>Updates the Google Sheet with the mapped email verification results. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Point:** Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`).
2. **Add Data Reader:** Create a **Google Sheets** node (`n8n-nodes-base.googleSheets`), connect it to the Manual Trigger, set the operation to read/get rows, and configure your target spreadsheet document and sheet name. Ensure your spreadsheet contains headers: `email`, `status`, `sub_status`, `score`, `catch_all`, `disposable`, and `verified_at`.
3. **Add Preprocessing Logic:** Add a **Code** node (`n8n-nodes-base.code`) connected after the Google Sheets read node. Paste the preprocessing script that filters out empty emails, skips already verified rows, drops duplicates using a JavaScript `Set`, and splits items into batches of 100 (`BATCH_SIZE = 100`).
4. **Configure API Call:** Connect an **HTTP Request** node (`n8n-nodes-base.httpRequest`) to the Code node. 
   - Set Method to `POST`.
   - Set URL to `https://api.apify.com/v2/acts/sumitr_mardy~email-verifier/run-sync-get-dataset-items`.
   - Set Body Content Type to JSON, using expression: `={{ JSON.stringify({ emails: $json.emails }) }}`.
   - Set request timeout to `300000` ms.
   - Configure authentication using a **Header Auth** credential named **Apify API** with header name `Authorization` and value `Bearer <YOUR_APIFY_TOKEN>`.
5. **Add Result Mapping Logic:** Add another **Code** node (`n8n-nodes-base.code`) to map the API results. Use the script that reads responses into a case-insensitive `Map`, cross-references them against the original sheet rows via `$('Read Unverified Rows in Sheets').all()`, and outputs combined objects containing status, sub-status, score, catch-all, disposable, and verified timestamp attributes.
6. **Add Data Writer:** Create a final **Google Sheets** node (`n8n-nodes-base.googleSheets`) connected to the mapping code node. Set operation to `update`, mapping mode to `autoMapInputData`, and matching column to `email`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Open-source repository, documentation, and additional templates (CSV cleaner, Signup webhook, AI agent tool) | https://github.com/sumitrmardy/email-verifier-api |
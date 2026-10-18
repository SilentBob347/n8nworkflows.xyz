Read Gmail receipt PDFs with Claude and reconcile card statements in Google Sheets

https://n8nworkflows.xyz/workflows/read-gmail-receipt-pdfs-with-claude-and-reconcile-card-statements-in-google-sheets-20855


# Read Gmail receipt PDFs with Claude and reconcile card statements in Google Sheets

### 1. Workflow Overview

This workflow automates the ingestion, verification, reconciliation, and reporting of financial receipts and card statements. It monitors Gmail for PDF receipts, extracts structured data using Anthropic Claude, performs deterministic code-based checks to validate totals and extracted numbers against the document text, stores validated records in Google Sheets, reconciles them against pasted credit card statements, and summarizes discrepancies via Slack.

The workflow logic is categorized into the following functional blocks:
- **1.1 Input Reception & Configuration:** Manages workflow initialization via a Gmail polling trigger or manual execution, merging input items with global system settings.
- **1.2 PDF Processing & AI Extraction:** Splits multi-attachment emails, extracts the raw text layer, and invokes Anthropic Claude to interpret receipt or invoice data into a structured JSON schema.
- **1.3 Receipt Validation & Storage:** Executes rigorous programmatic checks (validating arithmetic correctness, date sanity, and text-presence of all numbers) before appending or updating records in the Google Sheets `Receipts` tab.
- **1.4 Statement Reconciliation:** Reads both the `Receipts` and `Card` tabs from Google Sheets, executes deterministic date-window and amount-matching logic, clears the existing `Matches` tab, and writes the refreshed reconciliation output.
- **1.5 Reporting & Initialization:** Builds a concise status summary of items requiring manual intervention and posts it to Slack, alongside a one-time initialization routine to set up Google Sheets tabs and headers.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration

##### Overview
This block serves as the dual entry point for the workflow, handling automated receipt ingestion from Gmail or manual execution triggers, and standardizing execution parameters through a central configuration node.

##### Nodes Involved
- Gmail Trigger
- Reconcile now (Manual Trigger)
- Config
- Is it a new email? (If Node)

##### Node Details

- **Gmail Trigger**
  - **Type & Technical Role:** `n8n-nodes-base.gmailTrigger` — Polls a Gmail inbox for unread emails carrying PDF attachments and matching specific query filters.
  - **Configuration Choices:** Configured to trigger every 15 minutes (`item` mode, `15` minutes). Applies search filter `q: label:receipts has:attachment filename:pdf` and checks `readStatus: unread`. Automatically downloads attachments with the binary prefix `attachment_`. Sets `maxResults` to 10.
  - **Key Expressions / Variables:** None.
  - **Input & Output Connections:** Input: None (Trigger node). Output: Connects to `Config`.
  - **Version-Specific Requirements:** Version 1.4. Requires authenticated Gmail OAuth2 credentials.
  - **Edge Cases / Potential Failures:** Authentication token expiration, rate limiting by Google APIs, or emails lacking the proper label format causing skipped items.
  - **Sub-Workflow Reference:** None.

- **Reconcile now**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` — Initiates the workflow manually for statement reconciliation without processing new emails.
  - **Configuration Choices:** Default manual trigger setup with no execution parameters.
  - **Key Expressions / Variables:** None.
  - **Input & Output Connections:** Input: None (Trigger node). Output: Connects to `Config`.
  - **Version-Specific Requirements:** Version 1.
  - **Edge Cases / Potential Failures:** None.
  - **Sub-Workflow Reference:** None.

- **Config**
  - **Type & Technical Role:** `n8n-nodes-base.set` — Establishes global configuration parameters for currency, table column mappings, date matching windows, and tolerances.
  - **Configuration Choices:** Manual assignment mode. Sets `base_currency` to `JPY`, `card_date_col` to `Date`, `card_vendor_col` to `Merchant`, `card_amount_col` to `Amount`, `days_before` to `3`, `days_after` to `7`, `tolerance` to `0`, and `near_pct` to `3`. Preserves other incoming fields.
  - **Key Expressions / Variables:** None.
  - **Input & Output Connections:** Input: Connected from `Gmail Trigger` and `Reconcile now`. Output: Connects to `Is it a new email?`.
  - **Version-Specific Requirements:** Version 3.4.
  - **Edge Cases / Potential Failures:** Missing or misspelled column names will cause validation errors during the statement matching phase.
  - **Sub-Workflow Reference:** None.

- **Is it a new email?**
  - **Type & Technical Role:** `n8n-nodes-base.if` — Routes execution based on whether an email ID is present, splitting the workflow between processing incoming receipts or performing statement reconciliation.
  - **Configuration Choices:** Evaluates condition using strict type validation. Left value checks `{{ $json.id }}` using the `notEmpty` string operation against an empty right value.
  - **Key Expressions / Variables:** `={{ $json.id }}`
  - **Input & Output Connections:** Input: Connected from `Config`. Output Index 0 (True branch): Connects to `Split attachments`. Output Index 1 (False branch): Connects to `Read Receipts tab`.
  - **Version-Specific Requirements:** Version 2.2.
  - **Edge Cases / Potential Failures:** Execution failure if incoming data structures do not include the expected `id` property when invoked via manual triggers.
  - **Sub-Workflow Reference:** None.

---

#### Block 1.2: PDF Processing & AI Extraction

##### Overview
This block isolates individual PDF attachments from incoming emails, extracts their text content, and submits the raw files to Anthropic Claude to interpret financial line items, taxes, and vendor details into a structured JSON schema.

##### Nodes Involved
- Split attachments
- Extract text from PDF
- AI: Read receipt

##### Node Details

- **Split attachments**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript code node that explodes multi-attachment emails into individual items, one per PDF file.
  - **Configuration Choices:** Iterates through all items from the `Gmail Trigger`, inspects binary attachments for PDF MIME types or `.pdf` extensions, and formats metadata objects while forwarding the raw binary data in the `data` field.
  - **Key Expressions / Variables:** `$('Config').first().json`, `$('Gmail Trigger').all()`
  - **Input & Output Connections:** Input: Connected from `Isit a new email?` (True branch). Output: Connects to `Extract text from PDF`.
  - **Version-Specific Requirements:** Version 2.
  - **Edge Cases / Potential Failures:** Emails containing only non-PDF attachments will output zero items, halting the downstream extraction pipeline gracefully.
  - **Sub-Workflow Reference:** None.

- **Extract text from PDF**
  - **Type & Technical Role:** `n8n-nodes-base.extractFromFile` — Extracts text content from binary PDF files for subsequent validation checks.
  - **Configuration Choices:** Operation set to `pdf`. Options enabled: `joinPages: true` and `keepSource: both`.
  - **Key Expressions / Variables:** Operates on the default binary property (`data`).
  - **Input & Output Connections:** Input: Connected from `Split attachments`. Output: Connects to `AI: Read receipt`.
  - **Version-Specific Requirements:** Version 1.1.
  - **Edge Cases / Potential Failures:** Scanned documents lacking a selectable text layer will yield empty text strings, which are flagged later in the validation code node.
  - **Sub-Workflow Reference:** None.

- **AI: Read receipt**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.anthropic` — Anthropic language model node that analyzes document binaries to extract receipt fields as structured JSON.
  - **Configuration Choices:** Uses model ID `claude-haiku-4-5-20251001`. Resource set to `document`, operation set to `analyze`, input type set to `binary` (`data`), and output simplified. Configured with max tokens `2000` and error handling set to `continueRegularOutput`.
  - **Key Expressions / Variables:** Detailed system prompt instructing Claude to extract fields (`document_type`, `vendor`, `issue_date`, `currency`, `lines`, `tax`, `subtotal`, `total`, `payment_method`) strictly matching printed text.
  - **Input & Output Connections:** Input: Connected from `Extract text from PDF`. Output: Connects to `Check receipt`.
  - **Version-Specific Requirements:** Version 1. Requires valid Anthropic API credentials.
  - **Edge Cases / Potential Failures:** Malformed JSON output or API service downtime. The `continueRegularOutput` error handling setting prevents crashing on model parse errors.
  - **Sub-Workflow Reference:** None.

---

#### Block 1.3: Receipt Validation & Storage

##### Overview
This block programmatically validates Claude's extracted JSON against the extracted PDF text and standard arithmetic rules, flagging discrepancies as "Needs review," and upserts the finalized record into the Google Sheets `Receipts` tab.

##### Nodes Involved
- Check receipt
- Add to Receipts tab

##### Node Details

- **Check receipt**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript code node executing exhaustive deterministic checks on extracted financial data.
  - **Configuration Choices:** Normalizes text using NFKC encoding, checks that all extracted numbers (total, line amounts, tax values) exist within the PDF text, verifies line multiplication ($quantity \times unit\_price = amount$), validates tax and subtotal additions against totals, checks date sanity, and generates a unique cryptographic hash (`receipt_id`) to prevent duplicate processing.
  - **Key Expressions / Variables:** `$('Config').first().json`, `$('Split attachments').all()`, `$('Extract text from PDF').all()`
  - **Input & Output Connections:** Input: Connected from `AI: Read receipt`. Output: Connects to `Add to Receipts tab`.
  - **Version-Specific Requirements:** Version 2.
  - **Edge Cases / Potential Failures:** Look-alike Unicode characters in PDF text are handled via loose matching functions (`containsLoose`), while missing text layers automatically flag the document as `Needs review`.
  - **Sub-Workflow Reference:** None.

- **Add to Receipts tab**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Appends or updates verified receipt records within a designated Google Sheet.
  - **Configuration Choices:** Resource set to `sheet`, operation set to `appendOrUpdate`. Sheet name set to `Receipts`. Mapping mode configured to auto-map input data using `row_id` as the matching column. Cell format set to `RAW`.
  - **Key Expressions / Variables:** Uses mapped input fields from `Check receipt`.
  - **Input & Output Connections:** Input: Connected from `Check receipt`. Output: None (Terminal node for the receipt branch).
  - **Version-Specific Requirements:** Version 4.7. Requires valid Google Sheets OAuth2 credentials and a selected Spreadsheet ID.
  - **Edge Cases / Potential Failures:** API rate limits, permission errors, or missing header configurations in the target sheet.
  - **Sub-Workflow Reference:** None.

---

#### Block 1.4: Statement Reconciliation

##### Overview
This block reads existing receipt and card statement records from Google Sheets, matches statement lines to eligible receipts based on amounts and configurable date windows, clears the `Matches` tab, and writes out the updated reconciliation state.

##### Nodes Involved
- Read Receipts tab
- Read Card tab
- Match statement
- Clear Matches tab
- Matches rows
- Add to Matches tab

##### Node Details

- **Read Receipts tab**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Reads all recorded entries from the `Receipts` sheet tab.
  - **Configuration Choices:** Resource set to `sheet`, operation set to `read`. Sheet name set to `Receipts`. Output formatting configured to output dates as `SERIAL_NUMBER` and general values as `UNFORMATTED_VALUE`. Configured with `executeOnce: true` and `alwaysOutputData: true`.
  - **Key Expressions / Variables:** None.
  - **Input & Output Connections:** Input: Connected from `Is it a new email?` (False branch). Output: Connects to `Read Card tab`.
  - **Version-Specific Requirements:** Version 4.7. Requires Google Sheets OAuth2 credentials.
  - **Edge Cases / Potential Failures:** Empty sheets will return empty arrays, handled gracefully by downstream matching code.
  - **Sub-Workflow Reference:** None.

- **Read Card tab**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Reads pasted credit card statement entries from the `Card` sheet tab.
  - **Configuration Choices:** Resource set to `sheet`, operation set to `read`. Sheet name set to `Card`. Output formatting configures dates as `SERIAL_NUMBER` and general values as `UNFORMATTED_VALUE`. Configured with `executeOnce: true` and `alwaysOutputData: true`.
  - **Key Expressions / Variables:** None.
  - **Input & Output Connections:** Input: Connected from `Read Receipts tab`. Output: Connects to `Match statement`.
  - **Version-Specific Requirements:** Version 4.7. Requires Google Sheets OAuth2 credentials.
  - **Edge Cases / Potential Failures:** Missing configured columns will throw explicit configuration errors in the subsequent code node.
  - **Sub-Workflow Reference:** None.

- **Match statement**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript code node executing deterministic receipt-to-card reconciliation logic without AI.
  - **Configuration Choices:** Validates required column headers from configuration settings, parses serial date numbers and amounts, filters eligible receipts against currency and payment methods, applies iterative greedy matching based on amount equality and date windows (`days_before` and `days_after`), scores vendor name similarities as tie-breakers, and categorizes unmatched rows into statuses (e.g., `Matched`, `Ambiguous`, `No receipt`, `Near match`, `Possible double charge`).
  - **Key Expressions / Variables:** `$('Config').first().json`, `$('Read Receipts tab').all()`, `$('Read Card tab').all()`
  - **Input & Output Connections:** Input: Connected from `Read Card tab`. Output: Connects to `Clear Matches tab`.
  - **Version-Specific Requirements:** Version 2.
  - **Edge Cases / Potential Failures:** Throws explicit errors if specified column names are missing from the `Card` sheet.
  - **Sub-Workflow Reference:** None.

- **Clear Matches tab**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Clears existing content from the `Matches` sheet tab prior to writing fresh results.
  - **Configuration Choices:** Resource set to `sheet`, operation set to `clear`. Sheet name set to `Matches`. Clear mode set to `wholeSheet` while retaining the first row (`keepFirstRow: true`). Configured with `executeOnce: true`.
  - **Key Expressions / Variables:** None.
  - **Input & Output Connections:** Input: Connected from `Match statement`. Output: Connects to `Matches rows`.
  - **Version-Specific Requirements:** Version 4.7. Requires Google Sheets OAuth2 credentials.
  - **Edge Cases / Potential Failures:** Permissions errors or sheet deletion.
  - **Sub-Workflow Reference:** None.

- **Matches rows**
  - **Type & Technical Role:** `n8n-nodes-base.code` — Passes matched statement rows forward in the execution flow.
  - **Configuration Choices:** Simple return statement wrapping upstream match results.
  - **Key Expressions / Variables:** `$('Match statement').all()`
  - **Input & Output Connections:** Input: Connected from `Clear Matches tab`. Output: Connects to `Add to Matches tab`.
  - **Version-Specific Requirements:** Version 2.
  - **Edge Cases / Potential Failures:** None.
  - **Sub-Workflow Reference:** None.

- **Add to Matches tab**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` — Appends newly calculated reconciliation rows to the `Matches` sheet tab.
  - **Configuration Choices:** Resource set to `sheet`, operation set to `append`. Sheet name set to `Matches`. Mapping mode set to auto-map input data. Cell format set to `RAW`.
  - **Key Expressions / Variables:** Mapped input fields from `Matches rows`.
  - **Input & Output Connections:** Input: Connected from `Matches rows`. Output: Connects to `Build summary`.
  - **Version-Specific Requirements:** Version 4.7. Requires Google Sheets OAuth2 credentials.
  - **Edge Cases / Potential Failures:** API rate limits or quota exhaustion.
  - **Sub-Workflow Reference:** None.

---

#### Block 1.5: Reporting & Initialization

##### Overview
This block compiles reconciliation results into a status summary message, posts the notification to Slack, and provides a standalone setup branch to initialize spreadsheet structure and headers.

##### Nodes Involved
- Build summary
- Post summary to Slack
- Run once: set up the sheet (Manual Trigger)
- Spreadsheet ID
- Add the Receipts, Card and Matches tabs
- Write the header rows

##### Node Details

- **Build summary**
  - **Type & Technical Role:** `n8n-nodes-base.code` — JavaScript code node that aggregates match statistics and formats a Slack notification string highlighting items requiring human attention.
  - **Configuration Choices:** Counts status occurrences, filters card lines needing review (`Ambiguous`, `No receipt`, `Possible double charge`, `Near match`, `Card line not understood`), and compiles a formatted text message.
  - **Key Expressions / Variables:** `$('Match statement').all()`
  - **Input & Output Connections:** Input: Connected from `Add to Matches tab`. Output: Connects to `Post summary to Slack`.
  - **Version-Specific Requirements:** Version 2.
  - **Edge Cases / Potential Failures:** None.
  - **Sub-Workflow Reference:** None.

- **Post summary to Slack**
  - **Type & Technical Role:** `n8n-nodes-base.slack` — Posts the generated reconciliation summary message to a specified Slack channel.
  - **Configuration Choices:** Resource set to `message`, operation set to `post`. Message type set to `text`. Channel selected dynamically via list selection. Excludes link to workflow in other options.
  - **Key Expressions / Variables:** `={{ $json.text }}`
  - **Input & Output Connections:** Input: Connected from `Build summary`. Output: None (Terminal node for reporting branch).
  - **Version-Specific Requirements:** Version 2.7. Requires valid Slack credentials and app membership in the target channel.
  - **Edge Cases / Potential Failures:** Bot permissions lacking `chat:write` scopes or posting to unjoined channels.
  - **Sub-Workflow Reference:** None.

- **Run once: set up the sheet**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` — Standalone manual trigger used exclusively to initialize the Google Sheet structure.
  - **Configuration Choices:** Standard manual trigger with no parameters.
  - **Key Expressions / Variables:** None.
  - **Input & Output Connections:** Input: None (Trigger node). Output: Connects to `Spreadsheet ID`.
  - **Version-Specific Requirements:** Version 1.
  - **Edge Cases / Potential Failures:** None.
  - **Sub-Workflow Reference:** None.

- **Spreadsheet ID**
  - **Type & Technical Role:** `n8n-nodes-base.set` — Holds the target Google Spreadsheet ID for the initialization routine.
  - **Configuration Choices:** Manual assignment mode. Assigns parameter `spreadsheet_id` with value `PASTE_YOUR_SPREADSHEET_ID_HERE`.
  - **Key Expressions / Variables:** None.
  - **Input & Output Connections:** Input: Connected from `Run once: set up the sheet`. Output: Connects to `Add the Receipts, Card and Matches tabs`.
  - **Version-Specific Requirements:** Version 3.4.
  - **Edge Cases / Potential Failures:** Failure if placeholder ID is not replaced with a valid Google Sheet ID before execution.
  - **Sub-Workflow Reference:** None.

- **Add the Receipts, Card and Matches tabs**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Makes a REST API call to the Google Sheets API to create the three required worksheet tabs.
  - **Configuration Choices:** Method set to `POST`. URL targets `https://sheets.googleapis.com/v4/spreadsheets/{{ $("Spreadsheet ID").first().json.spreadsheet_id }}:batchUpdate`. JSON body specifies requests to add sheets titled `Receipts`, `Card`, and `Matches`. Authenticated using predefined Google Sheets OAuth2 API credentials.
  - **Key Expressions / Variables:** `={{ $("Spreadsheet ID").first().json.spreadsheet_id }}`
  - **Input & Output Connections:** Input: Connected from `Spreadsheet ID`. Output: Connects to `Write the header rows`.
  - **Version-Specific Requirements:** Version 4.5. Requires Google Sheets OAuth2 credentials.
  - **Edge Cases / Potential Failures:** Running this branch twice will fail because the sheets already exist.
  - **Sub-Workflow Reference:** None.

- **Write the header rows**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` — Populates header rows across all three created worksheet tabs via the Google Sheets API.
  - **Configuration Choices:** Method set to `POST`. URL targets `https://sheets.googleapis.com/v4/spreadsheets/{{ $("Spreadsheet ID").first().json.spreadsheet_id }}/values:batchUpdate`. Sends raw value input data mapping column headers for `Receipts`, `Card`, and `Matches` tabs. Authenticated using predefined Google Sheets OAuth2 API credentials.
  - **Key Expressions / Variables:** `={{ $("Spreadsheet ID").first().json.spreadsheet_id }}`
  - **Input & Output Connections:** Input: Connected from `Add the Receipts, Card and Matches tabs`. Output: None (Terminal node for setup branch).
  - **Version-Specific Requirements:** Version 4.5. Requires Google Sheets OAuth2 credentials.
  - **Edge Cases / Potential Failures:** API formatting or permission errors.
  - **Sub-Workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Gmail Trigger | n8n-nodes-base.gmailTrigger | Polls Gmail for unread emails with receipt PDFs | None | Config | Receipts in, card statement matched<br>Two ways in |
| Reconcile now | n8n-nodes-base.manualTrigger | Manually triggers statement reconciliation | None | Config | Two ways in |
| Config | n8n-nodes-base.set | Sets global configuration variables and tolerances | Gmail Trigger, Reconcile now | Is it a new email? | Two ways in |
| Is it a new email? | n8n-nodes-base.if | Routes execution based on incoming email presence | Config | Split attachments, Read Receipts tab | Two ways in |
| Split attachments | n8n-nodes-base.code | Splits multi-attachment emails into individual PDF items | Is it a new email? | Extract text from PDF | Claude reads each PDF |
| Extract text from PDF | n8n-nodes-base.extractFromFile | Extracts text layer from binary PDF files | Split attachments | AI: Read receipt | Claude reads each PDF |
| AI: Read receipt | @n8n/n8n-nodes-langchain.anthropic | Extracts structured JSON receipt data using Claude | Extract text from PDF | Check receipt | Claude reads each PDF |
| Check receipt | n8n-nodes-base.code | Validates extracted data against PDF text and arithmetic rules | AI: Read receipt | Add to Receipts tab | Check by code, save |
| Add to Receipts tab | n8n-nodes-base.googleSheets | Appends or updates verified receipt records in Google Sheets | Check receipt | None | Check by code, save |
| Read Receipts tab | n8n-nodes-base.googleSheets | Reads records from the Receipts Google Sheet tab | Is it a new email? | Read Card tab | Match the card statement |
| Read Card tab | n8n-nodes-base.googleSheets | Reads card statement entries from the Card Google Sheet tab | Read Receipts tab | Match statement | Match the card statement |
| Match statement | n8n-nodes-base.code | Reconciles card statement lines against eligible receipts | Read Card tab | Clear Matches tab | Match the card statement |
| Clear Matches tab | n8n-nodes-base.googleSheets | Clears existing rows in the Matches Google Sheet tab | Match statement | Matches rows | Match the card statement |
| Matches rows | n8n-nodes-base.code | Passes matched reconciliation rows forward | Clear Matches tab | Add to Matches tab | Match the card statement |
| Add to Matches tab | n8n-nodes-base.googleSheets | Appends new reconciliation rows to the Matches Google Sheet tab | Matches rows | Build summary | Match the card statement |
| Build summary | n8n-nodes-base.code | Compiles reconciliation statistics into a notification message | Add to Matches tab | Post summary to Slack | Report |
| Post summary to Slack | n8n-nodes-base.slack | Posts reconciliation summary and alerts to Slack | Build summary | None | Report |
| Run once: set up the sheet | n8n-nodes-base.manualTrigger | Manually triggers Google Sheet initialization branch | None | Spreadsheet ID | Run once: set up the sheet |
| Spreadsheet ID | n8n-nodes-base.set | Defines target spreadsheet ID for sheet setup | Run once: set up the sheet | Add the Receipts, Card and Matches tabs | Run once: set up the sheet |
| Add the Receipts, Card and Matches tabs | n8n-nodes-base.httpRequest | Creates required worksheet tabs via Google Sheets API | Spreadsheet ID | Write the header rows | Run once: set up the sheet |
| Write the header rows | n8n-nodes-base.httpRequest | Populates table headers across initialized sheet tabs | Add the Receipts, Card and Matches tabs | None | Run once: set up the sheet |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Credentials:** Ensure you have active credentials configured in n8n for:
   - Gmail (OAuth2)
   - Google Sheets (OAuth2)
   - Anthropic (API Key)
   - Slack (OAuth2 / Bot API token)

2. **Set up Initialization Branch:**
   - Add a **Manual Trigger** node named `Run once: set up the sheet`.
   - Add a **Set** node named `Spreadsheet ID`. Configure mode to `Manual`, add string assignment `spreadsheet_id`, and paste your target Google Spreadsheet ID. Connect `Run once: set up the sheet` to `Spreadsheet ID`.
   - Add an **HTTP Request** node named `Add the Receipts, Card and Matches tabs`. Set method to `POST`, URL to `https://sheets.googleapis.com/v4/spreadsheets/{{ $("Spreadsheet ID").first().json.spreadsheet_id }}:batchUpdate`, and select `Google Sheets OAuth2 API` authentication. Specify body as JSON:
     ```json
     {"requests":[{"addSheet":{"properties":{"title":"Receipts"}}},{"addSheet":{"properties":{"title":"Card"}}},{"addSheet":{"properties":{"title":"Matches"}}}]}
     ```
     Connect `Spreadsheet ID` to this node.
   - Add an **HTTP Request** node named `Write the header rows`. Set method to `POST`, URL to `https://sheets.googleapis.com/v4/spreadsheets/{{ $("Spreadsheet ID").first().json.spreadsheet_id }}/values:batchUpdate`, and select `Google Sheets OAuth2 API` authentication. Specify body as JSON supplying headers for `Receipts`, `Card`, and `Matches`. Connect `Add the Receipts, Card and Matches tabs` to `Write the header rows`.

3. **Set up Main Entry & Configuration:**
   - Add a **Gmail Trigger** node named `Gmail Trigger`. Set event to `messageReceived`, filter query to `label:receipts has:attachment filename:pdf`, read status to `unread`, and enable attachment downloading (`attachment_` prefix, poll every 15 minutes).
   - Add a **Manual Trigger** node named `Reconcile now`.
   - Add a **Set** node named `Config`. Configure manual assignments for `base_currency` (string: `JPY`), `card_date_col` (string: `Date`), `card_vendor_col` (string: `Merchant`), `card_amount_col` (string: `Amount`), `days_before` (number: `3`), `days_after` (number: `7`), `tolerance` (number: `0`), and `near_pct` (number: `3`). Include other fields. Connect both `Gmail Trigger` and `Reconcile now` to `Config`.
   - Add an **If** node named `Is it a new email?`. Set condition to evaluate `{{ $json.id }}` as `notEmpty`. Connect `Config` to `Is it a new email?`.

4. **Set up Receipt Ingestion & AI Processing Branch (True Path):**
   - Add a **Code** node named `Split attachments` connected to True output (index 0) of `Is it a new email?`. Insert JavaScript code to iterate through binary PDF attachments and output one item per PDF.
   - Add an **Extract From File** node named `Extract text from PDF`. Set operation to `pdf`, with options `joinPages: true` and `keepSource: both`. Connect `Split attachments` to this node.
   - Add an **Anthropic** node named `AI: Read receipt`. Set model ID to `claude-haiku-4-5-20251001`, resource to `document`, operation to `analyze`, input type to `binary` (`data`), and enable error handling to `continueRegularOutput`. Paste the receipt reading prompt into the text parameter. Connect `Extract text from PDF` to this node.
   - Add a **Code** node named `Check receipt` to validate AI output against text and arithmetic rules. Connect `AI: Read receipt` to this node.
   - Add a **Google Sheets** node named `Add to Receipts tab`. Set resource to `sheet`, operation to `appendOrUpdate`, sheet name to `Receipts`, and matching column to `row_id`. Connect `Check receipt` to this node.

5. **Set up Reconciliation & Reporting Branch (False Path):**
   - Add a **Google Sheets** node named `Read Receipts tab`. Set operation to `read`, sheet name to `Receipts`, output formatting date to `SERIAL_NUMBER`, general to `UNFORMATTED_VALUE`, and enable `executeOnce` and `alwaysOutputData`. Connect False output (index 1) of `Is it a new email?` to this node.
   - Add a **Google Sheets** node named `Read Card tab`. Configure similarly for sheet `Card`. Connect `Read Receipts tab` to this node.
   - Add a **Code** node named `Match statement` to execute deterministic card-to-receipt matching logic. Connect `Read Card tab` to this node.
   - Add a **Google Sheets** node named `Clear Matches tab`. Set operation to `clear`, sheet name to `Matches`, clear mode to `wholeSheet`, and `keepFirstRow: true` (`executeOnce: true`). Connect `Match statement` to this node.
   - Add a **Code** node named `Matches rows` to pass match items forward. Connect `Clear Matches tab` to this node.
   - Add a **Google Sheets** node named `Add to Matches tab`. Set operation to `append`, sheet name to `Matches`. Connect `Matches rows` to this node.
   - Add a **Code** node named `Build summary` to format Slack notification text. Connect `Add to Matches tab` to this node.
   - Add a **Slack** node named `Post summary to Slack`. Set resource to `message`, operation to `post`, message type to `text`, text to `={{ $json.text }}`, and select your target Slack channel. Connect `Build summary` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Receipts in, card statement matched | High-level workflow purpose: automated PDF receipt parsing via Claude combined with deterministic card statement reconciliation. |
| Run once: set up the sheet | Required one-time initialization step to provision worksheet tabs (`Receipts`, `Card`, `Matches`) and headers before running automated or manual triggers. |
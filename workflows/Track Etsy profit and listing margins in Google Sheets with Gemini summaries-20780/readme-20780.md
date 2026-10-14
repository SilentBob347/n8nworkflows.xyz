Track Etsy profit and listing margins in Google Sheets with Gemini summaries

https://n8nworkflows.xyz/workflows/track-etsy-profit-and-listing-margins-in-google-sheets-with-gemini-summaries-20780


# Track Etsy profit and listing margins in Google Sheets with Gemini summaries

### 1. Workflow Overview

This workflow automates the calculation of monthly Profit & Loss (P&L), per-listing profit margins, data validation checks, and plain-language summaries for Etsy shop owners. It operates entirely without using the Etsy API by leveraging manual CSV exports or Google Sheets tabs containing monthly statements, order items, and Cost of Goods Sold (COGS). 

The workflow is logically structured into the following operational blocks:
- **1.1 Input Reception & Configuration:** Initializes global variables (modes, sources, thresholds) and determines whether to process built-in sample data, local CSV files, or Google Sheets tabs.
- **1.2 Data Ingestion & Bundling:** Gathers input data from the chosen source (Sample data node, Google Sheets reader nodes, or local file read/extract nodes) and unifies them into a single payload object.
- **1.3 Financial Calculation & Reconciliation:** Processes and parses raw monetary amounts, assigns categories, maps order items to listings, calculates COGS, computes net profits/margins, and performs reconciliation checks.
- **1.4 Summary Generation (Mock vs. AI):** Generates a rule-based summary by default, with an optional branch to rewrite the review via the Google Gemini API.
- **1.5 Data Output & Persistence:** Formats final datasets into tabular structures and conditionally writes them either to Google Sheets tabs or local CSV files.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Sets up user preferences, processing modes, and data source selections. It serves as the primary configuration center for the entire pipeline.
- **Nodes Involved:** `Run`, `Config`, `Source = sample?`, `Source = sheet?`
- **Node Details:**
  - **Run (`n8n-nodes-base.manualTrigger`)**
    - *Type and Role:* Manual trigger node used to initiate the workflow on demand.
    - *Configuration choices:* Default parameters (no webhooks or schedules).
    - *Key expressions:* None.
    - *Input/Output:* Output connects to `Config`.
    - *Edge cases:* None.
  - **Config (`n8n-nodes-base.code`)**
    - *Type and Role:* JavaScript code node that outputs configuration settings (`mode`, `source`, `output`, `filesFolder`, `outputFolder`, `currency`, `dateOrder`, `monthlyOtherCosts`, `lowMarginPct`, `summaryLanguage`, `geminiModel`).
    - *Configuration choices:* Hardcoded configuration object.
    - *Key expressions:* Returns JSON configuration payload.
    - *Input/Output:* Input from `Run`; output connects to `Source = sample?`.
    - *Edge cases:* Changing keys requires manual code updates.
  - **Source = sample? (`n8n-nodes-base.if`)**
    - *Type and Role:* Conditional router checking if `source` is set to `"sample"`.
    - *Configuration choices:* Evaluates `{{ $json.source }}` equals `"sample"`.
    - *Key expressions:* `{{ $json.source }}`
    - *Input/Output:* Input from `Config`; outputs route to `Sample Etsy exports (fake data)` or `Source = sheet?`.
    - *Edge cases:* Invalid string values route incorrectly to sheet logic.
  - **Source = sheet? (`n8n-nodes-base.if`)**
    - *Type and Role:* Conditional router checking if `source` is set to `"sheet"`.
    - *Configuration choices:* Evaluates `{{ $json.source }}` equals `"sheet"`.
    - *Key expressions:* `{{ $json.source }}`
    - *Input/Output:* Input from `Source = sample?` (false branch); outputs route to `Read Statement tab` or `Read statement CSVs`.
    - *Edge cases:* If source is `"files"`, it routes to local CSV reading nodes.

#### 2.2 Data Ingestion & Bundling
- **Overview:** Retrieves raw accounting and inventory data from either built-in mock data, Google Sheets, or local directory files, normalizing them into a unified structure.
- **Nodes Involved:** `Sample Etsy exports (fake data)`, `Read Statement tab`, `Read OrderItems tab`, `Read COGS tab`, `Read statement CSVs`, `Parse statement CSV`, `Read Order Items CSVs`, `Parse Order Items CSV`, `Read COGS CSV`, `Parse COGS CSV`, `Bundle inputs`
- **Node Details:**
  - **Sample Etsy exports (fake data) (`n8n-nodes-base.code`)**
    - *Type and Role:* Provides mock statement, order item, and COGS data structures for testing.
    - *Configuration choices:* Static JSON payload returned via JavaScript.
    - *Input/Output:* Input from `Source = sample?` (true branch); output connects to `Compute P&L`.
    - *Edge cases:* Static data only reflects mocked dates and categories.
  - **Read Statement tab (`n8n-nodes-base.googleSheets`)**
    - *Type and Role:* Reads statement records from a specified Google Sheets document tab.
    - *Configuration choices:* Uses Operation `read`, Document ID URL parameter, and Sheet Name `"Statement"`.
    - *Input/Output:* Input from `Source = sheet?` (true branch); output connects to `Read OrderItems tab`.
    - *Edge cases:* Authentication errors or invalid Sheet IDs will halt execution.
  - **Read OrderItems tab (`n8n-nodes-base.googleSheets`)**
    - *Type and Role:* Reads order item records from the Google Sheets tab.
    - *Configuration choices:* Operation `read`, Sheet Name `"OrderItems"`, `executeOnce: true`.
    - *Input/Output:* Input from `Read Statement tab`; output connects to `Read COGS tab`.
    - *Edge cases:* Missing columns will result in unmapped values during processing.
  - **Read COGS tab (`n8n-nodes-base.googleSheets`)**
    - *Type and Role:* Reads product cost data from the Google Sheets COGS tab.
    - *Configuration choices:* Operation `read`, Sheet Name `"COGS"`, `executeOnce: true`.
    - *Input/Output:* Input from `Read OrderItems tab`; output connects to `Bundle inputs`.
    - *Edge cases:* Missing cost entries default to zero, overstating profits.
  - **Read statement CSVs (`n8n-nodes-base.readWriteFile`)**
    - *Type and Role:* Reads local statement CSV files from disk.
    - *Configuration choices:* File selector using `{{ $('Config').first().json.filesFolder }}/etsy_statement*.csv`, `onError: continueRegularOutput`.
    - *Input/Output:* Input from `Source = sheet?` (false branch); output connects to `Parse statement CSV`.
    - *Edge cases:* File not found or path permissions errors.
  - **Parse statement CSV (`n8n-nodes-base.extractFromFile`)**
    - *Type and Role:* Parses raw statement file buffers into structured JSON rows.
    - *Configuration choices:* Operation `csv`.
    - *Input/Output:* Input from `Read statement CSVs`; output connects to `Read Order Items CSVs`.
    - *Edge cases:* Malformed CSV structures or encoding issues (e.g., BOM headers).
  - **Read Order Items CSVs (`n8n-nodes-base.readWriteFile`)**
    - *Type and Role:* Reads local order items CSV files from disk.
    - *Configuration choices:* File selector using `{{ $('Config').first().json.filesFolder }}/EtsySoldOrderItems*.csv`, `executeOnce: true`.
    - *Input/Output:* Input from `Parse statement CSV`; output connects to `Parse Order Items CSV`.
    - *Edge cases:* Missing files or incorrect file name patterns.
  - **Parse Order Items CSV (`n8n-nodes-base.extractFromFile`)**
    - *Type and Role:* Extracts tabular rows from the order items CSV.
    - *Configuration choices:* Operation `csv`.
    - *Input/Output:* Input from `Read Order Items CSVs`; output connects to `Read COGS CSV`.
    - *Edge cases:* Parsing failures due to delimiter mismatches.
  - **Read COGS CSV (`n8n-nodes-base.readWriteFile`)**
    - *Type and Role:* Reads local COGS CSV files from disk.
    - *Configuration choices:* File selector using `{{ $('Config').first().json.filesFolder }}/cogs*.csv`, `executeOnce: true`.
    - *Input/Output:* Input from `Parse Order Items CSV`; output connects to `Parse COGS CSV`.
    - *Edge cases:* Missing file defaults to uncosted inventory items.
  - **Parse COGS CSV (`n8n-nodes-base.extractFromFile`)**
    - *Type and Role:* Parses cost data from local COGS CSV files.
    - *Configuration choices:* Operation `csv`.
    - *Input/Output:* Input from `Read COGS CSV`; output connects to `Bundle inputs`.
    - *Edge cases:* Unrecognized header formats.
  - **Bundle inputs (`n8n-nodes-base.code`)**
    - *Type and Role:* Aggregates statement, order items, and COGS datasets into a single item payload.
    - *Configuration choices:* JavaScript safely evaluates execution states across branches.
    - *Input/Output:* Inputs from `Read COGS tab` or `Parse COGS CSV`; output connects to `Compute P&L`.
    - *Edge cases:* Empty input arrays if preceding nodes yielded no data.

#### 2.3 Financial Calculation & Reconciliation
- **Overview:** Executes core accounting logic—parsing dates and monetary strings, categorizing fees, matching orders to listings, calculating item-level and shop-level expenses, and flagging anomalies.
- **Nodes Involved:** `Compute P&L`
- **Node Details:**
  - **Compute P&L (`n8n-nodes-base.code`)**
    - *Type and Role:* Heavy computational script that transforms raw accounting rows into monthly P&L summaries, per-listing margins, flags (`LOSS`, `LOW_MARGIN`, `MISSING_COGS`, `REFUNDED`), rule-based text summaries, and AI prompts.
    - *Configuration choices:* Pure JavaScript business logic script.
    - *Key expressions:* Consumes input data object and configuration settings.
    - *Input/Output:* Input from `Bundle inputs`; output connects to `Mode = gemini?`.
    - *Edge cases:* Unparsed date formats, mismatched currencies, or missing COGS entries trigger issue logging arrays.

#### 2.4 Summary Generation (Mock vs. AI)
- **Overview:** Determines whether to output the rule-based summary directly or invoke the Google Gemini API to generate an AI-driven review.
- **Nodes Involved:** `Mode = gemini?`, `Gemini summary`, `Read Gemini reply`, `Rule-based summary`
- **Node Details:**
  - **Mode = gemini? (`n8n-nodes-base.if`)**
    - *Type and Role:* Conditional router checking if execution mode is set to `"gemini"`.
    - *Configuration choices:* Evaluates `{{ $json._cfg.mode }}` equals `"gemini"`.
    - *Key expressions:* `{{ $json._cfg.mode }}`
    - *Input/Output:* Input from `Compute P&L`; outputs route to `Gemini summary` (true) or `Rule-based summary` (false).
    - *Edge cases:* Incorrect credential configurations in Gemini mode will trigger HTTP fallback.
  - **Gemini summary (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* Sends structured prompt payloads to the Google Generative Language API.
    - *Configuration choices:* POST request to `https://generativelanguage.googleapis.com/v1beta/models/{{ $json._cfg.geminiModel }}:generateContent`, generic header authentication (`x-goog-api-key`), timeout set to 120,000ms, max tries: 3.
    - *Key expressions:* `={{ JSON.stringify({ contents: [{ role: 'user', parts: [{ text: $json.prompt }] }], generationConfig: { temperature: 0.3 } }) }}`
    - *Input/Output:* Input from `Mode = gemini?` (true branch); output connects to `Read Gemini reply`.
    - *Edge cases:* API rate limits (429), model availability errors (404), or invalid keys (401/403).
  - **Read Gemini reply (`n8n-nodes-base.code`)**
    - *Type and Role:* Parses API responses from Gemini, extracting generated text and handling potential AI service failures gracefully.
    - *Configuration choices:* JavaScript node parsing response candidates.
    - *Input/Output:* Input from `Gemini summary`; output fans out to `Rows: P&L`, `Rows: By Listing`, and `Rows: Summary`.
    - *Edge cases:* Empty candidate payloads fall back to rule-based summaries.
  - **Rule-based summary (`n8n-nodes-base.code`)**
    - *Type and Role:* Passes through rule-generated summaries when running in mock mode or when AI generation is bypassed.
    - *Configuration choices:* JavaScript assignment mapping `rulesSummary` to `summary`.
    - *Input/Output:* Input from `Mode = gemini?` (false branch); output fans out to `Rows: P&L`, `Rows: By Listing`, and `Rows: Summary`.
    - *Edge cases:* None.

#### 2.5 Data Output & Persistence
- **Overview:** Formats processed records into structured tabular arrays and writes them conditionally to Google Sheets tabs or local CSV output files.
- **Nodes Involved:** `Rows: P&L`, `P&L to sheet?`, `Write 'P&L' tab`, `P&L to CSV`, `Save etsy_profit_monthly.csv`, `Rows: By Listing`, `By Listing to sheet?`, `Write 'By Listing' tab`, `By Listing to CSV`, `Save etsy_profit_by_listing.csv`, `Rows: Summary`, `Summary to sheet?`, `Write 'Summary' tab`, `Summary to CSV`, `Save etsy_profit_summary.csv`
- **Node Details:**
  - **Rows: P&L (`n8n-nodes-base.code`)**
    - *Type and Role:* Flattens monthly P&L totals and appends generation timestamps.
    - *Input/Output:* Inputs from `Read Gemini reply` or `Rule-based summary`; output connects to `P&L to sheet?`.
  - **P&L to sheet? (`n8n-nodes-base.if`)**
    - *Type and Role:* Checks if output destination is set to `"sheet"`.
    - *Key expressions:* `{{ $('Config').first().json.output }}` equals `"sheet"`.
    - *Input/Output:* Input from `Rows: P&L`; outputs route to `Write 'P&L' tab` (true) or `P&L to CSV` (false).
  - **Write 'P&L' tab (`n8n-nodes-base.googleSheets`)**
    - *Type and Role:* Appends or updates P&L data rows in the Google Sheets `"P&L"` tab.
    - *Configuration choices:* Operation `appendOrUpdate`, matching columns `["month"]`.
  - **P&L to CSV (`n8n-nodes-base.convertToFile`)**
    - *Type and Role:* Converts P&L JSON objects into CSV format.
    - *Configuration choices:* File name `etsy_profit_monthly.csv`.
  - **Save etsy_profit_monthly.csv (`n8n-nodes-base.readWriteFile`)**
    - *Type and Role:* Writes the P&L CSV string to disk.
    - *Configuration choices:* Operation `write`, filename `{{ $('Config').first().json.outputFolder }}/etsy_profit_monthly.csv`.
  - **Rows: By Listing (`n8n-nodes-base.code`)**
    - *Type and Role:* Flattens listing-level profit records with unique composite keys.
    - *Input/Output:* Inputs from summary nodes; output connects to `By Listing to sheet?`.
  - **By Listing to sheet? (`n8n-nodes-base.if`)**
    - *Type and Role:* Checks if output destination is set to `"sheet"`.
    - *Key expressions:* `{{ $('Config').first().json.output }}` equals `"sheet"`.
  - **Write 'By Listing' tab (`n8n-nodes-base.googleSheets`)**
    - *Type and Role:* Appends or updates listing profit records in the Google Sheets `"By Listing"` tab.
    - *Configuration choices:* Operation `appendOrUpdate`, matching columns `["key"]`.
  - **By Listing to CSV (`n8n-nodes-base.convertToFile`)**
    - *Type and Role:* Converts listing dataset to CSV format (`etsy_profit_by_listing.csv`).
  - **Save etsy_profit_by_listing.csv (`n8n-nodes-base.readWriteFile`)**
    - *Type and Role:* Writes listing CSV file to disk.
  - **Rows: Summary (`n8n-nodes-base.code`)**
    - *Type and Role:* Formats text summaries, AI errors, and reconciliation check issues into structured log rows.
    - *Input/Output:* Inputs from summary nodes; output connects to `Summary to sheet?`.
  - **Summary to sheet? (`n8n-nodes-base.if`)**
    - *Type and Role:* Checks if output destination is set to `"sheet"`.
  - **Write 'Summary' tab (`n8n-nodes-base.googleSheets`)**
    - *Type and Role:* Appends summary records to the Google Sheets `"Summary"` tab.
    - *Configuration choices:* Operation `append`.
  - **Summary to CSV (`n8n-nodes-base.convertToFile`)**
    - *Type and Role:* Converts summary rows to CSV format (`etsy_profit_summary.csv`).
  - **Save etsy_profit_summary.csv (`n8n-nodes-base.readWriteFile`)**
    - *Type and Role:* Writes summary CSV file to disk.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run | n8n-nodes-base.manualTrigger | Initiates workflow execution manually. | None | Config | Profit Tracker for Etsy Shops v0.2 (free) · by Stallkit Studio<br>Etsy monthly statement CSV + Order Items CSV + your cost table → monthly P&L, profit per listing per month, reconciliation checks, and a short summary of which listings lost money.<br><br>**Try it first (no keys, no accounts):** click **Test workflow**. It runs on built-in FAKE data and writes 3 CSVs to `/home/node/.n8n-files/` (self-hosted). On n8n Cloud, look at the output of the **Rows:** nodes instead.<br><br>**Your own data**<br>1. Etsy Shop Manager → Finances → Payment account → *Monthly statements* → CSV (one per month) and Settings → Options → Download Data → Orders → *Order Items* CSV (same months).<br>2. Make a cost table with columns `sku, listing_id, item_name, unit_cost, extra_cost_per_unit` (fill sku OR listing_id OR item_name).<br>3. **Files** (self-hosted): put them in `filesFolder` as `etsy_statement*.csv`, `EtsySoldOrderItems*.csv`, `cogs.csv`, set `source: \"files\"`.<br>   **Google Sheet**: tabs `Statement`, `OrderItems`, `COGS` (File → Import → each CSV), plus empty tabs `P&L`, `By Listing`, `Summary`. Paste the sheet URL into all 6 Google Sheets nodes, set `source` and `output` to `\"sheet\"`.<br>4. Optional AI summary: Header Auth credential, name `x-goog-api-key`, value = your Gemini key; pick it in **Gemini summary**; set `mode: \"gemini\"`. Only totals and listing titles are sent, never buyer names or addresses.<br><br>n8n Cloud is a paid n8n subscription; self-hosting needs your own computer or server. Gemini API costs, if any, are billed to you by Google.<br>Not tax or accounting advice. Check numbers against your Etsy statement.<br>Etsy and n8n are trademarks of their respective owners. Stallkit Studio is not affiliated with or endorsed by them. |
| Config | n8n-nodes-base.code | Sets global configuration variables and runtime parameters. | Run | Source = sample? | Profit Tracker for Etsy Shops v0.2 (free) · by Stallkit Studio<br>Etsy monthly statement CSV + Order Items CSV + your cost table → monthly P&L, profit per listing per month, reconciliation checks, and a short summary of which listings lost money.<br><br>**Try it first (no keys, no accounts):** click **Test workflow**. It runs on built-in FAKE data and writes 3 CSVs to `/home/node/.n8n-files/` (self-hosted). On n8n Cloud, look at the output of the **Rows:** nodes instead.<br><br>**Your own data**<br>1. Etsy Shop Manager → Finances → Payment account → *Monthly statements* → CSV (one per month) and Settings → Options → Download Data → Orders → *Order Items* CSV (same months).<br>2. Make a cost table with columns `sku, listing_id, item_name, unit_cost, extra_cost_per_unit` (fill sku OR listing_id OR item_name).<br>3. **Files** (self-hosted): put them in `filesFolder` as `etsy_statement*.csv`, `EtsySoldOrderItems*.csv`, `cogs.csv`, set `source: \"files\"`.<br>   **Google Sheet**: tabs `Statement`, `OrderItems`, `COGS` (File → Import → each CSV), plus empty tabs `P&L`, `By Listing`, `Summary`. Paste the sheet URL into all 6 Google Sheets nodes, set `source` and `output` to `\"sheet\"`.<br>4. Optional AI summary: Header Auth credential, name `x-goog-api-key`, value = your Gemini key; pick it in **Gemini summary**; set `mode: \"gemini\"`. Only totals and listing titles are sent, never buyer names or addresses.<br><br>n8n Cloud is a paid n8n subscription; self-hosting needs your own computer or server. Gemini API costs, if any, are billed to you by Google.<br>Not tax or accounting advice. Check numbers against your Etsy statement.<br>Etsy and n8n are trademarks of their respective owners. Stallkit Studio is not affiliated with or endorsed by them. |
| Source = sample? | n8n-nodes-base.if | Routes execution based on whether source is set to sample. | Config | Sample Etsy exports (fake data), Source = sheet? | Profit Tracker for Etsy Shops v0.2 (free) · by Stallkit Studio<br>Etsy monthly statement CSV + Order Items CSV + your cost table → monthly P&L, profit per listing per month, reconciliation checks, and a short summary of which listings lost money.<br><br>**Try it first (no keys, no accounts):** click **Test workflow**. It runs on built-in FAKE data and writes 3 CSVs to `/home/node/.n8n-files/` (self-hosted). On n8n Cloud, look at the output of the **Rows:** nodes instead.<br><br>**Your own data**<br>1. Etsy Shop Manager → Finances → Payment account → *Monthly statements* → CSV (one per month) and Settings → Options → Download Data → Orders → *Order Items* CSV (same months).<br>2. Make a cost table with columns `sku, listing_id, item_name, unit_cost, extra_cost_per_unit` (fill sku OR listing_id OR item_name).<br>3. **Files** (self-hosted): put them in `filesFolder` as `etsy_statement*.csv`, `EtsySoldOrderItems*.csv`, `cogs.csv`, set `source: \"files\"`.<br>   **Google Sheet**: tabs `Statement`, `OrderItems`, `COGS` (File → Import → each CSV), plus empty tabs `P&L`, `By Listing`, `Summary`. Paste the sheet URL into all 6 Google Sheets nodes, set `source` and `output` to `\"sheet\"`.<br>4. Optional AI summary: Header Auth credential, name `x-goog-api-key`, value = your Gemini key; pick it in **Gemini summary**; set `mode: \"gemini\"`. Only totals and listing titles are sent, never buyer names or addresses.<br><br>n8n Cloud is a paid n8n subscription; self-hosting needs your own computer or server. Gemini API costs, if any, are billed to you by Google.<br>Not tax or accounting advice. Check numbers against your Etsy statement.<br>Etsy and n8n are trademarks of their respective owners. Stallkit Studio is not affiliated with or endorsed by them. |
| Sample Etsy exports (fake data) | n8n-nodes-base.code | Supplies built-in mock shop data for testing. | Source = sample? | Compute P&L | Profit Tracker for Etsy Shops v0.2 (free) · by Stallkit Studio<br>Etsy monthly statement CSV + Order Items CSV + your cost table → monthly P&L, profit per listing per month, reconciliation checks, and a short summary of which listings lost money.<br><br>**Try it first (no keys, no accounts):** click **Test workflow**. It runs on built-in FAKE data and writes 3 CSVs to `/home/node/.n8n-files/` (self-hosted). On n8n Cloud, look at the output of the **Rows:** nodes instead.<br><br>**Your own data**<br>1. Etsy Shop Manager → Finances → Payment account → *Monthly statements* → CSV (one per month) and Settings → Options → Download Data → Orders → *Order Items* CSV (same months).<br>2. Make a cost table with columns `sku, listing_id, item_name, unit_cost, extra_cost_per_unit` (fill sku OR listing_id OR item_name).<br>3. **Files** (self-hosted): put them in `filesFolder` as `etsy_statement*.csv`, `EtsySoldOrderItems*.csv`, `cogs.csv`, set `source: \"files\"`.<br>   **Google Sheet**: tabs `Statement`, `OrderItems`, `COGS` (File → Import → each CSV), plus empty tabs `P&L`, `By Listing`, `Summary`. Paste the sheet URL into all 6 Google Sheets nodes, set `source` and `output` to `\"sheet\"`.<br>4. Optional AI summary: Header Auth credential, name `x-goog-api-key`, value = your Gemini key; pick it in **Gemini summary**; set `mode: \"gemini\"`. Only totals and listing titles are sent, never buyer names or addresses.<br><br>n8n Cloud is a paid n8n subscription; self-hosting needs your own computer or server. Gemini API costs, if any, are billed to you by Google.<br>Not tax or accounting advice. Check numbers against your Etsy statement.<br>Etsy and n8n are trademarks of their respective owners. Stallkit Studio is not affiliated with or endorsed by them. |
| Source = sheet? | n8n-nodes-base.if | Routes execution based on whether source is set to sheet. | Source = sample? | Read Statement tab, Read statement CSVs | Profit Tracker for Etsy Shops v0.2 (free) · by Stallkit Studio<br>Etsy monthly statement CSV + Order Items CSV + your cost table → monthly P&L, profit per listing per month, reconciliation checks, and a short summary of which listings lost money.<br><br>**Try it first (no keys, no accounts):** click **Test workflow**. It runs on built-in FAKE data and writes 3 CSVs to `/home/node/.n8n-files/` (self-hosted). On n8n Cloud, look at the output of the **Rows:** nodes instead.<br><br>**Your own data**<br>1. Etsy Shop Manager → Finances → Payment account → *Monthly statements* → CSV (one per month) and Settings → Options → Download Data → Orders → *Order Items* CSV (same months).<br>2. Make a cost table with columns `sku, listing_id, item_name, unit_cost, extra_cost_per_unit` (fill sku OR listing_id OR item_name).<br>3. **Files** (self-hosted): put them in `filesFolder` as `etsy_statement*.csv`, `EtsySoldOrderItems*.csv`, `cogs.csv`, set `source: \"files\"`.<br>   **Google Sheet**: tabs `Statement`, `OrderItems`, `COGS` (File → Import → each CSV), plus empty tabs `P&L`, `By Listing`, `Summary`. Paste the sheet URL into all 6 Google Sheets nodes, set `source` and `output` to `\"sheet\"`.<br>4. Optional AI summary: Header Auth credential, name `x-goog-api-key`, value = your Gemini key; pick it in **Gemini summary**; set `mode: \"gemini\"`. Only totals and listing titles are sent, never buyer names or addresses.<br><br>n8n Cloud is a paid n8n subscription; self-hosting needs your own computer or server. Gemini API costs, if any, are billed to you by Google.<br>Not tax or accounting advice. Check numbers against your Etsy statement.<br>Etsy and n8n are trademarks of their respective owners. Stallkit Studio is not affiliated with or endorsed by them. |
| Read Statement tab | n8n-nodes-base.googleSheets | Reads statement records from Google Sheets. | Source = sheet? | Read OrderItems tab | |
| Read OrderItems tab | n8n-nodes-base.googleSheets | Reads order items from Google Sheets. | Read Statement tab | Read COGS tab | |
| Read COGS tab | n8n-nodes-base.googleSheets | Reads COGS table from Google Sheets. | Read OrderItems tab | Bundle inputs | |
| Read statement CSVs | n8n-nodes-base.readWriteFile | Reads local Etsy statement CSV files. | Source = sheet? | Parse statement CSV | |
| Parse statement CSV | n8n-nodes-base.extractFromFile | Parses statement CSV data into JSON. | Read statement CSVs | Read Order Items CSVs | |
| Read Order Items CSVs | n8n-nodes-base.readWriteFile | Reads local order items CSV files. | Parse statement CSV | Parse Order Items CSV | |
| Parse Order Items CSV | n8n-nodes-base.extractFromFile | Parses order items CSV data into JSON. | Read Order Items CSVs | Read COGS CSV | |
| Read COGS CSV | n8n-nodes-base.readWriteFile | Reads local COGS CSV files. | Parse Order Items CSV | Parse COGS CSV | |
| Parse COGS CSV | n8n-nodes-base.extractFromFile | Parses COGS CSV data into JSON. | Read COGS CSV | Bundle inputs | |
| Bundle inputs | n8n-nodes-base.code | Bundles all data inputs into a unified object. | Read COGS tab, Parse COGS CSV | Compute P&L | |
| Compute P&L | n8n-nodes-base.code | Computes P&L, listing profits, validation checks, and AI prompts. | Bundle inputs | Mode = gemini? | |
| Mode = gemini? | n8n-nodes-base.if | Routes execution depending on whether mode is gemini. | Compute P&L | Gemini summary, Rule-based summary | |
| Gemini summary | n8n-nodes-base.httpRequest | Calls Google Gemini API to generate an AI summary. | Mode = gemini? | Read Gemini reply | |
| Read Gemini reply | n8n-nodes-base.code | Parses Gemini response or falls back to rule-based summary. | Gemini summary | Rows: P&L, Rows: By Listing, Rows: Summary | |
| Rule-based summary | n8n-nodes-base.code | Generates rule-based text summary (mock mode / fallback). | Mode = gemini? | Rows: P&L, Rows: By Listing, Rows: Summary | |
| Rows: P&L | n8n-nodes-base.code | Formats monthly P&L rows for output. | Read Gemini reply, Rule-based summary | P&L to sheet? | |
| P&L to sheet? | n8n-nodes-base.if | Routes P&L data to Google Sheets or CSV output. | Rows: P&L | Write 'P&L' tab, P&L to CSV | |
| Write 'P&L' tab | n8n-nodes-base.googleSheets | Appends or updates P&L data in Google Sheets. | P&L to sheet? | None | |
| P&L to CSV | n8n-nodes-base.convertToFile | Converts P&L rows to CSV format. | P&L to sheet? | Save etsy_profit_monthly.csv | |
| Save etsy_profit_monthly.csv | n8n-nodes-base.readWriteFile | Writes monthly P&L CSV to disk. | P&L to CSV | None | |
| Rows: By Listing | n8n-nodes-base.code | Formats listing profit rows for output. | Read Gemini reply, Rule-based summary | By Listing to sheet? | |
| By Listing to sheet? | n8n-nodes-base.if | Routes listing data to Google Sheets or CSV output. | Rows: By Listing | Write 'By Listing' tab, By Listing to CSV | |
| Write 'By Listing' tab | n8n-nodes-base.googleSheets | Appends or updates listing profit rows in Google Sheets. | By Listing to sheet? | None | |
| By Listing to CSV | n8n-nodes-base.convertToFile | Converts listing rows to CSV format. | By Listing to sheet? | Save etsy_profit_by_listing.csv | |
| Save etsy_profit_by_listing.csv | n8n-nodes-base.readWriteFile | Writes listing profit CSV to disk. | By Listing to CSV | None | |
| Rows: Summary | n8n-nodes-base.code | Formats summary and reconciliation issues for output. | Read Gemini reply, Rule-based summary | Summary to sheet? | |
| Summary to sheet? | n8n-nodes-base.if | Routes summary data to Google Sheets or CSV output. | Rows: Summary | Write 'Summary' tab, Summary to CSV | |
| Write 'Summary' tab | n8n-nodes-base.googleSheets | Appends summary rows to Google Sheets. | Summary to sheet? | None | |
| Summary to CSV | n8n-nodes-base.convertToFile | Converts summary rows to CSV format. | Summary to sheet? | Save etsy_profit_summary.csv | |
| Save etsy_profit_summary.csv | n8n-nodes-base.readWriteFile | Writes summary CSV to disk. | Summary to CSV | None | |

---

### 4. Reproducing the Workflow from Scratch

To manually rebuild this workflow in n8n without using the JSON, follow these exact steps:

1. **Create the Entry Point:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`) named `Run`.

2. **Configure Settings:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `Config`.
   - Paste the configuration JavaScript code (defining `mode`, `source`, `output`, `filesFolder`, `outputFolder`, `currency`, `dateOrder`, etc.).
   - Connect `Run` to `Config`.

3. **Set Up Input Branching:**
   - Add an **If** node (`n8n-nodes-base.if`) named `Source = sample?`. Set condition: `{{ $json.source }}` equals `"sample"`. Connect `Config` to this node.
   - Add a second **If** node named `Source = sheet?`. Set condition: `{{ $json.source }}` equals `"sheet"`. Connect the `false` output of `Source = sample?` to this node.

4. **Build Sample Data & File Ingestion Branches:**
   - **Sample Branch (`true` from `Source = sample?`):** Add a **Code** node named `Sample Etsy exports (fake data)` containing the mock statement, order items, and COGS arrays. Connect output to `Compute P&L`.
   - **Google Sheets Branch (`true` from `Source = sheet?`):**
     - Add **Google Sheets** node `Read Statement tab` (Sheet Name: `Statement`).
     - Add **Google Sheets** node `Read OrderItems tab` (Sheet Name: `OrderItems`, executeOnce: true).
     - Add **Google Sheets** node `Read COGS tab` (Sheet Name: `COGS`, executeOnce: true).
     - Chain them sequentially: `Read Statement tab` → `Read OrderItems tab` → `Read COGS tab` → `Bundle inputs`.
   - **Local Files Branch (`false` from `Source = sheet?`):**
     - Add **Read/Write Files from Disk** node `Read statement CSVs` (`fileSelector`: `{{ $('Config').first().json.filesFolder }}/etsy_statement*.csv`).
     - Add **Extract from File** node `Parse statement CSV` (Operation: CSV).
     - Add **Read/Write Files from Disk** node `Read Order Items CSVs` (`fileSelector`: `{{ $('Config').first().json.filesFolder }}/EtsySoldOrderItems*.csv`).
     - Add **Extract from File** node `Parse Order Items CSV` (Operation: CSV).
     - Add **Read/Write Files from Disk** node `Read COGS CSV` (`fileSelector`: `{{ $('Config').first().json.filesFolder }}/cogs*.csv`).
     - Add **Extract from File** node `Parse COGS CSV` (Operation: CSV).
     - Chain them sequentially: `Read statement CSVs` → `Parse statement CSV` → `Read Order Items CSVs` → `Parse Order Items CSV` → `Read COGS CSV` → `Parse COGS CSV` → `Bundle inputs`.

5. **Bundle and Compute:**
   - Add a **Code** node named `Bundle inputs` that collects statement, orderItems, and cogs inputs into a single item object. Connect both `Read COGS tab` and `Parse COGS CSV` to this node.
   - Add a **Code** node named `Compute P&L` containing the core financial parsing, calculation, reconciliation, and prompt generation script. Connect `Bundle inputs` to this node.

6. **Handle AI vs. Rule-Based Summary:**
   - Add an **If** node named `Mode = gemini?`. Set condition: `{{ $json._cfg.mode }}` equals `"gemini"`. Connect `Compute P&L` to this node.
   - **Gemini Branch (`true`):**
     - Add an **HTTP Request** node named `Gemini summary`. Method: `POST`, URL: `https://generativelanguage.googleapis.com/v1beta/models/{{ $json._cfg.geminiModel }}:generateContent`. Authentication: Generic Header Auth (`x-goog-api-key`). Set JSON body containing the AI prompt. Enable retry on fail and error continuation.
     - Add a **Code** node named `Read Gemini reply` to parse the API response.
   - **Rule-Based Branch (`false`):**
     - Add a **Code** node named `Rule-based summary` to assign rule-based summaries.

7. **Format Output Data Rows:**
   - From both `Read Gemini reply` and `Rule-based summary`, connect outputs to three separate **Code** formatting nodes:
     - `Rows: P&L`
     - `Rows: By Listing`
     - `Rows: Summary`

8. **Handle Output Routing (Sheets vs. CSV):**
   - For each of the three output streams, add an **If** node (`P&L to sheet?`, `By Listing to sheet?`, `Summary to sheet?`) checking if `{{ $('Config').first().json.output }}` equals `"sheet"`.
   - **Sheet Branch (`true`):**
     - Add corresponding **Google Sheets** nodes (`Write 'P&L' tab`, `Write 'By Listing' tab`, `Write 'Summary' tab`) with appropriate operation types (`appendOrUpdate` or `append`) pointing to your Google Sheet document ID and respective tab names.
   - **CSV Branch (`false`):**
     - Add **Convert to File** nodes (`P&L to CSV`, `By Listing to CSV`, `Summary to CSV`).
     - Add **Read/Write Files from Disk** nodes (`Save etsy_profit_monthly.csv`, `Save etsy_profit_by_listing.csv`, `Save etsy_profit_summary.csv`) configured with write operation paths pointing to `filesFolder`.

9. **Credentials Setup:**
   - Configure **Google Sheets OAuth2 API** credentials for all Google Sheets nodes (if using sheet mode).
   - Configure **Header Auth** credentials named `Gemini API Key` with header name `x-goog-api-key` and your Google AI Studio API key (if using gemini mode).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Stallkit Studio Free Download | https://stallkitstudio.gumroad.com/l/profit-tracker |
| Stallkit Studio Store | https://stallkitstudio.gumroad.com |
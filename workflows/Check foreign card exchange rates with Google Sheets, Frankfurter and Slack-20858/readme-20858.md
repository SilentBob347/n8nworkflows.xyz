Check foreign card exchange rates with Google Sheets, Frankfurter and Slack

https://n8nworkflows.xyz/workflows/check-foreign-card-exchange-rates-with-google-sheets--frankfurter-and-slack-20858


# Check foreign card exchange rates with Google Sheets, Frankfurter and Slack

### 1. Workflow Overview

This workflow automates the monthly or on-demand auditing of foreign-currency card charges by comparing transaction amounts against historical market exchange rates. It targets individuals or businesses tracking travel or international card expenses who want to detect high exchange rate premiums, un-supported currencies, or data input errors without incurring API costs or utilizing AI models. 

The workflow logic is categorized into the following functional blocks:
- **1.1 Triggers and Configuration:** Handles scheduled or manual initiation and injects global configuration parameters (base currency, column mappings, premium thresholds, and API endpoints).
- **1.2 Data Ingestion and Planning:** Reads raw statement rows from Google Sheets, separates domestic transactions, parses dates and amounts, and groups foreign transactions by currency to build date-ranged API request URLs.
- **1.3 Rate Retrieval:** Fetches official market exchange rates from the external Frankfurter API.
- **1.4 Processing, Storage, and Reporting:** Calculates effective exchange rates, expected billed amounts, and premiums in custom JavaScript, updates transaction records in Google Sheets, builds a structured summary, and posts notifications to Slack.
- **1.5 Sheet Initialization (One-time setup):** Creates required Google Sheets tabs and default header columns.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggers and Configuration

- **Overview:** Initializes the execution flow on a monthly schedule or via manual user intervention, supplying global configuration settings to downstream nodes.
- **Nodes Involved:** `Every month`, `Check now`, `Config`
- **Node Details:**
  - **Every month**
    - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.4) — Cron-like trigger executing on the 1st day of every month at 09:00.
    - *Configuration choices:* Monthly frequency interval.
    - *Input/Output:* No inputs; outputs execution timestamp.
    - *Edge cases:* Missed executions if n8n instance is offline during the exact trigger hour.
  - **Check now**
    - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` (v1) — Interactive trigger for manual testing.
    - *Configuration choices:* None.
    - *Input/Output:* No inputs; outputs manual execution signal.
  - **Config**
    - *Type & Technical Role:* `n8n-nodes-base.set` (v3.4) — Injects static configuration parameters.
    - *Configuration choices:* Sets `base_currency` (`JPY`), column name mappings (`Date`, `Merchant`, `Billed`, `Foreign amount`, `Currency`), warning limits (`premium_warn_pct: 4`, `premium_low_pct: -2`), `lookback_days` (`7`), and `fx_api` (`https://api.frankfurter.dev/v1`).
    - *Input/Output:* Connected from triggers; outputs configuration object to `Read FX statement tab`.

#### 1.2 Data Ingestion and Planning

- **Overview:** Reads transaction statement rows from Google Sheets, filters out domestic records, validates and parses date and amount formats, and plans necessary HTTP requests for missing exchange rates by currency.
- **Nodes Involved:** `Read FX statement tab`, `Plan rates`, `Any foreign charges?`, `Nothing to check`
- **Node Details:**
  - **Read FX statement tab**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (v4.7) — Reads rows from the `FX statement` tab.
    - *Configuration choices:* Operation set to `read`, sheet name configured as `FX statement`, output formatting configured to preserve serial numbers and unformatted values. `executeOnce: true`.
    - *Input/Output:* Input from `Config`; outputs all row objects.
    - *Edge cases:* Authentication token expiration; missing sheet name causing errors.
  - **Plan rates**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript execution block containing custom normalization, date/amount parsing, transaction hashing, and batch rate planning.
    - *Configuration choices:* Executes inline JS verifying required columns exist, mapping transaction records, and creating distinct API URLs per currency covering historical date boundaries.
    - *Input/Output:* Input from Google Sheets; outputs an array of currency date ranges (`url`, `currency`, `from`, `to`) or a single `none: true` flag if no foreign transactions exist.
    - *Edge cases:* Throws descriptive errors if required sheet columns are missing.
  - **Any foreign charges?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (v2.2) — Conditional router.
    - *Configuration choices:* Evaluates whether `$json.none` is false (proceed to fetch rates) or true (route to `Nothing to check`).
    - *Input/Output:* Input from `Plan rates`; dual outputs.
  - **Nothing to check**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — Formats a notification payload when no foreign currencies are discovered.
    - *Configuration choices:* Builds text object string.
    - *Input/Output:* Input from `Any foreign charges?` (false branch); outputs text to `Post to Slack`.

#### 1.3 Rate Retrieval

- **Overview:** Executes HTTP GET requests against the public Frankfurter API for each distinct foreign currency detected in the transaction batch.
- **Nodes Involved:** `Get rates`
- **Node Details:**
  - **Get rates**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (v4.5) — Fetches historical exchange rate time-series data.
    - *Configuration choices:* Method `GET`, URL set dynamically to `={{ $json.url }}`, request timeout set to `30000ms`, batching enabled with batch size `1` and interval `500ms`. Error handling set to `continueRegularOutput`.
    - *Input/Output:* Input from `Any foreign charges?`; outputs API JSON response payloads.
    - *Edge cases:* API downtime, unsupported currency returning HTTP 404, or rate-limiting throttling.

#### 1.4 Processing, Storage, and Reporting

- **Overview:** Matches transaction dates against fetched rate series, computes premiums and deviations, updates individual transaction rows in Google Sheets using a unique row key, compiles a summary, and delivers alerts to Slack.
- **Nodes Involved:** `Check charges`, `Add to FX check tab`, `Build summary`, `Post to Slack`
- **Node Details:**
  - **Check charges**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — JavaScript processing block calculating effective rates, market rates, expected bills, percentage premiums, and assigning validation status flags (`OK`, `High premium`, `Below market`, `Rate unavailable`, `Check data`).
    - *Configuration choices:* Aggregates HTTP rate inputs with planned charges using custom lookup functions (`rateFor`).
    - *Input/Output:* Input from `Get rates`; outputs processed row objects.
  - **Add to FX check tab**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (v4.7) — Writes or updates transaction evaluation results in Google Sheets.
    - *Configuration choices:* Operation set to `appendOrUpdate`, sheet name `FX check`, matching column configured as `row_key`, cell format set to `RAW`.
    - *Input/Output:* Input from `Check charges`; outputs updated sheet row data.
    - *Edge cases:* Google Sheets API write limits or permission errors.
  - **Build summary**
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) — Summarizes totals, highest premium violations, and flagged data issues into a Slack-compatible markdown string.
    - *Configuration choices:* Filters and sorts extreme outliers.
    - *Input/Output:* Input from `Add to FX check tab`; outputs markdown text.
  - **Post to Slack**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (v2.7) — Posts formatted audit reports to a designated Slack channel.
    - *Configuration choices:* Resource `message`, operation `post`, message type `text`, channel ID selection, content mapped to `={{ $json.text }}`.
    - *Input/Output:* Inputs from `Build summary` or `Nothing to check`; outputs API response.
    - *Edge cases:* Invalid Slack channel ID or unauthenticated bot permissions.

#### 1.5 Sheet Initialization (One-time setup)

- **Overview:** Optional branch used to initialize a blank Google Spreadsheet with required tab names and header columns.
- **Nodes Involved:** `Run once: set up the sheet`, `Spreadsheet ID`, `Add the FX statement and FX check tabs`, `Write the header rows`
- **Node Details:**
  - **Run once: set up the sheet**
    - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` (v1) — Manual execution starter.
  - **Spreadsheet ID**
    - *Type & Technical Role:* `n8n-nodes-base.set` (v3.4) — Stores the target Google Spreadsheet ID string (`PASTE_YOUR_SPREADSHEET_ID_HERE`).
  - **Add the FX statement and FX check tabs**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (v4.5) — Calls Google Sheets v4 API (`batchUpdate`) to add sheets `FX statement` and `FX check`. Authenticated via Google Sheets OAuth2.
  - **Write the header rows**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (v4.5) — Calls Google Sheets v4 API (`values:batchUpdate`) to write exact column headers for both tabs.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every month | scheduleTrigger | Triggers workflow execution on the 1st of every month | None | Config | ## 1. Triggers and settings<br>Runs monthly or on demand. Config holds your currency, the column names and the premium limits. |
| Check now | manualTrigger | Manually triggers workflow execution | None | Config | ## 1. Triggers and settings<br>Runs monthly or on demand. Config holds your currency, the column names and the premium limits. |
| Config | set | Sets global configuration variables and thresholds | Every month, Check now | Read FX statement tab | ## 1. Triggers and settings<br>Runs monthly or on demand. Config holds your currency, the column names and the premium limits. |
| Read FX statement tab | googleSheets | Reads rows from the statement sheet | Config | Plan rates | ## 2. Which rows, which rates?<br>Finds the foreign rows and the date range of each currency. A wrong column name stops the run with a clear message. |
| Plan rates | code | Plans currency date ranges and parses rows | Read FX statement tab | Any foreign charges? | ## 2. Which rows, which rates?<br>Finds the foreign rows and the date range of each currency. A wrong column name stops the run with a clear message. |
| Any foreign charges? | if | Checks if any foreign transactions exist | Plan rates | Get rates, Nothing to check | ## 2. Which rows, which rates?<br>Finds the foreign rows and the date range of each currency. A wrong column name stops the run with a clear message. |
| Nothing to check | code | Prepares notification text when no foreign transactions exist | Any foreign charges? | Post to Slack | ## 2. Which rows, which rates?<br>Finds the foreign rows and the date range of each currency. A wrong column name stops the run with a clear message. |
| Get rates | httpRequest | Fetches historical FX rate data from Frankfurter API | Any foreign charges? | Check charges | ## 3. Market rates<br>One request per currency to the free Frankfurter API (no key). A currency it does not know is reported, not guessed. |
| Check charges | code | Calculates rates, premiums, and status flags | Get rates | Add to FX check tab | ## 4. Check in code, save, report<br>Code calculates the expected amount and the premium. Results are saved by row, so a re-run does not duplicate. Slack gets the summary. |
| Add to FX check tab | googleSheets | Updates or appends evaluated rows into Google Sheets | Check charges | Build summary | ## 4. Check in code, save, report<br>Code calculates the expected amount and the premium. Results are saved by row, so a re-run does not duplicate. Slack gets the summary. |
| Build summary | code | Builds a Slack summary message from processed rows | Add to FX check tab | Post to Slack | ## 4. Check in code, save, report<br>Code calculates the expected amount and the premium. Results are saved by row, so a re-run does not duplicate. Slack gets the summary. |
| Post to Slack | slack | Posts notifications or summaries to Slack | Build summary, Nothing to check | None | ## 4. Check in code, save, report<br>Code calculates the expected amount and the premium. Results are saved by row, so a re-run does not duplicate. Slack gets the summary. |
| Run once: set up the sheet | manualTrigger | Manual trigger for initial sheet setup | None | Spreadsheet ID | ## Run once: set up the sheet<br>Paste your spreadsheet id, run this branch once, and it adds the two tabs and the header rows. Running it twice fails, because the tabs exist. |
| Spreadsheet ID | set | Stores target spreadsheet ID for setup | Run once: set up the sheet | Add the FX statement and FX check tabs | ## Run once: set up the sheet<br>Paste your spreadsheet id, run this branch once, and it adds the two tabs and the header rows. Running it twice fails, because the tabs exist. |
| Add the FX statement and FX check tabs | httpRequest | Creates the required spreadsheet tabs via API | Spreadsheet ID | Write the header rows | ## Run once: set up the sheet<br>Paste your spreadsheet id, run this branch once, and it adds the two tabs and the header rows. Running it twice fails, because the tabs exist. |
| Write the header rows | httpRequest | Writes column headers for both configured tabs | Add the FX statement and FX check tabs | None | ## Run once: set up the sheet<br>Paste your spreadsheet id, run this branch once, and it adds the two tabs and the header rows. Running it twice fails, because the tabs exist. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers & Configuration:**
   - Add a **Schedule Trigger** node named `Every month`, setting its interval to monthly on day 1 at 09:00.
   - Add a **Manual Trigger** node named `Check now`.
   - Add an **Edit Fields (Set)** node named `Config`. Assign the following manual string/number fields:
     - `base_currency` = `JPY`
     - `date_col` = `Date`
     - `merchant_col` = `Merchant`
     - `billed_col` = `Billed`
     - `foreign_col` = `Foreign amount`
     - `currency_col` = `Currency`
     - `premium_warn_pct` = `4`
     - `premium_low_pct` = `-2`
     - `lookback_days` = `7`
     - `fx_api` = `https://api.frankfurter.dev/v1`
   - Connect both `Every month` and `Check now` outputs to `Config`.

2. **Ingest and Plan Statement Data:**
   - Add a **Google Sheets** node named `Read FX statement tab`. Set resource to `Sheet`, operation to `read`, and sheet name to `FX statement`. Configure output formatting options to serial numbers/unformatted values. Enable `executeOnce`.
   - Connect `Config` output to `Read FX statement tab`.
   - Add a **Code** node named `Plan rates`. Paste the planning script from the original workflow JSON into the JavaScript code parameter.
   - Connect `Read FX statement tab` output to `Plan rates`.
   - Add an **If** node named `Any foreign charges?`. Set condition to evaluate if `{{ $json.none }}` is `false`.
   - Connect `Plan rates` output to `Any foreign charges?`.
   - Add a **Code** node named `Nothing to check` for handling zero foreign transactions. Connect the `false` output of `Any foreign charges?` to this node.

3. **Fetch Exchange Rates:**
   - Add an **HTTP Request** node named `Get rates`. Set method to `GET`, URL to `={{ $json.url }}`, and timeout to `30000ms`. Enable batching (batch size 1, interval 500ms). Set error handling to continue regular output.
   - Connect the `true` output of `Any foreign charges?` to `Get rates`.

4. **Process, Store, and Report Results:**
   - Add a **Code** node named `Check charges`. Paste the evaluation and check script into the JavaScript execution block.
   - Connect `Get rates` output to `Check charges`.
   - Add a **Google Sheets** node named `Add to FX check tab`. Set resource to `Sheet`, operation to `appendOrUpdate`, sheet name to `FX check`, and matching column to `row_key`.
   - Connect `Check charges` output to `Add to FX check tab`.
   - Add a **Code** node named `Build summary`. Paste the summary formatting script into the JavaScript box.
   - Connect `Add to FX check tab` output to `Build summary`.
   - Add a **Slack** node named `Post to Slack`. Set resource to `Message`, operation to `Post`, select channel type, and bind text to `={{ $json.text }}`.
   - Connect the outputs of both `Build summary` and `Nothing to check` to `Post to Slack`.

5. **Configure Credentials:**
   - Configure **Google Sheets OAuth2 API** credentials for the Google Sheets nodes and HTTP setup requests.
   - Configure **Slack API** credentials for the `Post to Slack` node.

6. **Optional Sheet Initialization Setup Branch:**
   - Add a **Manual Trigger** node named `Run once: set up the sheet`.
   - Add a **Set** node named `Spreadsheet ID` containing string assignment `spreadsheet_id` with your target spreadsheet ID value.
   - Add an **HTTP Request** node named `Add the FX statement and FX check tabs` (POST to `https://sheets.googleapis.com/v4/spreadsheets/{{ $('Spreadsheet ID').first().json.spreadsheet_id }}:batchUpdate` with JSON body creating `FX statement` and `FX check` sheets). Authenticated via Google Sheets OAuth2.
   - Add an **HTTP Request** node named `Write the header rows` (POST to `https://sheets.googleapis.com/v4/spreadsheets/{{ $('Spreadsheet ID').first().json.spreadsheet_id }}/values:batchUpdate` with header arrays for both sheets).
   - Connect nodes sequentially: `Run once: set up the sheet` $\rightarrow$ `Spreadsheet ID` $\rightarrow$ `Add the FX statement and FX check tabs` $\rightarrow$ `Write the header rows`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Frankfurther API Overview | Free public ECB reference rate API requiring no authentication keys. Subject to fair-use rate limits and provider terms. |
| Financial Disclaimer | Market rates serve as a yardstick; actual card network rates incorporate issuer fees and posting date variations. |
| Sheet Initialization Notice | The one-time setup branch (`Run once: set up the sheet`) will fail if the target spreadsheet tabs already exist. |
Reprice gold chain stock daily and announce it on WhatsApp via Twilio and Google Sheets

https://n8nworkflows.xyz/workflows/reprice-gold-chain-stock-daily-and-announce-it-on-whatsapp-via-twilio-and-google-sheets-20938


# Reprice gold chain stock daily and announce it on WhatsApp via Twilio and Google Sheets

### 1. Workflow Overview

This workflow automates the daily repricing of a gold chain wholesale catalogue on weekdays, compiles updated stock information, constructs customer-facing messaging focusing purely on recent inventory changes, requests owner authorization via WhatsApp, and subsequently broadcasts the approved announcement while updating the underlying Google Sheets record. 

The primary target use cases are jewellery wholesalers and gold traders seeking to eliminate manual daily price recalculations and streamline outbound customer communications over WhatsApp.

The workflow logic is categorized into the following functional blocks:

- **1.1 Input Reception & Configuration:** Initializes execution parameters via manual or scheduled triggers and loads global business rules, pricing matrices, margins, and fallback demo data.
- **1.2 Stock Ingestion & Validation:** Pulls inventory rows from Google Sheets or defaults to built-in demo stock, validating row-level metrics such as weight, karat purity, and style codes.
- **1.3 Gold Rate Acquisition & Pricing:** Fetches the daily 24k gold spot rate via external API (or fallback demo rate), calculates metal and making charges across inventory items, and evaluates price movement tolerances.
- **1.4 Catalogue & Broadcast Compilation:** Generates structured outputs including customer catalogues, internal inventories, CSV exports, delta announcements, and pre-filtered recipient target lists.
- **1.5 Authorization, Distribution & Sync:** Manages owner approval loops via interactive webhook resume links, executes bulk Twilio WhatsApp dispatches, and logs updated pricing metrics back to Google Sheets.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration

**Overview:**  
This block initializes the execution flow via manual execution or a cron schedule, passing control to a centralized configuration engine that establishes operational rules, pricing variables, feature flags, and built-in fallback datasets.

**Nodes Involved:**
- `Every weekday, 08:00`
- `Run it now`
- `config`

**Node Details:**

- **Every weekday, 08:00**
  - **Type & Role:** Schedule Trigger (Version 1.2). Automatically initiates the workflow at 08:00 AM from Monday to Friday.
  - **Configuration Choices:** Cron expression rule configured to `0 8 * * 1-5`.
  - **Expressions/Variables:** None.
  - **Connections:** Output connects to `config`.
  - **Version Requirements:** Version 1.2.
  - **Edge Cases & Failures:** Missed executions if n8n instance is offline during the specific minute window.

- **Run it now**
  - **Type & Role:** Manual Trigger (Version 1). Allows administrators to test or force-run the workflow on demand.
  - **Configuration Choices:** Default settings (no parameters required).
  - **Expressions/Variables:** None.
  - **Connections:** Output connects to `config`.
  - **Version Requirements:** Version 1.

- **config**
  - **Type & Role:** Code Node (Version 2). Serves as the single source of truth for business parameters, margin multipliers, style definitions, API keys, safety switches, and hardcoded demo datasets.
  - **Configuration Choices:** Executes JavaScript evaluating global constants and constructing a unified `SETTINGS` dictionary object passed downstream.
  - **Expressions/Variables:** Evaluates incoming execution payloads via `$input.all()`.
  - **Connections:** Input from triggers; output connects to `stock sheet configured?`.
  - **Version Requirements:** Version 2.
  - **Edge Cases & Failures:** JavaScript syntax errors or invalid JSON structural updates will halt the entire execution at inception.

---

#### Block 1.2: Stock Ingestion & Validation

**Overview:**  
This block checks whether a live Google Sheets ID is provided, routing execution either to read live stock data or fall back to internal demo inventory, followed by rigorous data validation that isolates incomplete or corrupted rows.

**Nodes Involved:**
- `stock sheet configured?`
- `[cred] Sheets - read the stock`
- `use the demo stock`
- `stock-collect` (named: `collect the stock`)
- `check the stock rows`

**Node Details:**

- **stock sheet configured?**
  - **Type & Role:** If Node (Version 2.2). Evaluates whether the live Google Sheets integration is enabled.
  - **Configuration Choices:** Boolean check on `{{ $('config').first().json.sheet_enabled }}` equals `true`.
  - **Expressions/Variables:** Uses config JSON state.
  - **Connections:** True branch connects to `[cred] Sheets - read the stock`; False branch connects to `use the demo stock`.
  - **Version Requirements:** Version 2.2.

- **[cred] Sheets - read the stock**
  - **Type & Role:** Google Sheets Node (Version 4.5). Reads all rows from the designated stock worksheet tab.
  - **Configuration Choices:** Operation mode set to read rows; document ID and sheet name bound dynamically to configuration parameters.
  - **Expressions/Variables:** `documentId` uses `{{ $('config').first().json.sheet_id }}`; `sheetName` uses `{{ $('config').first().json.stock_tab }}`.
  - **Connections:** Input from `stock sheet configured?`; output connects to `collect the stock`.
  - **Version Requirements:** Version 4.5.
  - **Edge Cases & Failures:** Authentication failures, expired OAuth tokens, or deleted sheet tabs.

- **use the demo stock**
  - **Type & Role:** Code Node (Version 2). Supplies fallback mock inventory items when live sheet integration is disabled.
  - **Configuration Choices:** Returns predefined array of inventory records from the config scope.
  - **Expressions/Variables:** Reads `cfg.demo_stock`.
  - **Connections:** Input from `stock sheet configured?` (false branch); output connects to `check the stock rows`.
  - **Version Requirements:** Version 2.

- **collect the stock**
  - **Type & Role:** Code Node (Version 2). Aggregates individual row outputs from the Google Sheets read node into a single array structure.
  - **Configuration Choices:** Filters out blank SKUs.
  - **Expressions/Variables:** Maps `$input.all()`.
  - **Connections:** Input from `[cred] Sheets - read the stock`; output connects to `check the stock rows`.
  - **Version Requirements:** Version 2.

- **check the stock rows**
  - **Type & Role:** Code Node (Version 2). Validates row-level data integrity, ensuring items possess valid SKUs, recognized styles, supported purities, positive weights, and valid quantities.
  - **Configuration Choices:** Inline parsing rules and purity/style mapping validations. Holds back unpriceable rows without dropping them silently.
  - **Expressions/Variables:** Evaluates row attributes against configuration tables.
  - **Connections:** Input from `collect the stock` or `use the demo stock`; output connects to `gold rate configured?`.
  - **Version Requirements:** Version 2.

---

#### Block 1.3: Gold Rate Acquisition & Pricing

**Overview:**  
This block checks for a configured API key, fetches the current daily 24k gold spot rate via HTTP, validates rate plausibility against historical thresholds, and executes comprehensive pricing calculations across all valid stock rows.

**Nodes Involved:**
- `gold rate configured?`
- `Gold rate - today's price`
- `use the demo gold rate`
- `rate-read` (named: `read the gold rate`)
- `rate-fail` (named: `STOP: gold rate unavailable`)
- `price` (named: `price every chain`)
- `rate looks right?`
- `rate-wrong` (named: `STOP: rate looks wrong`)

**Node Details:**

- **gold rate configured?**
  - **Type & Role:** If Node (Version 2.2). Determines if live gold API credentials are provided.
  - **Configuration Choices:** Boolean check on `{{ $('config').first().json.gold_api_enabled }}` equals `true`.
  - **Connections:** True branch connects to `Gold rate - today's price`; False branch connects to `use the demo gold rate`.
  - **Version Requirements:** Version 2.2.

- **Gold rate - today's price**
  - **Type & Role:** HTTP Request Node (Version 4.2). Queries `goldapi.io` for current spot prices in the specified currency.
  - **Configuration Choices:** GET request with custom headers (`x-access-token`, `Accept`), timeout configured to 20,000ms. Error output configured to continue on error.
  - **Expressions/Variables:** URL constructed via `{{ $('config').first().json.gold_api_base + '/XAU/' + $('config').first().json.currency }}`.
  - **Connections:** Input from `gold rate configured?`; outputs connect to `read the gold rate` (success) and `STOP: gold rate unavailable` (error).
  - **Version Requirements:** Version 4.2.
  - **Edge Cases & Failures:** API rate limits, timeouts, invalid API keys, or unexpected JSON payload structures.

- **read the gold rate**
  - **Type & Role:** Code Node (Version 2). Normalizes gold API responses into a standardized gram rate structure.
  - **Configuration Choices:** Fallback calculations evaluating Troy ounce conversions if direct gram fields are missing.
  - **Connections:** Input from HTTP request success; output connects to `price every chain`.
  - **Version Requirements:** Version 2.

- **STOP: gold rate unavailable**
  - **Type & Role:** Code Node (Version 2). Halts execution when the gold rate API fails.
  - **Configuration Choices:** Returns descriptive stop payload explaining the failure.
  - **Connections:** Input from HTTP request error output.
  - **Version Requirements:** Version 2.

- **use the demo gold rate**
  - **Type & Role:** Code Node (Version 2). Supplies a labeled fallback mock gold rate when live API keys are omitted.
  - **Configuration Choices:** Reads `cfg.demo_gold_rate`.
  - **Connections:** Input from `gold rate configured?` (false branch); output connects to `price every chain`.
  - **Version Requirements:** Version 2.

- **price every chain**
  - **Type & Role:** Code Node (Version 2). Applies pricing algorithms factoring in metal weights, karat purity fractions, making charges, wholesale margins, and rounding rules. Evaluates inventory delta changes (new, back in stock, sold out, last few).
  - **Configuration Choices:** Computes gross values and determines variance movements against historical medians.
  - **Connections:** Input from `read the gold rate` or `use the demo gold rate`; output connects to `rate looks right?`.
  - **Version Requirements:** Version 2.

- **rate looks right?**
  - **Type & Role:** If Node (Version 2.2). Validates that calculated market rate shifts do not exceed configured percentage tolerance thresholds (`MAX_RATE_MOVE_PCT`).
  - **Configuration Choices:** Checks `{{ $json.rate_ok }}` equals `true`.
  - **Connections:** True branch connects to `build the catalogue`; False branch connects to `STOP: rate looks wrong`.
  - **Version Requirements:** Version 2.2.

- **STOP: rate looks wrong**
  - **Type & Role:** Code Node (Version 2). Halts workflow execution if spot rate fluctuations exceed plausibility thresholds, preventing anomalous pricing broadcasts.
  - **Configuration Choices:** Outputs descriptive termination JSON.
  - **Connections:** Input from `rate looks right?` (false branch).
  - **Version Requirements:** Version 2.

---

#### Block 1.4: Catalogue & Broadcast Compilation

**Overview:**  
This block transforms priced inventory data into multiple unified outputs (customer catalogue, internal stock sheets, CSV formats, and change-focused announcements) and filters recipient lists based on opt-out statuses, validity, and broadcast volume caps.

**Nodes Involved:**
- `catalogue` (named: `build the catalogue`)
- `cust-on` (named: `customer list configured?`)
- `[cred] Sheets - read the customers`
- `cust-collect` (named: `collect the customers`)
- `cust-demo` (named: `use the demo customers`)
- `plan` (named: `plan the broadcast`)

**Node Details:**

- **build the catalogue**
  - **Type & Role:** Code Node (Version 2). Generates synchronized text representations including customer-facing catalogues, accountant CSV strings, internal stock lists, and concise WhatsApp delta announcements highlighting additions or stock depletion.
  - **Configuration Choices:** Implements character truncation safeguards (1,500-character limits for WhatsApp).
  - **Connections:** Input from `rate looks right?` (true branch); output connects to `customer list configured?`.
  - **Version Requirements:** Version 2.

- **customer list configured?**
  - **Type & Role:** If Node (Version 2.2). Evaluates whether a live customer sheet is linked.
  - **Configuration Choices:** Boolean check on `{{ $('config').first().json.sheet_enabled }}` equals `true`.
  - **Connections:** True branch connects to `[cred] Sheets - read the customers`; False branch connects to `use the demo customers`.
  - **Version Requirements:** Version 2.

- **[cred] Sheets - read the customers**
  - **Type & Role:** Google Sheets Node (Version 4.5). Reads customer contact records and opt-out flags from the designated worksheet tab.
  - **Configuration Choices:** Bound dynamically to configuration sheet ID and customer tab name.
  - **Expressions/Variables:** Uses config parameters for ID and tab name.
  - **Connections:** Input from `customer list configured?`; output connects to `collect the customers`.
  - **Version Requirements:** Version 4.5.

- **collect the customers**
  - **Type & Role:** Code Node (Version 2). Aggregates sheet rows into a standardized customer array.
  - **Connections:** Input from `[cred] Sheets - read the customers`; output connects to `plan the broadcast`.
  - **Version Requirements:** Version 2.

- **use the demo customers**
  - **Type & Role:** Code Node (Version 2). Supplies fallback mock customer data when live sheet integration is disabled.
  - **Connections:** Input from `customer list configured?` (false branch); output connects to `plan the broadcast`.
  - **Version Requirements:** Version 2.

- **plan the broadcast**
  - **Type & Role:** Code Node (Version 2). Filters out opted-out contacts, malformed telephone numbers, and duplicate entries, enforcing hard broadcast volume ceilings (`BROADCAST_CAP`). Prepares the owner approval message payload.
  - **Configuration Choices:** Applies E.164 phone number formatting validations and constructs test-mode redirection payloads if enabled.
  - **Connections:** Input from `collect the customers` or `use the demo customers`; output connects to `sending on?`.
  - **Version Requirements:** Version 2.

---

#### Block 1.5: Authorization, Distribution & Sync

**Overview:**  
This block checks if live sending is enabled, sends an approval request via Twilio to the owner with an interactive webhook resume link, pauses execution pending approval, dispatches approved messages to customers, and optionally logs updated pricing back to Google Sheets.

**Nodes Involved:**
- `sending on?`
- `preview` (named: `STOP: preview - draft shown`)
- `ask` (named: `[cred] Twilio - ask the owner to approve`)
- `wait` (named: `wait for the owner's approval`)
- `approved?`
- `not-approved` (named: `STOP: not approved`)
- `per-customer` (named: `one message per customer`)
- `send` (named: `[cred] Twilio - send the announcement`)
- `sent` (named: `STOP: announcement sent`)
- `write-on` (named: `writing on?`)
- `rows` (named: `rows for the stock sheet`)
- `write` (named: `[cred] Sheets - write today's prices`)
- `written` (named: `STOP: prices written`)
- `not-written` (named: `STOP: preview - prices not written`)

**Node Details:**

- **sending on?**
  - **Type & Role:** If Node (Version 2.2). Evaluates whether Twilio dispatch permissions and operational modes permit live message sending.
  - **Configuration Choices:** Boolean check on `{{ $('config').first().json.send_enabled }}` equals `true`.
  - **Connections:** True branch connects to `[cred] Twilio - ask the owner to approve`; False branch connects to `STOP: preview - draft shown`.
  - **Version Requirements:** Version 2.2.

- **STOP: preview - draft shown**
  - **Type & Role:** Code Node (Version 2). Terminates execution in preview mode, outputting complete generated catalogues, stock counts, and draft approval messages without triggering external network dispatches.
  - **Connections:** Input from `sending on?` (false branch).
  - **Version Requirements:** Version 2.

- **[cred] Twilio - ask the owner to approve**
  - **Type & Role:** Twilio Node (Version 1). Sends a WhatsApp message to the business owner containing the announcement summary and an interactive approval link.
  - **Configuration Choices:** WhatsApp messaging enabled.
  - **Expressions/Variables:** `to` uses `{{ $json.owner_to }}`; `from` uses `{{ $('config').first().json.twilio_whatsapp_from }}`; message body incorporates `{{ $execution.resumeUrl + '?approve=yes' }}`.
  - **Connections:** Input from `sending on?` (true branch); output connects to `wait for the owner's approval`.
  - **Version Requirements:** Version 1.
  - **Edge Cases & Failures:** Twilio credential errors, incorrect recipient telephone formats, or rate limiting.

- **wait for the owner's approval**
  - **Type & Role:** Wait Node (Version 1.1). Pauses workflow execution awaiting webhook callback verification.
  - **Configuration Choices:** Resume type set to Webhook; limit wait time configured using config timeout parameters (`APPROVAL_WAIT_HOURS`).
  - **Expressions/Variables:** Resume unit in hours bound to `{{ $('config').first().json.approval_wait_hours }}`.
  - **Connections:** Input from Twilio approval node; output connects to `approved?`.
  - **Version Requirements:** Version 1.1.
  - **Edge Cases & Failures:** Expiration of the approval time window before owner interaction.

- **approved?**
  - **Type & Role:** If Node (Version 2.2). Validates query parameters returned from the webhook resume callback.
  - **Configuration Choices:** Evaluates boolean condition `{{ !!($json.query && $json.query.approve === 'yes') }}`.
  - **Connections:** True branch connects in parallel to `one message per customer` and `writing on?`; False branch connects to `STOP: not approved`.
  - **Version Requirements:** Version 2.2.

- **STOP: not approved**
  - **Type & Role:** Code Node (Version 2). Terminates execution when approval time windows expire or approval parameters are missing.
  - **Connections:** Input from `approved?` (false branch).
  - **Version Requirements:** Version 2.

- **one message per customer**
  - **Type & Role:** Code Node (Version 2). Splits recipient lists into individual item payloads for downstream processing.
  - **Connections:** Input from `approved?` (true branch); output connects to `[cred] Twilio - send the announcement`.
  - **Version Requirements:** Version 2.

- **[cred] Twilio - send the announcement**
  - **Type & Role:** Twilio Node (Version 1). Broadcasts approved WhatsApp announcement messages to each validated customer.
  - **Configuration Choices:** WhatsApp messaging enabled.
  - **Expressions/Variables:** `to` uses `{{ $json.to }}`; `from` uses `{{ $('config').first().json.twilio_whatsapp_from }}`; message body uses `{{ $json.body }}`.
  - **Connections:** Input from `one message per customer`; output connects to `STOP: announcement sent`.
  - **Version Requirements:** Version 1.
  - **Edge Cases & Failures:** Twilio dispatch failures, recipient blocking, or invalid destination numbers.

- **STOP: announcement sent**
  - **Type & Role:** Code Node (Version 2). Concludes successful broadcast execution runs.
  - **Connections:** Input from Twilio send node.
  - **Version Requirements:** Version 2.

- **writing on?**
  - **Type & Role:** If Node (Version 2.2). Evaluates whether live Google Sheets updating is permitted (`write_enabled`).
  - **Configuration Choices:** Boolean check on `{{ $('config').first().json.write_enabled }}` equals `true`.
  - **Connections:** True branch connects to `rows for the stock sheet`; False branch connects to `STOP: preview - prices not written`.
  - **Version Requirements:** Version 2.2.

- **rows for the stock sheet**
  - **Type & Role:** Code Node (Version 2). Formats updated pricing, gold rates, quantities, and timestamps mapped by SKU.
  - **Connections:** Input from `writing on?` (true branch); output connects to `[cred] Sheets - write today's prices`.
  - **Version Requirements:** Version 2.

- **[cred] Sheets - write today's prices**
  - **Type & Role:** Google Sheets Node (Version 4.5). Updates existing inventory rows in the Stock worksheet tab.
  - **Configuration Choices:** Operation set to `update`; cell format set to `RAW`; matching columns configured on `sku`.
  - **Expressions/Variables:** Document ID and sheet name bound to config parameters.
  - **Connections:** Input from `rows for the stock sheet`; output connects to `STOP: prices written`.
  - **Version Requirements:** Version 4.5.
  - **Edge Cases & Failures:** Sheet write permission errors, mismatched SKU headers, or API rate limitations.

- **STOP: prices written**
  - **Type & Role:** Code Node (Version 2). Concludes successful sheet persistence runs.
  - **Connections:** Input from Google Sheets write node.
  - **Version Requirements:** Version 2.

- **STOP: preview - prices not written**
  - **Type & Role:** Code Node (Version 2). Outputs simulated sheet update payloads without executing live sheet mutations during test or preview modes.
  - **Connections:** Input from `writing on?` (false branch).
  - **Version Requirements:** Version 2.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Run it now` | manualTrigger | Manual execution entry point | None | `config` | ## Reprice gold chain stock daily and announce it on WhatsApp<br><br>**Who's it for**<br>Jewellery wholesalers and gold traders who send a daily price list to customers on WhatsApp and rework every price by hand each morning.<br><br>**How it works**<br>Every weekday at 08:00, or by hand, it reads the Stock tab and checks every row: a chain with no weight or an unknown karat is held back with its reason. It fetches today's 24k rate from goldapi.io and prices each chain as weight x purity x rate, plus the style's making charge and your margin. If the rate moved more than `MAX_RATE_MOVE_PCT` since the last list, it stops. From the same priced rows it builds the catalogue, the stock list, a CSV and an announcement of what changed. The owner approves on WhatsApp before any customer receives it, then the prices are written back.<br><br>**How to set up**<br>1. Run **Run it now** first: demo stock, a labelled demo rate, nothing sent.<br>2. In `config`, set `PURITY`, `STYLES`, `WHOLESALE_MARGIN_PCT` and `SHEET_ID`.<br>3. Add `GOLD_API_KEY` and connect Google Sheets and Twilio on the `[cred]` nodes.<br>4. Set `TEST_WHATSAPP` to your number, then `TEST_RUN = false`.<br><br>**Requirements**<br>A Google Sheet with Stock and Customers tabs, a goldapi.io key, and a Twilio WhatsApp sender. WhatsApp only allows free-form messages within 24 hours of the customer's last message, so the broadcast needs an approved template.<br><br>**How to customize**<br>`STYLES` holds making charges per gram, `ROUND_TO` and `CURRENCY` shape the prices, and `BROADCAST_CAP` and `APPROVAL_WAIT_HOURS` bound the send. Import the companion enquiries workflow to answer replies at the approved price. |
| `Every weekday, 08:00` | scheduleTrigger | Scheduled weekday execution entry point | None | `config` | ## Reprice gold chain stock daily and announce it on WhatsApp<br><br>**Who's it for**<br>Jewellery wholesalers and gold traders who send a daily price list to customers on WhatsApp and rework every price by hand each morning.<br><br>**How it works**<br>Every weekday at 08:00, or by hand, it reads the Stock tab and checks every row: a chain with no weight or an unknown karat is held back with its reason. It fetches today's 24k rate from goldapi.io and prices each chain as weight x purity x rate, plus the style's making charge and your margin. If the rate moved more than `MAX_RATE_MOVE_PCT` since the last list, it stops. From the same priced rows it builds the catalogue, the stock list, a CSV and an announcement of what changed. The owner approves on WhatsApp before any customer receives it, then the prices are written back.<br><br>**How to set up**<br>1. Run **Run it now** first: demo stock, a labelled demo rate, nothing sent.<br>2. In `config`, set `PURITY`, `STYLES`, `WHOLESALE_MARGIN_PCT` and `SHEET_ID`.<br>3. Add `GOLD_API_KEY` and connect Google Sheets and Twilio on the `[cred]` nodes.<br>4. Set `TEST_WHATSAPP` to your number, then `TEST_RUN = false`.<br><br>**Requirements**<br>A Google Sheet with Stock and Customers tabs, a goldapi.io key, and a Twilio WhatsApp sender. WhatsApp only allows free-form messages within 24 hours of the customer's last message, so the broadcast needs an approved template.<br><br>**How to customize**<br>`STYLES` holds making charges per gram, `ROUND_TO` and `CURRENCY` shape the prices, and `BROADCAST_CAP` and `APPROVAL_WAIT_HOURS` bound the send. Import the companion enquiries workflow to answer replies at the approved price. |
| `config` | code | Centralized configuration and settings initialization engine | `Run it now`, `Every weekday, 08:00` | `stock sheet configured?` | ## Reprice gold chain stock daily and announce it on WhatsApp<br><br>**Who's it for**<br>Jewellery wholesalers and gold traders who send a daily price list to customers on WhatsApp and rework every price by hand each morning.<br><br>**How it works**<br>Every weekday at 08:00, or by hand, it reads the Stock tab and checks every row: a chain with no weight or an unknown karat is held back with its reason. It fetches today's 24k rate from goldapi.io and prices each chain as weight x purity x rate, plus the style's making charge and your margin. If the rate moved more than `MAX_RATE_MOVE_PCT` since the last list, it stops. From the same priced rows it builds the catalogue, the stock list, a CSV and an announcement of what changed. The owner approves on WhatsApp before any customer receives it, then the prices are written back.<br><br>**How to set up**<br>1. Run **Run it now** first: demo stock, a labelled demo rate, nothing sent.<br>2. In `config`, set `PURITY`, `STYLES`, `WHOLESALE_MARGIN_PCT` and `SHEET_ID`.<br>3. Add `GOLD_API_KEY` and connect Google Sheets and Twilio on the `[cred]` nodes.<br>4. Set `TEST_WHATSAPP` to your number, then `TEST_RUN = false`.<br><br>**Requirements**<br>A Google Sheet with Stock and Customers tabs, a goldapi.io key, and a Twilio WhatsApp sender. WhatsApp only allows free-form messages within 24 hours of the customer's last message, so the broadcast needs an approved template.<br><br>**How to customize**<br>`STYLES` holds making charges per gram, `ROUND_TO` and `CURRENCY` shape the prices, and `BROADCAST_CAP` and `APPROVAL_WAIT_HOURS` bound the send. Import the companion enquiries workflow to answer replies at the approved price. |
| `stock sheet configured?` | if | Evaluates whether Google Sheets stock integration is enabled | `config` | `[cred] Sheets - read the stock`, `use the demo stock` | ### 1. Read the stock, and check it before pricing<br><br>`stock sheet configured?` reads the Stock tab through `[cred] Sheets - read the stock`, or falls back to `use the demo stock` so the whole run works with nothing connected.<br><br>`check the stock rows` holds back a chain with no weight, a karat missing from `PURITY` or a style missing from `STYLES`, with its reason. A row is never priced on a guess and never silently dropped. |
| `[cred] Sheets - read the stock` | googleSheets | Reads stock inventory rows from Google Sheets | `stock sheet configured?` | `collect the stock` | ### 1. Read the stock, and check it before pricing<br><br>`stock sheet configured?` reads the Stock tab through `[cred] Sheets - read the stock`, or falls back to `use the demo stock` so the whole run works with nothing connected.<br><br>`check the stock rows` holds back a chain with no weight, a karat missing from `PURITY` or a style missing from `STYLES`, with its reason. A row is never priced on a guess and never silently dropped. |
| `collect the stock` | code | Aggregates read sheet rows into a standardized array | `[cred] Sheets - read the stock` | `check the stock rows` | ### 1. Read the stock, and check it before pricing<br><br>`stock sheet configured?` reads the Stock tab through `[cred] Sheets - read the stock`, or falls back to `use the demo stock` so the whole run works with nothing connected.<br><br>`check the stock rows` holds back a chain with no weight, a karat missing from `PURITY` or a style missing from `STYLES`, with its reason. A row is never priced on a guess and never silently dropped. |
| `use the demo stock` | code | Provides fallback mock stock inventory | `stock sheet configured?` | `check the stock rows` | ### 1. Read the stock, and check it before pricing<br><br>`stock sheet configured?` reads the Stock tab through `[cred] Sheets - read the stock`, or falls back to `use the demo stock` so the whole run works with nothing connected.<br><br>`check the stock rows` holds back a chain with no weight, a karat missing from `PURITY` or a style missing from `STYLES`, with its reason. A row is never priced on a guess and never silently dropped. |
| `check the stock rows` | code | Validates data integrity of inventory rows before pricing | `collect the stock`, `use the demo stock` | `gold rate configured?` | ### 1. Read the stock, and check it before pricing<br><br>`stock sheet configured?` reads the Stock tab through `[cred] Sheets - read the stock`, or falls back to `use the demo stock` so the whole run works with nothing connected.<br><br>`check the stock rows` holds back a chain with no weight, a karat missing from `PURITY` or a style missing from `STYLES`, with its reason. A row is never priced on a guess and never silently dropped. |
| `gold rate configured?` | if | Evaluates whether live gold API integration is enabled | `check the stock rows` | `Gold rate - today's price`, `use the demo gold rate` | ### 2. Price every chain at today's rate<br><br>`Gold rate - today's price` calls goldapi.io; without a key, `use the demo gold rate` supplies a rate labelled as not a market price everywhere it appears.<br><br>`price every chain` applies one formula: weight x purity x 24k rate, plus the making charge, plus margin.<br><br>`rate looks right?` refuses a rate that moved more than `MAX_RATE_MOVE_PCT` since the last list. A stale feed or a wrong currency is far likelier than the market, and a wrong price sent to every customer is the expensive failure. |
| `Gold rate - today's price` | httpRequest | Fetches today's 24k gold spot rate via external API | `gold rate configured?` | `read the gold rate`, `STOP: gold rate unavailable` | ### 2. Price every chain at today's rate<br><br>`Gold rate - today's price` calls goldapi.io; without a key, `use the demo gold rate` supplies a rate labelled as not a market price everywhere it appears.<br><br>`price every chain` applies one formula: weight x purity x 24k rate, plus the making charge, plus margin.<br><br>`rate looks right?` refuses a rate that moved more than `MAX_RATE_MOVE_PCT` since the last list. A stale feed or a wrong currency is far likelier than the market, and a wrong price sent to every customer is the expensive failure. |
| `read the gold rate` | code | Normalizes gold API spot rate responses | `Gold rate - today's price` | `price every chain` | ### 2. Price every chain at today's rate<br><br>`Gold rate - today's price` calls goldapi.io; without a key, `use the demo gold rate` supplies a rate labelled as not a market price everywhere it appears.<br><br>`price every chain` applies one formula: weight x purity x 24k rate, plus the making charge, plus margin.<br><br>`rate looks right?` refuses a rate that moved more than `MAX_RATE_MOVE_PCT` since the last list. A stale feed or a wrong currency is far likelier than the market, and a wrong price sent to every customer is the expensive failure. |
| `STOP: gold rate unavailable` | code | Halts execution when gold rate API fails | `Gold rate - today's price` | None | ### 2. Price every chain at today's rate<br><br>`Gold rate - today's price` calls goldapi.io; without a key, `use the demo gold rate` supplies a rate labelled as not a market price everywhere it appears.<br><br>`price every chain` applies one formula: weight x purity x 24k rate, plus the making charge, plus margin.<br><br>`rate looks right?` refuses a rate that moved more than `MAX_RATE_MOVE_PCT` since the last list. A stale feed or a wrong currency is far likelier than the market, and a wrong price sent to every customer is the expensive failure. |
| `use the demo gold rate` | code | Provides fallback mock gold spot rate | `gold rate configured?` | `price every chain` | ### 2. Price every chain at today's rate<br><br>`Gold rate - today's price` calls goldapi.io; without a key, `use the demo gold rate` supplies a rate labelled as not a market price everywhere it appears.<br><br>`price every chain` applies one formula: weight x purity x 24k rate, plus the making charge, plus margin.<br><br>`rate looks right?` refuses a rate that moved more than `MAX_RATE_MOVE_PCT` since the last list. A stale feed or a wrong currency is far likelier than the market, and a wrong price sent to every customer is the expensive failure. |
| `price every chain` | code | Calculates item prices and evaluates inventory status changes | `read the gold rate`, `use the demo gold rate` | `rate looks right?` | ### 2. Price every chain at today's rate<br><br>`Gold rate - today's price` calls goldapi.io; without a key, `use the demo gold rate` supplies a rate labelled as not a market price everywhere it appears.<br><br>`price every chain` applies one formula: weight x purity x 24k rate, plus the making charge, plus margin.<br><br>`rate looks right?` refuses a rate that moved more than `MAX_RATE_MOVE_PCT` since the last list. A stale feed or a wrong currency is far likelier than the market, and a wrong price sent to every customer is the expensive failure. |
| `rate looks right?` | if | Validates gold rate fluctuation tolerance thresholds | `price every chain` | `build the catalogue`, `STOP: rate looks wrong` | ### 2. Price every chain at today's rate<br><br>`Gold rate - today's price` calls goldapi.io; without a key, `use the demo gold rate` supplies a rate labelled as not a market price everywhere it appears.<br><br>`price every chain` applies one formula: weight x purity x 24k rate, plus the making charge, plus margin.<br><br>`rate looks right?` refuses a rate that moved more than `MAX_RATE_MOVE_PCT` since the last list. A stale feed or a wrong currency is far likelier than the market, and a wrong price sent to every customer is the expensive failure. |
| `STOP: rate looks wrong` | code | Halts execution on excessive gold rate fluctuations | `rate looks right?` | None | ### 2. Price every chain at today's rate<br><br>`Gold rate - today's price` calls goldapi.io; without a key, `use the demo gold rate` supplies a rate labelled as not a market price everywhere it appears.<br><br>`price every chain` applies one formula: weight x purity x 24k rate, plus the making charge, plus margin.<br><br>`rate looks right?` refuses a rate that moved more than `MAX_RATE_MOVE_PCT` since the last list. A stale feed or a wrong currency is far likelier than the market, and a wrong price sent to every customer is the expensive failure. |
| `build the catalogue` | code | Generates customer catalogues, CSV outputs, and delta announcements | `rate looks right?` | `customer list configured?` | ### 3. Four views of the same priced rows<br><br>`build the catalogue` makes the customer catalogue, the internal stock list, a CSV and the announcement from one set of rows, so they cannot disagree. The announcement says only what changed: new in, back in stock, last few, sold out.<br><br>`plan the broadcast` skips opted-out, unusable and duplicate numbers with a reason, and enforces `BROADCAST_CAP` in code. |
| `customer list configured?` | if | Evaluates whether Google Sheets customer list is enabled | `build the catalogue` | `[cred] Sheets - read the customers`, `use the demo customers` | ### 3. Four views of the same priced rows<br><br>`build the catalogue` makes the customer catalogue, the internal stock list, a CSV and the announcement from one set of rows, so they cannot disagree. The announcement says only what changed: new in, back in stock, last few, sold out.<br><br>`plan the broadcast` skips opted-out, unusable and duplicate numbers with a reason, and enforces `BROADCAST_CAP` in code. |
| `[cred] Sheets - read the customers` | googleSheets | Reads customer records from Google Sheets | `customer list configured?` | `collect the customers` | ### 3. Four views of the same priced rows<br><br>`build the catalogue` makes the customer catalogue, the internal stock list, a CSV and the announcement from one set of rows, so they cannot disagree. The announcement says only what changed: new in, back in stock, last few, sold out.<br><br>`plan the broadcast` skips opted-out, unusable and duplicate numbers with a reason, and enforces `BROADCAST_CAP` in code. |
| `collect the customers` | code | Aggregates customer rows into a standardized array | `[cred] Sheets - read the customers` | `plan the broadcast` | ### 3. Four views of the same priced rows<br><br>`build the catalogue` makes the customer catalogue, the internal stock list, a CSV and the announcement from one set of rows, so they cannot disagree. The announcement says only what changed: new in, back in stock, last few, sold out.<br><br>`plan the broadcast` skips opted-out, unusable and duplicate numbers with a reason, and enforces `BROADCAST_CAP` in code. |
| `use the demo customers` | code | Provides fallback mock customer list records | `customer list configured?` | `plan the broadcast` | ### 3. Four views of the same priced rows<br><br>`build the catalogue` makes the customer catalogue, the internal stock list, a CSV and the announcement from one set of rows, so they cannot disagree. The announcement says only what changed: new in, back in stock, last few, sold out.<br><br>`plan the broadcast` skips opted-out, unusable and duplicate numbers with a reason, and enforces `BROADCAST_CAP` in code. |
| `plan the broadcast` | code | Filters recipient lists and prepares owner approval payloads | `collect the customers`, `use the demo customers` | `sending on?` | ### 3. Four views of the same priced rows<br><br>`build the catalogue` makes the customer catalogue, the internal stock list, a CSV and the announcement from one set of rows, so they cannot disagree. The announcement says only what changed: new in, back in stock, last few, sold out.<br><br>`plan the broadcast` skips opted-out, unusable and duplicate numbers with a reason, and enforces `BROADCAST_CAP` in code. |
| `sending on?` | if | Evaluates whether Twilio send permissions are enabled | `plan the broadcast` | `[cred] Twilio - ask the owner to approve`, `STOP: preview - draft shown` | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |
| `STOP: preview - draft shown` | code | Outputs preview drafts when live sending is disabled | `sending on?` | None | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |
| `[cred] Twilio - ask the owner to approve` | twilio | Sends approval request and resume link to owner via WhatsApp | `sending on?` | `wait for the owner's approval` | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |
| `wait for the owner's approval` | wait | Pauses execution awaiting webhook approval callback | `[cred] Twilio - ask the owner to approve` | `approved?` | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |
| `approved?` | if | Evaluates approval webhook response query parameters | `wait for the owner's approval` | `one message per customer`, `STOP: not approved`, `writing on?` | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |
| `STOP: not approved` | code | Halts execution when approval times out or is denied | `approved?` | None | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |
| `one message per customer` | code | Splits planned broadcasts into individual recipient payloads | `approved?` | `[cred] Twilio - send the announcement` | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |
| `[cred] Twilio - send the announcement` | twilio | Broadcasts approved announcement to customers via WhatsApp | `one message per customer` | `STOP: announcement sent` | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |
| `STOP: announcement sent` | code | Concludes successful broadcast runs | `[cred] Twilio - send the announcement` | None | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |
| `writing on?` | if | Evaluates whether live Google Sheets writing is permitted | `approved?` | `rows for the stock sheet`, `STOP: preview - prices not written` | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |
| `rows for the stock sheet` | code | Formats updated pricing data mapped by SKU for persistence | `writing on?` | `[cred] Sheets - write today's prices` | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |
| `[cred] Sheets - write today's prices` | googleSheets | Writes updated prices, rates, quantities, and dates to Google Sheets | `rows for the stock sheet` | `STOP: prices written` | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |
| `STOP: prices written` | code | Concludes successful sheet writing execution runs | `[cred] Sheets - write today's prices` | None | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |
| `STOP: preview - prices not written` | code | Outputs simulated sheet write updates in preview or test modes | `writing on?` | None | ### 4. The owner approves, then it sends and writes back<br><br>`[cred] Twilio - ask the owner to approve` sends the full message with an approve link, and `wait for the owner's approval` holds the run. Nothing reaches a customer until the owner opens it, and if `APPROVAL_WAIT_HOURS` passes first nothing is sent: a morning price is wrong by the evening.<br><br>After approval the announcement goes out, and `[cred] Sheets - write today's prices` records price, rate and quantity. That write is how tomorrow's run knows what is new and what sold out. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n without importing the JSON, follow these exact step-by-step instructions:

#### Step 1: Create Triggers & Configuration
1. Add a **Schedule Trigger** node (`Every weekday, 08:00`), configure the interval rule to Cron expression `0 8 * * 1-5`.
2. Add a **Manual Trigger** node (`Run it now`).
3. Add a **Code** node (`config`). Paste the complete configuration JavaScript object defining business name, margins, purity dictionaries, style definitions, API keys, and demo data arrays. Connect both triggers to this node.

#### Step 2: Build Stock Ingestion & Validation Branch
1. Add an **If** node (`stock sheet configured?`). Set condition to evaluate boolean `{{ $('config').first().json.sheet_enabled }}` as true. Connect `config` output to this node.
2. Add a **Google Sheets** node (`[cred] Sheets - read the stock`). Set operation to read rows, binding document ID and sheet name to configuration expressions (`{{ $('config').first().json.sheet_id }}` and `{{ $('config').first().json.stock_tab }}` respectively). Connect to the *True* output of `stock sheet configured?`.
3. Add a **Code** node (`collect the stock`). Collects read sheet rows into an array. Connect `[cred] Sheets - read the stock` here.
4. Add a **Code** node (`use the demo stock`). Returns `cfg.demo_stock`. Connect to the *False* output of `stock sheet configured?`.
5. Add a **Code** node (`check the stock rows`). Validates inventory row attributes (weight, karat, style, qty, SKU). Connect outputs of both `collect the stock` and `use the demo stock` into this node.

#### Step 3: Implement Gold Rate Acquisition & Pricing Branch
1. Add an **If** node (`gold rate configured?`). Set condition to evaluate `{{ $('config').first().json.gold_api_enabled }}` as true. Connect from `check the stock rows`.
2. Add an **HTTP Request** node (`Gold rate - today's price`). Set method to GET, URL to `{{ $('config').first().json.gold_api_base + '/XAU/' + $('config').first().json.currency }}`, add headers `x-access-token` and `Accept`. Configure error handling to continue on error output. Connect to the *True* output of `gold rate configured?`.
3. Add a **Code** node (`read the gold rate`) to normalize API responses. Connect to HTTP request success output.
4. Add a **Code** node (`STOP: gold rate unavailable`) to handle HTTP request failures. Connect to HTTP request error output.
5. Add a **Code** node (`use the demo gold rate`) returning `cfg.demo_gold_rate`. Connect to the *False* output of `gold rate configured?`.
6. Add a **Code** node (`price every chain`). Applies pricing math (metal + making charge + margin + rounding) and computes change states. Connect outputs from `read the gold rate` and `use the demo gold rate`.
7. Add an **If** node (`rate looks right?`). Set condition to check `{{ $json.rate_ok }}` equals true. Connect from `price every chain`.
8. Add a **Code** node (`STOP: rate looks wrong`) connected to the *False* output of `rate looks right?`.

#### Step 4: Construct Catalogue & Broadcast Planning
1. Add a **Code** node (`build the catalogue`). Generates customer catalogue, CSV export string, internal stock list, and delta announcement text. Connect to the *True* output of `rate looks right?`.
2. Add an **If** node (`customer list configured?`). Evaluate `{{ $('config').first().json.sheet_enabled }}` equals true. Connect from `build the catalogue`.
3. Add a **Google Sheets** node (`[cred] Sheets - read the customers`). Read rows binding document ID and sheet name to config variables. Connect to the *True* output of `customer list configured?`.
4. Add a **Code** node (`collect the customers`) connected to `[cred] Sheets - read the customers`.
5. Add a **Code** node (`use the demo customers`) connected to the *False* output of `customer list configured?`.
6. Add a **Code** node (`plan the broadcast`). Filters recipients, enforces opt-outs, uniqueness, and `BROADCAST_CAP`, and constructs the owner approval message. Connect outputs from `collect the customers` and `use the demo customers`.

#### Step 5: Setup Authorization, Dispatch & Persistence
1. Add an **If** node (`sending on?`). Evaluate `{{ $('config').first().json.send_enabled }}` equals true. Connect from `plan the broadcast`.
2. Add a **Code** node (`STOP: preview - draft shown`) connected to the *False* output of `sending on?`.
3. Add a **Twilio** node (`[cred] Twilio - ask the owner to approve`). Configure WhatsApp messaging; bind `to` to `{{ $json.owner_to }}`, `from` to `{{ $('config').first().json.twilio_whatsapp_from }}`, and include resume URL in message body: `{{ $execution.resumeUrl + '?approve=yes' }}`. Connect to the *True* output of `sending on?`.
4. Add a **Wait** node (`wait for the owner's approval`). Set resume type to Webhook, limit wait time to `{{ $('config').first().json.approval_wait_hours }}` hours. Connect from the Twilio approval node.
5. Add an **If** node (`approved?`). Evaluate query condition `{{ !!($json.query && $json.query.approve === 'yes') }}` equals true. Connect from `wait for the owner's approval`.
6. Add a **Code** node (`STOP: not approved`) connected to the *False* output of `approved?`.
7. Add a **Code** node (`one message per customer`) connected to the *True* output of `approved?`.
8. Add a **Twilio** node (`[cred] Twilio - send the announcement`). Configure WhatsApp messaging; bind `to` to `{{ $json.to }}`, `from` to `{{ $('config').first().json.twilio_whatsapp_from }}`, and message to `{{ $json.body }}`. Connect from `one message per customer`.
9. Add a **Code** node (`STOP: announcement sent`) connected from the Twilio send node.
10. Add an **If** node (`writing on?`). Evaluate `{{ $('config').first().json.write_enabled }}` equals true. Connect from the *True* output of `approved?`.
11. Add a **Code** node (`rows for the stock sheet`) connected to the *True* output of `writing on?`.
12. Add a **Google Sheets** node (`[cred] Sheets - write today's prices`). Set operation to `update`, matching columns to `sku`, cell format to `RAW`. Connect from `rows for the stock sheet`.
13. Add a **Code** node (`STOP: prices written`) connected from the Google Sheets write node.
14. Add a **Code** node (`STOP: preview - prices not written`) connected to the *False* output of `writing on?`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Jewellery Wholesaler Daily Repricing & WhatsApp Broadcast Automation Engine | Designed for jewellery wholesalers and gold traders automating daily price list distribution. |
| Companion Workflow Integration | Import the companion WhatsApp enquiries workflow to handle inbound customer replies at approved prices. |
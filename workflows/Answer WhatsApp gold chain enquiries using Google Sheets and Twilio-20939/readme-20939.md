Answer WhatsApp gold chain enquiries using Google Sheets and Twilio

https://n8nworkflows.xyz/workflows/answer-whatsapp-gold-chain-enquiries-using-google-sheets-and-twilio-20939


# Answer WhatsApp gold chain enquiries using Google Sheets and Twilio

### 1. Workflow Overview

This workflow automates the handling of incoming WhatsApp enquiries for a gold chain wholesaler. It receives customer messages via Twilio, processes stock availability and approved pricing retrieved from Google Sheets (or built-in fallback demo data), uses an optional OpenAI-compatible model to parse customer intent, and dynamically replies with matching stock, confirms opt-out requests, or escalates complex or bulk inquiries to the owner.

The logic is organized into the following functional blocks:

- **1.1 Input Reception & Configuration:** Receives inbound Twilio webhook events or manual trigger executions, loading all business logic, pricing parameters, and demo stock/messages into a centralized configuration object.
- **1.2 Stock Data Retrieval:** Evaluates whether a Google Sheet ID is provided, branching to either pull the live `Stock` tab data or fall back to built-in demo inventory.
- **1.3 Message Parsing & Intent Extraction:** Normalizes and analyzes customer messages using keyword rules, and optionally queries an OpenAI-compatible model to extract specific attributes (style, karat, length, quantity) without exposing pricing or stock generation logic to the AI.
- **1.4 Decision Engine & Reply Generation:** Evaluates parsed user requests against inventory data, determining whether to confirm an opt-out, hand off an escalation/bulk order to the owner, or formulate a stock-based catalog recommendation.
- **1.5 Dispatch & Persistence Routing:** Routes messages according to the operational mode (preview, test, or live) to either send WhatsApp replies via Twilio or display execution previews, and records customer opt-outs to Google Sheets when applicable.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Acts as the primary entry point for the workflow, capturing incoming webhooks from Twilio or manual test runs, and centralizing all operational constants, rules, and demo data.
- **Nodes Involved:** 
  - `A WhatsApp message arrives`
  - `Run the demo messages`
  - `config`
- **Node Details:**
  - **A WhatsApp message arrives**
    - *Type & Technical Role:* `n8n-nodes-base.webhook` (Webhook trigger)
    - *Configuration:* Listens on path `gold-whatsapp` via HTTP POST with response mode set to `onReceived`.
    - *Input/Output:* No inputs; outputs to the `config` node.
    - *Edge Cases/Failures:* Incorrect webhook URL configuration on Twilio side will prevent incoming message ingestion.
  - **Run the demo messages**
    - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` (Manual trigger)
    - *Configuration:* Standard manual execution trigger.
    - *Input/Output:* No inputs; outputs to the `config` node.
  - **config**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Houses all business constants, product styles (`STYLES`), purity mapping (`PURITY`), thresholds (`HANDOFF_QTY`, `LOW_STOCK_AT`), API endpoints, demo stock data, and runtime mode switches (`TEST_RUN`, `TEST_WHATSAPP`).
    - *Key Expressions/Variables:* Evaluates `MODE` (`live`, `test`, or `preview`), `send_enabled`, and `sheet_enabled`.
    - *Input/Output:* Inputs from webhooks or manual triggers; outputs to `stock sheet configured?`.

---

#### 2.2 Stock Data Retrieval
- **Overview:** Determines whether to query an external Google Sheets document for inventory data or rely on built-in demo stock sets.
- **Nodes Involved:**
  - `stock sheet configured?`
  - `[cred] Sheets - read the stock`
  - `collect the stock`
  - `use the demo stock`
- **Node Details:**
  - **stock sheet configured?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional branching)
    - *Configuration:* Evaluates whether `{{ $('config').first().json.sheet_enabled }}` evaluates to `true`.
    - *Input/Output:* Input from `config`; outputs True branch to Google Sheets read, False branch to demo stock.
  - **[cred] Sheets - read the stock**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets integration)
    - *Configuration:* Reads rows from the configured Stock tab ID. Uses always-output-data mode.
    - *Credentials:* Requires Google Sheets OAuth2/Service Account credentials.
    - *Input/Output:* Input from `stock sheet configured?`; outputs to `collect the stock`.
    - *Edge Cases/Failures:* Authentication revocation or missing Google Sheet tabs causes read failures.
  - **collect the stock**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Aggregates sheet row items into a unified array structure.
    - *Input/Output:* Input from Google Sheets node; outputs to `read the message`.
  - **use the demo stock**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Loads built-in inventory arrays from the configuration node.
    - *Input/Output:* Input from `stock sheet configured?` (False); outputs to `read the message`.

---

#### 2.3 Message Parsing & Intent Extraction
- **Overview:** Normalizes incoming WhatsApp message bodies, executes baseline keyword matching, and optionally queries an AI model to refine parameters.
- **Nodes Involved:**
  - `read the message`
  - `AI configured?`
  - `build the AI request`
  - `AI - read what they want`
  - `apply the AI reading`
  - `keyword reading only`
- **Node Details:**
  - **read the message**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Normalizes Twilio payload structures (`Body`, `From`, `ProfileName`), applies regex-based extraction for karats, lengths, and quantities, and evaluates keyword flags (`opt_out_words`, `handoff_words`).
    - *Input/Output:* Inputs from stock collection nodes; outputs to `AI configured?`.
  - **AI configured?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional branching)
    - *Configuration:* Checks if `ai_enabled` is true based on API key configuration.
    - *Input/Output:* Input from `read the message`; outputs True to AI request builder, False to keyword-only path.
  - **build the AI request**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Formats prompt payloads restricting the AI to return strict JSON containing style, karat, length, quantity, and handoff flags.
    - *Input/Output:* Input from `AI configured?` (True); outputs to `AI - read what they want`.
  - **AI - read what they want**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (HTTP API Request)
    - *Configuration:* Sends POST requests to the OpenAI-compatible endpoint with a 30-second timeout.
    - *Key Expressions/Variables:* Uses dynamic URL and Bearer token headers derived from configuration settings.
    - *Input/Output:* Input from `build the AI request`; outputs to `apply the AI reading`.
    - *Edge Cases/Failures:* API timeouts, rate limits, or invalid keys trigger `continueRegularOutput`, falling back gracefully to keyword rules.
  - **apply the AI reading**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Parses JSON responses from the AI model, validates extracted styles against allowed inventory configuration, and merges attributes over baseline keyword extractions.
    - *Input/Output:* Input from HTTP request; outputs to `decide the reply`.
  - **keyword reading only**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Passes through baseline keyword extraction results when AI is disabled or fails.
    - *Input/Output:* Input from `AI configured?` (False); outputs to `decide the reply`.

---

#### 2.4 Decision Engine & Reply Generation
- **Overview:** Evaluates structured user intent against current inventory stock levels and pricing rules to formulate responses or owner escalations.
- **Nodes Involved:**
  - `decide the reply`
- **Node Details:**
  - **decide the reply**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Implements pricing calculation logic, checks opt-out status, flags wholesale/bulk orders for human handoff, searches inventory arrays for closest matching items, and formats multi-line customer responses.
    - *Input/Output:* Inputs from `apply the AI reading` or `keyword reading only`; outputs to `address the messages` and `rows for the customers tab`.

---

#### 2.5 Dispatch & Persistence Routing
- **Overview:** Directs outbound messaging based on system execution modes (live, test, or preview) and handles opt-out data persistence in Google Sheets.
- **Nodes Involved:**
  - `address the messages`
  - `sending on?`
  - `[cred] Twilio - send on WhatsApp`
  - `STOP: replies sent`
  - `reply-preview`
  - `rows for the customers tab`
  - `writing on?`
  - `[cred] Sheets - record the opt-outs`
  - `optout-done`
  - `optout-preview`
- **Node Details:**
  - **address the messages**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Applies routing rules for `test` mode (redirecting broadcasts to a test WhatsApp number up to a set cap), `live` mode, or preview handling.
    - *Input/Output:* Input from `decide the reply`; outputs to `sending on?`.
  - **sending on?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional branching)
    - *Configuration:* Evaluates `send_enabled` configuration flag.
    - *Input/Output:* Input from `address the messages`; outputs True to Twilio send, False to preview stop node.
  - **[cred] Twilio - send on WhatsApp**
    - *Type & Technical Role:* `n8n-nodes-base.twilio` (Twilio integration)
    - *Configuration:* Sends WhatsApp messages using dynamic recipient and sender numbers.
    - *Credentials:* Requires Twilio API credentials.
    - *Input/Output:* Input from `sending on?` (True); outputs to `STOP: replies sent`.
    - *Edge Cases/Failures:* Twilio API rate limits, invalid phone number formatting, or missing credentials.
  - **STOP: replies sent**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Terminates message distribution branches with a summary outcome object.
    - *Input/Output:* Input from Twilio send node; terminal node.
  - **reply-preview**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Outputs preview objects detailing simulated message dispatches when sending is disabled.
    - *Input/Output:* Input from `sending on?` (False); terminal node.
  - **rows for the customers tab**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Filters out customer opt-out records generated during reply decision processing.
    - *Input/Output:* Input from `decide the reply`; outputs to `writing on?`.
  - **writing on?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional branching)
    - *Configuration:* Evaluates `write_enabled` to ensure sheet persistence occurs strictly in live production mode.
    - *Input/Output:* Input from opt-out row selector; outputs True to sheet update, False to preview record stop node.
  - **[cred] Sheets - record the opt-outs**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets integration)
    - *Configuration:* Updates existing rows in the Customers tab matching on the `whatsapp` column using RAW cell formatting.
    - *Credentials:* Requires Google Sheets credentials.
    - *Input/Output:* Input from `writing on?` (True); outputs to `optout-done`.
  - **optout-done**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Returns success outcome metrics for recorded opt-outs.
    - *Input/Output:* Input from Google Sheets update; terminal node.
  - **optout-preview**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Generates preview datasets for unrecorded opt-outs when safety switches prevent live sheet modifications.
    - *Input/Output:* Input from `writing on?` (False); terminal node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| A WhatsApp message arrives | webhook | Inbound Twilio webhook receiver | None | config | Answer WhatsApp gold chain enquiries from a Google Sheets price list... |
| Run the demo messages | manualTrigger | Manual execution trigger for testing | None | config | Answer WhatsApp gold chain enquiries from a Google Sheets price list... |
| config | code | Centralized configuration engine | A WhatsApp message arrives, Run the demo messages | stock sheet configured? | Answer WhatsApp gold chain enquiries from a Google Sheets price list... |
| stock sheet configured? | if | Determines whether to use sheet or demo stock | config | [cred] Sheets - read the stock, use the demo stock | |
| [cred] Sheets - read the stock | googleSheets | Reads inventory stock list from Google Sheets | stock sheet configured? | collect the stock | 1. Take the message in, with the stock beside it |
| collect the stock | code | Aggregates spreadsheet row data | [cred] Sheets - read the stock | read the message | 1. Take the message in, with the stock beside it |
| use the demo stock | code | Fallback inventory dataset provider | stock sheet configured? | read the message | 1. Take the message in, with the stock beside it |
| read the message | code | Normalizes messages & applies keyword parsing | collect the stock, use the demo stock | AI configured? | 1. Take the message in, with the stock beside it |
| AI configured? | if | Checks if AI integration is enabled | read the message | build the AI request, keyword reading only | 2. Let a model help read, never answer |
| build the AI request | code | Formats prompt payload for LLM analysis | AI configured? | AI - read what they want | 2. Let a model help read, never answer |
| AI - read what they want | httpRequest | Queries OpenAI-compatible API for intent parsing | build the AI request | apply the AI reading | 2. Let a model help read, never answer |
| apply the AI reading | code | Merges LLM extraction results with base data | AI - read what they want | decide the reply | 2. Let a model help read, never answer |
| keyword reading only | code | Retains keyword-only parsing results | AI configured? | decide the reply | 2. Let a model help read, never answer |
| decide the reply | code | Calculates pricing & formulates customer/owner responses | apply the AI reading, keyword reading only | address the messages, rows for the customers tab | 3. Answer from the sheet, or hand it to a person |
| address the messages | code | Applies environment safety routing rules | decide the reply | sending on? | 3. Answer from the sheet, or hand it to a person |
| sending on? | if | Evaluates if live message sending is permitted | address the messages | [cred] Twilio - send on WhatsApp, STOP: preview - replies shown | 3. Answer from the sheet, or hand it to a person |
| [cred] Twilio - send on WhatsApp | twilio | Sends outbound WhatsApp messages via Twilio | sending on? | STOP: replies sent | 3. Answer from the sheet, or hand it to a person |
| STOP: replies sent | code | Terminal node confirming message delivery | [cred] Twilio - send on WhatsApp | None | 3. Answer from the sheet, or hand it to a person |
| STOP: preview - replies shown | code | Displays preview of formulated messages | sending on? | None | 3. Answer from the sheet, or hand it to a person |
| rows for the customers tab | code | Filters opt-out records for sheet updates | decide the reply | writing on? | 4. Record opt-outs |
| writing on? | if | Verifies if live sheet updates are permitted | rows for the customers tab | [cred] Sheets - record the opt-outs, optout-preview | 4. Record opt-outs |
| [cred] Sheets - record the opt-outs | googleSheets | Updates customer opt-out status in Google Sheets | writing on? | optout-done | 4. Record opt-outs |
| optout-done | code | Terminal node confirming opt-out persistence | [cred] Sheets - record the opt-outs | None | 4. Record opt-outs |
| optout-preview | code | Displays preview of unrecorded opt-outs | writing on? | None | 4. Record opt-outs |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, execute the following steps in sequence:

1. **Create Triggers and Configuration:**
   - Add a **Webhook** node named `A WhatsApp message arrives`. Set the HTTP Method to `POST`, path to `gold-whatsapp`, and response mode to `On Received`.
   - Add a **Manual Trigger** node named `Run the demo messages`.
   - Add a **Code** node named `config`. Paste the JavaScript configuration engine code establishing business settings, fallback inventory arrays, styles, pricing margins, and mode toggles (`TEST_RUN`, `TEST_WHATSAPP`). Connect both triggers to this node.

2. **Set Up Stock Branching:**
   - Add an **If** node named `stock sheet configured?`. Configure condition to check if `{{ $('config').first().json.sheet_enabled }}` equals `true`. Connect `config` output to this node.
   - Add a **Google Sheets** node named `[cred] Sheets - read the stock`. Set operation to read rows using Document ID and Sheet Name expressions referencing the config node. Configure required Google Sheets credentials. Connect the True branch of `stock sheet configured?` here.
   - Add a **Code** node named `collect the stock`. Map incoming Google Sheets rows into a standardized array. Connect `[cred] Sheets - read the stock` here.
   - Add a **Code** node named `use the demo stock` with `executeOnce` enabled. Load `demo_stock` data from config. Connect the False branch of `stock sheet configured?` here.

3. **Configure Message Parsing:**
   - Add a **Code** node named `read the message`. Implement normalization logic for Twilio message payloads (`Body`, `From`, `ProfileName`), run regex extractions for karats, lengths, and quantities, and detect keyword flags. Connect both `collect the stock` and `use the demo stock` outputs to this node.
   - Add an **If** node named `AI configured?`. Set condition to check if `{{ $('config').first().json.ai_enabled }}` is true. Connect `read the message` here.
   - Add a **Code** node named `build the AI request` to format OpenAI-compatible JSON prompt payloads. Connect to the True branch of `AI configured?`.
   - Add an **HTTP Request** node named `AI - read what they want`. Set method to `POST`, URL expression pointing to `/chat/completions`, and add Bearer token authorization headers. Set error handling to `Continue Regular Output`. Connect from `build the AI request`.
   - Add a **Code** node named `apply the AI reading` to parse LLM JSON responses and merge extracted values. Connect from the HTTP request node.
   - Add a **Code** node named `keyword reading only` to pass through baseline parsing results when AI is inactive. Connect to the False branch of `AI configured?`.

4. **Build the Reply Decision Engine:**
   - Add a **Code** node named `decide the reply`. Implement logic evaluating opt-outs, owner handoffs for bulk/custom requests, pricing formulas, and inventory stock matching. Connect outputs from both `apply the AI reading` and `keyword reading only` into this node.

5. **Configure Message Dispatch Routing:**
   - Add a **Code** node named `address the messages` to handle environment safety switches and test recipient redirection caps. Connect from `decide the reply`.
   - Add an **If** node named `sending on?` evaluating `{{ $('config').first().json.send_enabled }}`. Connect from `address the messages`.
   - Add a **Twilio** node named `[cred] Twilio - send on WhatsApp`. Configure `to`, `from`, and message body expressions, enabling WhatsApp options. Set up Twilio credentials. Connect to the True branch of `sending on?`.
   - Add a **Code** node named `STOP: replies sent`. Connect from the Twilio node.
   - Add a **Code** node named `STOP: preview - replies shown`. Connect to the False branch of `sending on?`.

6. **Configure Opt-Out Persistence:**
   - Add a **Code** node named `rows for the customers tab` with `executeOnce` enabled to filter opt-out customer records from `decide the reply`.
   - Add an **If** node named `writing on?` evaluating `{{ $('config').first().json.write_enabled }}`. Connect from the opt-out filter node.
   - Add a **Google Sheets** node named `[cred] Sheets - record the opt-outs`. Set operation to `Update`, matching columns on `whatsapp` using RAW cell formatting. Connect to the True branch of `writing on?`.
   - Add a **Code** node named `optout-done`. Connect from the Google Sheets update node.
   - Add a **Code** node named `optout-preview`. Connect to the False branch of `writing on?`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Designed for jewellery wholesalers managing incoming WhatsApp product and pricing enquiries. | Target business use case |
| Companion workflow available for daily metal repricing and broadcast price list generation. | Workflow ecosystem pairing |
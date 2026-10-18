Send Hyperliquid wallet fill alerts to Telegram

https://n8nworkflows.xyz/workflows/send-hyperliquid-wallet-fill-alerts-to-telegram-20926


# Send Hyperliquid wallet fill alerts to Telegram

### 1. Workflow Overview

The **Send Hyperliquid wallet fill alerts to Telegram** workflow monitors a single public Hyperliquid wallet address every five minutes. It fetches recent trading activity using the public Hyperliquid Info API, processes and deduplicates trade fills, and delivers clean, formatted summaries directly to a specified Telegram chat.

Target use cases include tracking personal or public DeFi trading wallets, receiving automated perpetual position updates, and auditing fills without needing to keep a trading dashboard open.

The workflow logic is divided into four functional blocks:
- **1.1 Input Reception & Configuration:** Triggers the workflow either on a 5-minute cron schedule or via manual test preview, loading target configuration parameters.
- **1.2 Data Preparation & API Retrieval:** Validates input parameters and stored checkpoints, constructs an HTTP POST request, and queries the public Hyperliquid Info API (`userFillsByTime`).
- **1.3 Data Processing & Deduplication:** Validates API payloads, parses decimal trade sizes, tracks perpetual position changes (e.g., opens, closes, reverses), filters out already seen trades, and formats a secure HTML Telegram message.
- **1.4 Conditional Dispatch & State Persistence:** Evaluates whether an alert should be dispatched, sends the message via Telegram if enabled, and updates the global workflow static data checkpoint upon successful execution.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration

##### Overview
This block initiates the execution cycle, handling both automated periodic triggers and on-demand manual previews, while setting up the required wallet and chat parameters.

##### Nodes Involved
- `Preview manually` (`n8n-nodes-base.manualTrigger`)
- `Every 5 minutes` (`n8n-nodes-base.scheduleTrigger`)
- `Wallet settings` (`n8n-nodes-base.set`)

##### Node Details

- **Preview manually**
  - **Type and technical role:** Manual Trigger (`n8n-nodes-base.manualTrigger`). Acts as an entry point for testing without saving state or sending external alerts.
  - **Configuration choices:** Default settings.
  - **Input and output connections:** Outputs to `Wallet settings`.
  - **Edge cases / Potential failures:** None.

- **Every 5 minutes**
  - **Type and technical role:** Schedule Trigger (`n8n-nodes-base.scheduleTrigger`). Triggers automated production runs every 5 minutes.
  - **Configuration choices:** Interval set to 5 minutes (`minutesInterval: 5`).
  - **Input and output connections:** Outputs to `Wallet settings`.
  - **Edge cases / Potential failures:** Concurrent executions can cause race conditions on static data storage; sequential execution configuration is mandatory.

- **Wallet settings**
  - **Type and technical role:** Set Node (`n8n-nodes-base.set`). Defines parameters for the target wallet and delivery destination.
  - **Configuration choices:** Sets three string assignments:
    - `walletAddress`: `PASTE_PUBLIC_WALLET_ADDRESS`
    - `walletLabel`: `Watched wallet`
    - `telegramChatId`: `PASTE_TELEGRAM_CHAT_ID`
  - **Input and output connections:** Inputs from both triggers; outputs to `Prepare request`.
  - **Edge cases / Potential failures:** Invalid addresses or chat IDs will be caught in the subsequent preparation node.
  - **Sticky Note Context:** Covered by the global setup note.

---

#### Block 1.2: Data Preparation & API Retrieval

##### Overview
This block validates input parameters, inspects stored state checkpoints to determine time boundaries, queries the public Hyperliquid API, and handles network or response retries.

##### Nodes Involved
- `Prepare request` (`n8n-nodes-base.code`)
- `Read Hyperliquid fills` (`n8n-nodes-base.httpRequest`)

##### Node Details

- **Prepare request**
  - **Type and technical role:** Code Node (`n8n-nodes-base.code`). Executes custom JavaScript to validate addresses, handle execution modes (manual vs. scheduled), compute temporal overlap windows, and retrieve state checkpoints.
  - **Configuration choices:** Custom JavaScript checking wallet validity (`0x...` format), validation of Telegram chat ID formats, static data integrity checks, and calculation of `startTime` and `endTime`.
  - **Key expressions / variables:** Uses `$input.first().json`, `$execution.mode`, and `$getWorkflowStaticData('global')`.
  - **Input and output connections:** Input from `Wallet settings`; output to `Read Hyperliquid fills`.
  - **Edge cases / Potential failures:** Throws explicit errors on malformed wallet addresses, invalid chat IDs, corrupted checkpoints, or gaps exceeding 24 hours.

- **Read Hyperliquid fills**
  - **Type and technical role:** HTTP Request Node (`n8n-nodes-base.httpRequest`). Queries the public Hyperliquid Info API.
  - **Configuration choices:** 
    - URL: `https://api.hyperliquid.xyz/info`
    - Method: `POST`
    - Timeout: 20,000 ms
    - Retries: Max 3 tries with 1,500 ms wait between tries.
    - Body Type: JSON stringified request payload from the previous node (`={{ JSON.stringify($json.request) }}`).
  - **Input and output connections:** Input from `Prepare request`; output to `Build alert`.
  - **Edge cases / Potential failures:** Network timeouts, API rate limits, or non-200 HTTP responses.
  - **Sticky Note Context:** Accompanied by the guide read note emphasizing that responses exceeding 500 rows or malformed responses will halt execution rather than skip history.

---

#### Block 1.3: Data Processing & Deduplication

##### Overview
This block validates raw API output, deduplicates trade fills, calculates perpetual position transitions (such as opens, closes, or reversals), filters out previously observed trades, and formats an HTML-ready message for Telegram.

##### Nodes Involved
- `Build alert` (`n8n-nodes-base.code`)

##### Node Details

- **Build alert**
  - **Type and technical role:** Code Node (`n8n-nodes-base.code`). Implements robust parsing logic to filter, sort, and format up to six recent fills.
  - **Configuration choices:** Custom JavaScript performing decimal validation, unique trade ID parsing (`tid`), unit scaling with `BigInt`, position delta analysis, HTML escaping, and message length checking.
  - **Key expressions / variables:** References `$('Prepare request').first().json` and `$input.first().json`.
  - **Input and output connections:** Input from `Read Hyperliquid fills`; output to `Send a new fill alert?`.
  - **Edge cases / Potential failures:** Throws an error if payload length is $\ge$ 500 rows (pagination gap), if trade IDs conflict, if records fall outside acceptable validation schemas, or if the generated message exceeds 4,000 characters.
  - **Sticky Note Context:** Accompanied by the guide send note highlighting that spot codes, HIP-3 prefixes, and position labels are processed here, with a link to the Market Depth Lab tracker.

---

#### Block 1.4: Conditional Dispatch & State Persistence

##### Overview
This block evaluates whether an alert needs to be sent, dispatches the message via Telegram during active scheduled runs, and safely saves checkpoint data to persistent workflow storage.

##### Nodes Involved
- `Send a new fill alert?` (`n8n-nodes-base.if`)
- `Telegram alert` (`n8n-nodes-base.telegram`)
- `Save checkpoint or preview` (`n8n-nodes-base.code`)

##### Node Details

- **Send a new fill alert?**
  - **Type and technical role:** If Node (`n8n-nodes-base.if`). Directs workflow execution based on whether new fills require notification.
  - **Configuration choices:** Evaluates condition `={{ $json.send }}` (boolean true).
  - **Input and output connections:** Input from `Build alert`; outputs to `Telegram alert` (true branch) and `Save checkpoint or preview` (false branch).
  - **Edge cases / Potential failures:** Misconfigured booleans resulting in skipped alerts or incorrect baseline behaviors.

- **Telegram alert**
  - **Type and technical role:** Telegram Node (`n8n-nodes-base.telegram`). Posts the formatted fill summary to the designated Telegram chat.
  - **Configuration choices:**
    - Resource: `message`
    - Operation: `sendMessage`
    - Chat ID: `={{ $json.chatId }}`
    - Text: `={{ $json.message }}`
    - Additional Fields: Parse Mode set to `HTML`, append attribution disabled, web page previews disabled.
  - **Input and output connections:** Input from the true branch of `Send a new fill alert?`; output to `Save checkpoint or preview`.
  - **Edge cases / Potential failures:** Telegram API authentication errors, invalid chat permissions, or markdown/HTML parsing failures.

- **Save checkpoint or preview**
  - **Type and technical role:** Code Node (`n8n-nodes-base.code`). Validates successful message delivery or dry-run preview states, updates global static data checkpoints, and prevents concurrent state updates.
  - **Configuration choices:** Custom JavaScript verifying Telegram delivery receipts (`message_id`), checking for concurrent checkpoint modifications, and writing the updated checkpoint object to `$getWorkflowStaticData('global')`.
  - **Key expressions / variables:** References `$('Build alert').first().json`, `$input.first().json`, and `$getWorkflowStaticData('global')`.
  - **Input and output connections:** Inputs from both the false branch of `Send a new fill alert?` and the output of `Telegram alert`; terminal node.
  - **Edge cases / Potential failures:** Concurrent execution collisions throwing explicit safety errors if state parameters drift.
  - **Sticky Note Context:** Accompanied by limits notes regarding static-data persistence behavior and execution constraints.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Preview manually` | `n8n-nodes-base.manualTrigger` | Manual test entry point | None | `Wallet settings` | # Send Hyperliquid wallet fill alerts to Telegram<br><br>## Who is this for?<br><br>Anyone following a public Hyperliquid trading address who wants alerts without repeatedly checking a dashboard. Use one lower-volume wallet per workflow.<br><br>## How it works<br><br>Every five minutes, the workflow reads Hyperliquid’s public Info API. The first scheduled run silently establishes a baseline. Later runs deduplicate fills and send a Telegram summary showing up to six recent trades, including perpetual position changes, prices, sizes, and available fees.<br><br>Each alert links to the same address in the [Market Depth Lab Hyperliquid Wallet Tracker](https://marketdepthlab.com/tools/hyperliquid-wallet-tracker/). Market Depth Lab created this template and companion tool.<br><br>## Requirements and setup<br><br>1. Import into n8n. Enter a public master-account or subaccount address, label, and Telegram chat ID in **Wallet settings**.<br>2. Select your Telegram bot credential in **Telegram alert**.<br>3. Run **Preview manually** and inspect the final output; manual runs never send messages or save checkpoints.<br>4. Activate when ready. Check execution history for failures.<br><br>No wallet connection, private key, or Hyperliquid trading API key is needed.<br><br>## Customization and limits<br><br>Change the address and label for another wallet. Keep executions sequential. This monitors fills, not deposits, withdrawals, or funding.<br><br>Static-data persistence is experimental. Interrupted deliveries can duplicate alerts; late fills can be missed. Responses of 500+ fills or checkpoint gaps over 24 hours stop with an error. See workflow notes for details. |
| `Every 5 minutes` | `n8n-nodes-base.scheduleTrigger` | Periodic production trigger (cron interval) | None | `Wallet settings` | # Send Hyperliquid wallet fill alerts to Telegram<br><br>## Who is this for?<br><br>Anyone following a public Hyperliquid trading address who wants alerts without repeatedly checking a dashboard. Use one lower-volume wallet per workflow.<br><br>## How it works<br><br>Every five minutes, the workflow reads Hyperliquid’s public Info API. The first scheduled run silently establishes a baseline. Later runs deduplicate fills and send a Telegram summary showing up to six recent trades, including perpetual position changes, prices, sizes, and available fees.<br><br>Each alert links to the same address in the [Market Depth Lab Hyperliquid Wallet Tracker](https://marketdepthlab.com/tools/hyperliquid-wallet-tracker/). Market Depth Lab created this template and companion tool.<br><br>## Requirements and setup<br><br>1. Import into n8n. Enter a public master-account or subaccount address, label, and Telegram chat ID in **Wallet settings**.<br>2. Select your Telegram bot credential in **Telegram alert**.<br>3. Run **Preview manually** and inspect the final output; manual runs never send messages or save checkpoints.<br>4. Activate when ready. Check execution history for failures.<br><br>No wallet connection, private key, or Hyperliquid trading API key is needed.<br><br>## Customization and limits<br><br>Change the address and label for another wallet. Keep executions sequential. This monitors fills, not deposits, withdrawals, or funding.<br><br>Static-data persistence is experimental. Interrupted deliveries can duplicate alerts; late fills can be missed. Responses of 500+ fills or checkpoint gaps over 24 hours stop with an error. See workflow notes for details. |
| `Wallet settings` | `n8n-nodes-base.set` | Define wallet address, label, and chat ID | `Preview manually`, `Every 5 minutes` | `Prepare request` | # Send Hyperliquid wallet fill alerts to Telegram<br><br>## Who is this for?<br><br>Anyone following a public Hyperliquid trading address who wants alerts without repeatedly checking a dashboard. Use one lower-volume wallet per workflow.<br><br>## How it works<br><br>Every five minutes, the workflow reads Hyperliquid’s public Info API. The first scheduled run silently establishes a baseline. Later runs deduplicate fills and send a Telegram summary showing up to six recent trades, including perpetual position changes, prices, sizes, and available fees.<br><br>Each alert links to the same address in the [Market Depth Lab Hyperliquid Wallet Tracker](https://marketdepthlab.com/tools/hyperliquid-wallet-tracker/). Market Depth Lab created this template and companion tool.<br><br>## Requirements and setup<br><br>1. Import into n8n. Enter a public master-account or subaccount address, label, and Telegram chat ID in **Wallet settings**.<br>2. Select your Telegram bot credential in **Telegram alert**.<br>3. Run **Preview manually** and inspect the final output; manual runs never send messages or save checkpoints.<br>4. Activate when ready. Check execution history for failures.<br><br>No wallet connection, private key, or Hyperliquid trading API key is needed.<br><br>## Customization and limits<br><br>Change the address and label for another wallet. Keep executions sequential. This monitors fills, not deposits, withdrawals, or funding.<br><br>Static-data persistence is experimental. Interrupted deliveries can duplicate alerts; late fills can be missed. Responses of 500+ fills or checkpoint gaps over 24 hours stop with an error. See workflow notes for details. |
| `Prepare request` | `n8n-nodes-base.code` | Validate inputs, load checkpoints, and format request payload | `Wallet settings` | `Read Hyperliquid fills` | # Send Hyperliquid wallet fill alerts to Telegram<br><br>## Who is this for?<br><br>Anyone following a public Hyperliquid trading address who wants alerts without repeatedly checking a dashboard. Use one lower-volume wallet per workflow.<br><br>## How it works<br><br>Every five minutes, the workflow reads Hyperliquid’s public Info API. The first scheduled run silently establishes a baseline. Later runs deduplicate fills and send a Telegram summary showing up to six recent trades, including perpetual position changes, prices, sizes, and available fees.<br><br>Each alert links to the same address in the [Market Depth Lab Hyperliquid Wallet Tracker](https://marketdepthlab.com/tools/hyperliquid-wallet-tracker/). Market Depth Lab created this template and companion tool.<br><br>## Requirements and setup<br><br>1. Import into n8n. Enter a public master-account or subaccount address, label, and Telegram chat ID in **Wallet settings**.<br>2. Select your Telegram bot credential in **Telegram alert**.<br>3. Run **Preview manually** and inspect the final output; manual runs never send messages or save checkpoints.<br>4. Activate when ready. Check execution history for failures.<br><br>No wallet connection, private key, or Hyperliquid trading API key is needed.<br><br>## Customization and limits<br><br>Change the address and label for another wallet. Keep executions sequential. This monitors fills, not deposits, withdrawals, or funding.<br><br>Static-data persistence is experimental. Interrupted deliveries can duplicate alerts; late fills can be missed. Responses of 500+ fills or checkpoint gaps over 24 hours stop with an error. See workflow notes for details. |
| `Read Hyperliquid fills` | `n8n-nodes-base.httpRequest` | Query Hyperliquid Info API for user fills | `Prepare request` | `Build alert` | ## 2. Read public fills<br>Every five minutes, query Hyperliquid with a two-minute overlap and a ten-second delay. First scheduled success creates a silent baseline.<br><br>500+ rows, malformed responses or a gap over 24 hours stop the run instead of silently skipping history. Intended for lower-volume wallets, not high-frequency accounts. |
| `Build alert` | `n8n-nodes-base.code` | Parse, deduplicate, calculate positions, and format Telegram message | `Read Hyperliquid fills` | `Send a new fill alert?` | ## 3. Alert, then checkpoint<br>One message per batch, latest six fills shown. Spot codes and returned HIP-3 prefixes are preserved. Position-change labels apply to perpetual fills.<br><br>Link opens the selected address in Market Depth Lab. No trading action is performed. |
| `Send a new fill alert?` | `n8n-nodes-base.if` | Determine if a new fill notification should be sent | `Build alert` | `Telegram alert`, `Save checkpoint or preview` | ## Before publishing this workflow in n8n<br>Use one sequential execution per workflow; timeout is 120s, schedule is 5m. Static data persists only after successful **published/active scheduled** runs. It is experimental and not suitable for high-frequency/overlapping runs.<br><br>Check Executions for API errors; attach your own error workflow if needed. Delivery is at-least-once: an interrupted send/checkpoint may duplicate a message. Late records beyond the overlap and upstream retention limits may be missed.<br><br>This monitors returned fills for one address, not deposits, withdrawals, funding, all subaccounts or a complete cross-DEX portfolio. Clone the workflow for another address; check limits before scaling. |
| `Telegram alert` | `n8n-nodes-base.telegram` | Send HTML-formatted message to Telegram chat | `Send a new fill alert?` | `Save checkpoint or preview` | ## Before publishing this workflow in n8n<br>Use one sequential execution per workflow; timeout is 120s, schedule is 5m. Static data persists only after successful **published/active scheduled** runs. It is experimental and not suitable for high-frequency/overlapping runs.<br><br>Check Executions for API errors; attach your own error workflow if needed. Delivery is at-least-once: an interrupted send/checkpoint may duplicate a message. Late records beyond the overlap and upstream retention limits may be missed.<br><br>This monitors returned fills for one address, not deposits, withdrawals, funding, all subaccounts or a complete cross-DEX portfolio. Clone the workflow for another address; check limits before scaling. |
| `Save checkpoint or preview` | `n8n-nodes-base.code` | Verify delivery and persist execution checkpoint state | `Send a new fill alert?`, `Telegram alert` | None | ## Before publishing this workflow in n8n<br>Use one sequential execution per workflow; timeout is 120s, schedule is 5m. Static data persists only after successful **published/active scheduled** runs. It is experimental and not suitable for high-frequency/overlapping runs.<br><br>Check Executions for API errors; attach your own error workflow if needed. Delivery is at-least-once: an interrupted send/checkpoint may duplicate a message. Late records beyond the overlap and upstream retention limits may be missed.<br><br>This monitors returned fills for one address, not deposits, withdrawals, funding, all subaccounts or a complete cross-DEX portfolio. Clone the workflow for another address; check limits before scaling. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Triggers and Configuration Nodes:**
   - **Node 1:** Create a **Manual Trigger** node named `Preview manually`. Leave parameters at default (v1).
   - **Node 2:** Create a **Schedule Trigger** node named `Every 5 minutes` (v1.2). Set rule interval to 5 minutes (`minutesInterval: 5`).
   - **Node 3:** Create a **Set** node named `Wallet settings` (v3.4). Add string assignments:
     - `walletAddress` = `PASTE_PUBLIC_WALLET_ADDRESS`
     - `walletLabel` = `Watched wallet`
     - `telegramChatId` = `PASTE_TELEGRAM_CHAT_ID`
   - *Connections:* Connect both `Preview manually` and `Every 5 minutes` to `Wallet settings`.

2. **Set Up API Request Preparation:**
   - **Node 4:** Create a **Code** node named `Prepare request` (v2). Paste the JavaScript logic that validates wallet addresses, handles manual/scheduled execution contexts, checks static data state integrity, and constructs the API request JSON object for `userFillsByTime`.
   - *Connection:* Connect `Wallet settings` to `Prepare request`.

3. **Configure API Retrieval:**
   - **Node 5:** Create an **HTTP Request** node named `Read Hyperliquid fills` (v4.2).
     - Set method to `POST`.
     - Set URL to `https://api.hyperliquid.xyz/info`.
     - Configure Options: Timeout `20000` ms, Response Format `JSON` with full response enabled.
     - Configure Body: Specify body type as `JSON` and insert expression: `={{ JSON.stringify($json.request) }}`.
     - Configure Retry: Enable retry on fail with max 3 tries and 1,500 ms wait between tries.
   - *Connection:* Connect `Prepare request` to `Read Hyperliquid fills`.

4. **Implement Data Processing & Formatting:**
   - **Node 6:** Create a **Code** node named `Build alert` (v2). Paste the JavaScript code that validates API responses, guards against pagination limits ($\ge$ 500 rows), validates decimals and trade IDs, deduplicates entries, calculates perpetual position changes, formats HTML strings, and builds the workflow output object containing checkpoint data.
   - *Connection:* Connect `Read Hyperliquid fills` to `Build alert`.

5. **Configure Conditional Routing and Telegram Delivery:**
   - **Node 7:** Create an **If** node named `Send a new fill alert?` (v2.2). Set condition: left value `={{ $json.send }}` equals boolean `true`.
   - *Connection:* Connect `Build alert` to `Send a new fill alert?`.
   - **Node 8:** Create a **Telegram** node named `Telegram alert` (v1.2).
     - Resource: `message`, Operation: `sendMessage`.
     - Chat ID: `={{ $json.chatId }}`
     - Text: `={{ $json.message }}`
     - Additional Fields: Parse Mode = `HTML`, Append Attribution = `false`, Disable Web Page Preview = `true`.
     - Credentials: Connect a valid Telegram Bot API credential.
   - *Connection:* Connect the `true` output branch of `Send a new fill alert?` to `Telegram alert`.

6. **Implement State Persistence:**
   - **Node 9:** Create a **Code** node named `Save checkpoint or preview` (v2). Paste the JavaScript code that validates Telegram message receipt IDs, checks for concurrent checkpoint modifications, and updates global static data (`$getWorkflowStaticData('global').hlWalletMonitor`).
   - *Connections:* Connect the `false` output branch of `Send a new fill alert?` and the output of `Telegram alert` to `Save checkpoint or preview`.

7. **Final Settings & Verification:**
   - Set workflow timezone to `UTC`.
   - Configure execution timeout to `120` seconds.
   - Enable `saveManualExecutions`, `saveDataErrorExecution` (`all`), and `saveDataSuccessExecution` (`all`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Created by Market Depth Lab | [Market Depth Lab Hyperliquid Wallet Tracker](https://marketdepthlab.com/tools/hyperliquid-wallet-tracker/) |
| Hyperliquid API Reference Documentation | [Hyperliquid Info API Documentation](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint) |
| n8n Built-in Code Cookbook Reference | [n8n Static-Data Documentation](https://docs.n8n.io/code/cookbook/builtin/get-workflow-static-data/) |
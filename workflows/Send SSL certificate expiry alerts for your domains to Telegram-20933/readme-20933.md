Send SSL certificate expiry alerts for your domains to Telegram

https://n8nworkflows.xyz/workflows/send-ssl-certificate-expiry-alerts-for-your-domains-to-telegram-20933


# Send SSL certificate expiry alerts for your domains to Telegram

### 1. Workflow Overview

This workflow automates daily checks of SSL certificate expiration statuses for a user-defined list of web domains. Its primary purpose is to warn administrators before certificates expire, helping to prevent unexpected site outages, security warnings for visitors, and manual oversight failures. It requires no paid API keys or spreadsheets, making it ideal for freelancers, agencies, and small IT teams.

The workflow logic is grouped into two main functional blocks:
- **1.1 Initialization and Configuration:** Triggers daily on a fixed schedule, defines runtime parameters (domain list, chat ID, thresholds, and reporting options), and sanitizes and validates the targeted domains.
- **1.2 Certificate Inspection, Evaluation, and Dispatch:** Queries a public domain certificate API for each target domain, evaluates the results against predefined warning thresholds and error conditions, tracks persistent state to avoid spamming duplicate alerts, consolidates the findings, and dispatches a prioritized HTML-formatted report via Telegram.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Initialization and Configuration

- **Overview:** This block establishes the execution schedule, injects operational parameters, and processes the raw input string of websites into a sanitized, deduplicated array ready for network querying.
- **Nodes Involved:** 
  - `Every day at 8:00`
  - `Settings`
  - `List domains`

- **Node Details:**
  - **Every day at 8:00**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node). Initiates the workflow automatically every day at 08:00 AM.
    - *Configuration Choices:* Trigger interval set to 1 day at hour 8.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Output connects to the `Settings` node.
    - *Version-Specific Requirements:* Type version 1.2.
    - *Edge Cases or Potential Failure Types:* None.
  - **Settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Parameter Definition Node). Sets operational variables for downstream logic.
    - *Configuration Choices:* Configures static parameters including the domain list, Telegram chat ID, alert threshold days, and a full-report toggle.
    - *Key Expressions or Variables:* 
      - `domains`: `"example.com, my-shop.com"`
      - `telegram_chat_id`: `"YOUR_CHAT_ID"`
      - `alert_days_before`: `"30, 14, 7, 3, 1"`
      - `report_all`: `false`
    - *Input and Output Connections:* Input from `Every day at 8:00`; output connects to `List domains`.
    - *Version-Specific Requirements:* Type version 3.4.
    - *Edge Cases or Potential Failure Types:* Invalid parameter types or unconfigured chat IDs will cause downstream issues.
  - **List domains**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation Node). Parses, cleans, validates, and deduplicates the raw domain string.
    - *Configuration Choices:* Executes custom JavaScript to strip protocols, paths, query strings, and ports, validating names against standard domain regex patterns. Limits output to a maximum of 50 domains.
    - *Key Expressions or Variables:* Reads `$input.first().json.domains`.
    - *Input and Output Connections:* Input from `Settings`; output connects to `Read certificate`.
    - *Version-Specific Requirements:* Type version 2.
    - *Edge Cases or Potential Failure Types:* Throws a workflow error if no valid websites remain after parsing. Ignores malformed domain entries while logging them.

---

#### Block 1.2: Certificate Inspection, Evaluation, and Dispatch

- **Overview:** This block executes individual HTTP requests to inspect the SSL certificate of each domain, compares expiration dates and domain coverage against historical state data, compiles an ordered notification string, and sends it to a Telegram channel.
- **Nodes Involved:**
  - `Read certificate`
  - `Evaluate certificates`
  - `Send Telegram alert`

- **Node Details:**
  - **Read certificate**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Request Node). Fetches SSL/TLS certificate details from a public API endpoint for each parsed domain.
    - *Configuration Choices:* Uses a GET request against `https://networkcalc.com/api/security/certificate/{{ $json.domain }}`. Configured with a 30-second timeout, batch size of 1 with a 400ms interval, 3 maximum retries with 2-second waits, and configured to never error out (`neverError: true`) so that unresolvable domains can be handled gracefully in code.
    - *Key Expressions or Variables:* Uses `{{ $json.domain }}` in the URL path.
    - *Input and Output Connections:* Input from `List domains`; output connects to `Evaluate certificates`.
    - *Version-Specific Requirements:* Type version 4.2.
    - *Edge Cases or Potential Failure Types:* Network drops, rate limits, or API downtime. Handled via built-in retry configurations and `continueRegularOutput` error handling.
  - **Evaluate certificates**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Logic and Formatting Node). Evaluates certificate validity, expiration thresholds, wildcard/exact name matching, and records historical alert states to prevent duplicate notifications.
    - *Configuration Choices:* Custom JavaScript processing input items alongside node-to-node data lookups (`$('Settings')`, `$('List domains')`) and static global workflow data (`$getWorkflowStaticData('global')`). Compiles a prioritized HTML-formatted message string.
    - *Key Expressions or Variables:* Accesses static workflow storage and configuration options.
    - *Input and Output Connections:* Input from `Read certificate`; output connects to `Send Telegram alert`.
    - *Version-Specific Requirements:* Type version 2.
    - *Edge Cases or Potential Failure Types:* Malformed JSON responses from the external API or invalid threshold date formats.
  - **Send Telegram alert**
    - *Type and Technical Role:* `n8n-nodes-base.telegram` (Messaging Integration Node). Sends the consolidated alert payload to the specified Telegram chat.
    - *Configuration Choices:* Sends a text message with HTML parse mode enabled, web page previews disabled, and attribution disabled.
    - *Key Expressions or Variables:* 
      - `text`: `={{ $json.message }}`
      - `chatId`: `={{ $('Settings').first().json.telegram_chat_id }}`
    - *Input and Output Connections:* Input from `Evaluate certificates`; execution ends here.
    - *Version-Specific Requirements:* Type version 1.2.
    - *Edge Cases or Potential Failure Types:* Invalid Telegram bot token, incorrect chat ID, or exceeding Telegram's character limits (mitigated by truncation logic in the evaluation script). Requires configured Telegram credentials.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Template overview | `n8n-nodes-base.stickyNote` | Documentation and setup instructions | None | None | ## Send SSL certificate expiry alerts for your websites to Telegram<br><br>An expired SSL certificate shows visitors a scary browser warning and can take a shop or a client site offline overnight. This workflow checks the certificates of your websites every day and warns you on Telegram **before** they expire, so renewal never catches you by surprise.<br><br>**Who is it for?** Freelancers, agencies, site owners and small IT teams who look after several domains. No API key, no spreadsheet and no paid service needed.<br><br>### How it works<br>1. Every day at 8:00 it reads your list of websites.<br>2. It reads the certificate of each site with a free public API.<br>3. It decides what deserves an alert: expiring soon (30, 14, 7, 3 and 1 days before), already expired, wrong name on the certificate, or a site that cannot be checked.<br>4. It sends **one** Telegram message, most urgent first. Each warning is sent once, not every day. Expired certificates are repeated daily until renewed. All fine = no message.<br><br>### Setup<br>1. Create a Telegram bot with @BotFather and add its token as a credential on **Send Telegram alert**.<br>2. Get your chat ID (for example with @userinfobot).<br>3. Open **Settings**: list your websites in `domains` (comma separated) and fill `telegram_chat_id`.<br>4. Run once with `report_all` set to true to see every site, then set it back to false and activate the workflow.<br><br>### Customization tips<br>- `alert_days_before`: the days before expiry that trigger a warning, for example `45, 30, 14, 7, 1`.<br>- Change the hour in **Every day at 8:00**.<br>- Up to 50 websites per run. Public sites only (the check is made from the internet). |
| Section 1 | `n8n-nodes-base.stickyNote` | Visual grouping for Start and settings | None | None | ## 1. Start and settings<br>Runs every day and reads your list of websites. |
| Section 2 | `n8n-nodes-base.stickyNote` | Visual grouping for Check and alert | None | None | ## 2. Check and alert<br>Reads each certificate, decides what deserves an alert and sends one message. |
| Every day at 8:00 | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow daily | None | Settings | |
| Settings | `n8n-nodes-base.set` | Configures domains, chat ID, and rules | Every day at 8:00 | List domains | |
| List domains | `n8n-nodes-base.code` | Parses and validates domain list | Settings | Read certificate | |
| Read certificate | `n8n-nodes-base.httpRequest` | Fetches SSL certificates via public API | List domains | Evaluate certificates | |
| Evaluate certificates | `n8n-nodes-base.code` | Analyzes expiry, errors, and creates message | Read certificate | Send Telegram alert | |
| Send Telegram alert | `n8n-nodes-base.telegram` | Delivers consolidated alert to Telegram | Evaluate certificates | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Set the interval rule to trigger every `1` days at hour `8`.
   - Name the node: `Every day at 8:00`.

2. **Create the Settings Node:**
   - Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`).
   - Connect the output of `Every day at 8:00` to this node.
   - Configure the following string/boolean assignments:
     - `domains` (String): `example.com, my-shop.com`
     - `telegram_chat_id` (String): `YOUR_CHAT_ID`
     - `alert_days_before` (String): `30, 14, 7, 3, 1`
     - `report_all` (Boolean): `false`
   - Name the node: `Settings`.

3. **Create the Domain Parsing Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Connect the output of `Settings` to this node.
   - Set the mode to run JavaScript and paste the domain sanitization, regex validation, and deduplication logic (which caps input arrays at 50 valid items).
   - Name the node: `List domains`.

4. **Create the HTTP Request Node:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Connect the output of `List domains` to this node.
   - Configure parameters:
     - Method: `GET`
     - URL: `https://networkcalc.com/api/security/certificate/{{ $json.domain }}`
     - Options > Timeout: `30000` ms
     - Options > Batching: Batch Size `1`, Batch Interval `400` ms
     - Options > Response > Never Error: Enabled (`true`)
     - Retry On Fail: Enabled (`true`)
     - Max Tries: `3`
     - Wait Between Tries: `2000` ms
   - Name the node: `Read certificate`.

5. **Create the Evaluation Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Connect the output of `Read certificate` to this node.
   - Insert the evaluation script that reads workflow static data (`global`), processes expiration thresholds, compares domain wildcard/exact naming coverage, and builds a consolidated HTML message structure.
   - Name the node: `Evaluate certificates`.

6. **Create the Telegram Integration Node:**
   - Add a **Telegram** node (`n8n-nodes-base.telegram`).
   - Connect the output of `Evaluate certificates` to this node.
   - Configure credentials: Set up a valid Telegram Bot API token credential.
   - Configure parameters:
     - Resource: `Message`
     - Operation: `Send`
     - Chat ID: `={{ $('Settings').first().json.telegram_chat_id }}`
     - Text: `={{ $json.message }}`
     - Additional Fields > Parse Mode: `HTML`
     - Additional Fields > Disable Web Page Preview: Enabled (`true`)
     - Additional Fields > Append Attribution: Disabled (`false`)
   - Name the node: `Send Telegram alert`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Created using n8n for automated SSL tracking without paid services. | Workflow Metadata |
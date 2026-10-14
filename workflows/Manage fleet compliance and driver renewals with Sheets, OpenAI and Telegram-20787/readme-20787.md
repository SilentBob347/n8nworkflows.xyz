Manage fleet compliance and driver renewals with Sheets, OpenAI and Telegram

https://n8nworkflows.xyz/workflows/manage-fleet-compliance-and-driver-renewals-with-sheets--openai-and-telegram-20787


# Manage fleet compliance and driver renewals with Sheets, OpenAI and Telegram

### 1. Workflow Overview

This workflow automates fleet compliance management for transport operations. Its core purpose is to monitor vehicle document expirations (Registration Certificate, Insurance, PUC, Fitness, and Permits), fetch pending e-challans via an external API, calculate risk scores, restrict dispatch for non-compliant vehicles, and manage the lifecycle of compliance renewals through multi-channel notifications (Telegram, Slack, ClickUp, and Gmail). 

The automation is divided into two primary execution loops:
- **1.1 Daily Compliance Scan & Notification Loop:** Initiated every morning via a schedule trigger, this pipeline evaluates spreadsheet records against alert thresholds, queries the e-challan API, updates dispatch statuses, generates AI-assisted driver notifications, creates task items, notifies operations personnel, and compiles fleet manager digests.
- **1.2 Driver Document Upload & Verification Loop:** Triggered asynchronously when a driver sends a document photograph via Telegram, this pipeline authenticates the sender, downloads and analyzes the image utilizing a vision-capable AI model, validates the extracted information against existing records, updates the database, and confirms the upload status.

---

### 2. Block-by-Block Analysis

#### Block 1: Input Reception and Daily Initialization
- **Overview:** Initiates the scheduled daily compliance check, fetches current vehicle records from the fleet inventory, and loads previous alert history to prevent duplicate notifications.
- **Nodes Involved:** 
  - `Daily Fleet Scan at 7 AM`
  - `Read Fleet Register`
  - `Read Alert Log`

- **Node Details:**
  - `Daily Fleet Scan at 7 AM`
    - **Type & Role:** `n8n-nodes-base.scheduleTrigger` (Triggers execution daily at 07:00 AM).
    - **Configuration:** Cron expression `0 7 * * *`.
    - **Connections:** Output connects to `Read Fleet Register`.
    - **Edge Cases:** Missed executions if the n8n instance is offline at the scheduled minute.
  - `Read Fleet Register`
    - **Type & Role:** `n8n-nodes-base.googleSheets` (Retrieves vehicle registry rows).
    - **Configuration:** Sheet name `Fleet_Register`, Document ID parameter set.
    - **Connections:** Input from `Daily Fleet Scan at 7 AM`; output connects to `Read Alert Log`.
    - **Credentials:** Google Sheets OAuth2 / Service Account.
    - **Edge Cases:** Spreadsheet API rate limits or connection timeouts.
  - `Read Alert Log`
    - **Type & Role:** `n8n-nodes-base.googleSheets` (Retrieves historical alert keys to deduplicate notifications).
    - **Configuration:** Sheet name `Alert_Log`, Document ID parameter set, executed once (`alwaysOutputData: true`).
    - **Connections:** Input from `Read Fleet Register`; output connects to `Evaluate Document Expiry & Dedupe`.
    - **Credentials:** Google Sheets OAuth2 / Service Account.
    - **Edge Cases:** Empty sheets returning null data, handled via string splitting protections in downstream code.

#### Block 2: Expiry Evaluation and e-Challan Processing
- **Overview:** Parses date strings, calculates remaining validity days, checks deduplication stages, queries external APIs for traffic violations, and computes an aggregate risk score.
- **Nodes Involved:**
  - `Evaluate Document Expiry & Dedupe`
  - `Check Pending e-Challans`
  - `Score Risk & Dispatch Gate`

- **Node Details:**
  - `Evaluate Document Expiry & Dedupe`
    - **Type & Role:** `n8n-nodes-base.code` (JavaScript execution block processing date parsing and alert stage checks).
    - **Configuration:** Custom JS code handling ISO, localized, and Excel serial date numbers against thresholds (0, 3, 7, 15, 30 days).
    - **Connections:** Input from `Read Alert Log`; output connects to `Check Pending e-Challans`.
    - **Edge Cases:** Malformed date formats resulting in null evaluations.
  - `Check Pending e-Challans`
    - **Type & Role:** `n8n-nodes-base.httpRequest` (Calls external e-challan API).
    - **Configuration:** POST method, endpoint URL configured, batch size of 5 items with 1500ms intervals, timeout 20,000ms. Error handling set to continue regular output with retries.
    - **Connections:** Input from `Evaluate Document Expiry & Dedupe`; output connects to `Score Risk & Dispatch Gate`.
    - **Credentials:** HTTP Header Authentication.
    - **Edge Cases:** API downtime, network timeouts, or invalid JSON responses.
  - `Score Risk & Dispatch Gate`
    - **Type & Role:** `n8n-nodes-base.code` (Calculates financial penalties, risk scores from 0–100, and evaluates dispatch restrictions).
    - **Configuration:** Custom JS evaluating expiry counts, warning windows, and unpaid fines.
    - **Connections:** Input from `Check Pending e-Challans`; outputs branch to `Needs Driver Action?` and `Dispatch Status Changed?`.
    - **Edge Cases:** Division by zero or missing child objects in HTTP responses.

#### Block 3: Decision Branching and AI Message Generation
- **Overview:** Filters records requiring intervention, uses language models to draft communication templates, and determines if project management tracking tasks need creation.
- **Nodes Involved:**
  - `Needs Driver Action?`
  - `AI Compose Driver Alerts`
  - `New Renewal Task Needed?`

- **Node Details:**
  - `Needs Driver Action?`
    - **Type & Role:** `n8n-nodes-base.if` (Conditional gate).
    - **Configuration:** Evaluates boolean flag `needs_action`.
    - **Connections:** Input from `Score Risk & Dispatch Gate`; output connects to `AI Compose Driver Alerts`.
  - `AI Compose Driver Alerts`
    - **Type & Role:** `@n8n/n8n-nodes-langchain.openAi` (Generates structured JSON messages via LLM).
    - **Configuration:** Model `gpt-4o-mini`, temperature `0.2`, structured JSON output enabled.
    - **Connections:** Input from `Needs Driver Action?`; outputs connect to `New Renewal Task Needed?` and `Build Manager Digests`.
    - **Credentials:** OpenAI API.
    - **Edge Cases:** Token limit exhaustion or invalid JSON generation from the model.
  - `New Renewal Task Needed?`
    - **Type & Role:** `n8n-nodes-base.if` (Conditional check).
    - **Configuration:** Evaluates length of `new_task_keys` array > 0.
    - **Connections:** Input from `AI Compose Driver Alerts`; true branch connects to `Create ClickUp Renewal Task`, false branch routes to `Alert Driver on Telegram`.
  - `Create ClickUp Renewal Task`
    - **Type & Role:** `n8n-nodes-base.clickUp` (Creates project management issues).
    - **Configuration:** Team, space, and list IDs configured; maps task title, description, due date, and priority.
    - **Connections:** Input from `New Renewal Task Needed?` (true); output connects to `Alert Driver on Telegram`.
    - **Credentials:** ClickUp API.
    - **Edge Cases:** Invalid workspace IDs or permission restrictions.

#### Block 4: Notifications, Dispatch Enforcement, and Logging
- **Overview:** Dispatches messages to drivers, logs delivery status, handles fleet manager email digests, and updates operational channels regarding dispatch state changes.
- **Nodes Involved:**
  - `Alert Driver on Telegram`
  - `Log Alert Sent`
  - `Dispatch Status Changed?`
  - `Update Dispatch Status in Register`
  - `Alert Ops on Dispatch Change`
  - `Build Manager Digests`
  - `Email Daily Digest to Fleet Manager`

- **Node Details:**
  - `Alert Driver on Telegram`
    - **Type & Role:** `n8n-nodes-base.telegram` (Sends text messages to mobile chats).
    - **Configuration:** Uses chat ID from item data, text mapped from AI response. Error handling set to continue execution.
    - **Connections:** Input from `New Renewal Task Needed?` (false) or `Create ClickUp Renewal Task`; output connects to `Log Alert Sent`.
    - **Credentials:** Telegram Bot API.
    - **Edge Cases:** Blocked bots, invalid chat IDs, or Telegram rate limits.
  - `Log Alert Sent`
    - **Type & Role:** `n8n-nodes-base.googleSheets` (Appends audit rows to audit log).
    - **Configuration:** Sheet name `Alert_Log`, appends timestamp, risk score, keys, and delivery metrics.
    - **Connections:** Input from `Alert Driver on Telegram`.
    - **Credentials:** Google Sheets OAuth2 / Service Account.
  - `Dispatch Status Changed?`
    - **Type & Role:** `n8n-nodes-base.if` (Evaluates state modification).
    - **Configuration:** Checks boolean flag `dispatch_changed`.
    - **Connections:** Input from `Score Risk & Dispatch Gate`; true branch connects to `Update Dispatch Status in Register`.
  - `Update Dispatch Status in Register`
    - **Type & Role:** `n8n-nodes-base.googleSheets` (Updates vehicle state).
    - **Configuration:** Operation `update`, matching column `vehicle_no`, updates status and block reason.
    - **Connections:** Input from `Dispatch Status Changed?`; output connects to `Alert Ops on Dispatch Change`.
    - **Credentials:** Google Sheets OAuth2 / Service Account.
  - `Alert Ops on Dispatch Change`
    - **Type & Role:** `n8n-nodes-base.slack` (Sends operational alerts).
    - **Configuration:** Channel ID configured, formats message based on clearance state.
    - **Connections:** Input from `Update Dispatch Status in Register`.
    - **Credentials:** Slack OAuth2 / Bot.
  - `Build Manager Digests`
    - **Type & Role:** `n8n-nodes-base.code` (Aggregates reports by manager email address).
    - **Configuration:** JavaScript grouping and HTML template compilation.
    - **Connections:** Input from `AI Compose Driver Alerts`; output connects to `Email Daily Digest to Fleet Manager`.
  - `Email Daily Digest to Fleet Manager`
    - **Type & Role:** `n8n-nodes-base.gmail` (Sends compiled HTML email reports).
    - **Configuration:** Maps recipient from group key, subject line, and HTML body.
    - **Connections:** Input from `Build Manager Digests`.
    - **Credentials:** Gmail OAuth2.
    - **Edge Cases:** SMTP rejection or invalid recipient email formats.

#### Block 5: Driver Upload Processing and Document Verification Loop
- **Overview:** Processes inbound messages from Telegram, validates vehicle ownership, extracts compliance data from uploaded images via multimodal AI, validates dates, updates storage records, and notifies operators.
- **Nodes Involved:**
  - `On Driver Telegram Message`
  - `Look Up Driver Vehicle`
  - `Accept Only Driver Photos`
  - `Download Document Photo`
  - `AI Read Document Photo`
  - `Validate Document Details`
  - `Document Valid?`
  - `Update Expiry in Register`
  - `Ask Driver to Resend`
  - `Ask Ops to Verify Upload`
  - `Confirm Update to Driver`

- **Node Details:**
  - `On Driver Telegram Message`
    - **Type & Role:** `n8n-nodes-base.telegramTrigger` (Webhook entry point for incoming bot messages).
    - **Configuration:** Listens to `message` update events.
    - **Connections:** Output connects to `Look Up Driver Vehicle`.
    - **Credentials:** Telegram Bot API.
  - `Look Up Driver Vehicle`
    - **Type & Role:** `n8n-nodes-base.googleSheets` (Filters register by sender chat ID).
    - **Configuration:** Filter criteria matching `message.chat.id` against `driver_telegram_chat_id`, output always enabled.
    - **Connections:** Input from `On Driver Telegram Message`; output connects to `Accept Only Driver Photos`.
    - **Credentials:** Google Sheets OAuth2 / Service Account.
  - `Accept Only Driver Photos`
    - **Type & Role:** `n8n-nodes-base.code` (Filters messages containing valid image attachments from authorized drivers).
    - **Configuration:** JavaScript extraction of `file_id` from photo arrays or document nodes.
    - **Connections:** Input from `Look Up Driver Vehicle`; output connects to `Download Document Photo`.
  - `Download Document Photo`
    - **Type & Role:** `n8n-nodes-base.telegram` (Downloads file binary data).
    - **Configuration:** Resource `file`, parameter `file_id`.
    - **Connections:** Input from `Accept Only Driver Photos`; output connects to `AI Read Document Photo`.
    - **Credentials:** Telegram Bot API.
  - `AI Read Document Photo`
    - **Type & Role:** `@n8n/n8n-nodes-langchain.openAi` (Multimodal vision analysis).
    - **Configuration:** Model `gpt-4o`, operation `analyze`, high detail mode, constrained output schema for extraction.
    - **Connections:** Input from `Download Document Photo`; output connects to `Validate Document Details`.
    - **Credentials:** OpenAI API.
  - `Validate Document Details`
    - **Type & Role:** `n8n-nodes-base.code` (Validates extracted document metrics against system policies).
    - **Configuration:** Custom JS checking vehicle registration matching, expiration date sanity, and confidence thresholds (>0.6).
    - **Connections:** Input from `AI Read Document Photo`; output connects to `Document Valid?`.
  - `Document Valid?`
    - **Type & Role:** `n8n-nodes-base.if` (Conditional evaluation).
    - **Configuration:** Checks boolean flag `ctx.valid`.
    - **Connections:** Input from `Validate Document Details`; true branch connects to `Update Expiry in Register`, false branch connects to `Ask Driver to Resend`.
  - `Update Expiry in Register`
    - **Type & Role:** `n8n-nodes-base.googleSheets` (Persists validated updates).
    - **Configuration:** Operation `update`, mapping mode auto-map input data, matching column `vehicle_no`.
    - **Connections:** Input from `Document Valid?` (true); output connects to `Ask Ops to Verify Upload`.
    - **Credentials:** Google Sheets OAuth2 / Service Account.
  - `Ask Driver to Resend`
    - **Type & Role:** `n8n-nodes-base.telegram` (Sends rejection notification).
    - **Configuration:** Maps chat ID and failure reasons from context.
    - **Connections:** Input from `Document Valid?` (false).
    - **Credentials:** Telegram Bot API.
  - `Ask Ops to Verify Upload`
    - **Type & Role:** `n8n-nodes-base.slack` (Sends verification task to operators).
    - **Configuration:** Channel ID configured, maps document details and AI confidence scores.
    - **Connections:** Input from `Update Expiry in Register`; output connects to `Confirm Update to Driver`.
    - **Credentials:** Slack OAuth2 / Bot.
  - `Confirm Update to Driver`
    - **Type & Role:** `n8n-nodes-base.telegram` (Sends completion confirmation).
    - **Configuration:** Maps chat ID, updated document label, and operational status.
    - **Connections:** Input from `Ask Ops to Verify Upload`.
    - **Credentials:** Telegram Bot API.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note – Overview | n8n-nodes-base.stickyNote | Workflow documentation | None | None | ## 🚛 Fleet Compliance OS<br><br>For Indian fleet owners and transport managers who track RC, insurance, PUC, fitness and permit dates in a spreadsheet, and find out a document lapsed when a truck is stopped or a challan arrives.<br><br>### How it works<br>Every morning each vehicle's document dates are checked, and pending e-challans are pulled from your challan provider. Each vehicle gets a risk score, and any vehicle with an expired document is blocked from dispatch, with ops told in Slack. Drivers get an AI-written Telegram reminder at 30, 15, 7, 3 and 0 days, a ClickUp renewal task is opened, and fleet managers get a daily email digest.<br><br>The loop closes on Telegram. A driver sends a photo of the renewed document to the bot, GPT-4o reads it, and if the vehicle number and new expiry check out, the register is updated and the driver gets a confirmation.<br><br>### Setup steps<br>1. Create a Google Sheet with **Fleet_Register** and **Alert_Log** tabs. The register needs one row per vehicle, with expiry columns and the driver's Telegram chat ID.<br>2. Connect Google Sheets, OpenAI, Telegram (bot), Slack, ClickUp and Gmail.<br>3. Add your e-challan API as Header Auth and set its URL in **Check Pending e-Challans**.<br>4. Replace `YOUR_GOOGLE_SHEET_ID`, the Slack channel and the ClickUp list, then run the daily scan once manually. |
| Sticky Note – Daily Scan & Expiry Check | n8n-nodes-base.stickyNote | Block documentation | None | None | ## 🗓️ Daily Scan & Expiry Check<br>At 7 AM every vehicle's RC, insurance, PUC, fitness and permit dates are checked. Alerts go out at 30, 15, 7, 3 and 0 days, and the alert log makes sure each stage fires only once. |
| Sticky Note – e-Challan & Risk Score | n8n-nodes-base.stickyNote | Block documentation | None | None | ## 🚦 e-Challan & Risk Score<br>Pulls pending e-challans for each vehicle from your provider, then gives the vehicle a 0–100 risk score. Any expired document blocks the vehicle from dispatch. |
| Sticky Note – AI Alerts & Renewal Tasks | n8n-nodes-base.stickyNote | Block documentation | None | None | ## 🧠 AI Alerts & Renewal Tasks<br>GPT-4o-mini writes a short Telegram message for the driver about what's due. A ClickUp renewal task is created once per document, and every alert is logged. |
| Sticky Note – Fleet Manager Digest | n8n-nodes-base.stickyNote | Block documentation | None | None | ## 📧 Fleet Manager Digest<br>Each fleet manager gets one email listing blocked vehicles, upcoming renewals and new challans. |
| Sticky Note – Dispatch Gate | n8n-nodes-base.stickyNote | Block documentation | None | None | ## ⛔ Dispatch Gate<br>When a vehicle flips between allowed and blocked, the register is updated and ops are told in Slack, so nobody dispatches a truck with lapsed papers. |
| Sticky Note – Driver Upload & AI Read | n8n-nodes-base.stickyNote | Block documentation | None | None | ## 📸 Driver Upload & AI Read<br>Drivers renew by sending a photo of the new document to the Telegram bot. Messages from unknown chats or without an image are ignored. GPT-4o reads the type, vehicle number and expiry date. |
| Sticky Note – Validate & Update | n8n-nodes-base.stickyNote | Block documentation | None | None | ## ✅ Validate & Update<br>The new expiry must beat both today and the date on file, the vehicle number must match and the photo must be clear. Good uploads update the register, ops get a Slack note to spot-check and the driver is told. Otherwise they're asked to resend. |
| Sticky Note – Credentials & Security | n8n-nodes-base.stickyNote | Block documentation | None | None | ## 🔐 Credentials & Security<br>Keep the challan API key in Header Auth and the Telegram bot token as a credential. The bot only accepts photos from chat IDs in your register. Document photos include personal details, so don't forward them outside ops. |
| Daily Fleet Scan at 7 AM | n8n-nodes-base.scheduleTrigger | Schedule entry point | None | Read Fleet Register | |
| Read Fleet Register | n8n-nodes-base.googleSheets | Read vehicle inventory | Daily Fleet Scan at 7 AM | Read Alert Log | |
| Read Alert Log | n8n-nodes-base.googleSheets | Read notification history | Read Fleet Register | Evaluate Document Expiry & Dedupe | |
| Evaluate Document Expiry & Dedupe | n8n-nodes-base.code | Process expiration dates | Read Alert Log | Check Pending e-Challans | |
| Check Pending e-Challans | n8n-nodes-base.httpRequest | Query external challan API | Evaluate Document Expiry & Dedupe | Score Risk & Dispatch Gate | |
| Score Risk & Dispatch Gate | n8n-nodes-base.code | Calculate risk & restrictions | Check Pending e-Challans | Needs Driver Action?, Dispatch Status Changed? | |
| Needs Driver Action? | n8n-nodes-base.if | Filter actionable records | Score Risk & Dispatch Gate | AI Compose Driver Alerts | |
| Dispatch Status Changed? | n8n-nodes-base.if | Detect state modifications | Score Risk & Dispatch Gate | Update Dispatch Status in Register | |
| AI Compose Driver Alerts | @n8n/n8n-nodes-langchain.openAi | Generate structured AI text | Needs Driver Action? | New Renewal Task Needed?, Build Manager Digests | |
| Update Dispatch Status in Register | n8n-nodes-base.googleSheets | Update database status | Dispatch Status Changed? | Alert Ops on Dispatch Change | |
| New Renewal Task Needed? | n8n-nodes-base.if | Evaluate task creation | AI Compose Driver Alerts | Create ClickUp Renewal Task, Alert Driver on Telegram | |
| Build Manager Digests | n8n-nodes-base.code | Aggregate manager summaries | AI Compose Driver Alerts | Email Daily Digest to Fleet Manager | |
| Alert Ops on Dispatch Change | n8n-nodes-base.slack | Notify operations team | Update Dispatch Status in Register | None | |
| Create ClickUp Renewal Task | n8n-nodes-base.clickUp | Create tracking task | New Renewal Task Needed? | Alert Driver on Telegram | |
| Email Daily Digest to Fleet Manager | n8n-nodes-base.gmail | Send summary email | Build Manager Digests | None | |
| Alert Driver on Telegram | n8n-nodes-base.telegram | Send driver notification | New Renewal Task Needed?, Create ClickUp Renewal Task | Log Alert Sent | |
| Log Alert Sent | n8n-nodes-base.googleSheets | Log alert execution | Alert Driver on Telegram | None | |
| On Driver Telegram Message | n8n-nodes-base.telegramTrigger | Webhook for incoming messages | None | Look Up Driver Vehicle | |
| Look Up Driver Vehicle | n8n-nodes-base.googleSheets | Authenticate sender | On Driver Telegram Message | Accept Only Driver Photos | |
| Accept Only Driver Photos | n8n-nodes-base.code | Filter image attachments | Look Up Driver Vehicle | Download Document Photo | |
| Download Document Photo | n8n-nodes-base.telegram | Download binary file | Accept Only Driver Photos | AI Read Document Photo | |
| AI Read Document Photo | @n8n/n8n-nodes-langchain.openAi | Multimodal image reading | Download Document Photo | Validate Document Details | |
| Validate Document Details | n8n-nodes-base.code | Validate parsed document | AI Read Document Photo | Document Valid? | |
| Document Valid? | n8n-nodes-base.if | Evaluate validation status | Validate Document Details | Update Expiry in Register, Ask Driver to Resend | |
| Update Expiry in Register | n8n-nodes-base.googleSheets | Update document expiration | Document Valid? | Ask Ops to Verify Upload | |
| Ask Driver to Resend | n8n-nodes-base.telegram | Request document resubmission | Document Valid? | None | |
| Ask Ops to Verify Upload | n8n-nodes-base.slack | Notify ops of upload | Update Expiry in Register | Confirm Update to Driver | |
| Confirm Update to Driver | n8n-nodes-base.telegram | Confirm successful upload | Ask Ops to Verify Upload | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Triggers:**
   - Add a **Schedule Trigger** node (`Daily Fleet Scan at 7 AM`) and set the cron expression to `0 7 * * *`.
   - Add a **Telegram Trigger** node (`On Driver Telegram Message`) configured to listen to `message` updates.

2. **Set up Data Retrieval (Fleet Scan Pipeline):**
   - Connect `Daily Fleet Scan at 7 AM` to a **Google Sheets** node named `Read Fleet Register`. Set the sheet name to `Fleet_Register` and configure your document ID.
   - Connect `Read Fleet Register` to another **Google Sheets** node named `Read Alert Log`. Set the sheet name to `Alert_Log`, check `Always Output Data`, and execute once.

3. **Configure Expiry Evaluation and API Verification:**
   - Connect `Read Alert Log` to a **Code** node named `Evaluate Document Expiry & Dedupe`. Insert the JavaScript snippet that processes date calculations across `rc`, `insurance`, `puc`, `fitness`, and `permit` columns.
   - Connect `Evaluate Document Expiry & Dedupe` to an **HTTP Request** node named `Check Pending e-Challans`. Set method to `POST`, URL to your e-challan provider endpoint, authentication to Header Auth, and configure batching (size 5, interval 1500ms).
   - Connect `Check Pending e-Challans` to a **Code** node named `Score Risk & Dispatch Gate` to calculate risk scores and evaluate dispatch gating.

4. **Configure Conditional Branching and AI Processing:**
   - Connect `Score Risk & Dispatch Gate` to an **If** node (`Needs Driver Action?`) checking if `needs_action` is true.
   - Connect the true branch of `Needs Driver Action?` to an **OpenAI** node named `AI Compose Driver Alerts`. Select model `gpt-4o-mini`, set temperature to `0.2`, and enable JSON output with the required system prompt.
   - Connect `Score Risk & Dispatch Gate` to a second **If** node (`Dispatch Status Changed?`) checking `dispatch_changed`.

5. **Set up Task Management and Notifications:**
   - From `AI Compose Driver Alerts`, branch to:
     - An **If** node (`New Renewal Task Needed?`) checking if `new_task_keys.length > 0`.
       - If true, connect to a **ClickUp** node named `Create ClickUp Renewal Task` (configure team, space, list, and mapping fields).
       - Route both true/false paths of the task check to a **Telegram** node named `Alert Driver on Telegram` (configure chat ID and text mapping).
       - Connect `Alert Driver on Telegram` to a **Google Sheets** node named `Log Alert Sent` (append mode, mapping audit columns).
     - A **Code** node named `Build Manager Digests`, connected to a **Gmail** node named `Email Daily Digest to Fleet Manager` (mapping recipient, subject, and HTML body).
   - From `Dispatch Status Changed?` (true branch), connect to a **Google Sheets** node named `Update Dispatch Status in Register` (update mode, matching on `vehicle_no`), followed by a **Slack** node named `Alert Ops on Dispatch Change`.

6. **Configure the Inbound Driver Upload Loop:**
   - Connect `On Driver Telegram Message` to a **Google Sheets** node named `Look Up Driver Vehicle` (filter: `message.chat.id` matches `driver_telegram_chat_id`).
   - Connect `Look Up Driver Vehicle` to a **Code** node named `Accept Only Driver Photos` to extract valid file IDs.
   - Connect `Accept Only Driver Photos` to a **Telegram** node named `Download Document Photo` (resource: `file`).
   - Connect `Download Document Photo` to an **OpenAI** node named `AI Read Document Photo` (model `gpt-4o`, operation `analyze`, image resource, high detail).
   - Connect `AI Read Document Photo` to a **Code** node named `Validate Document Details` to verify vehicle registration matching and expiration date validity.
   - Connect `Validate Document Details` to an **If** node (`Document Valid?`) checking `ctx.valid`.
     - **True Branch:** Connect to a **Google Sheets** node named `Update Expiry in Register` (update mode, auto-map input data, matching on `vehicle_no`), followed by a **Slack** node named `Ask Ops to Verify Upload`, and finally a **Telegram** node named `Confirm Update to Driver`.
     - **False Branch:** Connect to a **Telegram** node named `Ask Driver to Resend`.

7. **Attach Credentials:**
   - Assign Google Sheets credentials to all Google Sheets nodes.
   - Assign OpenAI API credentials to `AI Compose Driver Alerts` and `AI Read Document Photo`.
   - Assign Telegram Bot credentials to `Alert Driver on Telegram`, `On Driver Telegram Message`, `Download Document Photo`, `Ask Driver to Resend`, and `Confirm Update to Driver`.
   - Assign Slack API credentials to `Alert Ops on Dispatch Change` and `Ask Ops to Verify Upload`.
   - Assign ClickUp API credentials to `Create ClickUp Renewal Task`.
   - Assign Gmail OAuth2 credentials to `Email Daily Digest to Fleet Manager`.
   - Assign HTTP Header Auth credentials to `Check Pending e-Challans`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Maintain the e-challan API key within n8n Header Auth credentials and keep the Telegram bot token secure. Document photos contain sensitive personal or vehicle details and should not be forwarded outside operational channels. | Credentials & Security |
| Ensure each vehicle's Telegram chat ID is populated correctly in the `Fleet_Register` spreadsheet to prevent unauthorized senders from triggering update loops. | Telegram Bot Security & Webhook Setup |
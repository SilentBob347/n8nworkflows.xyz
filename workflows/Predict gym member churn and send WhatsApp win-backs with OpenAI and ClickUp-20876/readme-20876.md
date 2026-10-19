Predict gym member churn and send WhatsApp win-backs with OpenAI and ClickUp

https://n8nworkflows.xyz/workflows/predict-gym-member-churn-and-send-whatsapp-win-backs-with-openai-and-clickup-20876


# Predict gym member churn and send WhatsApp win-backs with OpenAI and ClickUp

### 1. Workflow Overview

This workflow automates gym member retention by predicting churn risk using historical visit and membership data stored in Google Sheets. Every morning at 7:00 AM, the system analyzes active members, scores their risk level, filters out cooled-down or opted-out members, and generates personalized WhatsApp win-back or renewal offers via OpenAI. It dispatches the messages, logs the offers, creates follow-up tasks in ClickUp for assigned trainers, and sends operational summaries to the gym owner and trainers. 

Additionally, the workflow includes an incoming webhook listener that handles incoming WhatsApp responses from members in real time. It identifies member intent using OpenAI (classifying messages as accept, freeze, callback, stop, or question/other), updates Google Sheets records, triggers relevant tasks, replies directly on WhatsApp, and escalates requests to staff members when manual follow-up is necessary.

The workflow logic is divided into the following functional blocks:
- **1.1 Daily Data Ingestion & Churn Risk Scoring:** Reads gym settings, member data, check-ins, and past offers from Google Sheets, then evaluates each member's churn score and segmentation profile.
- **1.2 Win-Back Outreach Generation & Execution:** Uses OpenAI to personalize messages for eligible at-risk members, sends offers via WhatsApp, creates trainer tasks in ClickUp, and logs outreach attempts.
- **1.3 Management & Trainer Digests:** Compiles and dispatches daily summary notifications to trainers and gym owners over WhatsApp, appending daily retention snapshots to Google Sheets.
- **1.4 Incoming Reply Intake & Intent Classification:** Listens for inbound WhatsApp member replies, retrieves matching member and offer data from Google Sheets, and uses OpenAI to categorize member intent and draft responses.
- **1.5 Record Updates, Escalations & Replies:** Updates Google Sheets status fields, creates administrative or follow-up ClickUp tasks, replies to members on WhatsApp, and notifies staff when human intervention is required.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Data Ingestion & Churn Risk Scoring
- **Overview:** This block establishes configuration rules, pulls raw operational data from Google Sheets, and calculates a data-driven churn risk score and outreach eligibility status for every active gym member.
- **Nodes Involved:** `Daily 7 AM1`, `GymOS Config1`, `Read Members1`, `Read Check-ins1`, `Read Offer Log1`, `Score Churn Risk1`, `Route: Outreach or Summary1`
- **Node Details:**
  - **Daily 7 AM1**
    - Type and technical role: `n8n-nodes-base.scheduleTrigger` (Schedule Trigger node)
    - Configuration choices: Configured to run daily via cron expression `0 7 * * *`.
    - Input/Output: No inputs; outputs cron tick payload to `GymOS Config1`.
    - Edge cases: System downtime at 7 AM will skip the daily execution until the next scheduled window.
  - **GymOS Config1**
    - Type and technical role: `n8n-nodes-base.code` (JavaScript execution node)
    - Configuration choices: Establishes global gym settings, timezones, thresholds, staff contact numbers, cooldown windows, and tiered offer definitions.
    - Input/Output: Input from `Daily 7 AM1`; output consumed by downstream scoring and AI prompt nodes.
    - Edge cases: Ensure placeholder values (like `REPLACE_WITH_OWNER_NUMBER`) are replaced with valid phone formats.
  - **Read Members1**, **Read Check-ins1**, **Read Offer Log1**
    - Type and technical role: `n8n-nodes-base.googleSheets` (Google Sheets integration nodes)
    - Configuration choices: Uses Google Sheets API to read rows from the “Members”, “Checkins”, and “Offer Log” tabs respectively. Set to execute once (`executeOnce: true`).
    - Input/Output: Sequential dependency chain linking configuration data through each sheet reader.
    - Edge cases: Authentication token expiration or invalid spreadsheet IDs will halt data extraction.
  - **Score Churn Risk1**
    - Type and technical role: `n8n-nodes-base.code` (JavaScript calculation engine)
    - Configuration choices: Parses check-in frequencies, tenure, membership expiry dates, and past offers to assign a churn risk score (0–100), severity tier, and segmentation status.
    - Input/Output: Consumes arrays from the read nodes; outputs target outreach payloads and a consolidated daily summary object.
    - Edge cases: Malformed date formats in Google Sheets can cause parsing failures or invalid scoring metrics.
  - **Route: Outreach or Summary1**
    - Type and technical role: `n8n-nodes-base.switch` (Flow control switch node)
    - Configuration choices: Evaluates the `type` property (`outreach` or `summary`) to split processing branches.
    - Input/Output: Input from `Score Churn Risk1`; branches into the outreach processing chain and the digest generation chain.

---

#### 1.2 Win-Back Outreach Generation & Execution
- **Overview:** This block takes eligible at-risk members, generates localized and contextualized win-back messaging via OpenAI, delivers the message over WhatsApp, schedules a trainer task in ClickUp, and logs the event.
- **Nodes Involved:** `AI Personalise Message1`, `Compose Outreach1`, `WhatsApp: Send Offer1`, `ClickUp: Trainer Follow-up1`, `Log Offer1`
- **Node Details:**
  - **AI Personalise Message1**
    - Type and technical role: `@n8n/n8n-nodes-langchain.openAi` (OpenAI Language Model node)
    - Configuration choices: Uses model `gpt-4o-mini` with a temperature of `0.6` to generate personalized WhatsApp text and trainer talking points based on member data.
    - Input/Output: Input from `Route: Outreach or Summary1`; output feeds `Compose Outreach1`.
    - Edge cases: Rate limits or API credit depletion on OpenAI will fail generation.
  - **Compose Outreach1**
    - Type and technical role: `n8n-nodes-base.code` (JavaScript transformation node)
    - Configuration choices: Combines OpenAI output with local template strings to format final WhatsApp text payloads and ClickUp task details.
    - Input/Output: Input from `AI Personalise Message1`; output feeds `WhatsApp: Send Offer1`.
  - **WhatsApp: Send Offer1**
    - Type and technical role: `n8n-nodes-base.whatsApp` (WhatsApp Business Cloud node)
    - Configuration choices: Sends text messages using the Cloud API with error handling set to continue regular execution (`onError: continueRegularOutput`).
    - Input/Output: Input from `Compose Outreach1`; output feeds `ClickUp: Trainer Follow-up1`.
    - Edge cases: Invalid phone numbers or unverified recipient devices will trigger delivery errors.
  - **ClickUp: Trainer Follow-up1**
    - Type and technical role: `n8n-nodes-base.clickUp` (ClickUp project management node)
    - Configuration choices: Creates a task in the specified team, space, and list IDs with priority levels and due dates calculated dynamically.
    - Input/Output: Input from `WhatsApp: Send Offer1`; output feeds `Log Offer1`.
  - **Log Offer1**
    - Type and technical role: `n8n-nodes-base.googleSheets` (Google Sheets append node)
    - Configuration choices: Appends a tracking row containing member identifiers, offer details, expiration dates, and status codes to the “Offer Log” tab.
    - Input/Output: Input from `ClickUp: Trainer Follow-up1`.

---

#### 1.3 Management & Trainer Digests
- **Overview:** This block aggregates scoring data and trainer lists into formatted daily summary reports, transmits them via WhatsApp to staff and owners, and logs a retention metrics snapshot to Google Sheets.
- **Nodes Involved:** `Build Trainer & Owner Digests1`, `WhatsApp: Send Digests1`, `Append Retention Snapshot1`
- **Node Details:**
  - **Build Trainer & Owner Digests1**
    - Type and technical role: `n8n-nodes-base.code` (JavaScript aggregation node)
    - Configuration choices: Formats targeted lists for individual trainers and a high-level revenue and risk dashboard for the gym owner.
    - Input/Output: Input from `Route: Outreach or Summary1`; output feeds `WhatsApp: Send Digests1`.
  - **WhatsApp: Send Digests1**
    - Type and technical role: `n8n-nodes-base.whatsApp` (WhatsApp Business Cloud node)
    - Configuration choices: Dispatches summary text messages to staff phone numbers with error handling set to continue regular execution.
    - Input/Output: Input from `Build Trainer & Owner Digests1`; output feeds `Append Retention Snapshot1`.
  - **Append Retention Snapshot1**
    - Type and technical role: `n8n-nodes-base.googleSheets` (Google Sheets append node)
    - Configuration choices: Appends daily snapshot analytics (active member counts, risk tiers, revenue at risk) to the “Retention Daily” tab.
    - Input/Output: Input from `WhatsApp: Send Digests1`.

---

#### 1.4 Incoming Reply Intake & Intent Classification
- **Overview:** This block acts as a webhook receiver for incoming WhatsApp messages from members, queries Google Sheets to match records and offer logs, and uses OpenAI to determine member intent and draft replies.
- **Nodes Involved:** `Member Reply (WhatsApp)1`, `Parse Reply1`, `Find Member1`, `Find Member Offers1`, `Build Reply Context1`, `AI Classify Intent1`, `Route by Intent1`
- **Node Details:**
  - **Member Reply (WhatsApp)1**
    - Type and technical role: `n8n-nodes-base.whatsAppTrigger` (WhatsApp Webhook Trigger node)
    - Configuration choices: Listens for incoming WhatsApp message updates on configured webhooks.
    - Input/Output: External trigger; outputs raw webhook payloads to `Parse Reply1`.
  - **Parse Reply1**
    - Type and technical role: `n8n-nodes-base.code` (JavaScript parser node)
    - Configuration choices: Normalizes inbound webhook structures, extracting phone numbers, message text, and profile names.
    - Input/Output: Input from `Member Reply (WhatsApp)1`; output feeds `Find Member1`.
  - **Find Member1**, **Find Member Offers1**
    - Type and technical role: `n8n-nodes-base.googleSheets` (Google Sheets lookup nodes)
    - Configuration choices: Looks up member records and offer logs matching the sender's phone number.
    - Input/Output: Sequential lookup chain starting from `Parse Reply1`; output feeds `Build Reply Context1`.
  - **Build Reply Context1**
    - Type and technical role: `n8n-nodes-base.code` (JavaScript context builder node)
    - Configuration choices: Consolidates member profiles, expiration data, and the most recent offer code into a unified processing context.
    - Input/Output: Input from `Find Member Offers1`; output feeds `AI Classify Intent1`.
  - **AI Classify Intent1**
    - Type and technical role: `@n8n/n8n-nodes-langchain.openAi` (OpenAI Language Model node)
    - Configuration choices: Evaluates incoming messages against specific intents (`accept`, `freeze`, `callback`, `stop`, `question`, `other`), assigns sentiment, and drafts an appropriate reply using `gpt-4o-mini` at temperature `0.3`.
    - Input/Output: Input from `Build Reply Context1`; output feeds `Route by Intent1`.
  - **Route by Intent1**
    - Type and technical role: `n8n-nodes-base.switch` (Flow control switch node)
    - Configuration choices: Routes execution paths based on the classified intent (`Accept`, `Freeze`, `Callback`, `Stop`, with fallback routing for `Question / Other`).
    - Input/Output: Input from `AI Classify Intent1`; outputs feed respective database updates or direct replies.

---

#### 1.5 Record Updates, Escalations & Replies
- **Overview:** This block executes intent-specific actions (updating offer logs, flagging member opt-outs, creating ClickUp support tasks), sends the AI-drafted reply to the member, and notifies designated staff members when escalation is needed.
- **Nodes Involved:** `Offer → Accepted1`, `ClickUp: Process Renewal1`, `Offer → Freeze Requested1`, `ClickUp: Trainer Callback1`, `Member → Opted Out1`, `WhatsApp: Reply to Member1`, `Needs Staff?1`, `WhatsApp: Notify Trainer / Desk1`
- **Node Details:**
  - **Offer → Accepted1**, **Offer → Freeze Requested1**
    - Type and technical role: `n8n-nodes-base.googleSheets` (Google Sheets update nodes)
    - Configuration choices: Updates offer log records matching the active offer code to status values like `accepted` or `freeze_requested`.
    - Input/Output: Triggered by `Route by Intent1`; outputs feed subsequent task creation or direct reply nodes.
  - **ClickUp: Process Renewal1**, **ClickUp: Trainer Callback1**
    - Type and technical role: `n8n-nodes-base.clickUp` (ClickUp task management nodes)
    - Configuration choices: Creates urgent tasks for renewals or trainer callbacks with contextual member data and due dates.
    - Input/Output: Triggered by intent routes; output feeds `WhatsApp: Reply to Member1`.
  - **Member → Opted Out1**
    - Type and technical role: `n8n-nodes-base.googleSheets` (Google Sheets update node)
    - Configuration choices: Updates member records to flag `opted_out` as `yes` based on the member ID.
    - Input/Output: Triggered by the `Stop` intent route from `Route by Intent1`; output feeds `WhatsApp: Reply to Member1`.
  - **WhatsApp: Reply to Member1**
    - Type and technical role: `n8n-nodes-base.whatsApp` (WhatsApp Business Cloud node)
    - Configuration choices: Dispatches the AI-drafted response text back to the member's verified WhatsApp phone number.
    - Input/Output: Aggregates inputs from various intent handling paths; output feeds `Needs Staff?1`.
  - **Needs Staff?1**
    - Type and technical role: `n8n-nodes-base.if` (Conditional evaluation node)
    - Configuration choices: Evaluates whether the intent requires operational escalation (`accept`, `freeze`, or `callback`).
    - Input/Output: Input from `WhatsApp: Reply to Member1`; output feeds `WhatsApp: Notify Trainer / Desk1` if true.
  - **WhatsApp: Notify Trainer / Desk1**
    - Type and technical role: `n8n-nodes-base.whatsApp` (WhatsApp Business Cloud node)
    - Configuration choices: Sends alert notifications to assigned trainers or falls back to front desk contact numbers.
    - Input/Output: Input from `Needs Staff?1`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Daily 7 AM1` | n8n-nodes-base.scheduleTrigger | Triggers the daily morning workflow execution at 7 AM. | None | `GymOS Config1` | ## Schedule & Scoring<nA 7 AM schedule loads your settings, members, check-ins and past offers, then scores every active member on visit drops, plan expiry and lapses. Gym name, offers and thresholds all live in the config node. |
| `GymOS Config1` | n8n-nodes-base.code | Defines global variables, configuration thresholds, and offer structures. | `Daily 7 AM1` | `Read Members1` | ## Schedule & Scoring<nA 7 AM schedule loads your settings, members, check-ins and past offers, then scores every active member on visit drops, plan expiry and lapses. Gym name, offers and thresholds all live in the config node. |
| `Read Members1` | n8n-nodes-base.googleSheets | Reads member records from the Google Sheet. | `GymOS Config1` | `Read Check-ins1` | ## Schedule & Scoring<nA 7 AM schedule loads your settings, members, check-ins and past offers, then scores every active member on visit drops, plan expiry and lapses. Gym name, offers and thresholds all live in the config node. |
| `Read Check-ins1` | n8n-nodes-base.googleSheets | Reads check-in history records from the Google Sheet. | `Read Members1` | `Read Offer Log1` | ## Schedule & Scoring<nA 7 AM schedule loads your settings, members, check-ins and past offers, then scores every active member on visit drops, plan expiry and lapses. Gym name, offers and thresholds all live in the config node. |
| `Read Offer Log1` | n8n-nodes-base.googleSheets | Reads past offer history logs from the Google Sheet. | `Read Check-ins1` | `Score Churn Risk1` | ## Schedule & Scoring<nA 7 AM schedule loads your settings, members, check-ins and past offers, then scores every active member on visit drops, plan expiry and lapses. Gym name, offers and thresholds all live in the config node. |
| `Score Churn Risk1` | n8n-nodes-base.code | Computes churn scores, risk tiers, and target segments. | `Read Offer Log1` | `Route: Outreach or Summary1` | ## Schedule & Scoring<nA 7 AM schedule loads your settings, members, check-ins and past offers, then scores every active member on visit drops, plan expiry and lapses. Gym name, offers and thresholds all live in the config node. |
| `Route: Outreach or Summary1` | n8n-nodes-base.switch | Branches execution between individual outreach items and daily summaries. | `Score Churn Risk1` | `AI Personalise Message1`, `Build Trainer & Owner Digests1` | ## Schedule & Scoring<nA 7 AM schedule loads your settings, members, check-ins and past offers, then scores every active member on visit drops, plan expiry and lapses. Gym name, offers and thresholds all live in the config node. |
| `AI Personalise Message1` | @n8n/n8n-nodes-langchain.openAi | Generates customized win-back messaging and trainer talking points. | `Route: Outreach or Summary1` | `Compose Outreach1` | ## Win-back Outreach<nOpenAI writes a short WhatsApp offer in the member's language, WhatsApp sends it, ClickUp gives the trainer a call task, and the offer is logged so the same member isn't messaged again during the cooldown. |
| `Compose Outreach1` | n8n-nodes-base.code | Composes final WhatsApp message payloads and ClickUp task templates. | `AI Personalise Message1` | `WhatsApp: Send Offer1` | ## Win-back Outreach<nOpenAI writes a short WhatsApp offer in the member's language, WhatsApp sends it, ClickUp gives the trainer a call task, and the offer is logged so the same member isn't messaged again during the cooldown. |
| `WhatsApp: Send Offer1` | n8n-nodes-base.whatsApp | Sends outbound WhatsApp win-back offers to eligible members. | `Compose Outreach1` | `ClickUp: Trainer Follow-up1` | ## Win-back Outreach<nOpenAI writes a short WhatsApp offer in the member's language, WhatsApp sends it, ClickUp gives the trainer a call task, and the offer is logged so the same member isn't messaged again during the cooldown. |
| `ClickUp: Trainer Follow-up1` | n8n-nodes-base.clickUp | Creates follow-up call tasks assigned to trainers in ClickUp. | `WhatsApp: Send Offer1` | `Log Offer1` | ## Win-back Outreach<nOpenAI writes a short WhatsApp offer in the member's language, WhatsApp sends it, ClickUp gives the trainer a call task, and the offer is logged so the same member isn't messaged again during the cooldown. |
| `Log Offer1` | n8n-nodes-base.googleSheets | Appends sent offer metadata to the Google Sheets offer log. | `ClickUp: Trainer Follow-up1` | None | ## Win-back Outreach<nOpenAI writes a short WhatsApp offer in the member's language, WhatsApp sends it, ClickUp gives the trainer a call task, and the offer is logged so the same member isn't messaged again during the cooldown. |
| `Build Trainer & Owner Digests1` | n8n-nodes-base.code | Aggregates daily performance metrics for owners and trainers. | `Route: Outreach or Summary1` | `WhatsApp: Send Digests1` | ## Daily Digests<nBuilds one summary for the owner and one call list per trainer, sends both on WhatsApp, then saves the day's numbers to a sheet so you can track retention over time. |
| `WhatsApp: Send Digests1` | n8n-nodes-base.whatsApp | Sends digest summaries to trainers and gym owners over WhatsApp. | `Build Trainer & Owner Digests1` | `Append Retention Snapshot1` | ## Daily Digests<nBuilds one summary for the owner and one call list per trainer, sends both on WhatsApp, then saves the day's numbers to a sheet so you can track retention over time. |
| `Append Retention Snapshot1` | n8n-nodes-base.googleSheets | Appends daily aggregated retention statistics to Google Sheets. | `WhatsApp: Send Digests1` | None | ## Daily Digests<nBuilds one summary for the owner and one call list per trainer, sends both on WhatsApp, then saves the day's numbers to a sheet so you can track retention over time. |
| `Member Reply (WhatsApp)1` | n8n-nodes-base.whatsAppTrigger | Webhook trigger capturing incoming WhatsApp messages from members. | None | `Parse Reply1` | ## Reply Intake & Intent<nWhen a member replies, the workflow finds their record and latest offer, then OpenAI labels the message as accept, freeze, callback, stop or a question and drafts the answer. |
| `Parse Reply1` | n8n-nodes-base.code | Normalizes raw inbound webhook payloads into standard message structures. | `Member Reply (WhatsApp)1` | `Find Member1` | ## Reply Intake & Intent<nWhen a member replies, the workflow finds their record and latest offer, then OpenAI labels the message as accept, freeze, callback, stop or a question and drafts the answer. |
| `Find Member1` | n8n-nodes-base.googleSheets | Looks up member profiles in Google Sheets matching the sender phone. | `Parse Reply1` | `Find MemberOffers1` | ## Reply Intake & Intent<nWhen a member replies, the workflow finds their record and latest offer, then OpenAI labels the message as accept, freeze, callback, stop or a question and drafts the answer. |
| `Find Member Offers1` | n8n-nodes-base.googleSheets | Retrieves historical offer logs matching the sender phone. | `Find Member1` | `Build Reply Context1` | ## Reply Intake & Intent<nWhen a member replies, the workflow finds their record and latest offer, then OpenAI labels the message as accept, freeze, callback, stop or a question and drafts the answer. |
| `Build Reply Context1` | n8n-nodes-base.code | Compiles profile, offer context, and reply text for AI intent analysis. | `Find Member Offers1` | `AI Classify Intent1` | ## Reply Intake & Intent<nWhen a member replies, the workflow finds their record and latest offer, then OpenAI labels the message as accept, freeze, callback, stop or a question and drafts the answer. |
| `AI Classify Intent1` | @n8n/n8n-nodes-langchain.openAi | Classifies member intent, analyzes sentiment, and drafts an auto-reply. | `Build Reply Context1` | `Route by Intent1` | ## Reply Intake & Intent<nWhen a member replies, the workflow finds their record and latest offer, then OpenAI labels the message as accept, freeze, callback, stop or a question and drafts the answer. |
| `Route by Intent1` | n8n-nodes-base.switch | Routes execution paths based on classified member intent. | `AI Classify Intent1` | `Offer → Accepted1`, `Offer → Freeze Requested1`, `ClickUp: Trainer Callback1`, `Member → Opted Out1`, `WhatsApp: Reply to Member1` | ## Update Records & Tasks<nEach intent updates the right place: accepted and freeze requests mark the offer log, renewals and callbacks become ClickUp tasks, and opt-outs are flagged on the member so they stop getting messages. |
| `Offer → Accepted1` | n8n-nodes-base.googleSheets | Updates the offer log status to accepted in Google Sheets. | `Route by Intent1` | `ClickUp: Process Renewal1` | ## Update Records & Tasks<nEach intent updates the right place: accepted and freeze requests mark the offer log, renewals and callbacks become ClickUp tasks, and opt-outs are flagged on the member so they stop getting messages. |
| `ClickUp: Process Renewal1` | n8n-nodes-base.clickUp | Creates a priority renewal task in ClickUp. | `Offer → Accepted1` | `WhatsApp: Reply to Member1` | ## Update Records & Tasks<nEach intent updates the right place: accepted and freeze requests mark the offer log, renewals and callbacks become ClickUp tasks, and opt-outs are flagged on the member so they stop getting messages. |
| `Offer → Freeze Requested1` | n8n-nodes-base.googleSheets | Updates the offer log status to freeze requested in Google Sheets. | `Route by Intent1` | `WhatsApp: Reply to Member1` | ## Update Records & Tasks<nEach intent updates the right place: accepted and freeze requests mark the offer log, renewals and callbacks become ClickUp tasks, and opt-outs are flagged on the member so they stop getting messages. |
| `ClickUp: Trainer Callback1` | n8n-nodes-base.clickUp | Creates a follow-up callback task for trainers in ClickUp. | `Route by Intent1` | `WhatsApp: Reply to Member1` | ## Update Records & Tasks<nEach intent updates the right place: accepted and freeze requests mark the offer log, renewals and callbacks become ClickUp tasks, and opt-outs are flagged on the member so they stop getting messages. |
| `Member → Opted Out1` | n8n-nodes-base.googleSheets | Flags the member record as opted out in Google Sheets. | `Route by Intent1` | `WhatsApp: Reply to Member1` | ## Update Records & Tasks<nEach intent updates the right place: accepted and freeze requests mark the offer log, renewals and callbacks become ClickUp tasks, and opt-outs are flagged on the member so they stop getting messages. |
| `WhatsApp: Reply to Member1` | n8n-nodes-base.whatsApp | Sends the AI-drafted reply text back to the member on WhatsApp. | `Route by Intent1`, `ClickUp: Process Renewal1`, `Offer → Freeze Requested1`, `ClickUp: Trainer Callback1`, `Member → Opted Out1` | `Needs Staff?1` | ## Reply & Escalate<nSends the member the drafted answer. Renewals, freezes and callbacks also alert the assigned trainer, or the front desk if none is set, so a person follows up. |
| `Needs Staff?1` | n8n-nodes-base.if | Checks if member intent requires staff escalation. | `WhatsApp: Reply to Member1` | `WhatsApp: Notify Trainer / Desk1` | ## Reply & Escalate<nSends the member the drafted answer. Renewals, freezes and callbacks also alert the assigned trainer, or the front desk if none is set, so a person follows up. |
| `WhatsApp: Notify Trainer / Desk1` | n8n-nodes-base.whatsApp | Sends staff escalation alerts to trainers or front desk numbers. | `Needs Staff?1` | None | ## Reply & Escalate<nSends the member the drafted answer. Renewals, freezes and callbacks also alert the assigned trainer, or the front desk if none is set, so a person follows up. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Infrastructure:** Set up a Google Spreadsheet with four exact tabs: `Members`, `Checkins`, `Offer Log`, and `Retention Daily`. Ensure column headers match expected parameters (`member_id`, `status`, `phone`, `join_date`, `expiry_date`, `plan`, `plan_price`, `trainer_name`, `trainer_phone`, `language`, `opted_out`, `checkin_at`, `sent_at`, `offer_code`, `offer_title`).
2. **Configure Credentials:** Set up n8n credentials for Google Sheets (OAuth2), OpenAI (API Key), WhatsApp Business Cloud (Access Token & Phone Number ID), and ClickUp (API Token).
3. **Build the Daily Ingestion and Scoring Flow:**
   - Create a **Schedule Trigger** node (`Daily 7 AM1`) configured with cron expression `0 7 * * *`.
   - Add a **Code** node (`GymOS Config1`) and populate it with your gym configuration parameters (gym name, timezone, staff numbers, renewal link, thresholds, and offer definitions). Replace all placeholder numbers.
   - Add three sequential **Google Sheets** nodes (`Read Members1`, `Read Check-ins1`, `Read Offer Log1`) pointing to your spreadsheet ID and respective sheet names. Set `executeOnce` to true on all three.
   - Add a **Code** node (`Score Churn Risk1`) to ingest sheet records, calculate check-in drops, evaluate tenure/expiry, assign risk scores, and segment eligible members.
   - Add a **Switch** node (`Route: Outreach or Summary1`) to route items based on the `type` property (`outreach` or `summary`).
4. **Build the Win-Back Outreach Branch:**
   - Connect the outreach output to an **OpenAI** node (`AI Personalise Message1`) using model `gpt-4o-mini` with system prompts instructing message generation in the member's preferred language.
   - Attach a **Code** node (`Compose Outreach1`) to assemble formatted WhatsApp messages and ClickUp task bodies.
   - Add a **WhatsApp** node (`WhatsApp: Send Offer1`) set to `send` operation using parameter `REPLACE_WITH_WHATSAPP_PHONE_NUMBER_ID`, enabling `onError: continueRegularOutput`.
   - Add a **ClickUp** node (`ClickUp: Trainer Follow-up1`) configured with your ClickUp team, space, and list IDs.
   - Add a **Google Sheets** node (`Log Offer1`) set to `append` operation on the `Offer Log` sheet tab.
5. **Build the Daily Digests Branch:**
   - Connect the summary output of the switch node to a **Code** node (`Build Trainer & Owner Digests1`) to format trainer call lists and owner metric dashboards.
   - Add a **WhatsApp** node (`WhatsApp: Send Digests1`) to dispatch digests.
   - Add a **Google Sheets** node (`Append Retention Snapshot1`) set to `append` operation on the `Retention Daily` sheet tab.
6. **Build the Reply Intake and Intent Branch:**
   - Create a **WhatsApp Trigger** node (`Member Reply (WhatsApp)1`) listening to incoming messages.
   - Add a **Code** node (`Parse Reply1`) to extract text and sender phone numbers.
   - Add two **Google Sheets** lookup nodes (`Find Member1` and `Find Member Offers1`) querying rows by the sender's phone number.
   - Add a **Code** node (`Build Reply Context1`) to structure context for intent classification.
   - Attach an **OpenAI** node (`AI Classify Intent1`) using model `gpt-4o-mini` to categorize intents (`accept`, `freeze`, `callback`, `stop`, `question`).
   - Add a **Switch** node (`Route by Intent1`) with output branches for `Accept`, `Freeze`, `Callback`, `Stop`, and a fallback output for `Question / Other`.
7. **Build Intent Handling and Escalation Branches:**
   - Connect the `Accept` branch to a **Google Sheets** update node (`Offer → Accepted1`) updating status on matching `offer_code`, followed by a **ClickUp** node (`ClickUp: Process Renewal1`).
   - Connect the `Freeze` branch to a **Google Sheets** update node (`Offer → Freeze Requested1`).
   - Connect the `Callback` branch to a **ClickUp** node (`ClickUp: Trainer Callback1`).
   - Connect the `Stop` branch to a **Google Sheets** update node (`Member → Opted Out1`).
   - Route all completed intent branches and the fallback question branch into a common **WhatsApp** node (`WhatsApp: Reply to Member1`) to dispatch automated responses.
   - Attach an **IF** node (`Needs Staff?1`) evaluating whether intent is in `['accept', 'freeze', 'callback']`.
   - If true, route to a final **WhatsApp** node (`WhatsApp: Notify Trainer / Desk1`) targeting assigned trainer phones or front desk fallback numbers.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Replace all instance configuration placeholders | Ensure all occurrences of `REPLACE_WITH_GOOGLE_SHEET_ID`, `REPLACE_WITH_WHATSAPP_PHONE_NUMBER_ID`, `REPLACE_WITH_CLICKUP_TEAM_ID`, `REPLACE_WITH_CLICKUP_SPACE_ID`, and `REPLACE_WITH_CLICKUP_LIST_ID` are updated with valid integration parameters before activation. |
| Security and privacy guideline | Keep real member phone numbers and private credentials out of publicly shared workflow templates or repository copies. |
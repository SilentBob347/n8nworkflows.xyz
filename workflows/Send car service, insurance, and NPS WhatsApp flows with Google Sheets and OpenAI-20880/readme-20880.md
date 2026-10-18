Send car service, insurance, and NPS WhatsApp flows with Google Sheets and OpenAI

https://n8nworkflows.xyz/workflows/send-car-service--insurance--and-nps-whatsapp-flows-with-google-sheets-and-openai-20880


# Send car service, insurance, and NPS WhatsApp flows with Google Sheets and OpenAI

### 1. Workflow Overview

This workflow automates post-sale customer communications and operational tracking for car dealerships. It coordinates daily vehicle service reminders, insurance renewals, test-drive follow-ups via OpenAI, closed job card synchronization with a Dealer Management System (DMS), and post-service Net Promoter Score (NPS) collection and triage. 

The logic is organized into the following functional blocks:
- **1.1 Schedule & Configuration Setup:** Initializes the daily execution at 9:00 AM IST and injects dealership configurations (intervals, links, thresholds).
- **1.2 Data Ingestion & Queue Building:** Fetches vehicle, test-drive lead, and reminder log records from Google Sheets, computing due statuses and applying a daily cap of 150 messages.
- **1.3 Reminder Routing & Execution:** Directs queued actions to handle service reminders, insurance expiration notifications (with urgent desk escalation), test-drive follow-up generation via OpenAI, and manager daily digests.
- **1.4 DMS Job Card Integration:** Receives webhook payloads from the DMS when a job card is closed, updates the vehicle’s service history, and schedules an NPS survey invite 3 hours post-delivery.
- **1.5 NPS Feedback Processing & Recovery:** Captures feedback submitted through an n8n form, analyzes sentiment and root issues using OpenAI, classifies scores into bands (Detractor, Passive, Promoter), routes alerts to staff, and logs outcomes.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Schedule & Configuration Setup
**Overview:** Triggers the daily scan at 9:00 AM IST and sets global dealership parameters, service intervals, and notification thresholds.

**Nodes Involved:**
- `Every Day at 9 AM IST`
- `Set AutoOS Config`

**Node Details:**
- **Every Day at 9 AM IST**
  - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Cron execution trigger)
  - *Configuration Choices:* Configured with a cron expression (`0 9 * * *`) targeting the `Asia/Kolkata` timezone.
  - *Input/Output:* No inputs; outputs execution timestamp to `Set AutoOS Config`.
  - *Edge Cases/Failure Types:* Clock drift or server timezone misconfigurations could cause execution at unintended hours.

- **Set AutoOS Config**
  - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution node)
  - *Configuration Choices:* Establishes global parameters such as `dealer_name`, service intervals for diesel/EV models, overdue thresholds (`due_soon_km: 750`, `overdue_days: 30`), and max daily limits (`max_reminders_per_run: 150`).
  - *Key Expressions/Variables:* Returns a static array containing configuration objects and placeholder strings (`REPLACE_WITH_SERVICE_MANAGER_NUMBER`).
  - *Input/Output:* Input from `Every Day at 9 AM IST`; output to `Read Vehicles Sheet`.
  - *Edge Cases/Failure Types:* Syntax errors in JavaScript return objects can halt execution.

---

#### 2.2 Data Ingestion & Queue Building
**Overview:** Pulls data from multiple Google Sheets tabs, checks reminder histories to prevent duplicate sends, and compiles a capped daily queue.

**Nodes Involved:**
- `Read Vehicles Sheet`
- `Read Test-Drive Leads`
- `Read Reminder Log`
- `Build Reminder Queue`

**Node Details:**
- **Read Vehicles Sheet**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data retrieval)
  - *Configuration Choices:* Reads rows from the `Vehicles` sheet using a document ID placeholder.
  - *Input/Output:* Input from `Set AutoOS Config`; output to `Read Test-Drive Leads`.
  - *Edge Cases/Failure Types:* API rate limits or revoked Google Sheets credentials.

- **Read Test-Drive Leads**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data retrieval)
  - *Configuration Choices:* Reads rows from the `Test Drives` sheet with `alwaysOutputData` enabled.
  - *Input/Output:* Input from `Read Vehicles Sheet`; output to `Read Reminder Log`.
  - *Edge Cases/Failure Types:* Missing columns in the sheet can result in undefined property errors during downstream processing.

- **Read Reminder Log**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data retrieval)
  - *Configuration Choices:* Reads rows from the `Reminder Log` sheet to build a set of already sent reminder keys.
  - *Input/Output:* Input from `Read Test-Drive Leads`; output to `Build Reminder Queue`.
  - *Edge Cases/Failure Types:* Large logs may impact performance; pagination limits.

- **Build Reminder Queue**
  - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript data processor)
  - *Configuration Choices:* Evaluates vehicle mileage/time intervals, checks test drive follow-up schedules against configured days (`[1, 3, 7, 14]`), filters out previously sent keys, and caps the output at 150 items while appending a summary item.
  - *Key Expressions/Variables:* Accesses upstream node data via `$('Read Vehicles Sheet')`, `$('Read Test-Drive Leads')`, and `$('Read Reminder Log')`.
  - *Input/Output:* Input from `Read Reminder Log`; output to `Route by Reminder Type`.
  - *Edge Cases/Failure Types:* Invalid date formats in source sheets can cause parsing failures and drop records.

---

#### 2.3 Reminder Routing & Execution
**Overview:** Distributes queue items to specific communication channels (WhatsApp, AI generation, or manager alerts) based on reminder types.

**Nodes Involved:**
- `Route by Reminder Type`
- `Send Service Reminder`
- `Log Service Reminder`
- `Send Insurance Reminder`
- `Urgent Insurance?`
- `Alert Insurance Desk`
- `Log Insurance Reminder`
- `AI Write Test-Drive Follow-up`
- `Send Test-Drive Follow-up`
- `Update Test-Drive Lead`
- `Lead Gone Cold?`
- `Alert Salesperson on Cold Lead`
- `Send Daily Summary to Manager`

**Node Details:**
- **Route by Reminder Type**
  - *Type and Technical Role:* `n8n-nodes-base.switch` (Conditional router)
  - *Configuration Choices:* Evaluates `{{ $json.type }}` against four outputs: `service`, `insurance`, `test_drive`, and `summary`.
  - *Input/Output:* Input from `Build Reminder Queue`; outputs to respective execution branches.

- **Send Service Reminder**
  - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Messaging)
  - *Configuration Choices:* Sends WhatsApp text messages using `REPLACE_WITH_WHATSAPP_PHONE_NUMBER_ID`. `onError` set to `continueRegularOutput`.
  - *Input/Output:* Input from `Route by Reminder Type` (Service); output to `Log Service Reminder`.
  - *Edge Cases/Failure Types:* Invalid phone numbers or WhatsApp Cloud API outages.

- **Log Service Reminder**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data logging)
  - *Configuration Choices:* Appends a row to the `Reminder Log` sheet detailing delivery status (`sent` or `failed`).
  - *Input/Output:* Input from `Send Service Reminder`.

- **Send Insurance Reminder**
  - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Messaging)
  - *Configuration Choices:* Sends insurance renewal notifications. `onError` set to `continueRegularOutput`.
  - *Input/Output:* Input from `Route by Reminder Type` (Insurance); output to `Urgent Insurance?`.

- **Urgent Insurance?**
  - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional evaluation)
  - *Configuration Choices:* Checks if `{{ $('Route by Reminder Type').item.json.urgent }}` is true.
  - *Input/Output:* Input from `Send Insurance Reminder`; outputs to `Alert Insurance Desk` or `Log Insurance Reminder`.

- **Alert Insurance Desk**
  - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Messaging)
  - *Configuration Choices:* Forwards urgent insurance notices to the internal insurance desk number.
  - *Input/Output:* Input from `Urgent Insurance?` (True); output to `Log Insurance Reminder`.

- **Log Insurance Reminder**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data logging)
  - *Configuration Choices:* Appends insurance reminder execution details to `Reminder Log`.
  - *Input/Output:* Inputs from `Urgent Insurance?` (False) and `Alert Insurance Desk`.

- **AI Write Test-Drive Follow-up**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (AI text generation)
  - *Configuration Choices:* Uses `gpt-4o-mini` with a temperature of `0.6` to generate personalized follow-up messages and salesperson tips in strict JSON format.
  - *Input/Output:* Input from `Route by Reminder Type` (Test Drive); output to `Send Test-Drive Follow-up`.
  - *Edge Cases/Failure Types:* OpenAI API rate limits, invalid JSON responses from the model.

- **Send Test-Drive Follow-up**
  - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Messaging)
  - *Configuration Choices:* Sends the AI-generated message content to the lead.
  - *Input/Output:* Input from `AI Write Test-Drive Follow-up`; output to `Update Test-Drive Lead`.

- **Update Test-Drive Lead**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data update)
  - *Configuration Choices:* Updates lead status, incremented follow-up count, and timestamp in the `Test Drives` sheet matching on `lead_id`.
  - *Input/Output:* Input from `Send Test-Drive Follow-up`; output to `Lead Gone Cold?`.

- **Lead Gone Cold?**
  - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional evaluation)
  - *Configuration Choices:* Checks if `{{ $('Route by Reminder Type').item.json.cold }}` evaluates to true.
  - *Input/Output:* Input from `Update Test-Drive Lead`; output to `Alert Salesperson on Cold Lead`.

- **Alert Salesperson on Cold Lead**
  - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Messaging)
  - *Configuration Choices:* Sends an alert to the assigned salesperson (or service manager fallback) with sales tips.
  - *Input/Output:* Input from `Lead Gone Cold?` (True).

- **Send Daily Summary to Manager**
  - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Messaging)
  - *Configuration Choices:* Sends aggregate metrics of the run to the service manager's WhatsApp.
  - *Input/Output:* Input from `Route by Reminder Type` (Summary).

---

#### 2.4 DMS Job Card Integration
**Overview:** Listens for closed job card webhooks from the DMS, normalizes payload data, updates vehicle records, and schedules an NPS survey invite.

**Nodes Involved:**
- `On Job Card Closed (DMS)`
- `Normalize Job Card`
- `Upsert Service Job`
- `Look Up Vehicle`
- `Update Vehicle Service Record`
- `Wait 3 Hours After Delivery`
- `Send NPS Survey Invite`

**Node Details:**
- **On Job Card Closed (DMS)**
  - *Type and Technical Role:* `n8n-nodes-base.webhook` (Webhook trigger)
  - *Configuration Choices:* Listens for incoming `POST` requests on path `autoos-job-closed`.
  - *Input/Output:* External trigger; output to `Normalize Job Card`.
  - *Edge Cases/Failure Types:* Unauthenticated payloads or malformed JSON structures.

- **Normalize Job Card**
  - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript data transformer)
  - *Configuration Choices:* Standardizes field names from various DMS structures and generates an NPS survey link incorporating the form URL and job card ID.
  - *Input/Output:* Input from DMS webhook; output to `Upsert Service Job`.

- **Upsert Service Job**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data upsert)
  - *Configuration Choices:* Appends or updates service job records in the `Service Jobs` sheet matched by `job_card`.
  - *Input/Output:* Input from `Normalize Job Card`; output to `Look Up Vehicle`.

- **Look Up Vehicle**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data retrieval)
  - *Configuration Choices:* Searches the `Vehicles` sheet for a record matching `vehicle_id`. `alwaysOutputData` enabled.
  - *Input/Output:* Input from `Upsert Service Job`; output to `Update Vehicle Service Record`.

- **Update Vehicle Service Record**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data update)
  - *Configuration Choices:* Increments `services_done` and updates `last_service_km` and `last_service_date`.
  - *Input/Output:* Input from `Look Up Vehicle`; output to `Wait 3 Hours After Delivery`.

- **Wait 3 Hours After Delivery**
  - *Type and Technical Role:* `n8n-nodes-base.wait` (Execution pause)
  - *Configuration Choices:* Pauses workflow execution for 3 hours.
  - *Input/Output:* Input from `Update Vehicle Service Record`; output to `Send NPS Survey Invite`.
  - *Edge Cases/Failure Types:* Workflow restarts or instance reboots during the wait period may clear active wait states depending on n8n persistence settings.

- **Send NPS Survey Invite**
  - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Messaging)
  - *Configuration Choices:* Sends the NPS survey link to the customer via WhatsApp.
  - *Input/Output:* Input from `Wait 3 Hours After Delivery`.

---

#### 2.5 NPS Feedback Processing & Recovery
**Overview:** Captures form submissions, analyzes feedback using OpenAI, routes responses based on NPS bands, notifies staff, and logs results.

**Nodes Involved:**
- `Post-Service NPS Form`
- `Look Up Service Job`
- `AI Analyze Feedback`
- `Classify NPS Band`
- `Route by NPS Band`
- `Email Service Manager (Detractor)`
- `Alert Service Manager on WhatsApp`
- `Send Apology & Callback Note`
- `Thank Passive Customer`
- `Ask Promoter for Google Review`
- `Update Job NPS in Sheet`
- `Append to NPS Log`

**Node Details:**
- **Post-Service NPS Form**
  - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (Form input trigger)
  - *Configuration Choices:* Hosted form collecting recommendation score (0–10), reason, and callback preference, with a hidden `job_card` field.
  - *Input/Output:* External form submission; output to `Look Up Service Job`.

- **Look Up Service Job**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data retrieval)
  - *Configuration Choices:* Queries `Service Jobs` using `job_card` from form parameters.
  - *Input/Output:* Input from `Post-Service NPS Form`; output to `AI Analyze Feedback`.

- **AI Analyze Feedback**
  - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (AI analysis)
  - *Configuration Choices:* Uses `gpt-4o-mini` with temperature `0.2` to evaluate sentiment, categorize issues, determine urgency, and draft apology messages in JSON.
  - *Input/Output:* Input from `Look Up Service Job`; output to `Classify NPS Band`.

- **Classify NPS Band**
  - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript data processor)
  - *Configuration Choices:* Combines form data, job details, and AI outputs, classifying scores into `detractor` (0–6), `passive` (7–8), or `promoter` (9–10).
  - *Input/Output:* Input from `AI Analyze Feedback`; output to `Route by NPS Band`.

- **Route by NPS Band**
  - *Type and Technical Role:* `n8n-nodes-base.switch` (Conditional router)
  - *Configuration Choices:* Routes execution based on `{{ $json.band }}` value.
  - *Input/Output:* Input from `Classify NPS Band`; outputs to Detractor, Passive, and Promoter handling branches.

- **Email Service Manager (Detractor)**
  - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email notification)
  - *Configuration Choices:* Sends a detailed HTML email summary to the service manager for detractor scores.
  - *Input/Output:* Input from `Route by NPS Band` (Detractor); output to `Alert Service Manager on WhatsApp`.

- **Alert Service Manager on WhatsApp**
  - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Messaging)
  - *Configuration Choices:* Sends a high-priority WhatsApp alert to the service manager.
  - *Input/Output:* Input from `Email Service Manager (Detractor)`; output to `Send Apology & Callback Note`.

- **Send Apology & Callback Note**
  - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Messaging)
  - *Configuration Choices:* Sends an AI-generated or default apology message to the detractor customer.
  - *Input/Output:* Input from `Alert Service Manager on WhatsApp`; output to `Update Job NPS in Sheet`.

- **Thank Passive Customer**
  - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Messaging)
  - *Configuration Choices:* Sends a follow-up prompt to passive customers asking how to improve.
  - *Input/Output:* Input from `Route by NPS Band` (Passive); output to `Update Job NPS in Sheet`.

- **Ask Promoter for Google Review**
  - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Messaging)
  - *Configuration Choices:* Sends a Google review request link to promoter customers.
  - *Input/Output:* Input from `Route by NPS Band` (Promoter); output to `Update Job NPS in Sheet`.

- **Update Job NPS in Sheet**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data update)
  - *Configuration Choices:* Updates `Service Jobs` sheet with NPS scores, bands, and status (`escalated` for detractors, `closed` for others) matching on `job_card`.
  - *Input/Output:* Inputs from `Send Apology & Callback Note`, `Thank Passive Customer`, and `Ask Promoter for Google Review`; output to `Append to NPS Log`.

- **Append to NPS Log**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data logging)
  - *Configuration Choices:* Appends a complete feedback entry to the `NPS Log` sheet.
  - *Input/Output:* Input from `Update Job NPS in Sheet`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note – Overview | n8n-nodes-base.stickyNote | Documentation note | None | None | ## 🚘 AutoOS – Car Dealer Reminders & Service NPS... |
| Sticky Note – Load Data & Build Queue | n8n-nodes-base.stickyNote | Documentation note | None | None | ## 📥 Load Data & Build Queue... |
| Sticky Note – Service & Insurance Reminders | n8n-nodes-base.stickyNote | Documentation note | None | None | ## 🔧 Service & Insurance Reminders... |
| Sticky Note – Test-Drive Follow-ups & Summary | n8n-nodes-base.stickyNote | Documentation note | None | None | ## 🚗 Test-Drive Follow-ups & Summary... |
| Sticky Note – Job Card Sync & NPS Invite | n8n-nodes-base.stickyNote | Documentation note | None | None | ## 🧾 Job Card Sync & NPS Invite... |
| Sticky Note – NPS Response & AI Analysis | n8n-nodes-base.stickyNote | Documentation note | None | None | ## 🧠 NPS Response & AI Analysis... |
| Sticky Note – Recovery & Logging | n8n-nodes-base.stickyNote | Documentation note | None | None | ## 🤝 Recovery & Logging... |
| Sticky Note – Credentials & Security | n8n-nodes-base.stickyNote | Documentation note | None | None | ## 🔐 Credentials & Security... |
| Every Day at 9 AM IST | n8n-nodes-base.scheduleTrigger | Triggers workflow daily at 9 AM IST | None | Set AutoOS Config | |
| Set AutoOS Config | n8n-nodes-base.code | Sets global dealership configurations | Every Day at 9 AM IST | Read Vehicles Sheet | |
| Read Vehicles Sheet | n8n-nodes-base.googleSheets | Loads vehicle data | Set AutoOS Config | Read Test-Drive Leads | |
| Read Test-Drive Leads | n8n-nodes-base.googleSheets | Loads test-drive lead data | Read Vehicles Sheet | Read Reminder Log | |
| Read Reminder Log | n8n-nodes-base.googleSheets | Loads past reminder logs | Read Test-Drive Leads | Build Reminder Queue | |
| Build Reminder Queue | n8n-nodes-base.code | Evaluates due dates and builds queue | Read Reminder Log | Route by Reminder Type | |
| Route by Reminder Type | n8n-nodes-base.switch | Routes items by reminder type | Build Reminder Queue | Send Service Reminder, Send Insurance Reminder, AI Write Test-Drive Follow-up, Send Daily Summary to Manager | |
| Send Service Reminder | n8n-nodes-base.whatsApp | Sends service reminder via WhatsApp | Route by Reminder Type | Log Service Reminder | |
| Send Insurance Reminder | n8n-nodes-base.whatsApp | Sends insurance reminder via WhatsApp | Route by Reminder Type | Urgent Insurance? | |
| AI Write Test-Drive Follow-up | @n8n/n8n-nodes-langchain.openAi | Generates AI follow-up message | Route by Reminder Type | Send Test-Drive Follow-up | |
| Send Daily Summary to Manager | n8n-nodes-base.whatsApp | Sends daily metrics summary to manager | Route by Reminder Type | None | |
| Urgent Insurance? | n8n-nodes-base.if | Checks if insurance case is urgent | Send Insurance Reminder | Alert Insurance Desk, Log Insurance Reminder | |
| Send Test-Drive Follow-up | n8n-nodes-base.whatsApp | Sends AI test-drive follow-up | AI Write Test-Drive Follow-up | Update Test-Drive Lead | |
| Alert Insurance Desk | n8n-nodes-base.whatsApp | Alerts insurance desk on urgent renewals | Urgent Insurance? | Log Insurance Reminder | |
| Update Test-Drive Lead | n8n-nodes-base.googleSheets | Updates lead status in sheet | Send Test-Drive Follow-up | Lead Gone Cold? | |
| Log Service Reminder | n8n-nodes-base.googleSheets | Logs sent service reminder | Send Service Reminder | None | |
| Log Insurance Reminder | n8n-nodes-base.googleSheets | Logs sent insurance reminder | Urgent Insurance?, Alert Insurance Desk | None | |
| Lead Gone Cold? | n8n-nodes-base.if | Checks if test drive lead is cold | Update Test-Drive Lead | Alert Salesperson on Cold Lead | |
| Alert Salesperson on Cold Lead | n8n-nodes-base.whatsApp | Alerts salesperson on cold lead | Lead Gone Cold? | None | |
| On Job Card Closed (DMS) | n8n-nodes-base.webhook | Receives closed job card webhooks | None | Normalize Job Card | |
| Normalize Job Card | n8n-nodes-base.code | Normalizes DMS job card payload | On Job Card Closed (DMS) | Upsert Service Job | |
| Upsert Service Job | n8n-nodes-base.googleSheets | Upserts service job record | Normalize Job Card | Look Up Vehicle | |
| Look Up Vehicle | n8n-nodes-base.googleSheets | Looks up vehicle details | Upsert Service Job | Update Vehicle Service Record | |
| Update Vehicle Service Record | n8n-nodes-base.googleSheets | Updates vehicle service history | Look Up Vehicle | Wait 3 Hours After Delivery | |
| Wait 3 Hours After Delivery | n8n-nodes-base.wait | Waits 3 hours before NPS invite | Update Vehicle Service Record | Send NPS Survey Invite | |
| Send NPS Survey Invite | n8n-nodes-base.whatsApp | Sends NPS survey link via WhatsApp | Wait 3 Hours After Delivery | None | |
| Post-Service NPS Form | n8n-nodes-base.formTrigger | Receives NPS form submissions | None | Look Up Service Job | |
| Look Up Service Job | n8n-nodes-base.googleSheets | Looks up job details from form input | Post-Service NPS Form | AI Analyze Feedback | |
| AI Analyze Feedback | @n8n/n8n-nodes-langchain.openAi | Analyzes feedback sentiment and issues | Look Up Service Job | Classify NPS Band | |
| Classify NPS Band | n8n-nodes-base.code | Bandifies NPS score | AI Analyze Feedback | Route by NPS Band | |
| Route by NPS Band | n8n-nodes-base.switch | Routes feedback by NPS band | Classify NPS Band | Email Service Manager (Detractor), Thank Passive Customer, Ask Promoter for Google Review | |
| Email Service Manager (Detractor) | n8n-nodes-base.gmail | Emails service manager for detractors | Route by NPS Band | Alert Service Manager on WhatsApp | |
| Alert Service Manager on WhatsApp | n8n-nodes-base.whatsApp | Sends WhatsApp alert for detractors | Email Service Manager (Detractor) | Send Apology & Callback Note | |
| Thank Passive Customer | n8n-nodes-base.whatsApp | Thanks passive customers | Route by NPS Band | Update Job NPS in Sheet | |
| Ask Promoter for Google Review | n8n-nodes-base.whatsApp | Asks promoters for Google review | Route by NPS Band | Update Job NPS in Sheet | |
| Send Apology & Callback Note | n8n-nodes-base.whatsApp | Sends apology message to detractors | Alert Service Manager on WhatsApp | Update Job NPS in Sheet | |
| Update Job NPS in Sheet | n8n-nodes-base.googleSheets | Updates job NPS status in sheet | Send Apology & Callback Note, Thank Passive Customer, Ask Promoter for Google Review | Append to NPS Log | |
| Append to NPS Log | n8n-nodes-base.googleSheets | Appends final feedback to NPS log | Update Job NPS in Sheet | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these numbered steps to recreate the workflow manually in n8n:

1. **Create Base Structure & Credentials:**
   - Prepare a Google Sheet containing tabs named: `Vehicles`, `Test Drives`, `Reminder Log`, `Service Jobs`, and `NPS Log`.
   - Set up active credentials in n8n for **Google Sheets**, **WhatsApp Business Cloud**, **OpenAI**, and **Gmail**.

2. **Set Up the Daily Trigger & Configuration:**
   - Add a **Schedule Trigger** node (`Every Day at 9 AM IST`), set to cron expression `0 9 * * *` in timezone `Asia/Kolkata`.
   - Add a **Code** node (`Set AutoOS Config`), connect it downstream from the Schedule Trigger, and paste the configuration JavaScript (defining dealership name, phone numbers, intervals, and thresholds).

3. **Build the Ingestion and Queue Logic:**
   - Add three **Google Sheets** nodes sequentially (`Read Vehicles Sheet`, `Read Test-Drive Leads`, `Read Reminder Log`), targeting the respective sheet names and replacing sheet ID placeholders.
   - Add a **Code** node (`Build Reminder Queue`) connected after the reminder log read node to evaluate due dates, check logs, and generate the daily array.

4. **Configure Routing and Reminder Execution:**
   - Add a **Switch** node (`Route by Reminder Type`) with four output rules targeting `service`, `insurance`, `test_drive`, and `summary`.
   - **Service Branch:** Connect output 0 to a **WhatsApp** node (`Send Service Reminder`) and then to a **Google Sheets** append node (`Log Service Reminder`).
   - **Insurance Branch:** Connect output 1 to a **WhatsApp** node (`Send Insurance Reminder`), followed by an **If** node (`Urgent Insurance?`). Connect true output to a **WhatsApp** node (`Alert Insurance Desk`), and feed both paths into a **Google Sheets** append node (`Log Insurance Reminder`).
   - **Test Drive Branch:** Connect output 2 to an **OpenAI** node (`AI Write Test-Drive Follow-up` using `gpt-4o-mini`), then to a **WhatsApp** node (`Send Test-Drive Follow-up`), a **Google Sheets** update node (`Update Test-Drive Lead`), an **If** node (`Lead Gone Cold?`), and a **WhatsApp** node (`Alert Salesperson on Cold Lead`).
   - **Summary Branch:** Connect output 3 to a **WhatsApp** node (`Send Daily Summary to Manager`).

5. **Configure DMS Webhook & Job Card Sync:**
   - Add a **Webhook** node (`On Job Card Closed (DMS)`) configured for `POST` requests on path `autoos-job-closed`.
   - Add a **Code** node (`Normalize Job Card`), followed by a **Google Sheets** upsert node (`Upsert Service Job`), a **Google Sheets** lookup node (`Look Up Vehicle`), a **Google Sheets** update node (`Update Vehicle Service Record`), a **Wait** node (`Wait 3 Hours After Delivery` set to `3` hours), and a **WhatsApp** node (`Send NPS Survey Invite`).

6. **Configure NPS Form & Feedback Processing:**
   - Add a **Form Trigger** node (`Post-Service NPS Form`) with path `autoos-nps`, collecting recommendation score, reason, and callback preference.
   - Connect it to a **Google Sheets** lookup node (`Look Up Service Job`), an **OpenAI** node (`AI Analyze Feedback`), a **Code** node (`Classify NPS Band`), and a **Switch** node (`Route by NPS Band`).
   - **Detractor Branch:** Connect to **Gmail** (`Email Service Manager (Detractor)`), **WhatsApp** (`Alert Service Manager on WhatsApp`), and **WhatsApp** (`Send Apology & Callback Note`).
   - **Passive Branch:** Connect to **WhatsApp** (`Thank Passive Customer`).
   - **Promoter Branch:** Connect to **WhatsApp** (`Ask Promoter for Google Review`).
   - Converge all three feedback branches into a **Google Sheets** update node (`Update Job NPS in Sheet`), followed finally by a **Google Sheets** append node (`Append to NPS Log`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Sensitive customer data protection | Vehicle and job sheets hold customer PII; ensure Google Sheets permissions are strictly controlled and DMS webhooks are secured with authentication headers. |
| WhatsApp opt-in compliance | Only message customers who have explicitly opted in to receive automated updates via WhatsApp. |
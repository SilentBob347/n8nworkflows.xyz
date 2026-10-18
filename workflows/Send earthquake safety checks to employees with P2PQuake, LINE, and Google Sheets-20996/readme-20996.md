Send earthquake safety checks to employees with P2PQuake, LINE, and Google Sheets

https://n8nworkflows.xyz/workflows/send-earthquake-safety-checks-to-employees-with-p2pquake--line--and-google-sheets-20996


# Send earthquake safety checks to employees with P2PQuake, LINE, and Google Sheets

### 1. Workflow Overview

This workflow provides an automated earthquake safety check and response tracking system for organizations. It integrates the P2PQuake seismic API, LINE Messaging API for employee communication, and Google Sheets for data storage and state management. 

The application logic is divided into three primary functional blocks:
- **1.1 Earthquake Detection and Notification:** Periodically polls the P2PQuake API, compares the latest seismic event against stored control records to prevent duplicate triggers, checks the employee directory against affected prefectures (seismic intensity 4 or higher), and dispatches automated LINE safety check messages.
- **1.2 Safety Response Tracking:** Captures incoming webhook replies from employees (1 for Safe, 2 for Needs Help, 3 for Other) and updates their respective safety statuses and response timestamps in Google Sheets.
- **1.3 Employee Registration:** Handles new contact parsing from inbound LINE messages (format: `Name, Home Prefecture, Work Prefecture`) and automatically appends new records to the employee directory.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Earthquake Detection and Notification
- **Overview:** This block handles the automated polling schedule, queries the external earthquake tracking API, evaluates whether a seismic event is novel, filters impacted employees based on geographic criteria, sends out broadcast alerts via LINE, and updates the control ledger.
- **Nodes Involved:** 
  - `Earthquake Check Schedule`
  - `Fetch Earthquake Information`
  - `Get Latest Earthquake`
  - `Get Last Earthquake ID`
  - `Is New Earthquake?`
  - `Get Employees`
  - `Is Employee in Affected Area?`
  - `Send Safety Check to LINE`
  - `Update Last Earthquake ID`

- **Node Details:**
  - **Earthquake Check Schedule**
    - *Type & Technical Role:* Schedule Trigger (`n8n-nodes-base.scheduleTrigger`) initiating the polling cycle.
    - *Configuration Choices:* Interval set to every 30 minutes.
    - *Expressions:* None.
    - *Connections:* Input: None (Trigger); Output: `Fetch Earthquake Information`.
    - *Failure Modes:* Missed executions if the n8n instance is offline.
  - **Fetch Earthquake Information**
    - *Type & Technical Role:* HTTP Request (`n8n-nodes-base.httpRequest`) pulling data from the P2PQuake API.
    - *Configuration Choices:* GET method targeting `https://api.p2pquake.net/v2/jma/quake` with query parameters `order = -1` (descending) and `min_scale = 40` (intensity 4+).
    - *Expressions:* None.
    - *Connections:* Input: `Earthquake Check Schedule`; Output: `Get Latest Earthquake`.
    - *Failure Modes:* API downtime, network timeout, or schema changes from the provider.
  - **Get Latest Earthquake**
    - *Type & Technical Role:* Limit Node (`n8n-nodes-base.limit`) to isolate the most recent seismic event.
    - *Configuration Choices:* Defaults to returning the first item.
    - *Expressions:* None.
    - *Connections:* Input: `Fetch Earthquake Information`; Output: `Get Last Earthquake ID`.
    - *Failure Modes:* Empty payload arrays if the API returns no items.
  - **Get Last Earthquake ID**
    - *Type & Technical Role:* Google Sheets Node (`n8n-nodes-base.googleSheets`) fetching the control state.
    - *Configuration Choices:* Operation: Get/Read (default list operation on spreadsheet tab `Earthquake Control`, gid=0).
    - *Expressions:* None.
    - *Connections:* Input: `Get Latest Earthquake`; Output: `Is New Earthquake?`.
    - *Failure Modes:* Google API authentication expiration, missing sheet, or API rate limits.
  - **Is New Earthquake?**
    - *Type & Technical Role:* If Node (`n8n-nodes-base.if`) for conditional branching.
    - *Configuration Choices:* Evaluates string inequality.
    - *Expressions:* `={{ $('Get Latest Earthquake').first().json.id }}` not equals `={{ $json["Last Earthquake ID"] }}`.
    - *Connections:* Input: `Get Last Earthquake ID`; Output (true branch): `Get Employees`.
    - *Failure Modes:* Expression evaluation errors if upstream nodes return null objects.
  - **Get Employees**
    - *Type & Technical Role:* Google Sheets Node (`n8n-nodes-base.googleSheets`) retrieving directory data.
    - *Configuration Choices:* Reads the `Employee Directory` sheet tab.
    - *Expressions:* None.
    - *Connections:* Input: `Is New Earthquake?` (True); Output: `Is Employee in Affected Area?`.
    - *Failure Modes:* Auth revocation or sheet name alteration.
  - **Is Employee in Affected Area?**
    - *Type & Technical Role:* If Node (`n8n-nodes-base.if`) verifying geographic exposure.
    - *Configuration Choices:* JavaScript array evaluation using boolean logic.
    - *Expressions:* `={{ $('Get Latest Earthquake').first().json.points.some(p => p.scale >= 40 && (p.pref === $json['Home Prefecture'] || p.pref === $json['Work Prefecture'])) }}`.
    - *Connections:* Input: `Get Employees`; Output (true branch): `Send Safety Check to LINE`.
    - *Failure Modes:* Data type mismatches between API prefecture strings and spreadsheet entries.
  - **Send Safety Check to LINE**
    - *Type & Technical Role:* HTTP Request (`n8n-nodes-base.httpRequest`) dispatching a push message via the LINE API.
    - *Configuration Choices:* POST method to `https://api.line.me/v2/bot/message/push` using generic HTTP Header Authentication.
    - *Expressions:* Constructs JSON payload mapping `to` to `{{$json['LINE User ID']}}` and embeds localized check text.
    - *Connections:* Input: `Is Employee in Affected Area?`; Output: `Update Last Earthquake ID`.
    - *Failure Modes:* Invalid LINE User IDs, expired channel access tokens, or rate throttling by LINE.
  - **Update Last Earthquake ID**
    - *Type & Technical Role:* Google Sheets Node (`n8n-nodes-base.googleSheets`) committing the latest state.
    - *Configuration Choices:* Operation: Update row 2 of the `Earthquake Control` sheet.
    - *Expressions:* 
      - `Last Earthquake ID`: `={{ $('Get Latest Earthquake').first().json.id }}`
      - `Last Checked At`: `={{ $now.toFormat('yyyy-LL-dd HH:mm') }}`
    - *Connections:* Input: `Send Safety Check to LINE`; Output: None (Terminal node for this branch).
    - *Failure Modes:* Row locking or schema mismatch on the control sheet.

---

#### Block 1.2: Safety Response Tracking
- **Overview:** Listens for inbound webhook events from the LINE Messaging API, evaluates message payloads to isolate response codes (1, 2, or 3), and updates the corresponding employee's safety status and timestamp in Google Sheets.
- **Nodes Involved:**
  - `Receive LINE Event`
  - `Route LINE Message`
  - `Update Status - Safe`
  - `Update Status - Needs Help`
  - `Update Status - Other`

- **Node Details:**
  - **Receive LINE Event**
    - *Type & Technical Role:* Webhook Trigger (`n8n-nodes-base.webhook`) capturing incoming HTTP POST requests from LINE.
    - *Configuration Choices:* HTTP Method: POST, Path set to a unique webhook identifier.
    - *Expressions:* None.
    - *Connections:* Input: None (Trigger); Output: `Route LINE Message`.
    - *Failure Modes:* Unreachable webhook URL, misconfigured LINE Developer Console settings, or missing signature validation.
  - **Route LINE Message**
    - *Type & Technical Role:* Switch Node (`n8n-nodes-base.switch`) routing messages based on text content.
    - *Configuration Choices:* Four output rules (Rules for "1", "2", "3", and an extra fallback output).
    - *Expressions:* Checks `={{ $('Receive LINE Event').first().json.body.events[0].message.text }}`.
    - *Connections:* Input: `Receive LINE Event`; Output: Routes 1, 2, and 3 connect to their respective status update nodes. Fallback routes to registration.
    - *Failure Modes:* Non-text event payloads (e.g., stickers or images) falling into unexpected paths.
  - **Update Status - Safe**
    - *Type & Technical Role:* Google Sheets Node (`n8n-nodes-base.googleSheets`) recording a "Safe" response.
    - *Configuration Choices:* Operation: Update row matching `LINE User ID`.
    - *Expressions:* 
      - `LINE User ID`: `={{ $('Receive LINE Event').first().json.body.events[0].source.userId }}`
      - `Safety Status`: `Safe`
      - `Last Response At`: `={{ $now.toFormat('yyyy-LL-dd HH:mm') }}`
    - *Connections:* Input: `Route LINE Message` (Output 0); Output: None.
    - *Failure Modes:* Unregistered user IDs failing to match existing rows during update operations.
  - **Update Status - Needs Help**
    - *Type & Technical Role:* Google Sheets Node (`n8n-nodes-base.googleSheets`) recording a "Needs Help" response.
    - *Configuration Choices:* Operation: Update row matching `LINE User ID`.
    - *Expressions:* 
      - `LINE User ID`: `={{ $('Receive LINE Event').first().json.body.events[0].source.userId }}`
      - `Safety Status`: `Needs Help`
      - `Last Response At`: `={{ $now.toFormat('yyyy-LL-dd HH:mm') }}`
    - *Connections:* Input: `Route LINE Message` (Output 1); Output: None.
    - *Failure Modes:* Same as above.
  - **Update Status - Other**
    - *Type & Technical Role:* Google Sheets Node (`n8n-nodes-base.googleSheets`) recording an "Other" status.
    - *Configuration Choices:* Operation: Update row matching `LINE User ID`.
    - *Expressions:* 
      - `LINE User ID`: `={{ $('Receive LINE Event').first().json.body.events[0].source.userId }}`
      - `Safety Status`: `Other`
      - `Last Response At`: `={{ $now.toFormat('yyyy-LL-dd HH:mm') }}`
    - *Connections:* Input: `Route LINE Message` (Output 2); Output: None.
    - *Failure Modes:* Same as above.

---

#### Block 1.3: Employee Registration
- **Overview:** Parses fallback text messages containing comma- or ideographic-comma-separated registration data sent by employees, extracting names and prefectures to populate new records in the Google Sheets employee directory.
- **Nodes Involved:**
  - `Register Employee`

- **Node Details:**
  - **Register Employee**
    - *Type & Technical Role:* Google Sheets Node (`n8n-nodes-base.googleSheets`) appending new employee rows.
    - *Configuration Choices:* Operation: Append row to `Employee Directory`.
    - *Expressions:* 
      - `Name`: `={{ $('Receive LINE Event').first().json.body.events[0].message.text.split(/[、,，]/)[0].trim() }}`
      - `LINE User ID`: `={{ $('Receive LINE Event').first().json.body.events[0].source.userId }}`
      - `Safety Status`: `Registered`
      - `Home Prefecture`: `={{ $('Receive LINE Event').first().json.body.events[0].message.text.split(/[、,，]/)[1].trim() }}`
      - `Work Prefecture`: `={{ $('Receive LINE Event').first().json.body.events[0].message.text.split(/[、,，]/)[2].trim() }}`
    - *Connections:* Input: `Route LINE Message` (Fallback Output); Output: None.
    - *Failure Modes:* Malformed text input failing to split into the expected array indexes, resulting in undefined values or index out-of-bounds exceptions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Earthquake Check Schedule` | scheduleTrigger | Triggers workflow every 30 minutes | None | `Fetch Earthquake Information` | 1-Earthquake Detection and Notification<br><br>Checks earthquake information on a schedule.<br>If a new earthquake with seismic intensity 4 or higher affects an employee's home or work prefecture, a LINE safety check message is sent. |
| `Fetch Earthquake Information` | httpRequest | Fetches earthquake data from P2PQuake API | `Earthquake Check Schedule` | `Get Latest Earthquake` | 1-Earthquake Detection and Notification<br><br>Checks earthquake information on a schedule.<br>If a new earthquake with seismic intensity 4 or higher affects an employee's home or work prefecture, a LINE safety check message is sent. |
| `Get Latest Earthquake` | limit | Isolates the most recent earthquake item | `Fetch Earthquake Information` | `Get Last Earthquake ID` | 1-Earthquake Detection and Notification<br><br>Checks earthquake information on a schedule.<br>If a new earthquake with seismic intensity 4 or higher affects an employee's home or work prefecture, a LINE safety check message is sent. |
| `Get Last Earthquake ID` | googleSheets | Reads the last processed earthquake ID from sheet | `Get Latest Earthquake` | `Is New Earthquake?` | 1-Earthquake Detection and Notification<br><br>Checks earthquake information on a schedule.<br>If a new earthquake with seismic intensity 4 or higher affects an employee's home or work prefecture, a LINE safety check message is sent. |
| `Is New Earthquake?` | if | Evaluates if the latest earthquake is unprocessed | `Get Last Earthquake ID` | `Get Employees` | 1-Earthquake Detection and Notification<br><br>Checks earthquake information on a schedule.<br>If a new earthquake with seismic intensity 4 or higher affects an employee's home or work prefecture, a LINE safety check message is sent. |
| `Get Employees` | googleSheets | Retrieves employee directory records | `Is New Earthquake?` | `Is Employee in Affected Area?` | 1-Earthquake Detection and Notification<br><br>Checks earthquake information on a schedule.<br>If a new earthquake with seismic intensity 4 or higher affects an employee's home or work prefecture, a LINE safety check message is sent. |
| `Is Employee in Affected Area?` | if | Checks if employee home/work prefecture is affected (Scale >= 40) | `Get Employees` | `Send Safety Check to LINE` | 1-Earthquake Detection and Notification<br><br>Checks earthquake information on a schedule.<br>If a new earthquake with seismic intensity 4 or higher affects an employee's home or work prefecture, a LINE safety check message is sent. |
| `Send Safety Check to LINE` | httpRequest | Sends LINE push message safety check to affected employees | `Is Employee in Affected Area?` | `Update Last Earthquake ID` | 1-Earthquake Detection and Notification<br><br>Checks earthquake information on a schedule.<br>If a new earthquake with seismic intensity 4 or higher affects an employee's home or work prefecture, a LINE safety check message is sent. |
| `Update Last Earthquake ID` | googleSheets | Updates control sheet with latest earthquake ID and timestamp | `Send Safety Check to LINE` | None | 1-Earthquake Detection and Notification<br><br>Checks earthquake information on a schedule.<br>If a new earthquake with seismic intensity 4 or higher affects an employee's home or work prefecture, a LINE safety check message is sent. |
| `Receive LINE Event` | webhook | Receives incoming webhook events from LINE | None | `Route LINE Message` | 2-Safety Response Tracking<br><br>Employees reply with:<br>1 = Safe<br>2 = Needs Help<br>3 = Other<br><br>The workflow updates the employee's safety status and response time in Google Sheets. |
| `Route LINE Message` | switch | Routes incoming LINE messages based on option text (1, 2, 3, or registration) | `Receive LINE Event` | `Update Status - Safe`, `Update Status - Needs Help`, `Update Status - Other`, `Register Employee` | 2-Safety Response Tracking<br><br>Employees reply with:<br>1 = Safe<br>2 = Needs Help<br>3 = Other<br><br>The workflow updates the employee's safety status and response time in Google Sheets. |
| `Update Status - Safe` | googleSheets | Updates employee safety status to "Safe" in Google Sheets | `Route LINE Message` | None | 2-Safety Response Tracking<br><br>Employees reply with:<br>1 = Safe<br>2 = Needs Help<br>3 = Other<br><br>The workflow updates the employee's safety status and response time in Google Sheets. |
| `Update Status - Needs Help` | googleSheets | Updates employee safety status to "Needs Help" in Google Sheets | `Route LINE Message` | None | 2-Safety Response Tracking<br><br>Employees reply with:<br>1 = Safe<br>2 = Needs Help<br>3 = Other<br><br>The workflow updates the employee's safety status and response time in Google Sheets. |
| `Update Status - Other` | googleSheets | Updates employee safety status to "Other" in Google Sheets | `Route LINE Message` | None | 2-Safety Response Tracking<br><br>Employees reply with:<br>1 = Safe<br>2 = Needs Help<br>3 = Other<br><br>The workflow updates the employee's safety status and response time in Google Sheets. |
| `Register Employee` | googleSheets | Appends a new employee record parsed from LINE message text | `Route LINE Message` | None | 3-Employee Registration<br><br>Employees send their name, home prefecture, and work prefecture via LINE.<br>The workflow stores the employee's LINE User ID and registration information in Google Sheets. |

---

### 4. Reproducing the Workflow from Scratch

1. **Set up Credentials:**
   - Create a **Google Sheets OAuth2 API** credential in n8n.
   - Create an **HTTP Header Auth** credential for LINE with your Channel Access Token (`X-Line-Bot-Api-Token` or equivalent header structure used by the LINE Messaging API).

2. **Prepare Google Sheets:**
   - Create a Google Sheet named with two tabs:
     - Tab 1: `Earthquake Control` containing columns `Last Earthquake ID`, `Last Checked At`, and a pre-existing row 2.
     - Tab 2: `Employee Directory` containing columns `Name`, `LINE User ID`, `Home Prefecture`, `Work Prefecture`, `Safety Status`, and `Last Response At`.

3. **Build Block 1.1 (Earthquake Detection & Notification):**
   - **Step 1:** Create an **Schedule Trigger** node (`Earthquake Check Schedule`). Configure interval to 30 minutes.
   - **Step 2:** Create an **HTTP Request** node (`Fetch Earthquake Information`). Set method to `GET`, URL to `https://api.p2pquake.net/v2/jma/quake`, and add query parameters `order = -1` and `min_scale = 40`. Connect `Earthquake Check Schedule` to this node.
   - **Step 3:** Create a **Limit** node (`Get Latest Earthquake`) and connect it. Leave default configurations to cap output at 1 item.
   - **Step 4:** Create a **Google Sheets** node (`Get Last Earthquake ID`). Link your Google Sheets credential, select the `Earthquake Control` sheet, and use the read operation. Connect `Get Latest Earthquake` to it.
   - **Step 5:** Create an **If** node (`Is New Earthquake?`). Set condition string comparison: `={{ $('Get Latest Earthquake').first().json.id }}` not equals `={{ $json["Last Earthquake ID"] }}`. Connect `Get Last Earthquake ID` to it.
   - **Step 6:** Create a **Google Sheets** node (`Get Employees`). Set operation to read the `Employee Directory` sheet. Connect the true output of `Is New Earthquake?` to this node.
   - **Step 7:** Create an **If** node (`Is Employee in Affected Area?`). Set JavaScript boolean expression: `={{ $('Get Latest Earthquake').first().json.points.some(p => p.scale >= 40 && (p.pref === $json['Home Prefecture'] || p.pref === $json['Work Prefecture'])) }}`. Connect `Get Employees` to it.
   - **Step 8:** Create an **HTTP Request** node (`Send Safety Check to LINE`). Set method to `POST`, URL to `https://api.line.me/v2/bot/message/push`, authentication type to Generic Credential Type (using the LINE HTTP Header Auth credential), body type to JSON, and supply the structured message payload mapping `to` to `{{$json['LINE User ID']}}`. Connect the true output of `Is Employee in Affected Area?` to this node.
   - **Step 9:** Create a **Google Sheets** node (`Update Last Earthquake ID`). Set operation to update row 2 of the `Earthquake Control` sheet, mapping `Last Earthquake ID` to `={{ $('Get Latest Earthquake').first().json.id }}` and `Last Checked At` to `={{ $now.toFormat('yyyy-LL-dd HH:mm') }}`. Connect `Send Safety Check to LINE` to this node.

4. **Build Block 1.2 & 1.3 (Response Tracking & Registration):**
   - **Step 10:** Create a **Webhook** node (`Receive LINE Event`). Set HTTP method to `POST` and note the generated test/production webhook URL to configure inside your LINE Developer Console.
   - **Step 11:** Create a **Switch** node (`Route LINE Message`). Configure four output rules targeting message text values equal to `"1"`, `"2"`, `"3"`, and a fallback output. Connect `Receive LINE Event` to this node.
   - **Step 12:** Create three **Google Sheets** nodes (`Update Status - Safe`, `Update Status - Needs Help`, `Update Status - Other`). Set each to update the `Employee Directory` sheet matching column `LINE User ID` (evaluated via `={{ $('Receive LINE Event').first().json.body.events[0].source.userId }}`), setting `Safety Status` to respective values (`Safe`, `Needs Help`, `Other`) and `Last Response At` to `={{ $now.toFormat('yyyy-LL-dd HH:mm') }}`. Connect outputs 0, 1, and 2 of the Switch node respectively.
   - **Step 13:** Create a **Google Sheets** node (`Register Employee`). Set operation to append a row to `Employee Directory`, mapping columns using string split expressions against `={{ $('Receive LINE Event').first().json.body.events[0].message.text }}` to extract Name, Home Prefecture, and Work Prefecture, along with `LINE User ID` and a default status of `Registered`. Connect the fallback output of the Switch node to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| External Seismic Data Source | Powered by the [P2PQuake API](https://api.p2pquake.net/) |
| Messaging Platform Documentation | Integrates with the [LINE Messaging API Documentation](https://developers.line.biz/en/docs/messaging-api/) |
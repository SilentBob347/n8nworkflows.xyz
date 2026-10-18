Send new SAM.gov contract opportunities to Slack with Apify

https://n8nworkflows.xyz/workflows/send-new-sam-gov-contract-opportunities-to-slack-with-apify-20979


# Send new SAM.gov contract opportunities to Slack with Apify

### 1. Workflow Overview

This workflow automates the collection and notification of federal contract opportunities from SAM.gov. Its primary purpose is to help small businesses, federal contractors, and business development (BD) teams monitor new or amended federal notices without manually checking the SAM.gov portal daily. 

The workflow relies on an external Apify actor to fetch, filter, and track state changes, ensuring that only newly published or modified notices are processed. If new items are found, formatted notifications are automatically broadcast to a designated Slack channel.

The logic is grouped into the following functional blocks:
- **1.1 Trigger & Execution Initiation:** Automatically triggers the workflow on a daily schedule.
- **1.2 Data Retrieval & Filtering:** Calls the Apify Actor via an HTTP request to query SAM.gov opportunities using predefined criteria, filters out empty responses, and evaluates whether new items exist.
- **1.3 Notification Delivery:** Formats the contract details into a structured message and posts them to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Execution Initiation
- **Overview:** Initiates the automation on a fixed daily timetable.
- **Nodes Involved:** `Schedule`
- **Node Details:**
  - **Schedule (`n8n-nodes-base.scheduleTrigger`)**
    - *Technical Role:* Time-based workflow trigger.
    - *Configuration:* Configured to trigger daily at hour 7:00.
    - *Input/Output:* No inputs; outputs execution flow to the HTTP Request node.
    - *Potential Edge Cases:* Clock drift or skipped executions if the n8n instance is offline during the scheduled hour.

#### 2.2 Data Retrieval & Filtering
- **Overview:** Communicates with the Apify platform to run the SAM.gov opportunity radar actor, retrieves filtered results, and checks if any notices were returned.
- **Nodes Involved:** `Run SAM.gov Opportunity Radar`, `Has notice?`
- **Node Details:**
  - **Run SAM.gov Opportunity Radar (`n8n-nodes-base.httpRequest`)**
    - *Technical Role:* Synchronous HTTP POST request to execute an Apify Actor and fetch dataset items.
    - *Configuration:* Uses Generic Header Authentication (`Authorization: Bearer <token>`). Sends a JSON body containing parameters like `naicsCodes` ("541512"), `setAside` ("SBA"), `lookbackDays` (2), `maxItems` (25), and `monitorName` ("n8n-daily"). Sends query parameters for `timeout` (280s), `format` (json), and `clean` (true).
    - *Key Expressions/Variables:* None (uses static JSON payload).
    - *Input/Output:* Input from `Schedule`; output to `Has notice?`.
    - *Version Requirements:* v4.2.
    - *Potential Edge Cases:* Authentication failures (invalid/expired Apify token), request timeouts if SAM.gov or Apify is slow, or changes to the actor's schema.
  - **Has notice? (`n8n-nodes-base.if`)**
    - *Technical Role:* Conditional branching node to stop execution if no opportunities are returned.
    - *Configuration:* Evaluates whether the property `noticeId` exists in the incoming data stream.
    - *Key Expressions/Variables:* `={{ $json.noticeId }}`
    - *Input/Output:* Input from `Run SAM.gov Opportunity Radar`; outputs to `Post to Slack` (true branch) or terminates the execution branch (false branch).
    - *Version Requirements:* v2.

#### 2.3 Notification Delivery
- **Overview:** Formats the contract data and sends an alert message to a communication channel.
- **Nodes Involved:** `Post to Slack`
- **Node Details:**
  - **Post to Slack (`n8n-nodes-base.slack`)**
    - *Technical Role:* Sends formatted chat messages to a target Slack channel via API integration.
    - *Configuration:* Posts messages to a channel named `#govcon-leads` with link unfurling disabled.
    - *Key Expressions/Variables:* Uses inline expressions to construct the message body:
      - `={{ $json.title }}`
      - `={{ $json.noticeType }} · {{ $json.department }}{{ $json.subTier ? ' / ' + $json.subTier : '' }}`
      - `=NAICS {{ $json.naicsCode }} · Set-aside: {{ $json.setAside || 'none' }} · Due: {{ $json.responseDeadline || 'n/a' }}`
      - `={{ ($json.description || '').slice(0, 300) }}`
      - `=<{{ $json.samUrl }}|Open on SAM.gov> ({{ $json.changeType }})`
    - *Input/Output:* Input from the true branch of `Has notice?`; no downstream outputs.
    - *Potential Edge Cases:* Slack credential expiration, missing channel permissions, or text truncation limits if descriptions exceed expectations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview and setup | n8n-nodes-base.stickyNote | Documentation | None | None | ## Send new SAM.gov contract opportunities to Slack daily<br><br>### Who it's for<br>Small federal contractors and BD teams who check SAM.gov every morning and still miss notices.<br><br>### How it works<br>1. **Schedule** fires every morning at 7:00.<br>2. **Run SAM.gov Opportunity Radar** calls the Apify Actor (run-sync-get-dataset-items). It returns only notices that are **new or amended** since the last run, filtered by NAICS, set-aside, agency and state.<br>3. **Has notice?** stops quietly on days with nothing new.<br>4. **Post to Slack** sends one message per notice: title, type, agency, NAICS, set-aside, due date, short description, SAM.gov link.<br><br>### How to set up<br>1. Get your Apify API token: Apify Console > Settings > API & Integrations.<br>2. In *Run SAM.gov Opportunity Radar*, create a **Header Auth** credential: Name `Authorization`, Value `Bearer YOUR_APIFY_TOKEN`. Never paste the token into the URL or body.<br>3. Connect your **Slack** credential in *Post to Slack* and pick the channel.<br>4. Run once manually, check the output, then activate.<br><br>### Customize the input<br>Edit the JSON body of *Run SAM.gov Opportunity Radar*: `naicsCodes`, `setAside` (e.g. SBA, 8A, SDVOSBC, WOSB), `state`, `keywords`. Keep `monitorName` the same between runs, because that's what makes it "new only".<br><br>### Requirements<br>Apify account, pay per result: $0.01 per run + $0.004 per new or changed notice. No SAM.gov API key needed (official public extract); add your own key for real-time data.<br>Actor: https://apify.com/cce_partners/samgov-opportunity-radar<br><br>### How to extend<br>Swap Slack for Microsoft Teams, Gmail or Discord; add an IF on `setAside` or `responseDeadline`; change the schedule. |
| Step 1: Schedule | n8n-nodes-base.stickyNote | Documentation | None | None | ### 1. Schedule<br>Change the cron/interval here. Runs once per period; the Actor itself skips anything already sent. |
| Step 2: Run the Actor | n8n-nodes-base.stickyNote | Documentation | None | None | ### 2. Run SAM.gov Opportunity Radar on Apify<br>POST to run-sync-get-dataset-items with your filters in the JSON body. Auth: your Header Auth credential (`Authorization: Bearer <token>`). Timeout 300 s. |
| Step 3: Deliver | n8n-nodes-base.stickyNote | Documentation | None | None | ### 3. Filter and deliver to Slack<br>*Has notice?* stops when there's nothing new. Then the results go to Slack. Set your own credential and target (sheet ID / channel) on the last node. |
| Schedule | n8n-nodes-base.scheduleTrigger | Triggers workflow daily | None | Run SAM.gov Opportunity Radar | |
| Run SAM.gov Opportunity Radar | n8n-nodes-base.httpRequest | Fetches contract data from Apify | Schedule | Has notice? | |
| Has notice? | n8n-nodes-base.if | Checks if any notices were returned | Run SAM.gov Opportunity Radar | Post to Slack (True), [None] (False) | |
| Post to Slack | n8n-nodes-base.slack | Sends notification to Slack channel | Has notice? | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Set the trigger rule to fire by days at hour `7`.
2. **Create the HTTP Request Node (Apify Integration):**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Connect the output of the **Schedule Trigger** node to this node.
   - Configure parameters:
     - Method: `POST`
     - URL: `https://api.apify.com/v2/acts/cce_partners~samgov-opportunity-radar/run-sync-get-dataset-items`
     - Authentication: Select Generic Credential Type -> Header Auth. Create a credential with Name: `Authorization` and Value: `Bearer <your-apify-token>`.
     - Specify Body: `JSON`
     - JSON Body:
       ```json
       {
         "naicsCodes": ["541512"],
         "setAside": "SBA",
         "lookbackDays": 2,
         "maxItems": 25,
         "monitorName": "n8n-daily"
       }
       ```
     - Query Parameters: Add parameters `timeout` = `280`, `format` = `json`, and `clean` = `true`.
     - Options -> Timeout: Set to `300000` ms.
3. **Create the IF Condition Node:**
   - Add an **If** node (`n8n-nodes-base.if`).
   - Connect the output of the **HTTP Request** node to this node.
   - Configure conditions: Set left value to `={{ $json.noticeId }}` with the operator **Exists**.
4. **Create the Slack Notification Node:**
   - Add a **Slack** node (`n8n-nodes-base.slack`).
   - Connect the `true` output branch of the **If** node to this node.
   - Configure credentials: Authenticate using your Slack account.
   - Set resource/action parameters to post a message to a channel.
   - Channel Name: Select mode by name and input `#govcon-leads` (or your preferred channel).
   - Text Message Content: Paste the template expression:
     ```text
     =*{{ $json.title }}*
     {{ $json.noticeType }} · {{ $json.department }}{{ $json.subTier ? ' / ' + $json.subTier : '' }}
     NAICS {{ $json.naicsCode }} · Set-aside: {{ $json.setAside || 'none' }} · Due: {{ $json.responseDeadline || 'n/a' }}
     {{ ($json.description || '').slice(0, 300) }}
     <{{ $json.samUrl }}|Open on SAM.gov> ({{ $json.changeType }})
     ```
   - Other Options: Set `unfurl_links` to `false`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify Actor documentation and repository for the SAM.gov Opportunity Radar | [https://apify.com/cce_partners/samgov-opportunity-radar](https://apify.com/cce_partners/samgov-opportunity-radar) |
Collect event feedback with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/collect-event-feedback-with-google-sheets-and-gmail-20973


# Collect event feedback with Google Sheets and Gmail

### 1. Workflow Overview

This workflow automates the process of identifying event participants who have missing information (specifically, food preferences), generating a unique tracking identifier for them, and preparing personalized Gmail draft emails containing a pre-filled Google Forms feedback link. 

The workflow is structured into four functional blocks:
- **1.1 Initialization and Ingestion:** Manually starts the process and pulls the full participant roster from a Google Sheets spreadsheet.
- **1.2 Filtering:** Isolates participants who have not yet provided their food preferences.
- **1.3 Iteration and Database Update:** Processes filtered participants in batches, assigns a unique hexadecimal ID if one does not already exist, and writes the ID back to the Google Sheets spreadsheet.
- **1.4 Email Preparation and Dispatch:** Constructs a personalized email body using the participant's name, assigned ID, and a pre-filled Google Form URL, then creates a draft in Gmail before looping to the next participant.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Ingestion
- **Overview:** Initiates the workflow via a manual trigger and fetches all participant records from the master Google Sheets spreadsheet.
- **Nodes Involved:** `Manual Email Trigger`, `Read Participants Sheet`
- **Node Details:**
  - **Manual Email Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` — Acts as the manual entry point to execute the workflow on demand.
    - *Configuration Choices:* Standard default configuration (no parameters required).
    - *Input/Output Connections:* Output connects to `Read Participants Sheet`.
    - *Edge Cases / Failure Types:* None (user-driven execution).
  - **Read Participants Sheet**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Connects to Google Sheets to read rows from a specific worksheet.
    - *Configuration Choices:* Operation set to read (default list rows), targeting document ID `1KblaK_alQlvJ-wvbPWv3Ug5QrH01ukUrc-h9gVCUhSw` and sheet name `Participants master` (`gid=0`).
    - *Input/Output Connections:* Input from `Manual Email Trigger`; output connects to `Filter Participants`.
    - *Credentials Required:* Google Sheets OAuth2 API (`googleSheetsOAuth2Api`).
    - *Edge Cases / Failure Types:* API rate limits, invalid spreadsheet IDs, or permission errors on the connected Google account.

#### 2.2 Filtering
- **Overview:** Evaluates the retrieved participant list and filters out records where the required data field is already populated.
- **Nodes Involved:** `Filter Participants`
- **Node Details:**
  - **Filter Participants**
    - *Type and Technical Role:* `n8n-nodes-base.filter` — Evaluates incoming items against defined conditions to split or restrict the data stream.
    - *Configuration Choices:* Uses a strict string comparison condition checking if the `Food pref` column is empty (`==""`).
    - *Key Expressions:* `={{ $json['Food pref'] }}`
    - *Input/Output Connections:* Input from `Read Participants Sheet`; output connects to `Loop Over Participants`.
    - *Edge Cases / Failure Types:* Expression evaluation failures if column headers change in the source sheet.

#### 2.3 Iteration and Database Update
- **Overview:** Loops through the filtered participants one by one, generates a unique ID if missing, and updates the corresponding row in Google Sheets.
- **Nodes Involved:** `Loop Over Participants`, `Generate Unique ID`, `Update Participant ID`
- **Node Details:**
  - **Loop Over Participants**
    - *Type and Technical Role:* `n8n-nodes-base.splitInBatches` — Iterates through items sequentially to control batch flow.
    - *Configuration Choices:* Uses default batch settings (batch size of 1).
    - *Input/Output Connections:* Input from `Filter Participants` (and loop-back from `Send Email Draft`); output branch 0 terminates the loop, output branch 1 routes to `Generate Unique ID`.
    - *Edge Cases / Failure Types:* Infinite loops if loop completion logic is improperly re-routed.
  - **Generate Unique ID**
    - *Type and Technical Role:* `n8n-nodes-base.code` — Executes custom JavaScript to inspect or mutate item properties.
    - *Configuration Choices:* Evaluates whether an `ID` already exists; if not, generates a 12-character random hexadecimal string using `Math.random()`.
    - *Key Expressions / Code:* 
      ```javascript
      $json.ID = $json.ID || Math.round(Math.random() * 1_000_000_000_000_000).toString(16);
      return $input.all();
      ```
    - *Input/Output Connections:* Input from `Loop Over Participants`; output connects to `Update Participant ID`.
    - *Edge Cases / Failure Types:* JavaScript runtime exceptions if input structure lacks expected JSON context.
  - **Update Participant ID**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` — Updates specific cells in an existing Google Sheet row.
    - *Configuration Choices:* Operation set to `update`. Maps values using `row_number` as the matching column to update the `ID` field.
    - *Key Expressions:* `ID` assigned to `={{ $json.ID }}`, matching column `row_number` assigned to `={{ $json.row_number }}`.
    - *Input/Output Connections:* Input from `Generate Unique ID`; output connects to `Set Mail Message`.
    - *Credentials Required:* Google Sheets OAuth2 API (`googleSheetsOAuth2Api`).
    - *Edge Cases / Failure Types:* Missing `row_number` metadata or write-permission constraints on the target sheet.

#### 2.4 Email Preparation and Dispatch
- **Overview:** Assembles a custom email message containing pre-filled parameters for a Google Form and creates a draft inside the user's Gmail account.
- **Nodes Involved:** `Set Mail Message`, `Send Email Draft`
- **Node Details:**
  - **Set Mail Message**
    - *Type and Technical Role:* `n8n-nodes-base.set` — Creates or modifies workflow data fields.
    - *Configuration Choices:* Assigns a constructed multi-line string to a new variable named `mailmessage`.
    - *Key Expressions:* Uses `.split(" ")[0]` to extract the participant's first name and `encodeURIComponent()` to safely inject the participant's ID and name into Google Form pre-fill URL entry parameters (`entry.663774070` and `entry.833568434`). It references properties from the upstream `Generate Unique ID` and `Loop Over Participants` nodes.
    - *Input/Output Connections:* Input from `Update Participant ID`; output connects to `Send Email Draft`.
    - *Edge Cases / Failure Types:* Expression errors if upstream node paths change or if names contain unexpected formatting.
  - **Send Email Draft**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` — Interacts with the Gmail API to manage messages and drafts.
    - *Configuration Choices:* Resource set to `draft`, subject configured as `Help us create a great xmas party`, message body set to `={{ $json.mailmessage }}`, and recipient set via options using `={{ $('Generate Unique ID').item.json.Email }}`.
    - *Input/Output Connections:* Input from `Set Mail Message`; output loops back to `Loop Over Participants` to process the next item.
    - *Credentials Required:* Gmail OAuth2 (`gmailOAuth2`).
    - *Edge Cases / Failure Types:* Invalid recipient email addresses, expired Gmail OAuth tokens, or Google API quota restrictions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation & setup reference | None | None | ## Chase feedback<br><br>(for Events, Operations, etc.)<br><br>### How it works<br><br>1. Starts with manual trigger to initiate the run.<br>2. Retrieves all participants from Google Sheets.<br>3. Filters the participants based on specific criteria.<br>4. Processes each participant in batches, creating unique IDs.<br>5. Matches IDs and sets up email drafts in Gmail.<br>6. Loops back if more participants remain.<br><br>### Setup steps<br><br>- [ ] Configure an OAuth 2 client (type "Web application" in the Google console<br>- [ ] Then in n8n, we need a Google sheets client<br>- [ ] Create your own sheet and form (can use the sample here to see how things work with XLOOKUP, ID, etc.). Select your sheet in "Get all participants" and "Set ID to match later"<br>- [ ] Create a fillable form link ("Get pre-filled link" in G-Form)<br>- [ ] Change "Create Mail message" to include your message - and also your form link (use from link from previous step, adjust similar to sample)<br><br>### Customization<br><br>Pls. customize sheet and form and message sent |
| `Sticky Note1` | `n8n-nodes-base.stickyNote` | Section documentation for trigger & fetch | None | None | ## Trigger and fetch participants<br><br>Starts the flow by manually triggering and retrieving participants from Google Sheets. |
| `Sticky Note2` | `n8n-nodes-base.stickyNote` | Section documentation for filtering | None | None | ## Filter participants<br><br>Filters participants based on preset conditions. |
| `Sticky Note3` | `n8n-nodes-base.stickyNote` | Section documentation for looping & IDs | None | None | ## Loop and create IDs<br><br>Processes items in batches creating unique IDs for each. |
| `Sticky Note4` | `n8n-nodes-base.stickyNote` | Section documentation for email prep | None | None | ## Set IDs and prepare email<br><br>Matches IDs and prepares email drafts for each participant. |
| `Manual Email Trigger` | `n8n-nodes-base.manualTrigger` | Initiates workflow execution manually | None | `Read Participants Sheet` | ## Trigger and fetch participants<br><br>Starts the flow by manually triggering and retrieving participants from Google Sheets. |
| `Read Participants Sheet` | `n8n-nodes-base.googleSheets` | Reads all participant rows from Google Sheets | `Manual Email Trigger` | `Filter Participants` | ## Trigger and fetch participants<br><br>Starts the flow by manually triggering and retrieving participants from Google Sheets. |
| `Filter Participants` | `n8n-nodes-base.filter` | Filters rows where food preference is empty | `Read Participants Sheet` | `Loop Over Participants` | ## Filter participants<br><br>Filters participants based on preset conditions. |
| `Loop Over Participants` | `n8n-nodes-base.splitInBatches` | Iterates through filtered participants sequentially | `Filter Participants`, `Send Email Draft` | `Generate Unique ID` (Branch 1), Loop End (Branch 0) | ## Loop and create IDs<br><br>Processes items in batches creating unique IDs for each. |
| `Generate Unique ID` | `n8n-nodes-base.code` | Generates a random hexadecimal ID if missing | `Loop Over Participants` | `Update Participant ID` | ## Loop and create IDs<br><br>Processes items in batches creating unique IDs for each. |
| `Update Participant ID` | `n8n-nodes-base.googleSheets` | Writes generated ID back to Google Sheets | `Generate Unique ID` | `Set Mail Message` | ## Set IDs and prepare email<br><br>Matches IDs and prepares email drafts for each participant. |
| `Set Mail Message` | `n8n-nodes-base.set` | Formats personalized email body with form link | `Update Participant ID` | `Send Email Draft` | ## Set IDs and prepare email<br><br>Matches IDs and prepares email drafts for each participant. |
| `Send Email Draft` | `n8n-nodes-base.gmail` | Creates a Gmail draft message for the participant | `Set Mail Message` | `Loop Over Participants` | ## Set IDs and prepare email<br><br>Matches IDs and prepares email drafts for each participant. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Point:**
   - Add a **Manual Trigger** (`n8n-nodes-base.manualTrigger`) node named `Manual Email Trigger`.
2. **Add Google Sheets Data Ingestion:**
   - Add a **Google Sheets** (`n8n-nodes-base.googleSheets`) node named `Read Participants Sheet`.
   - Connect `Manual Email Trigger` output to `Read Participants Sheet`.
   - Configure credentials: Authenticate using a **Google Sheets OAuth2 API** credential.
   - Set parameters: Operation = `Read` (Get Many), Document ID = your target spreadsheet ID, Sheet Name = select your participant worksheet.
3. **Filter Participants:**
   - Add a **Filter** (`n8n-nodes-base.filter`) node named `Filter Participants`.
   - Connect `Read Participants Sheet` output to `Filter Participants`.
   - Set condition: Left Value = `={{ $json['Food pref'] }}`, Operator = `Is Empty`, Right Value = `=""`.
4. **Setup Iteration Loop:**
   - Add a **Split In Batches** (`n8n-nodes-base.splitInBatches`) node named `Loop Over Participants`.
   - Connect `Filter Participants` output to the input of `Loop Over Participants`.
5. **Generate Unique Identifiers:**
   - Add a **Code** (`n8n-nodes-base.code`) node named `Generate Unique ID`.
   - Connect output branch 1 (`Loop Over Participants`) to `Generate Unique ID`.
   - Set Mode to JavaScript and paste:
     ```javascript
     $json.ID = $json.ID || Math.round(Math.random() * 1_000_000_000_000_000).toString(16);
     return $input.all();
     ```
6. **Update Google Sheets Record:**
   - Add a **Google Sheets** (`n8n-nodes-base.googleSheets`) node named `Update Participant ID`.
   - Connect `Generate Unique ID` output to `Update Participant ID`.
   - Reuse your Google Sheets OAuth2 credentials.
   - Set parameters: Operation = `Update`, Document ID = your target spreadsheet ID, Sheet Name = select your participant worksheet.
   - Configure mapping: Matching Columns = `row_number`. Define columns below: Map `ID` to `={{ $json.ID }}` and `row_number` to `={{ $json.row_number }}`.
7. **Construct Email Content:**
   - Add a **Set (Edit Fields)** (`n8n-nodes-base.set`) node named `Set Mail Message`.
   - Connect `Update Participant ID` output to `Set Mail Message`.
   - Add string assignment named `mailmessage`:
     ```text
     =Hi {{ $('Generate Unique ID').item.json.Name.split(" ")[0] }}, 

     thanks for signing up for the Springfield Xmas party! We're happy to welcome you. 

     To help us prepare, can you pls. share a bit about you and your preferences - which food you like et al. It helps us create the best event. 

     You're done in a minute - just fill out this form: https://docs.google.com/forms/d/e/1FAIpQLSdP8Ru0dRvKE1Vj3nic9iC03Rwko-16xkom79eqxegAfE85tA/viewform?usp=pp_url&entry.663774070={{ encodeURIComponent($('Generate Unique ID').item.json.ID) }}&entry.833568434={{ encodeURIComponent($('Loop Over Participants').item.json.Name) }}

     Thanks and cu there
        Xmas team
     ```
8. **Create Gmail Drafts:**
   - Add a **Gmail** (`n8n-nodes-base.gmail`) node named `Send Email Draft`.
   - Connect `Set Mail Message` output to `Send Email Draft`.
   - Configure credentials: Authenticate using a **Gmail OAuth2** credential.
   - Set parameters: Resource = `Draft`, Subject = `Help us create a great xmas party`, Message = `={{ $json.mailmessage }}`. Under Options, add `Send To` = `={{ $('Generate Unique ID').item.json.Email }}`.
9. **Complete the Loop:**
   - Connect the output of `Send Email Draft` back to the input of `Loop Over Participants` (`Loop Over Participants`) to continue processing remaining rows.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Target Use Case | Designed for event and operations workflows to automate data completion and follow-up communications. |
| Google Cloud Configuration | Requires an OAuth 2 client configured as a "Web application" within the Google Console with permissions for Google Sheets and Gmail APIs. |
| Form Pre-fill Parameters | Ensure Google Form entry IDs (`entry.663774070` and `entry.833568434`) correspond to the actual form field IDs generated by your specific Google Form. |
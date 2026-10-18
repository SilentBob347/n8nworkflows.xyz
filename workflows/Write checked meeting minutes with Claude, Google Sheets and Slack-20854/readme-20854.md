Write checked meeting minutes with Claude, Google Sheets and Slack

https://n8nworkflows.xyz/workflows/write-checked-meeting-minutes-with-claude--google-sheets-and-slack-20854


# Write checked meeting minutes with Claude, Google Sheets and Slack

### 1. Workflow Overview

This workflow automates the generation, validation, and archiving of professional meeting minutes. It accepts raw meeting inputs via an interactive form, leverages an LLM to extract structured data alongside verbatim supporting quotes, and runs deterministic code-based guards to verify every extracted claim against the original transcript. Validated data is archived in a Google Sheets database, and an executive summary is posted to a designated Slack channel.

The workflow logic is categorized into the following operational blocks:

- **1.1 Input Reception & Configuration:** Captures meeting metadata and transcripts via a form interface, sets operational parameters, and normalizes the raw inputs into a stable, hashed format.
- **1.2 Duplicate Detection & Pre-Flight Checks:** Queries the destination spreadsheet to detect duplicate submissions and evaluates length and formatting constraints before proceeding.
- **1.3 AI Processing & Code-Based Validation:** Sends transcripts to Anthropic Claude for structured entity extraction, followed by a robust validation script that verifies quote authenticity, number integrity, attendee matching, and date parsing.
- **1.4 Data Persistence & Notifications:** Writes structured line items and aggregate meeting rows to Google Sheets, and pushes rich status summaries to Slack.
- **1.5 Spreadsheet Initialization (Setup Utility):** A manual provisioning branch used once to initialize the target Google Sheet with required tabs and headers.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Configuration

**Overview:** 
This block initializes the workflow execution upon form submission, configures global parsing and length constraints, and cleans the unstructured text payload into a structured schema with a unique cryptographic identifier.

**Nodes Involved:**
- Meeting form
- Config
- Prepare meeting

**Node Details:**

- **Meeting form**
  - **Type & Technical Role:** `n8n-nodes-base.formTrigger` (v2.6) – Acts as the primary HTTP webhook and interactive UI entry point for capturing form payloads.
  - **Configuration Choices:** Configured to respond with plain text upon completion (`formSubmittedText`). Exposes fields for meeting title, date, attendees, and transcript.
  - **Key Expressions / Variables:** Uses UI-defined schema fields.
  - **Input / Output Connections:** Input: None (Trigger). Output: Connects to `Config`.
  - **Edge Cases / Failure Types:** Publicly accessible by default; vulnerable to unauthorized submissions unless Basic Authentication is enabled on the trigger settings.

- **Config**
  - **Type & Technical Role:** `n8n-nodes-base.set` (v3.4) – Injects global configuration variables into the execution context.
  - **Configuration Choices:** Manually assigns values for `output_language`, `min_quote_chars` (8), `min_transcript_chars` (200), and `max_transcript_chars` (150000).
  - **Key Expressions / Variables:** Passes static constants downstream.
  - **Input / Output Connections:** Input: `Meeting form`. Output: Connects to `Prepare meeting`.
  - **Edge Cases / Failure Types:** None.

- **Prepare meeting**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) – Cleans input text, splits attendee strings into a normalized array, generates a deterministic meeting hash (`meetingId`), and performs pre-flight validation checks.
  - **Configuration Choices:** Custom JavaScript utilizing a cyrb53 hashing algorithm on title, date, and transcript concatenations.
  - **Key Expressions / Variables:** Reads `$input.first().json`, calculates `$now.toISO()`.
  - **Input / Output Connections:** Input: `Config`. Output: Connects to `Read Minutes tab`.
  - **Edge Cases / Failure Types:** Throws errors if JS execution fails on malformed input types; handles empty fields via fallback string coercions.

---

#### 2.2 Duplicate Detection & Pre-Flight Checks

**Overview:** 
This block prevents redundant processing by checking existing records in Google Sheets and routes the workflow conditionally based on whether the meeting passes initial constraints and uniqueness requirements.

**Nodes Involved:**
- Read Minutes tab
- Check for duplicates
- Process this meeting?
- Post skip notice to Slack

**Node Details:**

- **Read Minutes tab**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (v4.7) – Reads all existing records from the "Minutes" worksheet tab to check for prior processing.
  - **Configuration Choices:** Resource: `sheet`, Operation: `read`, Sheet Name: `Minutes`. `executeOnce` set to true.
  - **Key Expressions / Variables:** References target Document ID.
  - **Input / Output Connections:** Input: `Prepare meeting`. Output: Connects to `Check for duplicates`.
  - **Edge Cases / Failure Types:** API rate limits or authentication expiration with Google OAuth2.

- **Check for duplicates**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) – Evaluates read data against the current meeting ID to flag duplicates.
  - **Configuration Choices:** JavaScript using `Set` lookup over historical `meeting_id` fields.
  - **Key Expressions / Variables:** `$('Prepare meeting').first().json`, `$input.all()`.
  - **Input / Output Connections:** Input: `Read Minutes tab`. Output: Connects to `Process this meeting?`.
  - **Edge Cases / Failure Types:** Fails if the sheet schema lacks the `meeting_id` column.

- **Process this meeting?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (v2.2) – Conditional router evaluating whether `skip_reason` is empty.
  - **Configuration Choices:** Evaluates `{{ $json.skip_reason }}` against an empty string.
  - **Key Expressions / Variables:** Uses expression evaluation for condition routing.
  - **Input / Output Connections:** Input: `Check for duplicates`. Outputs: Branch A (True) -> `AI: Write minutes`; Branch B (False) -> `Post skip notice to Slack`.
  - **Edge Cases / Failure Types:** None.

- **Post skip notice to Slack**
  - **Type & Technical Role:** `n8n-nodes-base.slack` (v2.7) – Posts an informational warning to Slack when a meeting is skipped.
  - **Configuration Choices:** Resource: `message`, Operation: `post`.
  - **Key Expressions / Variables:** `={{:no_entry_sign: Minutes skipped for "{{ $json.title }}": {{ $json.skip_reason }}}`.
  - **Input / Output Connections:** Input: `Process this meeting?` (False branch). Output: Terminal node.
  - **Edge Cases / Failure Types:** Bot lacks permissions to post in the configured channel.

---

#### 2.3 AI Processing & Code-Based Validation

**Overview:** 
This block uses Anthropic Claude to draft structured minutes from the raw transcript, followed by a deterministic validation script that audits every extracted claim for quote veracity, proper attribution, temporal parsing, and numerical consistency.

**Nodes Involved:**
- AI: Write minutes
- Check minutes

**Node Details:**

- **AI: Write minutes**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.anthropic` (v1) – Calls Anthropic Claude to extract structured JSON data from unstructured text transcripts.
  - **Configuration Choices:** Model: `claude-haiku-4-5-20251001`, Temperature: `0`, Max Tokens: `4000`. Error handling set to `continueRegularOutput`.
  - **Key Expressions / Variables:** Uses prompt templates mapping `{{ $json.title }}`, `{{ $json.meeting_date }}`, `{{ $json.attendees_text }}`, `{{ $json.output_language }}`, and `{{ $json.transcript }}`.
  - **Input / Output Connections:** Input: `Process this meeting?` (True branch). Output: Connects to `Check minutes`.
  - **Edge Cases / Failure Types:** API timeouts, rate limiting, or malformed JSON responses from the model (mitigated downstream).

- **Check minutes**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) – Comprehensive validation engine executing custom JavaScript routines for normalization, quote anchoring, number checking, attendee matching, and natural language date resolution.
  - **Configuration Choices:** Executes custom scripts including NFKC normalization, fuzzy string squashing, regex-based due date parsing (supporting English and Japanese idioms), and multi-tier constraint checks.
  - **Key Expressions / Variables:** `$input.first().json.content?.[0]?.text`, `$('Check for duplicates').first().json`.
  - **Input / Output Connections:** Input: `AI: Write minutes`. Output: Connects to `To item rows`.
  - **Edge Cases / Failure Types:** Script execution errors if the AI returns non-parsable structures or unexpected types. Handled via try/catch blocks assigning "Needs review" statuses.

---

#### 2.4 Data Persistence & Notifications

**Overview:** 
This block formats validated outputs into individual item rows and an aggregate meeting row, persists them into separate Google Sheets worksheets, and sends a formatted status report to Slack.

**Nodes Involved:**
- To item rows
- Add to Items tab
- To minutes row
- Add to Minutes tab
- Post minutes to Slack

**Node Details:**

- **To item rows**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) – Flattens the array of extracted items (`items`) into individual workflow items for batch processing.
  - **Configuration Choices:** JavaScript map transformation.
  - **Key Expressions / Variables:** `$input.first().json.items`.
  - **Input / Output Connections:** Input: `Check minutes`. Output: Connects to `Add to Items tab`.
  - **Edge Cases / Failure Types:** Empty item arrays if validation fails catastrophically.

- **Add to Items tab**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (v4.7) – Appends or updates individual item records in the "Items" worksheet.
  - **Configuration Choices:** Resource: `sheet`, Operation: `appendOrUpdate`, Sheet Name: `Items`, Matching Columns: `item_id`, Mapping Mode: `autoMapInputData`.
  - **Key Expressions / Variables:** Dynamic column mapping.
  - **Input / Output Connections:** Input: `To item rows`. Output: Connects to `To minutes row`.
  - **Edge Cases / Failure Types:** Missing schema columns or sheet lockups.

- **To minutes row**
  - **Type & Technical Role:** `n8n-nodes-base.code` (v2) – Strips out line-item arrays and slack text objects, leaving only the aggregate meeting row properties.
  - **Configuration Choices:** JavaScript object destructuring.
  - **Key Expressions / Variables:** Destructures `items` and `slack_text` from `$('Check minutes').first().json`.
  - **Input / Output Connections:** Input: `Add to Items tab`. Output: Connects to `Add to Minutes tab`.
  - **Edge Cases / Failure Types:** None.

- **Add to Minutes tab**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (v4.7) – Appends or updates the aggregate meeting record in the "Minutes" worksheet tab.
  - **Configuration Choices:** Resource: `sheet`, Operation: `appendOrUpdate`, Sheet Name: `Minutes`, Matching Columns: `meeting_id`, Mapping Mode: `autoMapInputData`.
  - **Key Expressions / Variables:** Dynamic column mapping.
  - **Input / Output Connections:** Input: `To minutes row`. Output: Connects to `Post minutes to Slack`.
  - **Edge Cases / Failure Types:** Failure here prevents confirmation tracking since meeting completion is marked strictly by the existence of this row.

- **Post minutes to Slack**
  - **Type & Technical Role:** `n8n-nodes-base.slack` (v2.7) – Publishes the generated summary and validation warning indicators to the selected Slack channel.
  - **Configuration Choices:** Resource: `message`, Operation: `post`, Message Type: `text`.
  - **Key Expressions / Variables:** `={{ $('Check minutes').first().json.slack_text }}`.
  - **Input / Output Connections:** Input: `Add to Minutes tab`. Output: Terminal node.
  - **Edge Cases / Failure Types:** Channel permission errors or payload size constraints.

---

#### 2.5 Spreadsheet Initialization (Setup Utility)

**Overview:** 
A manual configuration branch used once during initial setup to provision the target Google Sheet with required worksheet tabs ("Minutes" and "Items") and corresponding header rows.

**Nodes Involved:**
- Run once: set up the sheet
- Spreadsheet ID
- Add the Minutes and Items tabs
- Write the header rows

**Node Details:**

- **Run once: set up the sheet**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` (v1) – Manual execution trigger for administrative setup tasks.
  - **Configuration Choices:** Default setup.
  - **Input / Output Connections:** Input: None. Output: Connects to `Spreadsheet ID`.
  - **Edge Cases / Failure Types:** Manual invocation only.

- **Spreadsheet ID**
  - **Type & Technical Role:** `n8n-nodes-base.set` (v3.4) – Holds the target Google Spreadsheet identifier.
  - **Configuration Choices:** Assigns `spreadsheet_id` parameter (`PASTE_YOUR_SPREADSHEET_ID_HERE`).
  - **Input / Output Connections:** Input: `Run once: set up the sheet`. Output: Connects to `Add the Minutes and Items tabs`.
  - **Edge Cases / Failure Types:** Failure to replace the placeholder string will result in API 404 errors.

- **Add the Minutes and Items tabs**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (v4.5) – Makes a Google Sheets API batchUpdate call to create the "Minutes" and "Items" tabs.
  - **Configuration Choices:** Method: `POST`, URL: `https://sheets.googleapis.com/v4/spreadsheets/{{ $("Spreadsheet ID").first().json.spreadsheet_id }}:batchUpdate`, Authentication: Predefined Google Sheets OAuth2 API.
  - **Key Expressions / Variables:** Dynamically inserts the spreadsheet ID.
  - **Input / Output Connections:** Input: `Spreadsheet ID`. Output: Connects to `Write the header rows`.
  - **Edge Cases / Failure Types:** Fails if the tabs already exist in the target spreadsheet.

- **Write the header rows**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (v4.5) – Makes a Google Sheets API values:batchUpdate call to populate column headers on both newly created tabs.
  - **Configuration Choices:** Method: `POST`, URL: `https://sheets.googleapis.com/v4/spreadsheets/{{ $("Spreadsheet ID").first().json.spreadsheet_id }}/values:batchUpdate`, Authentication: Predefined Google Sheets OAuth2 API.
  - **Key Expressions / Variables:** Inserts schema headers for both `Minutes` and `Items` tabs.
  - **Input / Output Connections:** Input: `Add the Minutes and Items tabs`. Output: Terminal node.
  - **Edge Cases / Failure Types:** Authentication failures or incorrect spreadsheet IDs.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Meeting form | n8n-nodes-base.formTrigger | Captures meeting metadata and transcript input via an interactive form | None | Config | ## 1. Collect the meeting<br>A form takes the title, date, attendees and transcript. Code cleans them, makes an ID from the text, and rejects empty, too short or too long transcripts. |
| Config | n8n-nodes-base.set | Assigns global limits, parsing variables, and quote thresholds | Meeting form | Prepare meeting | ## 1. Collect the meeting<br>A form takes the title, date, attendees and transcript. Code cleans them, makes an ID from the text, and rejects empty, too short or too long transcripts. |
| Prepare meeting | n8n-nodes-base.code | Normalizes inputs, generates a meeting hash ID, and evaluates pre-flight length constraints | Config | Read Minutes tab | ## 1. Collect the meeting<br>A form takes the title, date, attendees and transcript. Code cleans them, makes an ID from the text, and rejects empty, too short or too long transcripts. |
| Read Minutes tab | n8n-nodes-base.googleSheets | Fetches existing meeting records to check for duplicates | Prepare meeting | Check for duplicates | ## 2. Skip what is already done<br>A meeting that is already in the Minutes tab is skipped, and Slack says why. A rejected transcript is reported the same way. |
| Check for duplicates | n8n-nodes-base.code | Compares the current meeting ID against historical spreadsheet records | Read Minutes tab | Process this meeting? | ## 2. Skip what is already done<br>A meeting that is already in the Minutes tab is skipped, and Slack says why. A rejected transcript is reported the same way. |
| Process this meeting? | n8n-nodes-base.if | Routes execution based on whether a skip reason was triggered | Check for duplicates | AI: Write minutes, Post skip notice to Slack | ## 2. Skip what is already done<br>A meeting that is already in the Minutes tab is skipped, and Slack says why. A rejected transcript is reported the same way. |
| AI: Write minutes | @n8n/n8n-nodes-langchain.anthropic | Uses Claude to draft structured JSON minutes with verbatim quotes from the transcript | Process this meeting? | Check minutes | ## 3. Claude writes the minutes<br>Claude returns a summary, decisions, action items and open questions as JSON. Every item must carry a quote copied from the transcript. |
| Check minutes | n8n-nodes-base.code | Validates quotes, dates, numbers, and attendee attributions against the source transcript | AI: Write minutes | To item rows | ## 4. Code checks every claim<br>Quotes, owners, numbers and due words are checked against the transcript. Date words become dates in code. Failed items are kept and marked Needs review. |
| To item rows | n8n-nodes-base.code | Flattens the validated output items into individual items for batch database writes | Check minutes | Add to Items tab | ## 5. Save and post<br>Items are saved first, the Minutes row last, so a failed run is not counted as done. Slack gets the summary with a warning on doubtful items. |
| Add to Items tab | n8n-nodes-base.googleSheets | Appends or updates individual item records in the Google Sheets "Items" tab | To item rows | To minutes row | ## 5. Save and post<br>Items are saved first, the Minutes row last, so a failed run is not counted as done. Slack gets the summary with a warning on doubtful items. |
| To minutes row | n8n-nodes-base.code | Prepares the aggregate meeting summary row for insertion | Add to Items tab | Add to Minutes tab | ## 5. Save and post<br>Items are saved first, the Minutes row last, so a failed run is not counted as done. Slack gets the summary with a warning on doubtful items. |
| Add to Minutes tab | n8n-nodes-base.googleSheets | Appends or updates the aggregate meeting record in the Google Sheets "Minutes" tab | To minutes row | Post minutes to Slack | ## 5. Save and post<br>Items are saved first, the Minutes row last, so a failed run is not counted as done. Slack gets the summary with a warning on doubtful items. |
| Post minutes to Slack | n8n-nodes-base.slack | Posts the formatted executive summary and validation warnings to Slack | Add to Minutes tab | None | ## 5. Save and post<br>Items are saved first, the Minutes row last, so a failed run is not counted as done. Slack gets the summary with a warning on doubtful items. |
| Post skip notice to Slack | n8n-nodes-base.slack | Posts a skip notification to Slack if pre-flight checks fail | Process this meeting? | None | ## 2. Skip what is already done<br>A meeting that is already in the Minutes tab is skipped, and Slack says why. A rejected transcript is reported the same way. |
| Run once: set up the sheet | n8n-nodes-base.manualTrigger | Manual entry point for spreadsheet provisioning | None | Spreadsheet ID | ## Run once: set up the sheet<br>Paste your spreadsheet ID, run this branch once, and it adds the two tabs and the header rows. Running it twice fails, because the tabs exist. |
| Spreadsheet ID | n8n-nodes-base.set | Stores the target spreadsheet ID for provisioning operations | Run once: set up the sheet | Add the Minutes and Items tabs | ## Run once: set up the sheet<br>Paste your spreadsheet ID, run this branch once, and it adds the two tabs and the header rows. Running it twice fails, because the tabs exist. |
| Add the Minutes and Items tabs | n8n-nodes-base.httpRequest | Makes a Google Sheets API batchUpdate call to create worksheet tabs | Spreadsheet ID | Write the header rows | ## Run once: set up the sheet<br>Paste your spreadsheet ID, run this branch once, and it adds the two tabs and the header rows. Running it twice fails, because the tabs exist. |
| Write the header rows | n8n-nodes-base.httpRequest | Populates table headers for the Minutes and Items tabs | Add the Minutes and Items tabs | None | ## Run once: set up the sheet<br>Paste your spreadsheet ID, run this branch once, and it adds the two tabs and the header rows. Running it twice fails, because the tabs exist. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initial Setup & Credentials
1. Create a blank Google Sheets document and copy its Spreadsheet ID from the URL.
2. In n8n, configure credentials for:
   - **Google Sheets OAuth2 API**
   - **Anthropic API**
   - **Slack API** (ensure the bot user is added to your target notification channel)

#### Step 2: Build the Setup Utility Branch
1. Create a **Manual Trigger** node (`Run once: set up the sheet`).
2. Add a **Set** node (`Spreadsheet ID`):
   - Set mode to *Manual*.
   - Assign a string parameter named `spreadsheet_id` with your actual Google Sheets ID value.
3. Add an **HTTP Request** node (`Add the Minutes and Items tabs`):
   - Method: `POST`
   - URL: `https://sheets.googleapis.com/v4/spreadsheets/{{ $("Spreadsheet ID").first().json.spreadsheet_id }}:batchUpdate`
   - Authentication: *Predefined Credential Type* -> `googleSheetsOAuth2Api`
   - Specify Body: `JSON`
   - JSON Body:
     ```json
     {
       "requests": [
         {"addSheet": {"properties": {"title": "Minutes"}}},
         {"addSheet": {"properties": {"title": "Items"}}}
       ]
     }
     ```
4. Add an **HTTP Request** node (`Write the header rows`):
   - Method: `POST`
   - URL: `https://sheets.googleapis.com/v4/spreadsheets/{{ $("Spreadsheet ID").first().json.spreadsheet_id }}/values:batchUpdate`
   - Authentication: *Predefined Credential Type* -> `googleSheetsOAuth2Api`
   - Specify Body: `JSON`
   - JSON Body:
     ```json
     {
       "valueInputOption": "RAW",
       "data": [
         {"range": "'Minutes'!A1", "values": [["meeting_id", "meeting_date", "title", "attendees", "summary", "decisions", "actions", "open_questions", "needs_review", "status", "problems", "minuted_at"]]},
         {"range": "'Items'!A1", "values": [["item_id", "meeting_id", "meeting_date", "meeting_title", "type", "text", "owner", "due_text", "due_date", "quote", "status", "problems"]]}
       ]
     }
     ```
5. Connect: `Run once: set up the sheet` ➔ `Spreadsheet ID` ➔ `Add the Minutes and Items tabs` ➔ `Write the header rows`. Run this branch **once** to provision your sheet.

#### Step 3: Build the Main Pipeline
1. Create a **Form Trigger** node (`Meeting form`):
   - Configure form title as `Meeting minutes`.
   - Add fields:
     - Text field: `Meeting title` (Required)
     - Date field: `Meeting date` (Required)
     - Textarea: `Attendees` (Optional)
     - Textarea: `Transcript` (Required)
   - Set response mode to *On Received*, rendering a custom text message on submission.

2. Add a **Set** node (`Config`):
   - Assign assignments:
     - `output_language` (String): `the same language as the transcript`
     - `min_quote_chars` (Number): `8`
     - `min_transcript_chars` (Number): `200`
     - `max_transcript_chars` (Number): `150000`

3. Add a **Code** node (`Prepare meeting`):
   - Insert JavaScript to clean inputs, split attendees, calculate the cyrb53 `meeting_id` hash, and evaluate initial length/date skip conditions.

4. Add a **Google Sheets** node (`Read Minutes tab`):
   - Resource: `Sheet`, Operation: `Read`
   - Sheet Name: `Minutes`
   - Select your configured spreadsheet. Enable *Always Output Data*.

5. Add a **Code** node (`Check for duplicates`):
   - Insert JavaScript to cross-reference the generated `meeting_id` against retrieved spreadsheet rows and update `skip_reason` if found.

6. Add an **If** node (`Process this meeting?`):
   - Add condition checking if `{{ $json.skip_reason }}` is empty (`IsEmpty`).

7. Connect the **False** branch of `Process this meeting?` to a **Slack** node (`Post skip notice to Slack`):
   - Resource: `Message`, Operation: `Post`
   - Text: `={{:no_entry_sign: Minutes skipped for "{{ $json.title }}": {{ $json.skip_reason }}}`

8. Connect the **True** branch of `Process this meeting?` to an **Anthropic Chat Model / Node** (`AI: Write minutes`):
   - Model: `claude-haiku-4-5-20251001`
   - Temperature: `0`, Max Tokens: `4000`
   - Error Handling: Set to *Continue Regular Output*.
   - Prompt user message template with strict JSON output schema rules requiring exact transcript quotes for summaries, decisions, action items, and open questions.

9. Add a **Code** node (`Check minutes`):
   - Insert the comprehensive validation script that parses Claude's JSON response, normalizes text strings, verifies quotes via substring index checks, ensures numerical consistency, validates owner attributions against attendees, and parses temporal phrases into ISO date formats.

10. Add a **Code** node (`To item rows`):
    - Insert JavaScript returning `$input.first().json.items.map((i) => ({ json: i }))`.

11. Add a **Google Sheets** node (`Add to Items tab`):
    - Resource: `Sheet`, Operation: `Append or Update`
    - Sheet Name: `Items`, Matching Columns: `item_id`, Mapping Mode: `Auto-map input data`.

12. Add a **Code** node (`To minutes row`):
    - Insert JavaScript destructuring `items` and `slack_text` away from the payload, returning only aggregate meeting properties.

13. Add a **Google Sheets** node (`Add to Minutes tab`):
    - Resource: `Sheet`, Operation: `Append or Update`
    - Sheet Name: `Minutes`, Matching Columns: `meeting_id`, Mapping Mode: `Auto-map input data`.

14. Add a **Slack** node (`Post minutes to Slack`):
    - Resource: `Message`, Operation: `Post`
    - Text: `={{ $('Check minutes').first().json.slack_text }}`

#### Step 4: Final Wire-up
Connect the nodes sequentially:
`Meeting form` ➔ `Config` ➔ `Prepare meeting` ➔ `Read Minutes tab` ➔ `Check for duplicates` ➔ `Process this meeting?` 
- *(True branch)* ➔ `AI: Write minutes` ➔ `Check minutes` ➔ `To item rows` ➔ `Add to Items tab` ➔ `To minutes row` ➔ `Add to Minutes tab` ➔ `Post minutes to Slack`
- *(False branch)* ➔ `Post skip notice to Slack`

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Source workflow description and design pattern | Workflow configuration originating from n8n automation templates. |
| LLM constraints and token estimates | Designed for Anthropic Claude Haiku; average test run uses ~750 input and ~350 output tokens for a 15-line meeting transcript. |
| Temporal parsing support | Code-level due date resolution supports English and Romance/Japanese date idioms; extendable within the `resolveDue` function inside the `Check minutes` node. |
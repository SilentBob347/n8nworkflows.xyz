Draft sales call follow-up emails from Fireflies with OpenAI and Gmail

https://n8nworkflows.xyz/workflows/draft-sales-call-follow-up-emails-from-fireflies-with-openai-and-gmail-20963


# Draft sales call follow-up emails from Fireflies with OpenAI and Gmail

### 1. Workflow Overview

This workflow automates the post-call administrative and follow-up process triggered by Fireflies.ai transcription events. Its primary purpose is to extract meeting transcripts via the Fireflies GraphQL API, analyze conversation dynamics and outcomes using OpenAI, prepare context-specific draft follow-up emails in Gmail, log structured call data to Google Sheets, and notify the meeting owner via email.

The logic is grouped into five functional blocks:
- **1.1 Input Reception & Configuration:** Receives webhook notifications from Fireflies and injects user profile settings and constants.
- **1.2 Event Validation & Transcript Retrieval:** Validates incoming webhook payload types, fetches raw transcript metrics and sentences, and filters out non-speech or duplicate runs.
- **1.3 AI Call Analysis:** Uses an OpenAI language model powered by a structured JSON schema output parser to analyze meeting dialogs and format summaries, next steps, and draft email text.
- **1.4 Email Draft Generation:** Routes workflow execution based on contact email presence to create categorized drafts in Gmail without automatic sending.
- **1.5 Logging & Internal Notification:** Formats structured spreadsheet rows, appends records to Google Sheets, and dispatches a summary digest to the meeting owner.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Captures incoming webhook events from Fireflies when a call transcription completes and provides runtime user configuration variables.
- **Nodes Involved:** `When Call Transcribed`, `Set Personal Settings`, `Verify Settings and Event`
- **Node Details:**
  - **When Call Transcribed**
    - *Type/Role:* Webhook trigger node (`n8n-nodes-base.webhook`).
    - *Configuration:* Listens for HTTP POST requests on path `fireflies-call-notes`.
    - *Expressions/Variables:* None.
    - *Connections:* Input: None; Output: `Set Personal Settings`.
    - *Edge Cases/Failures:* Network-level timeouts or misconfigured external webhooks pointing to incorrect production URLs.
  - **Set Personal Settings**
    - *Type/Role:* Set/Edit Fields node (`n8n-nodes-base.set`).
    - *Configuration:* Manually sets string assignments (`yourName`, `businessName`, `yourEmail`, `whatYouSell`, `bookingLink`, `callLogSheetUrl`).
    - *Expressions/Variables:* Static fallback string values that must be overwritten by the user prior to activation.
    - *Connections:* Input: `When Call Transcribed`; Output: `Verify Settings and Event`.
    - *Edge Cases/Failures:* Failure to replace placeholder strings causes downstream validation errors.
  - **Verify Settings and Event**
    - *Type/Role:* Code node (`n8n-nodes-base.code`, JavaScript).
    - *Configuration:* Runs a script verifying that placeholders are replaced and that incoming payload formats match expected Fireflies V1/V2 structures.
    - *Expressions/Variables:* Uses `$('Set Personal Settings').first().json` and `$('When Call Transcribed').first().json.body`.
    - *Connections:* Input: `Set Personal Settings`; Output: `If New Transcript Event`.
    - *Edge Cases/Failures:* Throws a runtime JavaScript error if configuration fields contain invalid syntax or missing emails.

#### 2.2 Event Validation & Transcript Retrieval
- **Overview:** Evaluates whether the webhook event represents a valid new transcription and calls the Fireflies GraphQL API to pull transcript metadata, participant details, and dialogue sentences.
- **Nodes Involved:** `If New Transcript Event`, `Fetch Transcript From Fireflies`, `Prepare Transcript Details`, `If Speech Detected`
- **Node Details:**
  - **If New Transcript Event**
    - *Type/Role:* Conditional router node (`n8n-nodes-base.if`).
    - *Configuration:* Evaluates `{{ $json.process }}` as true.
    - *Expressions/Variables:* `={{ $json.process }}`
    - *Connections:* Input: `Verify Settings and Event`; Output: `Fetch Transcript From Fireflies`.
    - *Edge Cases/Failures:* Non-matching event types or reprocessing tags halt execution paths silently.
  - **Fetch Transcript From Fireflies**
    - *Type/Role:* HTTP Request node (`n8n-nodes-base.httpRequest`).
    - *Configuration:* POST request to `https://api.fireflies.ai/graphql` using Bearer Token authentication.
    - *Expressions/Variables:* `={{ JSON.stringify({ query: '...', variables: { id: $json.meetingId } }) }}`
    - *Connections:* Input: `If New Transcript Event`; Output: `Prepare Transcript Details`.
    - *Credentials:* `httpBearerAuth` (Fireflies API Key).
    - *Edge Cases/Failures:* 401/403 authorization errors, API rate limit status codes (429), or missing meeting IDs return formatted workflow exceptions.
  - **Prepare Transcript Details**
    - *Type/Role:* Code node (`n8n-nodes-base.code`, JavaScript).
    - *Configuration:* Parses GraphQL output, identifies external contact emails by filtering out internal aliases, and compresses raw sentences into text blocks.
    - *Expressions/Variables:* Interrogates API payloads, calculates meeting durations, formats local timestamps, and constructs Fireflies viewer URLs.
    - *Connections:* Input: `Fetch Transcript From Fireflies`; Output: `If Speech Detected`.
    - *Edge Cases/Failures:* Malformed JSON string bodies or empty sentences arrays.
  - **If Speech Detected**
    - *Type/Role:* Conditional router node (`n8n-nodes-base.if`).
    - *Configuration:* Validates that `hasSpeech` equals true.
    - *Expressions/Variables:* `={{ $json.hasSpeech }}`
    - *Connections:* Input: `Prepare Transcript Details`; Output: `Call Analysis Agent`.
    - *Edge Cases/Failures:* Skips processing for empty transcripts or cancelled calls.

#### 2.3 AI Call Analysis
- **Overview:** Sends the processed meeting transcript to OpenAI to evaluate sales outcomes, extract action items, and draft contextual follow-up emails according to specific rule constraints.
- **Nodes Involved:** `Call Analysis Agent`, `OpenAI GPT-5 Model`, `Define Notes Format`
- **Node Details:**
  - **Call Analysis Agent**
    - *Type/Role:* LangChain AI Agent node (`@n8n/n8n-nodes-langchain.agent`).
    - *Configuration:* Configured with system instructions matching business contexts, call metrics, and email style parameters.
    - *Expressions/Variables:* `={{ Call: {{ $json.title }}... }}` and dynamic system message configurations referencing personal settings.
    - *Connections:* Input: `If Speech Detected` (main), `OpenAI GPT-5 Model` (ai_languageModel), `Define Notes Format` (ai_outputParser); Output: `If Contact Email Exists`.
    - *Edge Cases/Failures:* Model timeouts, token limit overruns on exceptionally long transcripts, or rate limiting from OpenAI API providers.
  - **OpenAI GPT-5 Model**
    - *Type/Role:* Chat Model node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`).
    - *Configuration:* Uses model `gpt-5-mini`.
    - *Expressions/Variables:* None.
    - *Connections:* Output: Connected to `Call Analysis Agent` via `ai_languageModel`.
    - *Credentials:* OpenAI API Account.
    - *Edge Cases/Failures:* Invalid API keys, depleted quota credits.
  - **Define Notes Format**
    - *Type/Role:* Structured Output Parser node (`@n8n/n8n-nodes-langchain.outputParserStructured`).
    - *Configuration:* Defines a manual JSON schema specifying properties for summary, open_questions, next_step, outcome, email_subject, and email_body.
    - *Expressions/Variables:* Strict JSON schema definition string.
    - *Connections:* Output: Connected to `Call Analysis Agent` via `ai_outputParser`.
    - *Edge Cases/Failures:* Schema validation failures if the model returns non-conforming structures.

#### 2.4 Email Draft Generation
- **Overview:** Checks whether a valid external contact email was found in the transcript metadata, branching the flow to create a Gmail draft either addressed to the lead or left blank.
- **Nodes Involved:** `If Contact Email Exists`, `Create Follow-up Draft in Gmail`, `Create Draft Without Recipient`
- **Node Details:**
  - **If Contact Email Exists**
    - *Type/Role:* Conditional router node (`n8n-nodes-base.if`).
    - *Configuration:* Evaluates whether `contactEmail` is non-empty.
    - *Expressions/Variables:* `={{ $('Prepare Transcript Details').first().json.contactEmail }}`
    - *Connections:* Input: `Call Analysis Agent`; Outputs: `Create Follow-up Draft in Gmail` (true path), `Create Draft Without Recipient` (false path).
    - *Edge Cases/Failures:* Malformed email formats or missing participant data.
  - **Create Follow-up Draft in Gmail**
    - *Type/Role:* Gmail integration node (`n8n-nodes-base.gmail`).
    - *Configuration:* Resource `draft`, mapping generated email bodies and subjects with recipient headers set to contact addresses.
    - *Expressions/Variables:* `={{ $('Call Analysis Agent').first().json.output.email_body }}` and `={{ $('Prepare Transcript Details').first().json.contactEmail }}`
    - *Connections:* Input: `If Contact Email Exists` (true branch); Output: `Prepare Log Entry Data`.
    - *Credentials:* Gmail OAuth2 account.
    - *Edge Cases/Failures:* OAuth token expiration, quota exceeded limits, or invalid recipient addresses.
  - **Create Draft Without Recipient**
    - *Type/Role:* Gmail integration node (`n8n-nodes-base.gmail`).
    - *Configuration:* Resource `draft`, creating drafts without the `sendTo` option populated.
    - *Expressions/Variables:* `={{ $('Call Analysis Agent').first().json.output.email_body }}`
    - *Connections:* Input: `If Contact Email Exists` (false branch); Output: `Prepare Log Entry Data`.
    - *Credentials:* Gmail OAuth2 account.
    - *Edge Cases/Failures:* Authorization errors.

#### 2.5 Logging & Internal Notification
- **Overview:** Normalizes transcript and AI analysis outputs into tabular column mappings, appends them to a designated Google Sheet, and emails an internal summary report to the workflow owner.
- **Nodes Involved:** `Prepare Log Entry Data`, `Append Call Log to Sheets`, `Send Call Notes to Self`
- **Node Details:**
  - **Prepare Log Entry Data**
    - *Type/Role:* Set/Edit Fields node (`n8n-nodes-base.set`).
    - *Configuration:* Maps keys (`date`, `call`, `contact_name`, `contact_email`, `outcome`, `summary`, `next_step`, `open_questions`, `follow_up_draft`, `fireflies_link`) to downstream values.
    - *Expressions/Variables:* References data from `Prepare Transcript Details` and `Call Analysis Agent`.
    - *Connections:* Input: `Create Follow-up Draft in Gmail` or `Create Draft Without Recipient`; Output: `Append Call Log to Sheets`.
    - *Edge Cases/Failures:* Undefined field references if upstream JSON properties mismatch.
  - **Append Call Log to Sheets**
    - *Type/Role:* Google Sheets node (`n8n-nodes-base.googleSheets`).
    - *Configuration:* Operation `append`, mapping input items automatically with RAW cell formatting to a target document URL.
    - *Expressions/Variables:* `={{ $('Set Personal Settings').first().json.callLogSheetUrl }}`
    - *Connections:* Input: `Prepare Log Entry Data`; Output: `Send Call Notes to Self`.
    - *Credentials:* Google Sheets OAuth2 account.
    - *Edge Cases/Failures:* Sheet permission revocation, missing column headers, or invalid spreadsheet URLs.
  - **Send Call Notes to Self**
    - *Type/Role:* Gmail integration node (`n8n-nodes-base.gmail`).
    - *Configuration:* Sends plain-text notification emails to the owner's primary email address detailing call outcomes, summaries, next steps, and status warnings.
    - *Expressions/Variables:* Evaluates string templates combining settings and AI parser outputs.
    - *Connections:* Input: `Append Call Log to Sheets`; Output: None (Terminal node).
    - *Credentials:* Gmail OAuth2 account.
    - *Edge Cases/Failures:* Invalid recipient parsing or email delivery failures.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Call Transcribed | n8n-nodes-base.webhook | Catches Fireflies webhook requests | None | Set Personal Settings | Draft sales call follow-ups from Fireflies transcripts with OpenAI and Gmail<br><br>### How it works<br><br>1. Fireflies calls the webhook when a call is transcribed.<br>2. It checks your settings, fetches the transcript and picks the first attendee email that is not yours.<br>3. OpenAI writes a summary, open questions, a next step, an outcome and a follow-up email.<br>4. The follow-up is saved as a Gmail draft, never sent.<br>5. The call is logged in Google Sheets and the notes are emailed to you.<br><br>### Setup steps<br><br>- [ ] Add your Fireflies API key (Integrations, then Fireflies API) as a Bearer Auth credential on **Fetch Transcript From Fireflies**.<br>- [ ] Connect OpenAI, Gmail and Google Sheets credentials.<br>- [ ] Create a blank Google Sheet. Headers are added on the first run.<br>- [ ] Fill in **Set Personal Settings**, including the sheet link. Separate several emails of yours with commas.<br>- [ ] Publish, then copy the Production URL from **When Call Transcribed**.<br>- [ ] In Fireflies webhooks (app.fireflies.ai/integrations/api/webhook), paste the URL and tick **meeting.transcribed**.<br><br>### Customization<br><br>Edit **Call Analysis Agent** to change the outcomes or tone, add columns in **Prepare Log Entry Data**, or send the notes to Slack instead of email. |
| Set Personal Settings | n8n-nodes-base.set | Assigns user profile configurations | When Call Transcribed | Verify Settings and Event | Draft sales call follow-ups from Fireflies transcripts with OpenAI and Gmail<br><br>### How it works<br><br>1. Fireflies calls the webhook when a call is transcribed.<br>2. It checks your settings, fetches the transcript and picks the first attendee email that is not yours.<br>3. OpenAI writes a summary, open questions, a next step, an outcome and a follow-up email.<br>4. The follow-up is saved as a Gmail draft, never sent.<br>5. The call is logged in Google Sheets and the notes are emailed to you.<br><br>### Setup steps<br><br>- [ ] Add your Fireflies API key (Integrations, then Fireflies API) as a Bearer Auth credential on **Fetch Transcript From Fireflies**.<br>- [ ] Connect OpenAI, Gmail and Google Sheets credentials.<br>- [ ] Create a blank Google Sheet. Headers are added on the first run.<br>- [ ] Fill in **Set Personal Settings**, including the sheet link. Separate several emails of yours with commas.<br>- [ ] Publish, then copy the Production URL from **When Call Transcribed**.<br>- [ ] In Fireflies webhooks (app.fireflies.ai/integrations/api/webhook), paste the URL and tick **meeting.transcribed**.<br><br>### Customization<br><br>Edit **Call Analysis Agent** to change the outcomes or tone, add columns in **Prepare Log Entry Data**, or send the notes to Slack instead of email. |
| Verify Settings and Event | n8n-nodes-base.code | Validates settings and event payloads | Set Personal Settings | If New Transcript Event | Receive and validate event<br><br>Catches the Fireflies webhook and loads your settings. If a setting is still a placeholder, the run stops with a message saying what to fill in. |
| If New Transcript Event | n8n-nodes-base.if | Routes valid new webhook events | Verify Settings and Event | Fetch Transcript From Fireflies | Fetch transcript details<br><br>Only meeting.transcribed events continue; resends and other events stop here. It fetches the transcript and attendees and picks the first email that is not yours. A meeting Fireflies cannot find, or a call with no speech, ends the run quietly. |
| Fetch Transcript From Fireflies | n8n-nodes-base.httpRequest | Pulls full transcript data via GraphQL | If New Transcript Event | Prepare Transcript Details | Fetch transcript details<br><br>Only meeting.transcribed events continue; resends and other events stop here. It fetches the transcript and attendees and picks the first email that is not yours. A meeting Fireflies cannot find, or a call with no speech, ends the run quietly. |
| Prepare Transcript Details | n8n-nodes-base.code | Extracts participant data and plain-text dialogue | Fetch Transcript From Fireflies | If Speech Detected | Fetch transcript details<br><br>Only meeting.transcribed events continue; resends and other events stop here. It fetches the transcript and attendees and picks the first email that is not yours. A meeting Fireflies cannot find, or a call with no speech, ends the run quietly. |
| If Speech Detected | n8n-nodes-base.if | Verifies that sentences exist in the call | Prepare Transcript Details | Call Analysis Agent | Fetch transcript details<br><br>Only meeting.transcribed events continue; resends and other events stop here. It fetches the transcript and attendees and picks the first email that is not yours. A meeting Fireflies cannot find, or a call with no speech, ends the run quietly. |
| OpenAI GPT-5 Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides the underlying LLM | None | Call Analysis Agent | Analyze call with AI<br><br>OpenAI reads the transcript and returns a summary, open questions, the next step, an outcome and a follow-up email in a fixed format. It only uses facts from the transcript. |
| Define Notes Format | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces strict JSON schema parsing | None | Call Analysis Agent | Analyze call with AI<br><br>OpenAI reads the transcript and returns a summary, open questions, the next step, an outcome and a follow-up email in a fixed format. It only uses facts from the transcript. |
| Call Analysis Agent | @n8n/n8n-nodes-langchain.agent | Analyzes conversation text and generates email content | If Speech Detected, OpenAI GPT-5 Model, Define Notes Format | If Contact Email Exists | Analyze call with AI<br><br>OpenAI reads the transcript and returns a summary, open questions, the next step, an outcome and a follow-up email in a fixed format. It only uses facts from the transcript. |
| If Contact Email Exists | n8n-nodes-base.if | Checks if a contact email address is available | Call Analysis Agent | Create Follow-up Draft in Gmail, Create Draft Without Recipient | Draft follow-up email<br><br>Creates a Gmail draft for you to check and send. It is addressed to the contact from the attendee list, or has no recipient if none was found. Nothing is sent automatically. |
| Create Follow-up Draft in Gmail | n8n-nodes-base.gmail | Creates a Gmail draft addressed to the lead | If Contact Email Exists | Prepare Log Entry Data | Draft follow-up email<br><br>Creates a Gmail draft for you to check and send. It is addressed to the contact from the attendee list, or has no recipient if none was found. Nothing is sent automatically. |
| Create Draft Without Recipient | n8n-nodes-base.gmail | Creates a recipient-less Gmail draft | If Contact Email Exists | Prepare Log Entry Data | Draft follow-up email<br><br>Creates a Gmail draft for you to check and send. It is addressed to the contact from the attendee list, or has no recipient if none was found. Nothing is sent automatically. |
| Prepare Log Entry Data | n8n-nodes-base.set | Formats tabular schema values for logging | Create Follow-up Draft in Gmail, Create Draft Without Recipient | Append Call Log to Sheets | Log and notify owner<br><br>Builds the log row, appends it to your Google Sheet and emails you the call notes. |
| Append Call Log to Sheets | n8n-nodes-base.googleSheets | Appends rows into a Google Sheet | Prepare Log Entry Data | Send Call Notes to Self | Log and notify owner<br><br>Builds the log row, appends it to your Google Sheet and emails you the call notes. |
| Send Call Notes to Self | n8n-nodes-base.gmail | Sends a digest email to the workflow owner | Append Call Log to Sheets | None | Log and notify owner<br><br>Builds the log row, appends it to your Google Sheet and emails you the call notes. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node named `When Call Transcribed`.
   - Set HTTP Method to `POST`, and Path to `fireflies-call-notes`.
2. **Configure Personal Settings:**
   - Add a **Set** node named `Set Personal Settings`.
   - Create manual assignments for: `yourName` (string), `businessName` (string), `yourEmail` (string), `whatYouSell` (string), `bookingLink` (string), and `callLogSheetUrl` (string).
3. **Add Validation Code:**
   - Add a **Code** node named `Verify Settings and Event`.
   - Set mode to `Run Once for All Items` and paste the verification JavaScript code checking placeholder strings and webhook parameters.
4. **Setup Event Branching:**
   - Add an **If** node named `If New Transcript Event`.
   - Configure a condition testing `={{ $json.process }}` equals `true`.
5. **Fetch Transcript Data:**
   - Add an **HTTP Request** node named `Fetch Transcript From Fireflies`.
   - Set Method to `POST`, URL to `https://api.fireflies.ai/graphql`, specify body as `JSON`, and configure authentication type to `Header Auth` (Bearer token).
   - Set JSON body expression querying transcript ID, title, date, duration, participants, attendees, and sentences.
6. **Parse Transcript Details:**
   - Add a **Code** node named `Prepare Transcript Details`.
   - Insert script logic parsing errors, identifying external contacts, filtering out internal emails, and formatting conversation dialogue into plain text.
7. **Verify Speech Content:**
   - Add an **If** node named `If Speech Detected`.
   - Condition: `={{ $json.hasSpeech }}` equals `true`.
8. **Configure AI Agent and Models:**
   - Add an **OpenAI Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) configured with model `gpt-5-mini` using OpenAI credentials.
   - Add a **Structured Output Parser** node (`@n8n/n8n-nodes-langchain.outputParserStructured`) defining the JSON schema containing `summary`, `open_questions`, `next_step`, `outcome`, `email_subject`, and `email_body`.
   - Add an **AI Agent** node (`@n8n/n8n-nodes-langchain.agent`) named `Call Analysis Agent`. Connect the Chat Model and Output Parser to its respective AI inputs. Configure its system message with business rules and styling constraints.
9. **Branch for Email Recipient Presence:**
   - Add an **If** node named `If Contact Email Exists`.
   - Condition: `={{ $('Prepare Transcript Details').first().json.contactEmail }}` is not empty.
10. **Create Gmail Draft Nodes:**
    - Add a **Gmail** node named `Create Follow-up Draft in Gmail` (Resource: `Draft`, Subject mapped from agent output, Message mapped from agent body, and SendTo mapped from contact email). Configure Gmail OAuth2 credentials.
    - Add a second **Gmail** node named `Create Draft Without Recipient` (Resource: `Draft`, Subject and Message mapped from agent output, leaving recipient blank).
11. **Format Log Entries:**
    - Add a **Set** node named `Prepare Log Entry Data`.
    - Map output fields (`date`, `call`, `contact_name`, `contact_email`, `outcome`, `summary`, `next_step`, `open_questions`, `follow_up_draft`, `fireflies_link`) to respective upstream paths.
12. **Append to Google Sheets:**
    - Add a **Google Sheets** node named `Append Call Log to Sheets`.
    - Set operation to `Append`, reference `callLogSheetUrl` dynamically, and configure Google Sheets OAuth2 credentials.
13. **Send Internal Digest:**
    - Add a **Gmail** node named `Send Call Notes to Self`.
    - Configure recipient email targeting the primary owner address, mapping subject titles and plain-text body summaries. Connect Gmail OAuth2 credentials.
14. **Wire Connections:**
    - Connect nodes sequentially following the direct node dependencies detailed in Section 2.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Fireflies Webhook Configuration Portal | [Fireflies Integrations API Webhook](https://app.fireflies.ai/integrations/api/webhook) |
| Fireflies Transcript Viewer Base URL | `https://app.fireflies.ai/view/` |
| n8n Built-in Execution Order | Execution engine version `v1` |
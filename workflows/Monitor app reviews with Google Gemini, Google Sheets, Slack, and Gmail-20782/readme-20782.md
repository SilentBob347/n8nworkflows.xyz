Monitor app reviews with Google Gemini, Google Sheets, Slack, and Gmail

https://n8nworkflows.xyz/workflows/monitor-app-reviews-with-google-gemini--google-sheets--slack--and-gmail-20782


# Monitor app reviews with Google Gemini, Google Sheets, Slack, and Gmail

### 1. Workflow Overview

This workflow is designed to automate the collection, analysis, categorization, and reporting of mobile app reviews. Its primary use case is to help customer support, product, and engineering teams monitor user sentiment across app stores without manual intervention, ensuring critical issues receive immediate attention.

The logic is organized into three functional blocks:
- **1.1 Input Reception & Normalization:** Manually triggers the execution, pulls raw review data from an HTTP endpoint, and transforms nested JSON arrays into flat, standardized individual review objects.
- **1.2 AI Sentiment & Category Processing:** Utilizes Google Gemini models sequentially to evaluate the sentiment (score, confidence, and reasoning) and categorize the review type (e.g., Bug, Payment, Crash).
- **1.3 Filtering, Storage & Alerting:** Evaluates whether reviews are negative, logs them to Google Sheets, filters critical complaints to trigger Slack notifications, generates an AI-suggested customer response, and emails the summary to stakeholders.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
**Overview:** This block acts as the entry point of the workflow, retrieving raw review arrays from a designated API endpoint and flattening them into structured items for individual processing.

**Nodes Involved:**
- When clicking ‘Execute workflow’
- HTTP Request
- Code in JavaScript

**Node Details:**
- **When clicking ‘Execute workflow’**
  - *Type & Role:* Manual Trigger (`n8n-nodes-base.manualTrigger`). Initiates workflow runs on-demand.
  - *Configuration:* Default parameters.
  - *Inputs/Outputs:* Out: `HTTP Request`.
  - *Edge Cases:* None; manual triggers do not fail.
- **HTTP Request**
  - *Type & Role:* HTTP Request (`n8n-nodes-base.httpRequest`). Fetches review data from an external endpoint.
  - *Configuration:* Sends a `GET` request to `http://192.168.1.100:8000/reviews`.
  - *Inputs/Outputs:* In: `When clicking ‘Execute workflow’` | Out: `Code in JavaScript`.
  - *Edge Cases:* Network timeouts, incorrect endpoint URLs, or unreachable local servers (requires local network IP instead of `localhost` if running in Docker/n8n cloud).
- **Code in JavaScript**
  - *Type & Role:* Code Node (`n8n-nodes-base.code`). Normalizes nested API responses into flat items containing properties like `review_id`, `review_text`, `rating`, `app_version_code`, `device`, and `platform`.
  - *Configuration:* Custom JavaScript loops through incoming review items and extracts specific comment and device metadata attributes.
  - *Inputs/Outputs:* In: `HTTP Request` | Out: `Message a model`.
  - *Edge Cases:* Empty review arrays or altered API schemas causing missing fields.

---

#### 2.2 AI Sentiment & Category Processing
**Overview:** This block passes the normalized review text through two successive Google Gemini models to determine user sentiment (positive, neutral, negative) and assign a specific support category (e.g., Bug, Payment, Crash), parsing the outputs into structured JSON objects.

**Nodes Involved:**
- Message a model
- Code in JavaScript1
- Message a model1
- Code in JavaScript2

**Node Details:**
- **Message a model**
  - *Type & Role:* Google Gemini (`@n8n/n8n-nodes-langchain.googleGemini`). Evaluates the sentiment of the review.
  - *Configuration:* Uses model `models/gemini-3-flash-preview`. Prompt requires strictly formatted JSON output containing sentiment, sentiment score (-1 to 1), confidence, and reason.
  - *Credentials:* `Google Gemini (MaryPhelps) AiStudio Builder AWS` (`googlePalmApi`).
  - *Inputs/Outputs:* In: `Code in JavaScript` | Out: `Code in JavaScript1`.
  - *Edge Cases:* API rate limits, authentication errors, or the model returning markdown-wrapped text instead of raw JSON.
- **Code in JavaScript1**
  - *Type & Role:* Code Node (`n8n-nodes-base.code`). Parses the stringified JSON output from the first Gemini model.
  - *Configuration:* Strips markdown code fences (e.g., ```json ... ```) and executes `JSON.parse()`.
  - *Inputs/Outputs:* In: `Message a model` | Out: `Message a model1`.
  - *Edge Cases:* Malformed JSON strings resulting in parser failures.
- **Message a model1**
  - *Type & Role:* Google Gemini (`@n8n/n8n-nodes-langchain.googleGemini`). Classifies the review into a specific support category.
  - *Configuration:* Uses model `models/gemini-2.5-flash`. Forces selection from a predefined list (Bug, Performance, Login, Payment, UI/UX, Feature Request, Crash, Other). References review text via expression: `{{ $('Code in JavaScript').item.json.review_text }}`.
  - *Credentials:* `Google Gemini (MaryPhelps) AiStudio Builder AWS` (`googlePalmApi`).
  - *Inputs/Outputs:* In: `Code in JavaScript1` | Out: `Code in JavaScript2`.
  - *Edge Cases:* Invalid category strings returned by the model.
- **Code in JavaScript2**
  - *Type & Role:* Code Node (`n8n-nodes-base.code`). Parses the categorization response from the second Gemini model.
  - *Configuration:* Converts Gemini's text response into a structured JavaScript object containing `category`, `category_confidence`, and `reason`.
  - *Inputs/Outputs:* In: `Message a model1` | Out: `If`.
  - *Edge Cases:* Missing text fields in the model response object.

---

#### 2.3 Filtering, Storage & Alerting
**Overview:** This block processes negative reviews by storing them in a spreadsheet, evaluates if they meet critical thresholds to trigger Slack alerts, generates a draft response via AI, and emails the summary to support staff.

**Nodes Involved:**
- If
- Append or update row in sheet
- If1
- Send a message
- Message a model2
- Send a message1

**Node Details:**
- **If**
  - *Type & Role:* If Conditional (`n8n-nodes-base.if`). Filters reviews to continue only if the sentiment equals "Negative".
  - *Configuration:* Condition checks if `{{ $('Code in JavaScript1').item.json.sentiment }}` equals `Negative`.
  - *Inputs/Outputs:* In: `Code in JavaScript2` | Out: `Append or update row in sheet`.
  - *Edge Cases:* Sentiment label mismatches (case sensitivity issues).
- **Append or update row in sheet**
  - *Type & Role:* Google Sheets (`n8n-nodes-base.googleSheets`). Logs or updates review data in a connected spreadsheet.
  - *Configuration:* Operation set to `appendOrUpdate`, matching on `Review ID`. Maps columns: `Review ID`, `Platform`, `Rating`, `Review`, `Reviewer`, `Review Date`, `Sentimate`, `Category`, and sets `Suggested reply` to "Pending".
  - *Document ID:* `1XQYlY6eqen4MGalDtXKCi4siIpgoT5ZoG19Of5Uo31k` (Sheet1).
  - *Inputs/Outputs:* In: `If` | Out: `If1`.
  - *Edge Cases:* Mismatched column names or revoked spreadsheet access permissions.
- **If1**
  - *Type & Role:* If Conditional (`n8n-nodes-base.if`). Determines if the negative review meets criteria for escalation.
  - *Configuration:* Combines conditions with an OR operator: checks if rating is `>= 1` or category equals `Payment`.
  - *Inputs/Outputs:* In: `Append or update row in sheet` | Out: `Send a message`.
  - *Edge Cases:* Evaluation logic misconfigurations skipping critical alerts.
- **Send a message (Slack)**
  - *Type & Role:* Slack (`n8n-nodes-base.slack`). Posts an alert notification to a designated channel.
  - *Configuration:* Selects channel by ID `C0ATZFFKRUH` (`n8n-general`). Formats message text using review details, rating, category, and review snippet.
  - *Inputs/Outputs:* In: `If1` | Out: `Message a model2`.
  - *Edge Cases:* Expired Slack credentials or missing bot scopes for the channel.
- **Message a model2**
  - *Type & Role:* Google Gemini (`@n8n/n8n-nodes-langchain.googleGemini`). Drafts a polite, professional suggested customer response.
  - *Configuration:* Uses model `models/gemini-3-flash-preview`. Constrained rules: under 80 words, acknowledges problem, apologizes, and avoids blaming customers or promising release dates.
  - *Credentials:* `Google Gemini (MaryPhelps) AiStudio Builder AWS` (`googlePalmApi`).
  - *Inputs/Outputs:* In: `Send a message` | Out: `Send a message1`.
  - *Edge Cases:* Generation timeouts or empty prompt contexts.
- **Send a message1 (Gmail)**
  - *Type & Role:* Gmail (`n8n-nodes-base.gmail`). Emails review details and the AI-generated draft response to an administrator or team.
  - *Configuration:* Sets recipient (`sendTo`) to `user@example.com`, subject line with category, and HTML body mapping review details alongside Gemini's response text (`{{ $json.content.parts[0].text }}`).
  - *Inputs/Outputs:* In: `Message a model2` | Out: None (Terminal node).
  - *Edge Cases:* Gmail quota limits, authentication token expiration, or invalid recipient addresses.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Execute workflow’ | n8n-nodes-base.manualTrigger | Initiates workflow manually. | None | HTTP Request | Triggerr, HTTP , Code<br>1. Trigger node every day and call API of ANdroi dreview<br>2. Code node formatted JSON with basic data like review id, reviewr name, review tex, rating etc. |
| HTTP Request | n8n-nodes-base.httpRequest | Fetches review payload from HTTP API. | When clicking ‘Execute workflow’ | Code in JavaScript | Triggerr, HTTP , Code<br>1. Trigger node every day and call API of ANdroi dreview<br>2. Code node formatted JSON with basic data like review id, reviewr name, review tex, rating etc. |
| Code in JavaScript | n8n-nodes-base.code | Normalizes raw review items into flat objects. | HTTP Request | Message a model | Triggerr, HTTP , Code<br>1. Trigger node every day and call API of ANdroi dreview<br>2. Code node formatted JSON with basic data like review id, reviewr name, review tex, rating etc. |
| Message a model | @n8n/n8n-nodes-langchain.googleGemini | Analyzes review sentiment via Gemini. | Code in JavaScript | Code in JavaScript1 | Gemini AI , Code<br>1. Send review resposne to AI modal Gemeni and extract criticality and actual issue on that<br>2. Second Gemini MOdal decice category , sentiment score and Reason text with detail lable |
| Code in JavaScript1 | n8n-nodes-base.code | Parses sentiment JSON output. | Message a model | Message a model1 | Gemini AI , Code<br>1. Send review resposne to AI modal Gemeni and extract criticality and actual issue on that<br>2. Second Gemini MOdal decice category , sentiment score and Reason text with detail lable |
| Message a model1 | @n8n/n8n-nodes-langchain.googleGemini | Classifies review into a support category. | Code in JavaScript1 | Code in JavaScript2 | Gemini AI , Code<br>1. Send review resposne to AI modal Gemeni and extract criticality and actual issue on that<br>2. Second Gemini MOdal decice category , sentiment score and Reason text with detail lable |
| Code in JavaScript2 | n8n-nodes-base.code | Parses category JSON output. | Message a model1 | If | Gemini AI , Code<br>1. Send review resposne to AI modal Gemeni and extract criticality and actual issue on that<br>2. Second Gemini MOdal decice category , sentiment score and Reason text with detail lable |
| If | n8n-nodes-base.if | Filters for negative sentiment reviews. | Code in JavaScript2 | Append or update row in sheet | IF, Google Sheet, Slack, Gmail<br>1. If review is Negative then Procced firhter and store in App review sheet with required column<br>2. After Negative review detact send Slack message to respective team group<br>3. AI modal gemini prepare email body to send team email with category, casuse and review text<br>4. After modal generate response send email via Gmail node |
| Append or update row in sheet | n8n-nodes-base.googleSheets | Logs review details to Google Sheets. | If | If1 | IF, Google Sheet, Slack, Gmail<br>1. If review is Negative then Procced firhter and store in App review sheet with required column<br>2. After Negative review detact send Slack message to respective team group<br>3. AI modal gemini prepare email body to send team email with category, casuse and review text<br>4. After modal generate response send email via Gmail node |
| If1 | n8n-nodes-base.if | Evaluates critical thresholds. | Append or update row in sheet | Send a message | IF, Google Sheet, Slack, Gmail<br>1. If review is Negative then Procced firhter and store in App review sheet with required column<br>2. After Negative review detact send Slack message to respective team group<br>3. AI modal gemini prepare email body to send team email with category, casuse and review text<br>4. After modal generate response send email via Gmail node |
| Send a message | n8n-nodes-base.slack | Sends team alert to Slack. | If1 | Message a model2 | IF, Google Sheet, Slack, Gmail<br>1. If review is Negative then Procced firhter and store in App review sheet with required column<br>2. After Negative review detact send Slack message to respective team group<br>3. AI modal gemini prepare email body to send team email with category, casuse and review text<br>4. After modal generate response send email via Gmail node |
| Message a model2 | @n8n/n8n-nodes-langchain.googleGemini | Generates suggested customer reply. | Send a message | Send a message1 | IF, Google Sheet, Slack, Gmail<br>1. If review is Negative then Procced firhter and store in App review sheet with required column<br>2. After Negative review detact send Slack message to respective team group<br>3. AI modal gemini prepare email body to send team email with category, casuse and review text<br>4. After modal generate response send email via Gmail node |
| Send a message1 | n8n-nodes-base.gmail | Sends review report and reply via Gmail. | Message a model2 | None | IF, Google Sheet, Slack, Gmail<br>1. If review is Negative then Procced firhter and store in App review sheet with required column<br>2. After Negative review detact send Slack message to respective team group<br>3. AI modal gemini prepare email body to send team email with category, casuse and review text<br>4. After modal generate response send email via Gmail node |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Point:** Add a **Manual Trigger** node (`When clicking ‘Execute workflow’`).
2. **Configure Data Source:** Add an **HTTP Request** node. Set the method to `GET` and URL to your reviews API endpoint (e.g., `http://192.168.1.100:8000/reviews`). Connect the Trigger to this node.
3. **Normalize Payload:** Add a **Code** node in JavaScript. Paste the normalization script that loops through `.reviews` arrays and flattens metadata (`review_id`, `review_text`, `rating`, `platform`, etc.). Connect the HTTP Request to this node.
4. **Setup Sentiment Analysis (Gemini):**
   - Add a **Google Gemini** node (`Message a model`).
   - Configure credentials using a Google Palm API key (`Google Gemini (MaryPhelps) AiStudio Builder AWS`).
   - Select model: `models/gemini-3-flash-preview`.
   - Add a prompt requesting strict JSON output with `sentiment`, `sentiment_score`, `confidence`, and `reason`.
   - Connect the Code node output to this Gemini node.
5. **Parse Sentiment Response:** Add a **Code** node (`Code in JavaScript1`) to strip markdown fences (` ```json `) and parse the JSON string. Connect the sentiment Gemini node to it.
6. **Setup Category Classification (Gemini):**
   - Add a second **Google Gemini** node (`Message a model1`).
   - Use the same Gemini credentials.
   - Select model: `models/gemini-2.5-flash`.
   - In the prompt, reference `{{ $('Code in JavaScript').item.json.review_text }}` and restrict classification categories (Bug, Performance, Login, Payment, UI/UX, Feature Request, Crash, Other).
   - Connect the sentiment-parsing code node to this classification node.
7. **Parse Category Response:** Add a third **Code** node (`Code in JavaScript2`) to parse the category JSON response. Connect the classification Gemini node to it.
8. **Filter Negative Reviews:** Add an **If** node. Configure a condition to check if `{{ $('Code in JavaScript1').item.json.sentiment }}` equals `Negative`. Connect the category-parsing code node to this If node.
9. **Log to Google Sheets:**
   - Add a **Google Sheets** node (`Append or update row in sheet`).
   - Connect required Google Sheets OAuth2 credentials.
   - Set operation to `appendOrUpdate`, matching columns on `Review ID`. Map sheet columns to corresponding expression fields (`Rating`, `Review`, `Category`, `Platform`, `Reviewer`, `Review ID`, `Sentimate`, `Review Date`).
   - Connect the `true` branch of the sentiment If node to this node.
10. **Evaluate Critical Alerts:** Add a second **If** node (`If1`). Set an OR condition checking if rating `>= 1` or category equals `Payment`. Connect the Google Sheets node to this If node.
11. **Alert Slack Team:** Add a **Slack** node (`Send a message`). Connect Slack credentials, choose channel selection method by list, specify channel ID (`C0ATZFFKRUH`), and construct the notification text using review metadata expressions. Connect the `true` branch of the critical If node to this Slack node.
12. **Generate AI Suggested Reply:**
   - Add a fourth **Google Gemini** node (`Message a model2`).
   - Use the same Gemini credentials and model `models/gemini-3-flash-preview`.
   - Provide prompt instructions limiting responses to under 80 words, maintaining a professional tone without blaming customers.
   - Connect the Slack node to this Gemini node.
13. **Send Summary via Email:** Add a **Gmail** node (`Send a message1`). Connect Gmail credentials, supply recipient email (`user@example.com`), map the subject line dynamically, and assemble an HTML email body incorporating review data and Gemini's output text (`{{ $json.content.parts[0].text }}`). Connect the final Gemini node to this Gmail node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| App Store / Play Store Review Monitor Overview | Automates collection, sentiment analysis, complaint classification, Google Sheets logging, Slack alerts, and AI reply generation. |
| Professional n8n Development Support | If you need help customizing or extending this workflow, connecting real Google Play/App Store APIs, or monitoring multiple apps, assistance is available from [WeblineIndia's n8n development team](https://www.weblineindia.com/hire-n8n-developers/). |
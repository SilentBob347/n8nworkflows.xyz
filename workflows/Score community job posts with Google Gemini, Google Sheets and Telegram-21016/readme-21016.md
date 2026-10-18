Score community job posts with Google Gemini, Google Sheets and Telegram

https://n8nworkflows.xyz/workflows/score-community-job-posts-with-google-gemini--google-sheets-and-telegram-21016


# Score community job posts with Google Gemini, Google Sheets and Telegram

### 1. Workflow Overview

This workflow automates the monitoring of the n8n community forum’s "Jobs" category by polling its RSS feed every 15 minutes. It identifies unrated posts, tracks them to prevent duplicate processing using Google Sheets, evaluates their relevance and quality via Google Gemini, alerts the user on Telegram for high-value leads, and logs all evaluations back to Google Sheets.

The architecture is divided into three primary functional blocks:
- **1.1 Input Reception & Deduplication:** Periodically triggers, reads already-seen records from Google Sheets, fetches new RSS items, and filters out previously processed posts.
- **1.2 AI Processing:** Persists new post IDs immediately as "seen", structures the payload for Google Gemini according to custom profile rules, sends the batched API request, and parses the returned JSON rating.
- **1.3 Alerting & Logging:** Evaluates the AI's score and request flags, dispatches high-scoring leads to a Telegram chat, and comprehensively logs every analyzed post to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Deduplication
- **Overview:** This block handles the scheduled initialization, retrieves historical processing states, pulls the latest forum entries from the RSS feed, and isolates genuinely new items.
- **Nodes Involved:** 
  - `Every 15 minutes`
  - `Read seen post IDs`
  - `Fetch jobs RSS feed`
  - `Keep only new posts`
- **Node Details:**
  - **Every 15 minutes**
    - *Type and technical role:* Schedule Trigger. Initiates the workflow execution at a fixed interval.
    - *Configuration choices:* Interval-based rule set to execute every 15 minutes.
    - *Key expressions or variables:* None.
    - *Input / Output:* Output connects to `Read seen post IDs`.
    - *Edge cases:* None.
  - **Read seen post IDs**
    - *Type and technical role:* Google Sheets Node. Reads existing rows from a tracking tab to establish a memory of processed posts.
    - *Configuration choices:* Resource: `sheet`, Operation: `read`, Sheet Name: `seen`, Document ID: configured via spreadsheet URL (`YOUR_SHEET_ID`). `alwaysOutputData` is enabled to ensure subsequent nodes function even if the sheet is initially empty.
    - *Key expressions or variables:* None.
    - *Input / Output:* Input from `Every 15 minutes`; output connects to `Fetch jobs RSS feed`.
    - *Edge cases:* Authentication failure if Google OAuth2 credentials are invalid; missing sheet name will throw a runtime error.
  - **Fetch jobs RSS feed**
    - *Type and technical role:* RSS Feed Read Node. Fetches the latest community job postings.
    - *Configuration choices:* URL points to `https://community.n8n.io/c/jobs/13.rss`. `executeOnce` set to true.
    - *Key expressions or variables:* None.
    - *Input / Output:* Input from `Read seen post IDs`; output connects to `Keep only new posts`.
    - *Edge cases:* Network timeouts or feed unavailability will cause the node execution to fail.
  - **Keep only new posts**
    - *Type and technical role:* Code Node (JavaScript). Cross-references fetched RSS items against the list of seen post IDs, extracts unique identifiers, truncates descriptions, and filters out duplicates.
    - *Configuration choices:* Uses a JavaScript function parsing IDs via regex matching on the item links.
    - *Key expressions or variables:* `$('Read seen post IDs').all()`, `i.json.link`, `i.json.contentSnippet`.
    - *Input / Output:* Input from `Fetch jobs RSS feed`; output connects to `Mark posts as seen`.
    - *Edge cases:* Changes in the RSS link structure could break the regex ID extractor (`/\/(\d+)\/?$/`).

#### 2.2 AI Processing
- **Overview:** This block pre-saves new post IDs to prevent duplicate processing in future runs, prepares custom system prompts tailored to a specific freelancer profile, interacts with Google Gemini via HTTP request with rate-limiting, and standardizes the AI's JSON output.
- **Nodes Involved:**
  - `Mark posts as seen`
  - `Build AI request`
  - `Score post with Gemini`
  - `Parse AI rating`
- **Node Details:**
  - **Mark posts as seen**
    - *Type and technical role:* Google Sheets Node. Appends new post identifiers and dates to the tracking sheet before processing occurs.
    - *Configuration choices:* Resource: `sheet`, Operation: `append`, Sheet Name: `seen`.
    - *Key expressions or variables:* `id: '={{ $json.id }}'`, `date: '={{ $now.toISODate() }}'`.
    - *Input / Output:* Input from `Keep only new posts`; output connects to `Build AI request`.
    - *Edge cases:* Write permission errors or API rate limits on Google Sheets.
  - **Build AI request**
    - *Type and technical role:* Code Node (JavaScript). Constructs structured system instructions and user contents for the Gemini API call based on profile constraints and preferences.
    - *Configuration choices:* Defines `MY_PROFILE` and `PREFERRED_LANGUAGE` constants inside the script.
    - *Key expressions or variables:* `$('Keep only new posts').all()`, `i.json.title`, `i.json.text`.
    - *Input / Output:* Input from `Mark posts as seen`; output connects to `Score post with Gemini`.
    - *Edge cases:* None.
  - **Score post with Gemini**
    - *Type and technical role:* HTTP Request Node. Sends evaluation payloads to the Google Gemini Generative Language API.
    - *Configuration choices:* Method: `POST`, URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-flash-latest:generateContent`. Uses predefined Google Palm API credentials. Configured with batching (`batchSize: 1`, `batchInterval: 7000`) to respect API free-tier rate limits.
    - *Key expressions or variables:* `={{ JSON.stringify($json.geminiBody) }}`.
    - *Input / Output:* Input from `Build AI request`; output connects to `Parse AI rating`.
    - *Edge cases:* API quota exhaustion, invalid API keys, or timeout errors from Gemini.
  - **Parse AI rating**
    - *Type and technical role:* Code Node (JavaScript). Reads the raw candidate responses from Gemini, attempts to parse the JSON output, and applies fallback defaults if parsing fails.
    - *Configuration choices:* JavaScript try/catch block validating response structure.
    - *Key expressions or variables:* `$('Build AI request').all()`, `item.json.candidates[0].content.parts[0].text`.
    - *Input / Output:* Input from `Score post with Gemini`; output connects to `Good match?`.
    - *Edge cases:* Malformed JSON responses from the AI trigger the fallback object `{ isRequest: false, score: 0 }`.

#### 2.3 Alerting & Logging
- **Overview:** Evaluates the standardized classification score and request flags, dispatches high-value alerts via Telegram, and appends a complete execution log to Google Sheets.
- **Nodes Involved:**
  - `Good match?`
  - `Send Telegram alert`
  - `Log every rated post`
- **Node Details:**
  - **Good match?**
    - *Type and technical role:* If Node. Evaluates whether a post qualifies as a real client request with an acceptable score.
    - *Configuration choices:* Combines two conditions using an `and` combinator: `score >= 5` and `isRequest === true`.
    - *Key expressions or variables:* `={{ $json.score }}`, `={{ $json.isRequest }}`.
    - *Input / Output:* Input from `Parse AI rating`. True branch connects to `Send Telegram alert` and `Log every rated post`. False branch connects solely to `Log every rated post`.
    - *Edge cases:* Type mismatches if score values are passed as strings instead of numbers.
  - **Send Telegram alert**
    - *Type and technical role:* Telegram Node. Sends formatted notifications about viable client leads to a designated chat ID.
    - *Configuration choices:* Uses predefined Telegram bot credentials. Target chat set via `YOUR_TELEGRAM_CHAT_ID`.
    - *Key expressions or variables:* `Score {{ $json.score }}/10 - {{ $json.title }}`, `{{ $json.summary }}`, `{{ $json.link }}`.
    - *Input / Output:* Input from the true branch of `Good match?`.
    - *Edge cases:* Invalid bot token, incorrect chat ID, or bot blocked by the user.
  - **Log every rated post**
    - *Type and technical role:* Google Sheets Node. Records every evaluated post (both matches and non-matches) to a logging tab.
    - *Configuration choices:* Resource: `sheet`, Operation: `append`, Sheet Name: `log`.
    - *Key expressions or variables:* Mapped column expressions for `date`, `link`, `type`, `score`, `title`, `summary`, `language`, and `isRequest`.
    - *Input / Output:* Inputs from both branches of `Good match?`.
    - *Edge cases:* Sheet schema mismatches or Google API write errors.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every 15 minutes | scheduleTrigger | Triggers the workflow every 15 minutes | None | Read seen post IDs | AI job alerts from the n8n community forum (Telegram + Google Sheets) |
| Read seen post IDs | googleSheets | Reads existing post IDs from tracking sheet | Every 1-15 minutes | Fetch jobs RSS feed | AI job alerts from the n8n community forum (Telegram + Google Sheets)<br>1. Fetch new posts |
| Fetch jobs RSS feed | rssFeedRead | Fetches latest items from community job RSS | Read seen post IDs | Keep only new posts | AI job alerts from the n8n community forum (Telegram + Google Sheets)<br>1. Fetch new posts |
| Keep only new posts | code | Filters out previously processed posts | Fetch jobs RSS feed | Mark posts as seen | AI job alerts from the n8n community forum (Telegram + Google Sheets)<br>1. Fetch new posts |
| Mark posts as seen | googleSheets | Appends new post IDs to tracking sheet | Keep only new posts | Build AI request | AI job alerts from the n8n community forum (Telegram + Google Sheets)<br>2. Rate with AI |
| Build AI request | code | Formats custom prompt payloads for Gemini | Mark posts as seen | Score post with Gemini | AI job alerts from the n8n community forum (Telegram + Google Sheets)<br>2. Rate with AI |
| Score post with Gemini | httpRequest | Calls Google Gemini API for classification | Build AI request | Parse AI rating | AI job alerts from the n8n community forum (Telegram + Google Sheets)<br>2. Rate with AI |
| Parse AI rating | code | Parses and normalizes Gemini JSON responses | Score post with Gemini | Good match? | AI job alerts from the n8n community forum (Telegram + Google Sheets)<br>2. Rate with AI |
| Good match? | if | Filters posts scoring >= 5 and flagged as requests | Parse AI rating | Send Telegram alert, Log every rated post | AI job alerts from the n8n community forum (Telegram + Google Sheets)<br>3. Alert & log |
| Send Telegram alert | telegram | Sends qualified alerts to Telegram chat | Good match? (True) | None | AI job alerts from the n8n community forum (Telegram + Google Sheets)<br>3. Alert & log |
| Log every rated post | googleSheets | Logs all evaluated posts to Google Sheets | Good match? (True/False) | None | AI job alerts from the n8n community forum (Telegram + Google Sheets)<br>3. Alert & log |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Google Sheet:** Set up a Google Sheet with two tabs:
   - Tab named `seen` with columns: `id`, `date`
   - Tab named `log` with columns: `date`, `score`, `isRequest`, `type`, `language`, `summary`, `title`, `link`
2. **Configure Credentials:** Prepare credentials for Google Sheets (OAuth2/Service Account), Google Palm API (Generative Language API key), and a Telegram Bot.
3. **Add the Trigger Node:** Create a **Schedule Trigger** (`Every 15 minutes`) set to run every 15 minutes.
4. **Add Read Seen Node:** Create a **Google Sheets** node (`Read seen post IDs`). Set resource to `sheet`, operation to `read`, target sheet name to `seen`, and configure your Document ID. Enable `Always Output Data`. Connect `Every 15 minutes` to this node.
5. **Add RSS Node:** Create an **RSS Feed Read** node (`Fetch jobs RSS feed`). Set URL to `https://community.n8n.io/c/jobs/13.rss` and enable `Execute Once`. Connect `Read seen post IDs` to this node.
6. **Add Filtering Code Node:** Create a **Code** node (`Keep only new posts`). Insert JavaScript to extract numeric IDs from post links and filter out entries present in the `seen` sheet. Connect `Fetch jobs RSS feed` to this node.
7. **Add Mark Seen Node:** Create a **Google Sheets** node (`Mark posts as seen`). Set resource to `sheet`, operation to `append`, and sheet name to `seen`. Map columns: `id` to `={{ $json.id }}` and `date` to `={{ $now.toISODate() }}`. Connect `Keep only new posts` to this node.
8. **Add AI Payload Builder Node:** Create a **Code** node (`Build AI request`). Add code to construct system instructions (`MY_PROFILE`, `PREFERRED_LANGUAGE`) and wrap post titles and text into the Gemini body format. Connect `Mark posts as seen` to this node.
9. **Add HTTP Request Node:** Create an **HTTP Request** node (`Score post with Gemini`). Set Method to `POST`, URL to `https://generativelanguage.googleapis.com/v1beta/models/gemini-flash-latest:generateContent`, body content type to JSON, body expression to `={{ JSON.stringify($json.geminiBody) }}`, and authentication type to Google Palm API. Configure batching options (`batchSize: 1`, `batchInterval: 7000`). Connect `Build AI request` to this node.
10. **Add AI Parsing Code Node:** Create a **Code** node (`Parse AI rating`). Add logic to parse candidate responses and handle fallbacks. Connect `Score post with Gemini` to this node.
11. **Add If Node:** Create an **If** node (`Good match?`). Configure conditions: check if score is greater than or equal to `5` and `isRequest` equals `true`. Connect `Parse AI rating` to this node.
12. **Add Telegram Node:** Create a **Telegram** node (`Send Telegram alert`). Configure bot credentials, input your `chatId`, and format the text template using score, title, summary, and link expressions. Connect the **true** output branch of `Good match?` to this node.
13. **Add Logging Node:** Create a **Google Sheets** node (`Log every rated post`). Set resource to `sheet`, operation to `append`, and sheet name to `log`. Map columns (`date`, `link`, `type`, `score`, `title`, `summary`, `language`, `isRequest`) to their respective `$json` fields. Connect **both** outputs (true and false branches) of `Good match?` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google AI Studio API Key acquisition | Obtain a Google Gemini API key from [AI Studio](https://aistudio.google.com) |
| n8n Community Jobs RSS Source | Default feed monitored: [n8n Community Jobs RSS](https://community.n8n.io/c/jobs/13.rss) |
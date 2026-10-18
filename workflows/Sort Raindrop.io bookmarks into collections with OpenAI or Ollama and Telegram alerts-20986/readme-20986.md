Sort Raindrop.io bookmarks into collections with OpenAI or Ollama and Telegram alerts

https://n8nworkflows.xyz/workflows/sort-raindrop-io-bookmarks-into-collections-with-openai-or-ollama-and-telegram-alerts-20986


# Sort Raindrop.io bookmarks into collections with OpenAI or Ollama and Telegram alerts

### 1. Workflow Overview

This workflow automates the categorization of bookmarks from a Raindrop.io Inbox collection by leveraging a Large Language Model (LLM). Operating on a scheduled interval, it evaluates recent bookmarks, queries an LLM (via Ollama or an OpenAI-compatible API) to determine the most relevant destination collection based on natural language descriptions, moves the bookmarks accordingly, and dispatches Telegram notifications for low-confidence classifications.

The logic is grouped into the following functional blocks:
- **1.1 Trigger and Initialization:** Schedules the execution, retrieves collection metadata from an n8n Data Table, and loads/validates user configuration settings.
- **1.2 Data Retrieval:** Fetches recent bookmarks from the designated Raindrop.io Inbox and breaks them down into individual items for processing.
- **1.3 AI Processing and Validation:** Constructs structured prompts for the LLM, executes the API call, and validates the returned classification against predefined confidence thresholds and allowed collection IDs.
- **1.4 Execution and Notification:** Routes bookmarks to their final destination (target or fallback collection) within Raindrop.io and sends alerts via Telegram when confidence requirements are unmet.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initialization
- **Overview:** Initiates the workflow every 15 minutes, pulls the collection registry from an n8n Data Table, and merges those collections with global configuration parameters.
- **Nodes Involved:** `Every 15 min`, `Get Collections`, `Config`
- **Node Details:**
  - `Every 15 min`
    - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.2) - Time-based workflow trigger.
    - *Configuration:* Interval configured for every 15 minutes.
    - *Input/Output:* Output connects to `Get Collections`.
    - *Edge Cases/Failures:* Missed executions if n8n is offline; handles catch-up triggers depending on instance configuration.
  - `Get Collections`
    - *Type & Technical Role:* `n8n-nodes-base.dataTable` (v1) - Database query node.
    - *Configuration:* Operation set to "get", returning all rows from the data table named `collections`.
    - *Input/Output:* Input from `Every 15 min`; output connects to `Config`.
    - *Edge Cases/Failures:* Missing data table throws execution errors if the table does not exist.
  - `Config`
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) - JavaScript execution node for state management and validation.
    - *Configuration:* Loads defaults (`inboxId`, `fallbackId`, `confidenceThreshold: 0.7`, `llmProvider: 'ollama'`, `llmUrl`, `llmModel`), merges with static workflow data, and validates that `inboxId` and `fallbackId` exist in the `collections` table and are distinct.
    - *Key Expressions/Variables:* `$getWorkflowStaticData('global').config`, `$('Get Collections').all()`.
    - *Input/Output:* Input from `Get Collections`; output connects to `Get Inbox`.
    - *Edge Cases/Failures:* Throws explicit errors if `inboxId` or `fallbackId` are missing, invalid, or identical.

#### 2.2 Data Retrieval
- **Overview:** Requests the latest bookmarks from the Raindrop.io API using the configured Inbox ID and splits the resulting array into individual item streams.
- **Nodes Involved:** `Get Inbox`, `Split Items`
- **Node Details:**
  - `Get Inbox`
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) - External API call.
    - *Configuration:* GET request to `https://api.raindrop.io/rest/v1/raindrops/{{ $('Config').first().json.inboxId }}?perpage=50&sort=-created`. Uses generic Header Authentication (`Authorization: Bearer <token>`).
    - *Input/Output:* Input from `Config`; output connects to `Split Items`.
    - *Edge Cases/Failures:* Rate limiting by Raindrop.io, invalid API tokens, or network timeouts.
  - `Split Items`
    - *Type & Technical Role:* `n8n-nodes-base.splitOut` (v1) - Data transformation node.
    - *Configuration:* Splits the `items` array from the Raindrop API response into individual workflow items.
    - *Input/Output:* Input from `Get Inbox`; output connects to `Build Input`.

#### 2.3 AI Processing and Validation
- **Overview:** Formats bookmark data and collection schemas into a strict JSON-output prompt, executes the LLM request, and validates the model's classification.
- **Nodes Involved:** `Build Input`, `LLM Request`, `Validate`
- **Node Details:**
  - `Build Input`
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) - JavaScript execution node (runs once for each item).
    - *Configuration:* Generates system and user prompts incorporating available collection definitions, extra rules, and provider-specific payload schemas (Ollama vs. OpenAI).
    - *Input/Output:* Input from `Split Items`; output connects to `LLM Request`.
  - `LLM Request`
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) - External API call.
    - *Configuration:* POST request to dynamic URL (`{{ $json.url }}`). Timeout set to 120,000ms. Sends JSON body. Uses generic Header Authentication if required by the provider.
    - *Input/Output:* Input from `Build Input`; output connects to `Validate`.
    - *Edge Cases/Failures:* LLM endpoint downtime, response timeouts (120s limit), or invalid JSON string output from the LLM.
  - `Validate`
    - *Type & Technical Role:* `n8n-nodes-base.code` (v2) - JavaScript execution node for response parsing and validation.
    - *Configuration:* Parses LLM output, checks if the returned collection ID is in the allowed list, and evaluates whether the confidence score meets or exceeds the threshold. Falls back to the fallback ID if criteria fail.
    - *Input/Output:* Input from `LLM Request`; output connects to `Moved?`.
    - *Edge Cases/Failures:* Malformed JSON in LLM response handled safely via `try/catch`, defaulting to fallback behaviors.

#### 2.4 Execution and Notification
- **Overview:** Evaluates confidence results to route bookmarks to their final Raindrop collection and issues Telegram alerts for unconfident sorts.
- **Nodes Involved:** `Moved?`, `Move Bookmark`, `Send Summary`
- **Node Details:**
  - `Moved?`
    - *Type & Technical Role:* `n8n-nodes-base.if` (v2) - Conditional router.
    - *Configuration:* Evaluates if `{{ $json.confident }}` is true.
    - *Input/Output:* Input from `Validate`; outputs connect to `Move Bookmark` (true path) and branch to both `Send Summary` and `Move Bookmark` (false path).
  - `Move Bookmark`
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (v4.2) - External API call.
    - *Configuration:* PUT request to `https://api.raindrop.io/rest/v1/raindrop/{{ $json.id }}` updating the collection ID (`collection: { "$id": $json.collection }`). Uses generic Header Authentication.
    - *Input/Output:* Input from `Moved?` branches; terminal node for collection updating.
    - *Edge Cases/Failures:* API authentication failures or referencing invalid collection IDs.
  - `Send Summary`
    - *Type & Technical Role:* `n8n-nodes-base.telegram` (v1.2) - Messaging integration node.
    - *Configuration:* Sends a text message containing bookmark title and reason for fallback routing to `{{ $('Config').first().json.telegramChatId }}`.
    - *Input/Output:* Input from the low-confidence branch of `Moved?`; terminal notification node.
    - *Edge Cases/Failures:* Invalid Telegram chat IDs, missing credentials, or Telegram API downtime.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every 15 min | n8n-nodes-base.scheduleTrigger | Time-based workflow trigger | None | Get Collections | Raindrop Auto-Sorter<br><br>Runs every 15 minutes. Each new bookmark in the Inbox is sent to an LLM, which picks the collection it belongs in. Bookmarks the LLM isn't confident about go to the fallback collection, and you get a Telegram alert for each one.<br><br>Setup (one time)<br>1. Create a Data Table named `collections` with columns: collection_id (number), name (text), description (text).<br>2. Add one row per collection. The description is what the LLM uses to choose it. Edit rows any time in the table view.<br>3. Raindrop token: create it in Raindrop under Settings > Integrations > Developer. In n8n, add a Header Auth credential on Get Inbox and Move Bookmark: Name Authorization, Value Bearer YOUR_TOKEN.<br>4. Run the Setup Form: Inbox ID, Fallback ID, Telegram chat ID, LLM settings, optional extra rules.<br>5. Credentials: LLM Request (Ollama: any value) and Send Summary (Telegram). Delete Send Summary if you don't want alerts.<br>6. Activate.<br><br>Example rows: 10000001 \| Work \| Job postings, client research. 10000004 \| Unsorted \| Fallback for anything uncertain. |
| Get Collections | n8n-nodes-base.dataTable | Retrieves collection metadata | Every 15 min | Config | Raindrop Auto-Sorter<br><br>Runs every 15 minutes. Each new bookmark in the Inbox is sent to an LLM, which picks the collection it belongs in. Bookmarks the LLM isn't confident about go to the fallback collection, and you get a Telegram alert for each one.<br><br>Setup (one time)<br>1. Create a Data Table named `collections` with columns: collection_id (number), name (text), description (text).<br>2. Add one row per collection. The description is what the LLM uses to choose it. Edit rows any time in the table view.<br>3. Raindrop token: create it in Raindrop under Settings > Integrations > Developer. In n8n, add a Header Auth credential on Get Inbox and Move Bookmark: Name Authorization, Value Bearer YOUR_TOKEN.<br>4. Run the Setup Form: Inbox ID, Fallback ID, Telegram chat ID, LLM settings, optional extra rules.<br>5. Credentials: LLM Request (Ollama: any value) and Send Summary (Telegram). Delete Send Summary if you don't want alerts.<br>6. Activate.<br><br>Example rows: 10000001 \| Work \| Job postings, client research. 10000004 \| Unsorted \| Fallback for anything uncertain. |
| Config | n8n-nodes-base.code | Validates and merges settings | Get Collections | Get Inbox | Raindrop Auto-Sorter<br><br>Runs every 15 minutes. Each new bookmark in the Inbox is sent to an LLM, which picks the collection it belongs in. Bookmarks the LLM isn't confident about go to the fallback collection, and you get a Telegram alert for each one.<br><br>Setup (one time)<br>1. Create a Data Table named `collections` with columns: collection_id (number), name (text), description (text).<br>2. Add one row per collection. The description is what the LLM uses to choose it. Edit rows any time in the table view.<br>3. Raindrop token: create it in Raindrop under Settings > Integrations > Developer. In n8n, add a Header Auth credential on Get Inbox and Move Bookmark: Name Authorization, Value Bearer YOUR_TOKEN.<br>4. Run the Setup Form: Inbox ID, Fallback ID, Telegram chat ID, LLM settings, optional extra rules.<br>5. Credentials: LLM Request (Ollama: any value) and Send Summary (Telegram). Delete Send Summary if you don't want alerts.<br>6. Activate.<br><br>Example rows: 10000001 \| Work \| Job postings, client research. 10000004 \| Unsorted \| Fallback for anything uncertain. |
| Get Inbox | n8n-nodes-base.httpRequest | Fetches bookmarks from Raindrop Inbox | Config | Split Items | LLM Backend<br><br>Provider "ollama" (local default): URL http://ollama:11434/api/chat. Resolves only on your local network. Pull a model first, e.g. ollama pull qwen2.5:7b.<br><br>Provider "openai": any OpenAI-compatible chat API. URL example: https://api.openai.com/v1/chat/completions. Key: Header Auth on LLM Request.<br><br>The LLM is asked for one bookmark at a time and returns a collection and a confidence score. Scores below the threshold (default 0.7, set in Config) go to the fallback. |
| Split Items | n8n-nodes-base.splitOut | Splits bookmark list into individual items | Get Inbox | Build Input | LLM Backend<br><br>Provider "ollama" (local default): URL http://ollama:11434/api/chat. Resolves only on your local network. Pull a model first, e.g. ollama pull qwen2.5:7b.<br><br>Provider "openai": any OpenAI-compatible chat API. URL example: https://api.openai.com/v1/chat/completions. Key: Header Auth on LLM Request.<br><br>The LLM is asked for one bookmark at a time and returns a collection and a confidence score. Scores below the threshold (default 0.7, set in Config) go to the fallback. |
| Build Input | n8n-nodes-base.code | Formats prompt and payload for LLM | Split Items | LLM Request | LLM Backend<br><br>Provider "ollama" (local default): URL http://ollama:11434/api/chat. Resolves only on your local network. Pull a model first, e.g. ollama pull qwen2.5:7b.<br><br>Provider "openai": any OpenAI-compatible chat API. URL example: https://api.openai.com/v1/chat/completions. Key: Header Auth on LLM Request.<br><br>The LLM is asked for one bookmark at a time and returns a collection and a confidence score. Scores below the threshold (default 0.7, set in Config) go to the fallback. |
| LLM Request | n8n-nodes-base.httpRequest | Sends classification request to LLM API | Build Input | Validate | LLM Backend<br><br>Provider "ollama" (local default): URL http://ollama:11434/api/chat. Resolves only on your local network. Pull a model first, e.g. ollama pull qwen2.5:7b.<br><br>Provider "openai": any OpenAI-compatible chat API. URL example: https://api.openai.com/v1/chat/completions. Key: Header Auth on LLM Request.<br><br>The LLM is asked for one bookmark at a time and returns a collection and a confidence score. Scores below the threshold (default 0.7, set in Config) go to the fallback. |
| Validate | n8n-nodes-base.code | Parses and validates LLM classification | LLM Request | Moved? | LLM Backend<br><br>Provider "ollama" (local default): URL http://ollama:11434/api/chat. Resolves only on your local network. Pull a model first, e.g. ollama pull qwen2.5:7b.<br><br>Provider "openai": any OpenAI-compatible chat API. URL example: https://api.openai.com/v1/chat/completions. Key: Header Auth on LLM Request.<br><br>The LLM is asked for one bookmark at a time and returns a collection and a confidence score. Scores below the threshold (default 0.7, set in Config) go to the fallback. |
| Moved? | n8n-nodes-base.if | Routes based on confidence check | Validate | Move Bookmark, Send Summary | |
| Move Bookmark | n8n-nodes-base.httpRequest | Updates bookmark collection in Raindrop | Moved? | None | |
| Send Summary | n8n-nodes-base.telegram | Sends Telegram notification on low confidence | Moved? | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an n8n Data Table:**
   - Name the table `collections`.
   - Add three columns: `collection_id` (Number), `name` (Text), and `description` (Text). Populate with valid Raindrop collection metadata.
2. **Create the Trigger Node:**
   - Add a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`). Set interval to every 15 minutes. Name it `Every 15 min`.
3. **Add Data Table Node:**
   - Add a **Data Table** node (`n8n-nodes-base.dataTable`). Set operation to `get`, select all rows, and bind to the `collections` data table. Name it `Get Collections`. Connect `Every 15 min` -> `Get Collections`.
4. **Add Configuration Node:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Config`. Paste the initialization JavaScript snippet that manages default settings, reads static workflow configuration, and verifies collection ID existence. Connect `Get Collections` -> `Config`.
5. **Fetch Raindrop Inbox Bookmarks:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Get Inbox`. Method: `GET`. URL: `https://api.raindrop.io/rest/v1/raindrops/{{ $('Config').first().json.inboxId }}?perpage=50&sort=-created`. Configure **Generic Credential Type** -> **Header Auth** (`Authorization: Bearer YOUR_TOKEN`). Connect `Config` -> `Get Inbox`.
6. **Split Bookmarks Array:**
   - Add a **Split Out** node (`n8n-nodes-base.splitOut`). Name it `Split Items`. Set field to split out to `items`. Connect `Get Inbox` -> `Split Items`.
7. **Build LLM Prompt Input:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Build Input`. Set mode to `Run Once for Each Item`. Insert code generating system prompts, collection mappings, and provider-specific JSON request bodies. Connect `Split Items` -> `Build Input`.
8. **Execute LLM Request:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `LLM Request`. Method: `POST`. URL: `{{ $json.url }}`. Set body to JSON and pass `{{ JSON.stringify($json.body) }}`. Set request timeout to 120,000ms. Configure Header Auth if required. Connect `Build Input` -> `LLM Request`.
9. **Validate LLM Output:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Validate`. Paste JavaScript snippet to parse LLM text output, extract `collection_id` and `confidence`, check threshold limits, and assign fallback IDs if invalid. Connect `LLM Request` -> `Validate`.
10. **Add Conditional Router:**
    - Add an **If** node (`n8n-nodes-base.if`). Name it `Moved?`. Set condition to evaluate `{{ $json.confident }}` as true. Connect `Validate` -> `Moved?`.
11. **Update Raindrop Collection:**
    - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Move Bookmark`. Method: `PUT`. URL: `https://api.raindrop.io/rest/v1/raindrop/{{ $json.id }}`. Set body type to JSON with payload `{{ JSON.stringify({ collection: { "$id": $json.collection } }) }}`. Configure Header Auth (`Authorization: Bearer YOUR_TOKEN`). Connect both the `true` output and the `false` output of `Moved?` to `Move Bookmark`.
12. **Add Telegram Notification:**
    - Add a **Telegram** node (`n8n-nodes-base.telegram`). Name it `Send Summary`. Configure text message to display the title and fallback reason (`{{ $json.title }} was not confidently sorted, sent to fallback. Reason: {{ $json.reason }}`) and set chat ID to `{{ $('Config').first().json.telegramChatId }}`. Connect the `false` output of `Moved?` to `Send Summary`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Local LLM Setup (Ollama) | Default Ollama URL: `http://ollama:11434/api/chat`. Ensure models are pulled locally (e.g., `ollama pull qwen2.5:7b`). |
| Cloud LLM Setup (OpenAI-compatible) | Use standard endpoints like `https://api.openai.com/v1/chat/completions` with appropriate API keys configured via Header Auth. |
| Raindrop.io API Authentication | API tokens must be generated via Raindrop Settings > Integrations > Developer and applied using HTTP Bearer Token authentication. |
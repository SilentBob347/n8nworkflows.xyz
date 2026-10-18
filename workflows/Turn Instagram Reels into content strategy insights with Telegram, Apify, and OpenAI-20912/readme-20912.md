Turn Instagram Reels into content strategy insights with Telegram, Apify, and OpenAI

https://n8nworkflows.xyz/workflows/turn-instagram-reels-into-content-strategy-insights-with-telegram--apify--and-openai-20912


# Turn Instagram Reels into content strategy insights with Telegram, Apify, and OpenAI

### 1. Workflow Overview

This workflow functions as an automated Instagram Reel intelligence assistant operated via a Telegram bot. Users send an Instagram Reel link to the bot, which extracts the media metadata, transcribes the audio, generates a 22-field content strategy breakdown using OpenAI, and replies directly inside Telegram. 

To optimize performance and minimize API operational costs, the workflow incorporates a strict gatekeeping pipeline and a 24-hour execution cache. Non-video links (such as profiles or stories) are rejected instantly prior to incurring external service charges, and identical Reels requested within a 24-hour window bypass external processing entirely by serving cached results.

The execution logic is structured into six functional blocks:
- **1.1 Input Reception & Intent Routing:** Receives incoming Telegram events, normalizes payload envelopes, and instantly routes system commands (`/start`, `/help`) to a static help response.
- **1.2 Validation & Caching Gate:** Parses and validates incoming URLs against supported Instagram patterns, drops unsupported links, and checks n8n's workflow static storage for active 24-hour cached insights.
- **1.3 Transcript Extraction & Verification:** Triggers the Apify transcription actor, handles asynchronous polling delays, extracts dataset results, and validates whether the transcript contains sufficient spoken content.
- **1.4 AI Prompt Generation & Structuring:** Formats metadata and truncated transcripts into a prompt enforcing strict originality and analytical neutrality rules, paired with an OpenAI LLM and an automated structured output parser.
- **1.5 Telegram Response Formatting & Delivery:** Normalizes outputs across success, cache-hit, and failure states, chunks long messages to respect Telegram's 4096-character limit, and dispatches messages to the user.
- **1.6 Cache Persistence:** Writes successful analysis runs back to global workflow static data for subsequent cache hits.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Intent Routing
- **Overview:** Captures Telegram bot updates, structures message attributes into a uniform data envelope, and branches execution flows based on recognized command strings.
- **Nodes Involved:** 
  - `01 — Receive Reel From Telegram`
  - `02 — Read Message & Classify Intent`
  - `03 — Route Request`
- **Node Details:**
  - **01 — Receive Reel From Telegram**
    - *Type & Technical Role:* `n8n-nodes-base.telegramTrigger` (Trigger). Listens for incoming webhook messages sent to the configured Telegram bot.
    - *Configuration Choices:* Set to watch `message` updates.
    - *Input/Output:* No input connections; outputs raw message JSON to Node 02.
    - *Edge Cases / Failures:* Fails if the Telegram webhook URL is unreachable due to missing public HTTPS exposure on self-hosted instances.
  - **02 — Read Message & Classify Intent**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Normalizes raw Telegram payloads into a consistent envelope containing safe string identifiers (`chatId`, `userId`) and classifies text into commands or processing routes.
    - *Configuration Choices:* Runs once for all items using custom JavaScript.
    - *Key Expressions / Variables:* `item.message`, `msg.chat.id`, `msg.from.id`.
    - *Input/Output:* Input from Node 01; output connected to Node 03.
    - *Edge Cases / Failures:* Handles missing text safely by checking types before executing string modifications.
  - **03 — Route Request**
    - *Type & Technical Role:* `n8n-nodes-base.switch` (Flow Control). Evaluates the classified route to separate help commands from standard processing paths.
    - *Configuration Choices:* Configured with a matching rule for `route === 'help'` directing to the formatting node, and a fallback (`extra`) directing to URL extraction.
    - *Key Expressions / Variables:* `={{ $json.route }}`
    - *Input/Output:* Input from Node 02; outputs branch to Node 04 (fallback) or Node 17 (`help`).
    - *Edge Cases / Failures:* Unrecognized commands safely fall through to the processing pipeline.

#### 1.2 Validation & Caching Gate
- **Overview:** Extracts Instagram links using regex matching (avoiding unavailable browser globals), validates host and path structures, drops invalid content types, and checks the local cache.
- **Nodes Involved:**
  - `04 — Extract & Validate Reel URL`
  - `05 — Reel URL Valid?`
  - `06 — Check Recent Cache`
  - `07 — Cached Result Available?`
- **Node Details:**
  - **04 — Extract & Validate Reel URL**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Validation). Extracts Instagram URLs via regular expressions, stripping query parameters and confirming support for allowed hosts and path formats (`/reel/`, `/reels/`, `/p/`).
    - *Configuration Choices:* Runs once for all items without utilizing browser-dependent globals (e.g., `URL`).
    - *Input/Output:* Input from Node 03; output connected to Node 05.
    - *Edge Cases / Failures:* Sets `failureCode` to `NO_URL` or `UNSUPPORTED_URL` if patterns fail to match.
  - **05 — Reel URL Valid?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Evaluates the boolean flag determining if a valid Instagram URL was parsed.
    - *Key Expressions / Variables:* `={{ $json.urlValid }}`
    - *Input/Output:* Input from Node 04; outputs branch to Node 06 (true) or Node 17 (false, formatting failure message).
  - **06 — Check Recent Cache**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Cache Inspection). Accesses workflow static storage to verify whether identical content was successfully processed within the last 24 hours.
    - *Configuration Choices:* TTL set to `24 * 60 * 60 * 1000` ms; max entries limited to 15.
    - *Key Expressions / Variables:* Uses `$getWorkflowStaticData('global')`.
    - *Input/Output:* Input from Node 05; output connected to Node 07.
    - *Edge Cases / Failures:* Automatically purges expired cache keys during traversal.
  - **07 — Cached Result Available?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Checks whether a valid unexpired cache entry was identified.
    - *Key Expressions / Variables:* `={{ $json.cacheHit }}`
    - *Input/Output:* Input from Node 06; outputs branch to Node 17 (true, serving cache) or Node 08 (false, initiating fresh API processing).

#### 1.3 Transcript Extraction & Verification
- **Overview:** Interacts with the Apify platform to execute video transcription, polls for completion status, fetches dataset items, and verifies transcript usability.
- **Nodes Involved:**
  - `08 — Start Transcript Run`
  - `09 — Transcript Run Ready?`
  - `10 — Wait & Re-check Transcript Run`
  - `09B — Run Ready After Wait?`
  - `10E — Mark Failure: Transcript Run Took Too Long`
  - `11 — Read Transcript Result`
  - `12 — Validate Transcript`
  - `13 — Transcript Usable?`
- **Node Details:**
  - **08 — Start Transcript Run**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (External API Call). Triggers the Apify Actor (`lance_api/instagram-video-ai-transcript-scraper-api`).
    - *Configuration Choices:* POST request with JSON payload containing URL, text format, and metadata flags. Uses Generic Header Auth (`Authorization: Bearer <token>`). Query parameters set `waitForFinish=60`, `timeout=600`, and `maxTotalChargeUsd=1`.
    - *Key Expressions / Variables:* `={{ $json.normalizedUrl }}`
    - *Input/Output:* Input from Node 07; output connected to Node 09.
    - *Edge Cases / Failures:* Retries up to 3 times on failure; timeouts handled via subsequent polling nodes.
  - **09 — Transcript Run Ready?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Validates whether the Apify run status has concluded.
    - *Key Expressions / Variables:* `={{ ['SUCCEEDED','FAILED','ABORTED','TIMED-OUT'].includes($json.data.status) }}`
    - *Input/Output:* Input from Node 08; outputs branch to Node 11 (true) or Node 10 (false, waiting).
  - **10 — Wait & Re-check Transcript Run**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (External API Call). Polls Apify for run status updates if initial execution took longer than 60 seconds.
    - *Configuration Choices:* GET request to actor-runs endpoint with `waitForFinish=55`.
    - *Key Expressions / Variables:* `={{ $json.data.id }}`
    - *Input/Output:* Input from Node 09; output connected to Node 09B.
  - **09B — Run Ready After Wait?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Checks status following the secondary polling wait period.
    - *Key Expressions / Variables:* `={{ ['SUCCEEDED','FAILED','ABORTED','TIMED-OUT'].includes($json.data.status) }}`
    - *Input/Output:* Input from Node 10; outputs branch to Node 11 (true) or Node 10E (false, flagging timeout).
  - **10E — Mark Failure: Transcript Run Took Too Long**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation). Assigns an `ACTOR_TIMEOUT` failure code.
    - *Input/Output:* Input from Node 09B; output connected to Node 17.
  - **11 — Read Transcript Result**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (External API Call). Retrieves dataset items from the completed Apify run.
    - *Configuration Choices:* GET request with `clean=true` query parameter pointing to the dataset endpoint.
    - *Key Expressions / Variables:* `={{ $json.data.defaultDatasetId }}`
    - *Input/Output:* Input from Nodes 09 or 09B; output connected to Node 12.
  - **12 — Validate Transcript**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Validation). Reads dataset outputs, maps status flags to internal failure codes, and verifies minimum length thresholds (8 characters).
    - *Input/Output:* Input from Node 11; output connected to Node 13.
  - **13 — Transcript Usable?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Evaluates the boolean flag `transcriptUsable`.
    - *Key Expressions / Variables:* `={{ $json.transcriptUsable }}`
    - *Input/Output:* Input from Node 12; outputs branch to Node 14 (true) or Node 17 (false, handling extraction errors).

#### 1.4 AI Prompt Generation & Structuring
- **Overview:** Assembles core system instructions, originality constraints, metadata, and transcript text into a single combined prompt block processed by OpenAI with structured output mapping.
- **Nodes Involved:**
  - `14 — Build AI Prompt`
  - `15 — Analyze Reel With AI`
  - `OpenAI Chat Model`
  - `Structured Output Parser`
  - `16 — AI Result Complete?`
- **Node Details:**
  - **14 — Build AI Prompt**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Combines core strategist system instructions, originality enforcement clauses, dynamic metadata, and truncated transcript text into a single prompt string.
    - *Input/Output:* Input from Node 13; output connected to Node 15.
  - **15 — Analyze Reel With AI**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Agent / LLM Chain). Executes the language model chain utilizing attached sub-nodes for model parameters and output structuring.
    - *Configuration Choices:* Input configured via prompt definition referencing combined system/user strings.
    - *Key Expressions / Variables:* `={{ $json.prompt }}`
    - *Input/Output:* Inputs connected from Node 14 (main), OpenAI Chat Model (AI language model), and Structured Output Parser (AI output parser); output connected to Node 16.
  - **OpenAI Chat Model**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Sub-node Model Provider). Configures the underlying OpenAI language model.
    - *Configuration Choices:* Model set to `gpt-4o-mini`; temperature set to `0.2`. Credential: OpenAI account.
    - *Input/Output:* Connected to Node 15 and Structured Output Parser.
  - **Structured Output Parser**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Sub-node Output Parser). Enforces a 22-field JSON schema on LLM outputs with auto-fix capabilities enabled.
    - *Configuration Choices:* Manual schema type with properties including `summary`, `opening_hook`, `hook_type`, `core_topic`, `content_angle`, `target_audience`, `problem_desire`, `key_message`, `cta`, `emotional_trigger`, `content_structure`, `why_it_works`, original hooks, content ideas, new script components, and language codes. All fields marked required.
    - *Input/Output:* Connected to Node 15 and OpenAI Chat Model.
  - **16 — AI Result Complete?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control). Verifies that essential AI fields (`summary` and `new_script_body`) are present in the model response.
    - *Key Expressions / Variables:* `={{ $json.summary || ($json.output || {}).summary }}`
    - *Input/Output:* Input from Node 15; outputs branch to Node 17 (true/false paths converge on formatting).

#### 1.5 Telegram Response Formatting & Delivery
- **Overview:** Centralizes message generation across all execution paths (success, cache hit, failure, or help), chunks long payloads to comply with character limits, and dispatches replies to Telegram.
- **Nodes Involved:**
  - `17 — Format Creator Insights`
  - `18 — Reply In Telegram`
- **Node Details:**
  - **17 — Format Creator Insights**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Normalizes wrapped AI outputs, generates formatted message blocks or error notices, splits text into chunks below 3800 characters, and outputs one item per chunk.
    - *Input/Output:* Inputs connected from Node 03, Node 05, Node 07, Node 13, and Node 16; output connected to Node 18.
    - *Edge Cases / Failures:* Handles extraction and JSON wrapping variants using robust fallback logic.
  - **18 — Reply In Telegram**
    - *Type & Technical Role:* `n8n-nodes-base.telegram` (Action). Sends formatted text chunks back to the user via Telegram chat.
    - *Configuration Choices:* Action set to send message. Additional fields configure reply-to message ID and disable web page previews.
    - *Key Expressions / Variables:* Chat ID: `={{ $json.chatId }}`, Text: `={{ $json.chunk }}`.
    - *Input/Output:* Input from Node 17; output connected to Node 19.
    - *Edge Cases / Failures:* Fails if message text is empty or chat identifiers are malformed.

#### 1.6 Cache Persistence
- **Overview:** Writes successful analysis runs back to workflow static storage for subsequent cache hits.
- **Nodes Involved:**
  - `19 — Save To Cache`
- **Node Details:**
  - **19 — Save To Cache**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Storage). Persists successful analysis results into workflow static storage with eviction safeguards based on entry count and byte size.
    - *Input/Output:* Input from Node 18; no outgoing connections.
    - *Edge Cases / Failures:* Wrapped in try-catch to bypass execution on paths where upstream validation did not run.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `01 — Receive Reel From Telegram` | `n8n-nodes-base.telegramTrigger` | Listens for incoming Telegram bot messages. | None | `02 — Read Message & Classify Intent` | ## TELEGRAM INPUT<br><br>Receives the message, pulls the Reel link out of the surrounding text, and answers `/start` and `/help` instantly without calling any paid API. |
| `02 — Read Message & Classify Intent` | `n8n-nodes-base.code` | Normalizes payload envelopes and classifies routing commands. | `01 — Receive Reel From Telegram` | `03 — Route Request` | ## TELEGRAM INPUT<br><br>Receives the message, pulls the Reel link out of the surrounding text, and answers `/start` and `/help` instantly without calling any paid API. |
| `03 — Route Request` | `n8n-nodes-base.switch` | Routes help commands versus processing workflows. | `02 — Read Message & Classify Intent` | `04 — Extract & Validate Reel URL`, `17 — Format Creator Insights` | ## TELEGRAM INPUT<br><br>Receives the message, pulls the Reel link out of the surrounding text, and answers `/start` and `/help` instantly without calling any paid API. |
| `04 — Extract & Validate Reel URL` | `n8n-nodes-base.code` | Extracts and validates Instagram URLs via regex. | `03 — Route Request` | `05 — Reel URL Valid?` | ## VALIDATE & CACHE<br><br>Reels and video posts only - profiles, Stories and image-only posts stop here, before any paid call. A Reel analysed in the last 24 hours is answered from cache rather than paid for again. |
| `05 — Reel URL Valid?` | `n8n-nodes-base.if` | Verifies whether a valid Instagram URL was parsed. | `04 — Extract & Validate Reel URL` | `06 — Check Recent Cache`, `17 — Format Creator Insights` | ## VALIDATE & CACHE<br><br>Reels and video posts only - profiles, Stories and image-only posts stop here, before any paid call. A Reel analysed in the last 24 hours is answered from cache rather than paid for again. |
| `06 — Check Recent Cache` | `n8n-nodes-base.code` | Checks static workflow storage for recent cache entries. | `05 — Reel URL Valid?` | `07 — Cached Result Available?` | ## VALIDATE & CACHE<br><br>Reels and video posts only - profiles, Stories and image-only posts stop here, before any paid call. A Reel analysed in the last 24 hours is answered from cache rather than paid for again. |
| `07 — Cached Result Available?` | `n8n-nodes-base.if` | Determines whether to use cached results or trigger extraction. | `06 — Check Recent Cache` | `08 — Start Transcript Run`, `17 — Format Creator Insights` | ## VALIDATE & CACHE<br><br>Reels and video posts only - profiles, Stories and image-only posts stop here, before any paid call. A Reel analysed in the last 24 hours is answered from cache rather than paid for again. |
| `08 — Start Transcript Run` | `n8n-nodes-base.httpRequest` | Triggers the Apify Actor for video transcription. | `07 — Cached Result Available?` | `09 — Transcript Run Ready?` | ## INSTAGRAM TRANSCRIPT<br><br>Runs the Apify Actor, waits up to 60 s, then re-checks once. A second paid run is never started for the same message, and `maxTotalChargeUsd` caps one run at $1. |
| `09 — Transcript Run Ready?` | `n8n-nodes-base.if` | Evaluates if the Apify run status has completed. | `08 — Start Transcript Run` | `10 — Wait & Re-check Transcript Run`, `11 — Read Transcript Result` | ## INSTAGRAM TRANSCRIPT<br><br>Runs the Apify Actor, waits up to 60 s, then re-checks once. A second paid run is never started for the same message, and `maxTotalChargeUsd` caps one run at $1. |
| `10 — Wait & Re-check Transcript Run` | `n8n-nodes-base.httpRequest` | Polls Apify for run status after initial wait. | `09 — Transcript Run Ready?` | `09B — Run Ready After Wait?` | ## INSTAGRAM TRANSCRIPT<br><br>Runs the Apify Actor, waits up to 60 s, then re-checks once. A second paid run is never started for the same message, and `maxTotalChargeUsd` caps one run at $1. |
| `09B — Run Ready After Wait?` | `n8n-nodes-base.if` | Verifies run status following secondary polling. | `10 — Wait & Re-check Transcript Run` | `10E — Mark Failure: Transcript Run Took Too Long`, `11 — Read Transcript Result` | ## INSTAGRAM TRANSCRIPT<br><br>Runs the Apify Actor, waits up to 60 s, then re-checks once. A second paid run is never started for the same message, and `maxTotalChargeUsd` caps one run at $1. |
| `10E — Mark Failure: Transcript Run Took Too Long` | `n8n-nodes-base.set` | Assigns timeout failure code on uncompleted runs. | `09B — Run Ready After Wait?` | `17 — Format Creator Insights` | ## INSTAGRAM TRANSCRIPT<br><br>Runs the Apify Actor, waits up to 60 s, then re-checks once. A second paid run is never started for the same message, and `maxTotalChargeUsd` caps one run at $1. |
| `11 — Read Transcript Result` | `n8n-nodes-base.httpRequest` | Retrieves dataset items from completed Apify runs. | `09 — Transcript Run Ready?`, `09B — Run Ready After Wait?` | `12 — Validate Transcript` | ## INSTAGRAM TRANSCRIPT<br><br>Runs the Apify Actor, waits up to 60 s, then re-checks once. A second paid run is never started for the same message, and `maxTotalChargeUsd` caps one run at $1. |
| `12 — Validate Transcript` | `n8n-nodes-base.code` | Validates transcript usability and maps failure codes. | `11 — Read Transcript Result` | `13 — Transcript Usable?` | ## INSTAGRAM TRANSCRIPT<br><br>Runs the Apify Actor, waits up to 60 s, then re-checks once. A second paid run is never started for the same message, and `maxTotalChargeUsd` caps one run at $1. |
| `13 — Transcript Usable?` | `n8n-nodes-base.if` | Verifies whether extracted transcripts meet usability thresholds. | `12 — Validate Transcript` | `14 — Build AI Prompt`, `17 — Format Creator Insights` | ## INSTAGRAM TRANSCRIPT<br><br>Runs the Apify Actor, waits up to 60 s, then re-checks once. A second paid run is never started for the same message, and `maxTotalChargeUsd` caps one run at $1. |
| `14 — Build AI Prompt` | `n8n-nodes-base.code` | Assembles system instructions, rules, metadata, and prompt strings. | `13 — Transcript Usable?` | `15 — Analyze Reel With AI` | ## AI ANALYSIS<br><br>A Basic LLM Chain with a Structured Output Parser (Auto-Fix Format) turns the transcript into 22 structured fields. Swap the model sub-node for Claude or Gemini and nothing else changes. |
| `15 — Analyze Reel With AI` | `@n8n/n8n-nodes-langchain.chainLlm` | Executes LLM chain for structured content analysis. | `14 — Build AI Prompt`, `OpenAI Chat Model`, `Structured Output Parser` | `16 — AI Result Complete?` | ## AI ANALYSIS<br><br>A Basic LLM Chain with a Structured Output Parser (Auto-Fix Format) turns the transcript into 22 structured fields. Swap the model sub-node for Claude or Gemini and nothing else changes. |
| `OpenAI Chat Model` | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Configures the underlying OpenAI language model (`gpt-4o-mini`). | None | `15 — Analyze Reel With AI`, `Structured Output Parser` | ## AI ANALYSIS<br><br>A Basic LLM Chain with a Structured Output Parser (Auto-Fix Format) turns the transcript into 22 structured fields. Swap the model sub-node for Claude or Gemini and nothing else changes. |
| `Structured Output Parser` | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces the 22-field structured schema on LLM output. | `OpenAI Chat Model` | `15 — Analyze Reel With AI` | ## AI ANALYSIS<br><br>A Basic LLM Chain with a Structured Output Parser (Auto-Fix Format) turns the transcript into 22 structured fields. Swap the model sub-node for Claude or Gemini and nothing else changes. |
| `16 — AI Result Complete?` | `n8n-nodes-base.if` | Verifies essential fields are present in LLM responses. | `15 — Analyze Reel With AI` | `17 — Format Creator Insights` | ## AI ANALYSIS<br><br>A Basic LLM Chain with a Structured Output Parser (Auto-Fix Format) turns the transcript into 22 structured fields. Swap the model sub-node for Claude or Gemini and nothing else changes. |
| `17 — Format Creator Insights` | `n8n-nodes-base.code` | Formats responses and splits text into Telegram-safe chunks. | `03 — Route Request`, `05 — Reel URL Valid?`, `07 — Cached Result Available?`, `10E — Mark Failure: Transcript Run Took Too Long`, `13 — Transcript Usable?`, `16 — AI Result Complete?` | `18 — Reply In Telegram` | ## TELEGRAM REPLY & CACHE<br><br>Every path - success, cache hit or failure - ends here, so the user always gets an answer. Long replies are split under Telegram's 4096-character limit, then cached for 24 hours. |
| `18 — Reply In Telegram` | `n8n-nodes-base.telegram` | Dispatches message chunks back to the Telegram chat. | `17 — Format Creator Insights` | `19 — Save To Cache` | ## TELEGRAM REPLY & CACHE<br><br>Every path - success, cache hit or failure - ends here, so the user always gets an answer. Long replies are split under Telegram's 4096-character limit, then cached for 24 hours. |
| `19 — Save To Cache` | `n8n-nodes-base.code` | Persists successful analysis runs back to workflow static storage. | `18 — Reply In Telegram` | None | ## TELEGRAM REPLY & CACHE<br><br>Every path - success, cache hit or failure - ends here, so the user always gets an answer. Long replies are split under Telegram's 4096-character limit, then cached for 24 hours. |
| `Sticky Note 49008` | `n8n-nodes-base.stickyNote` | High-level overview and instructions note. | None | None | Paste an Instagram Reel link into your own Telegram bot and get a full content-strategy breakdown back in seconds - without opening Instagram or watching the video. Built for creators, social media managers, freelancers and small agencies who need to turn saved Reels into something they can post.<br><br>## How it works<br><br>1. You send a Reel link to the bot.<br>2. The Apify Actor `lance_api/instagram-video-ai-transcript-scraper-api` transcribes the audio.<br>3. OpenAI returns a structured analysis: what it is about and why it works, the hook type, target audience, key message, 3 original hooks, 3 content ideas and an original Reel script.<br>4. The reply is split into numbered chunks when it is long.<br>5. Results are cached for 24 hours, so re-sending the same Reel costs nothing.<br><br>Links that cannot work (profiles, Stories, image-only posts, private or speechless Reels) are rejected before any paid API is called.<br><br>## Setup<br><br>1. Create a bot with @BotFather and connect the token.<br>2. Add a Header Auth credential for Apify: Name `Authorization`, Value `Bearer <your Apify token>`.<br>3. Connect OpenAI (default model gpt-4o-mini).<br>4. Self-hosted: Telegram needs a public HTTPS URL to your n8n (ngrok, Cloudflare Tunnel, reverse proxy).<br>5. Recommended: restrict `01 — Receive Reel From Telegram` to your own user ID - anyone who finds the bot can spend your Apify credits.<br>6. Publish, then message your bot.<br><br>## Customization<br><br>Questions live in `14 — Build AI Prompt`, message wording in `17 — Format Creator Insights`, the model in the `OpenAI Chat Model` sub-node, cache settings in `06 — Check Recent Cache` and `19 — Save To Cache`.<br><br>## Requirements<br><br>Telegram bot token, Apify account, OpenAI API key. Transcription is billed to your Apify account at about $0.01 per Reel; failures are free. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create Node 01 (`01 — Receive Reel From Telegram`):**
   - Type: `n8n-nodes-base.telegramTrigger`
   - Parameters: Set updates to `message`.
   - Credentials: Connect a valid Telegram account credential using your bot token from `@BotFather`.
2. **Create Node 02 (`02 — Read Message & Classify Intent`):**
   - Type: `n8n-nodes-base.code`
   - Parameters: Mode = `runOnceForAllItems`. Paste the JavaScript code normalizing chat identifiers, text, and commands (`/start`, `/help`).
   - Connection: Connect Node 01 output to Node 02 input.
3. **Create Node 03 (`03 — Route Request`):**
   - Type: `n8n-nodes-base.switch`
   - Parameters: Add a rule matching `={{ $json.route }}` equals `help` (outputKey: `help`). Set fallback output to `extra`.
   - Connection: Connect Node 02 output to Node 03 input.
4. **Create Node 04 (`04 — Extract & Validate Reel URL`):**
   - Type: `n8n-nodes-base.code`
   - Parameters: Mode = `runOnceForAllItems`. Paste the regex-based URL extraction and validation script.
   - Connection: Connect Node 03 fallback output (`extra`) to Node 04 input.
5. **Create Node 05 (`05 — Reel URL Valid?`):**
   - Type: `n8n-nodes-base.if`
   - Parameters: Condition `={{ $json.urlValid }}` equals `true`.
   - Connection: Connect Node 04 output to Node 05 input. Route `true` to Node 06, route `false` to Node 17.
6. **Create Node 06 (`06 — Check Recent Cache`):**
   - Type: `n8n-nodes-base.code`
   - Parameters: Mode = `runOnceForAllItems`. Paste cache verification code checking `$getWorkflowStaticData('global')`.
   - Connection: Connect Node 05 `true` output to Node 06 input.
7. **Create Node 07 (`07 — Cached Result Available?`):**
   - Type: `n8n-nodes-base.if`
   - Parameters: Condition `={{ $json.cacheHit }}` equals `true`.
   - Connection: Connect Node 06 output to Node 07 input. Route `true` to Node 17, route `false` to Node 08.
8. **Create Node 08 (`08 — Start Transcript Run`):**
   - Type: `n8n-nodes-base.httpRequest`
   - Parameters: Method = `POST`, URL = `https://api.apify.com/v2/acts/lance_api~instagram-video-ai-transcript-scraper-api/runs`. Specify Body = JSON (`{"urls":["{{ $json.normalizedUrl }}"],"formats":["text"],"translate":false,"includeMediaMetadata":true}`). Query parameters: `waitForFinish=60`, `timeout=600`, `maxTotalChargeUsd=1`.
   - Credentials: Connect a Generic Header Auth credential named `Apify (Header Auth)` with Header Name `Authorization` and Value `Bearer <your_token>`. Set retry on fail to 3 tries.
   - Connection: Connect Node 07 `false` output to Node 08 input.
9. **Create Node 09 (`09 — Transcript Run Ready?`):**
   - Type: `n8n-nodes-base.if`
   - Parameters: Condition `={{ ['SUCCEEDED','FAILED','ABORTED','TIMED-OUT'].includes($json.data.status) }}` equals `true`.
   - Connection: Connect Node 08 output to Node 09 input. Route `true` to Node 11, route `false` to Node 10.
10. **Create Node 10 (`10 — Wait & Re-check Transcript Run`):**
    - Type: `n8n-nodes-base.httpRequest`
    - Parameters: Method = `GET`, URL = `https://api.apify.com/v2/actor-runs/{{ $json.data.id }}`. Query parameters: `waitForFinish=55`.
    - Credentials: Use the same Apify Header Auth credential.
    - Connection: Connect Node 09 `false` output to Node 10 input.
11. **Create Node 09B (`09B — Run Ready After Wait?`):**
    - Type: `n8n-nodes-base.if`
    - Parameters: Condition `={{ ['SUCCEEDED','FAILED','ABORTED','TIMED-OUT'].includes($json.data.status) }}` equals `true`.
    - Connection: Connect Node 10 output to Node 09B input. Route `true` to Node 11, route `false` to Node 10E.
12. **Create Node 10E (`10E — Mark Failure: Transcript Run Took Too Long`):**
    - Type: `n8n-nodes-base.set`
    - Parameters: Assignment `failureCode` = `ACTOR_TIMEOUT`.
    - Connection: Connect Node 09B `false` output to Node 10E input. Route output to Node 17.
13. **Create Node 11 (`11 — Read Transcript Result`):**
    - Type: `n8n-nodes-base.httpRequest`
    - Parameters: Method = `GET`, URL = `https://api.apify.com/v2/datasets/{{ $json.data.defaultDatasetId }}/items`. Query parameters: `clean=true`.
    - Credentials: Use Apify Header Auth.
    - Connection: Connect Node 09 (`true`) and Node 09B (`true`) outputs to Node 11 input.
14. **Create Node 12 (`12 — Validate Transcript`):**
    - Type: `n8n-nodes-base.code`
    - Parameters: Mode = `runOnceForAllItems`. Paste script validating dataset items, status mapping, and character thresholds.
    - Connection: Connect Node 11 output to Node 12 input.
15. **Create Node 13 (`13 — Transcript Usable?`):**
    - Type: `n8n-nodes-base.if`
    - Parameters: Condition `={{ $json.transcriptUsable }}` equals `true`.
    - Connection: Connect Node 12 output to Node 13 input. Route `true` to Node 14, route `false` to Node 17.
16. **Create Node 14 (`14 — Build AI Prompt`):**
    - Type: `n8n-nodes-base.code`
    - Parameters: Mode = `runOnceForAllItems`. Paste script assembling system instructions, originality rules, metadata, and prompt strings.
    - Connection: Connect Node 13 `true` output to Node 14 input.
17. **Create AI Sub-Nodes (`OpenAI Chat Model` & `Structured Output Parser`):**
    - *OpenAI Chat Model:* Type `@n8n/n8n-nodes-langchain.lmChatOpenAi`. Model = `gpt-4o-mini`, Temperature = `0.2`. Connect OpenAI API credentials.
    - *Structured Output Parser:* Type `@n8n/n8n-nodes-langchain.outputParserStructured`. Set manual schema type with the 22 required fields (`summary`, `opening_hook`, `hook_type`, `core_topic`, `content_angle`, `target_audience`, `problem_desire`, `key_message`, `cta`, `emotional_trigger`, `content_structure`, `why_it_works`, `original_hook_1`, `original_hook_2`, `original_hook_3`, `content_idea_1`, `content_idea_2`, `content_idea_3`, `new_script_hook`, `new_script_body`, `new_script_cta`, `language`) and enable auto-fix.
18. **Create Node 15 (`15 — Analyze Reel With AI`):**
    - Type: `@n8n/n8n-nodes-langchain.chainLlm`
    - Parameters: Text = `={{ $json.prompt }}`. Has Output Parser = `true`.
    - Connections: Connect Node 14 to main input; connect OpenAI Chat Model to AI language model input; connect Structured Output Parser to AI output parser input; connect OpenAI Chat Model to Structured Output Parser model input.
19. **Create Node 16 (`16 — AI Result Complete?`):**
    - Type: `n8n-nodes-base.if`
    - Parameters: Verify presence of `summary` and `new_script_body`.
    - Connection: Connect Node 15 output to Node 16 input. Route outputs to Node 17.
20. **Create Node 17 (`17 — Format Creator Insights`):**
    - Type: `n8n-nodes-base.code`
    - Parameters: Mode = `runOnceForAllItems`. Paste formatting, chunking, and error-handling code.
    - Connections: Connect Node 03 (`help`), Node 05 (`false`), Node 07 (`true`), Node 10E, Node 13 (`false`), and Node 16 outputs to Node 17 input.
21. **Create Node 18 (`18 — Reply In Telegram`):**
    - Type: `n8n-nodes-base.telegram`
    - Parameters: Action = send message, Chat ID = `={{ $json.chatId }}`, Text = `={{ $json.chunk }}`, Reply to message ID = `={{ $json.messageId }}`, Disable web page preview = `true`.
    - Credentials: Connect Telegram account credential.
    - Connection: Connect Node 17 output to Node 18 input.
22. **Create Node 19 (`19 — Save To Cache`):**
    - Type: `n8n-nodes-base.code`
    - Parameters: Mode = `runOnceForAllItems`. Paste cache persistence script writing successful outputs to workflow static storage.
    - Connection: Connect Node 18 output to Node 19 input.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify Actor documentation and usage reference for `lance_api/instagram-video-ai-transcript-scraper-api`. | Used for extracting media metadata and transcription. Billed at approximately $0.01 per successful run. |
| Telegram Bot API documentation regarding character length limitations and Markdown/HTML parsing. | Informs chunking limits (3800-character safety margin under 4096 limit) and bracket-stripping logic. |
Create LinkedIn, Facebook and Bluesky posts from Google Sheets with Gemini, Nano Banana and PostWire

https://n8nworkflows.xyz/workflows/create-linkedin--facebook-and-bluesky-posts-from-google-sheets-with-gemini--nano-banana-and-postwire-20917


# Create LinkedIn, Facebook and Bluesky posts from Google Sheets with Gemini, Nano Banana and PostWire

### 1. Workflow Overview

This workflow automates the creation and publication of multi-platform social media posts (LinkedIn, Facebook, and Bluesky) derived from a content queue stored in a Google Sheet. It executes every weekday at 09:00, fetching a pending content topic, generating tailored text copy and a matching AI illustration using Google Gemini models, publishing the assets across networks via PostWire, and dynamically logging success or failure states back to the sheet alongside email notifications if errors occur.

The execution logic is partitioned into five distinct functional blocks:

- **1.1 Schedule & Initialization:** Triggers execution on weekdays, establishes global configuration variables (voice tone, image style, notification email), and reads available records from the content source.
- **1.2 Topic Filtering & Claiming:** Evaluates retrieved spreadsheet rows to ensure valid data exists, selects the highest-priority single entry, and updates its tracking status to prevent duplicate processing.
- **1.3 AI Content & Asset Generation:** Interacts with Google Gemini to produce targeted social media copy structured as JSON and generates a corresponding editorial illustration via an image-generation model.
- **1.4 Multi-Platform Publishing:** Distributes the generated text formats and binary image assets to LinkedIn, Facebook, and Bluesky via PostWire, collecting and aggregating execution results for validation.
- **1.5 Logging & Exception Handling:** Evaluates publishing success; updates the source sheet with platform permalinks upon full success, or logs detailed error notes and sends an automated Gmail alert if any delivery step fails.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Initialization
- **Overview:** Initializes the automated pipeline on a weekday schedule, injects system configuration parameters, and queries the primary Google Sheet for queued items.
- **Nodes Involved:** `Every weekday at 9:00`, `Post settings`, `Read the topics`

##### Node Details:
- **Every weekday at 9:00**
  - **Type & Role:** `n8n-nodes-base.scheduleTrigger` (Trigger) — Sets the time-based execution cadence.
  - **Configuration:** Configured to run weekly on days 1 through 5 (Monday–Friday) at hour 09:00.
  - **Expressions:** None.
  - **Connections:** Input: None; Output: `Post settings`
  - **Failure Modes:** Missed executions due to n8n downtime or host service interruption.

- **Post settings**
  - **Type & Role:** `n8n-nodes-base.set` (Data Transformation) — Establishes persistent variables for voice tone, illustration art style, and target recipient email.
  - **Configuration:** Assigns three string properties: `voice`, `image_style`, and `email`.
  - **Expressions:** Static text strings defining the content generation guidelines.
  - **Connections:** Input: `Every weekday at 9:00`; Output: `Read the topics`
  - **Failure Modes:** Schema mismatch if downstream nodes fail to reference the `.first().json` accessor correctly.

- **Read the topics**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Data Input) — Queries the target spreadsheet for rows queued for publication.
  - **Configuration:** Operation set to `read`; filters rows where column `status` matches `ready`. Requires valid Google Sheets OAuth2 credentials.
  - **Expressions:** None.
  - **Connections:** Input: `Post settings`; Output: `Has a topic`
  - **Failure Modes:** API quota limits, authentication token expiration, or missing sheet headers.

---

#### 2.2 Topic Filtering & Claiming
- **Overview:** Validates that retrieved rows contain actual content, isolates the single next item to process, and locks the row status to prevent concurrent execution races.
- **Nodes Involved:** `Has a topic`, `Take the next topic`, `Mark the topic as writing`

##### Node Details:
- **Has a topic**
  - **Type & Role:** `n8n-nodes-base.filter` (Flow Control) — Validates that the fetched row contains a non-empty topic string.
  - **Configuration:** Evaluates string length and presence, trimming whitespace.
  - **Expressions:** `={{ String($json.topic || '').trim() }}` (Condition operator: `notEmpty`)
  - **Connections:** Input: `Read the topics`; Output: `Take the next topic`
  - **Failure Modes:** Data type discrepancies if incoming sheet cells contain boolean or numeric types instead of strings.

- **Take the next topic**
  - **Type & Role:** `n8n-nodes-base.limit` (Flow Control) — Limits downstream item batch sizes to exactly one row per execution cycle.
  - **Configuration:** `maxItems` set to `1`.
  - **Expressions:** None.
  - **Connections:** Input: `Has a topic`; Output: `Mark the topic as writing`
  - **Failure Modes:** None standard.

- **Mark the topic as writing**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Data Output) — Updates the source spreadsheet row status to claim it for processing.
  - **Configuration:** Operation set to `update`; updates `status` to `writing` matched against the `row_number` column.
  - **Expressions:** `={{ $json.row_number }}`
  - **Connections:** Input: `Take the next topic`; Output: `Write the posts with Gemini`
  - **Failure Modes:** Write permission errors on the Google Sheet or missing row identifiers.

---

#### 2.3 AI Content & Asset Generation
- **Overview:** Sends the topic facts and style parameters to Google Gemini to generate structured multi-platform text, parses the JSON payload, and requests an AI-generated image.
- **Nodes Involved:** `Write the posts with Gemini`, `Read the plan`, `Nano Banana image`

##### Node Details:
- **Write the posts with Gemini**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.googleGemini` (AI / LLM) — Generates platform-specific social copy and an image prompt based on input topics.
  - **Configuration:** Uses model `models/gemini-2.5-flash` with system instructions enforcing strict factual adherence and structured JSON output containing `linkedin`, `facebook`, `bluesky`, and `image_prompt` keys.
  - **Expressions:** Constructs multi-line prompt dynamically pulling topic, notes, voice, and image style from preceding nodes.
  - **Connections:** Input: `Mark the topic as writing`; Output: `Read the plan`
  - **Failure Modes:** API rate limits, invalid JSON generation, or network timeouts. Requires Google Gemini API credentials.

- **Read the plan**
  - **Type & Role:** `n8n-nodes-base.set` (Data Transformation) — Parses the raw JSON response from the LLM into structured n8n data objects.
  - **Configuration:** Assigns properties: `plan` (parsed JSON object), `topic` (string), and `row_number` (number).
  - **Expressions:** `={{ JSON.parse($('Write the posts with Gemini').item.json.content.parts.map(p => p.text || '').join('')) }}`
  - **Connections:** Input: `Write the posts with Gemini`; Output: `Nano Banana image`
  - **Failure Modes:** Syntax errors if Gemini returns malformed JSON outside markdown code blocks.

- **Nano Banana image**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.googleGemini` (AI / Media Generation) — Generates an editorial illustration based on the prompt returned by the text model.
  - **Configuration:** Resource set to `image`, operation `generate`, model `models/gemini-2.5-flash-image`. Error handling set to `continueRegularOutput` to capture generation failures gracefully.
  - **Expressions:** `={{ $json.plan.image_prompt || ($json.topic + '. ' + $('Post settings').first().json.image_style) }}`
  - **Connections:** Input: `Read the plan`; Output: `Post to LinkedIn, Facebook, Bluesky`
  - **Failure Modes:** Image generation service failure, safety filter blocks, or quota exhaustion.

---

#### 2.4 Multi-Platform Publishing
- **Overview:** Uploads the generated binary image asset along with network-specific text drafts to PostWire for synchronized publication across social platforms, then aggregates the results.
- **Nodes Involved:** `Post to LinkedIn, Facebook, Bluesky`, `Collect the results`, `Everything published?`

##### Node Details:
- **Post to LinkedIn, Facebook, Bluesky**
  - **Type & Role:** `n8n-nodes-postwire.postWire` (Social Integration) — Publishes multi-channel posts with media attachments via PostWire.
  - **Configuration:** Resource `post`, operation `publish`, platforms specified as `["linkedin", "facebook", "bluesky"]`, media source set to binary property `data`. Error handling set to `continueRegularOutput`.
  - **Expressions:** Drafts mapped dynamically via array mapping over platform keys; idempotency key generated using row and topic identifiers.
  - **Connections:** Input: `Nano Banana image`; Output: `Collect the results`
  - **Failure Modes:** PostWire API authentication failure, expired platform tokens within PostWire, or payload validation errors. Requires PostWire API credentials.

- **Collect the results**
  - **Type & Role:** `n8n-nodes-base.aggregate` (Data Transformation) — Combines multi-item platform delivery responses into a single aggregated array.
  - **Configuration:** Aggregate mode set to `aggregateAllItemData` with destination field name `results`.
  - **Expressions:** None.
  - **Connections:** Input: `Post to LinkedIn, Facebook, Bluesky`; Output: `Everything published?`
  - **Failure Modes:** None standard.

- **Everything published?**
  - **Type & Role:** `n8n-nodes-base.if` (Flow Control) — Verifies whether every target platform reported a successful publication status.
  - **Configuration:** Evaluates array length and checks if all response objects contain `ok: true`.
  - **Expressions:** `={{ $json.results.length > 0 && $json.results.every(r => r.ok === true) }}`
  - **Connections:** Input: `Collect the results`; Output: True branch (`Mark the topic as posted`), False branch (`Write the fix in the row`)
  - **Failure Modes:** None standard.

---

#### 2.5 Logging & Exception Handling
- **Overview:** Writes final outcome metadata back to the source Google Sheet—recording permalinks upon complete success or compiling error diagnostics and triggering an alert email if failures occur.
- **Nodes Involved:** `Mark the topic as posted`, `Write the fix in the row`, `Email me the fix`

##### Node Details:
- **Mark the topic as posted**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Data Output) — Updates the tracking row upon successful multi-platform publication.
  - **Configuration:** Operation set to `update`; sets status to `posted`, records current timestamp, formats published platform links, and clears error notes.
  - **Expressions:** Uses `$now.toFormat('yyyy-MM-dd HH:mm')` and maps platform URLs from aggregated results.
  - **Connections:** Input: `Everything published?` (True branch); Output: None (Terminal node)
  - **Failure Modes:** Google Sheets API write errors or permission restrictions.

- **Write the fix in the row**
  - **Type & Role:** `n8n-nodes-base.googleSheets` (Data Output) — Updates the spreadsheet row with failure details and partial success links if publishing encounters an error.
  - **Configuration:** Operation set to `update`; sets status to `needs fix` or `partly posted` based on partial success checks, logging descriptive error messages from Gemini or PostWire.
  - **Expressions:** Complex IIFE script extracting image generation error messages or network-specific API error hints.
  - **Connections:** Input: `Everything published?` (False branch); Output: `Email me the fix`
  - **Failure Modes:** Google Sheets API write errors.

- **Email me the fix**
  - **Type & Role:** `n8n-nodes-base.gmail` (Communication) — Sends an administrative email alert detailing publishing failures and instructions for remediation.
  - **Configuration:** Sends plaintext email via Gmail node using configured recipient address and dynamic subject/body content. Requires Gmail OAuth2 credentials.
  - **Expressions:** Dynamically constructs email subject line and body incorporating topic details, error summaries, and partial URLs.
  - **Connections:** Input: `Write the fix in the row`; Output: None (Terminal node)
  - **Failure Modes:** Gmail API authentication expiration, daily sending limits, or invalid recipient addresses.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Every weekday at 9:00` | `n8n-nodes-base.scheduleTrigger` | Schedule Trigger | None | `Post settings` | ## Write LinkedIn posts with an AI image and post them from a list of topics... |
| `Post settings` | `n8n-nodes-base.set` | Configuration Variable Assignment | `Every weekday at 9:00` | `Read the topics` | ## Write LinkedIn posts with an AI image and post them from a list of topics... |
| `Read the topics` | `n8n-nodes-base.googleSheets` | Read Ready Rows from Google Sheets | `Post settings` | `Has a topic` | ## Write LinkedIn posts with an AI image and post them from a list of topics... |
| `Has a topic` | `n8n-nodes-base.filter` | Validate Topic Content | `Read the topics` | `Take the next topic` | ## 1. Next topic<br>The first row marked ready, claimed before anything is written. |
| `Take the next topic` | `n8n-nodes-base.limit` | Limit Item Batch Size | `Has a topic` | `Mark the topic as writing` | ## 1. Next topic<br>The first row marked ready, claimed before anything is written. |
| `Mark the topic as writing` | `n8n-nodes-base.googleSheets` | Update Row Status to Writing | `Take the next topic` | `Write the posts with Gemini` | ## 1. Next topic<br>The first row marked ready, claimed before anything is written. |
| `Write the posts with Gemini` | `@n8n/n8n-nodes-langchain.googleGemini` | Generate Post Copy & Image Prompt | `Mark the topic as writing` | `Read the plan` | ## 2. Write with Gemini<br>One post per network and the brief for the image. |
| `Read the plan` | `n8n-nodes-base.set` | Parse LLM JSON Response | `Write the posts with Gemini` | `Nano Banana image` | ## 2. Write with Gemini<br>One post per network and the brief for the image. |
| `Nano Banana image` | `@n8n/n8n-nodes-langchain.googleGemini` | Generate AI Illustration | `Read the plan` | `Post to LinkedIn, Facebook, Bluesky` | ## 3. Image and post<br>Nano Banana draws the image; PostWire uploads it and publishes each text. |
| `Post to LinkedIn, Facebook, Bluesky` | `n8n-nodes-postwire.postWire` | Publish Content via PostWire | `Nano Banana image` | `Collect the results` | ## 3. Image and post<br>Nano Banana draws the image; PostWire uploads it and publishes each text. |
| `Collect the results` | `n8n-nodes-base.aggregate` | Aggregate Publishing Results | `Post to LinkedIn, Facebook, Bluesky` | `Everything published?` | ## 4. Log and alert<br>The post links in the row, or the fix in the row and by email. |
| `Everything published?` | `n8n-nodes-base.if` | Evaluate Success Status | `Collect the results` | `Mark the topic as posted`, `Write the fix in the row` | ## 4. Log and alert<br>The post links in the row, or the fix in the row and by email. |
| `Mark the topic as posted` | `n8n-nodes-base.googleSheets` | Log Success State & URLs | `Everything published?` (True) | None | ## 4. Log and alert<br>The post links in the row, or the fix in the row and by email. |
| `Write the fix in the row` | `n8n-nodes-base.googleSheets` | Log Error Details & Partial State | `Everything published?` (False) | `Email me the fix` | ## 4. Log and alert<br>The post links in the row, or the fix in the row and by email. |
| `Email me the fix` | `n8n-nodes-base.gmail` | Send Error Alert Email | `Write the fix in the row` | None | ## 4. Log and alert<br>The post links in the row, or the fix in the row and by email. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps sequentially to rebuild the workflow in n8n manually:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** (`n8n-nodes-base.scheduleTrigger`) named `Every weekday at 9:00`.
   - Configure the rule interval to run weekly on days `[1, 2, 3, 4, 5]` at hour `9`.

2. **Configure Global Settings:**
   - Add a **Set** node (`n8n-nodes-base.set`) named `Post settings`.
   - Create three string assignments:
     - `voice`: `Plain and specific, first person plural, short sentences, no hype and no emojis`
     - `image_style`: `Flat editorial illustration, landscape 16:9, calm colours, no text, no logos, no real people`
     - `email`: `you@example.com` (replace with your target alert address).
   - Connect `Every weekday at 9:00` $\rightarrow$ `Post settings`.

3. **Set Up Google Sheets Integration (Input):**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Read the topics`.
   - Set operation to `Read`, add a filter where `lookupColumn` is `status` and `lookupValue` is `ready`. Configure valid Google Sheets credentials.
   - Connect `Post settings` $\rightarrow$ `Read the topics`.

4. **Filter and Claim Topics:**
   - Add a **Filter** node (`n8n-nodes-base.filter`) named `Has a topic`.
   - Set condition: Left value `={{ String($json.topic || '').trim() }}`, operator `Not Empty`. Connect `Read the topics` $\rightarrow$ `Has a topic`.
   - Add a **Limit** node (`n8n-nodes-base.limit`) named `Take the next topic` with `maxItems` set to `1`. Connect `Has a topic` $\rightarrow$ `Take the next topic`.
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Mark the topic as writing`. Set operation to `Update`, matching columns on `row_number`, and setting status to `writing`. Connect `Take the next topic` $\rightarrow$ `Mark the topic as writing`.

5. **AI Content Generation:**
   - Add a **Google Gemini** node (`@n8n/n8n-nodes-langchain.googleGemini`) named `Write the posts with Gemini`.
   - Set resource to `Text`, operation to `Message`, model to `models/gemini-2.5-flash`, and enable `JSON Output`.
   - Configure system message instructing the LLM to return JSON with keys `linkedin`, `facebook`, `bluesky`, and `image_prompt`.
   - Set user message expression:
     `={{ 'Topic: ' + $('Take the next topic').item.json.topic + '\\nNotes and facts: ' + ($('Take the next topic').item.json.notes || 'none') + '\\nVoice: ' + $('Post settings').first().json.voice + '\\nImage style: ' + $('Post settings').first().json.image_style }}`
   - Connect `Mark the topic as writing` $\rightarrow$ `Write the posts with Gemini`.
   - Add a **Set** node (`n8n-nodes-base.set`) named `Read the plan` to parse the JSON response:
     - `plan`: `={{ JSON.parse($('Write the posts with Gemini').item.json.content.parts.map(p => p.text || '').join('')) }}`
     - `topic`: `={{ $('Take the next topic').item.json.topic }}`
     - `row_number`: `={{ $('Take the next topic').item.json.row_number }}`
   - Connect `Write the posts with Gemini` $\rightarrow$ `Read the plan`.

6. **AI Image Generation & Publishing:**
   - Add a **Google Gemini** node (`@n8n/n8n-nodes-langchain.googleGemini`) named `Nano Banana image`. Set resource to `Image`, operation to `Generate`, model to `models/gemini-2.5-flash-image`. Set node error handling to `Continue (Using Error Output)` or `Continue Regular Output`.
   - Configure prompt expression: `={{ $json.plan.image_prompt || ($json.topic + '. ' + $('Post settings').first().json.image_style) }}`.
   - Connect `Read the plan` $\rightarrow$ `Nano Banana image`.
   - Add a **PostWire** node (`n8n-nodes-postwire.postWire`) named `Post to LinkedIn, Facebook, Bluesky`. Configure resource to `Post`, operation to `Publish`, platforms to `["linkedin", "facebook", "bluesky"]`, media source to `Binary`, binary property name to `data`. Set error handling to continue execution on failure.
   - Configure drafts expression mapping network text content. Connect `Nano Banana image` $\rightarrow$ `Post to LinkedIn, Facebook, Bluesky`.

7. **Result Aggregation & Evaluation:**
   - Add an **Aggregate** node (`n8n-nodes-base.aggregate`) named `Collect the results` with aggregate mode `aggregateAllItemData` and destination field `results`. Connect `Post to LinkedIn, Facebook, Bluesky` $\rightarrow$ `Collect the results`.
   - Add an **If** node (`n8n-nodes-base.if`) named `Everything published?` checking if all results succeeded (`={{ $json.results.length > 0 && $json.results.every(r => r.ok === true) }}`). Connect `Collect the results` $\rightarrow$ `Everything published?`.

8. **Logging & Notifications:**
   - **True Branch:** Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Mark the topic as posted`. Operation `Update`, matching on `row_number`, setting status to `posted`, `posted_at` to timestamp, and formatting `links`. Connect `Everything published?` (True output) $\rightarrow$ `Mark the topic as posted`.
   - **False Branch:** Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`) named `Write the fix in the row`. Operation `Update`, matching on `row_number`, setting status to `needs fix` / `partly posted`, and logging error details. Connect `Everything published?` (False output) $\rightarrow$ `Write the fix in the row`.
   - Add a **Gmail** node (`n8n-nodes-base.gmail`) named `Email me the fix`. Set sendTo to `={{ $('Post settings').first().json.email }}`, configure subject and body expressions. Connect `Write the fix in the row` $\rightarrow$ `Email me the fix`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| PostWire Integration Setup | Requires a PostWire account (postwire.io) with connected social profiles and an installed PostWire API credential in n8n. |
| Google Sheet Column Schema | The target Google Sheet must contain headers: `topic`, `notes`, `status`, `posted_at`, `links`, and `note`. |
| AI Model Credentials | Requires active Google Gemini API credentials assigned to both text and image generation nodes. |
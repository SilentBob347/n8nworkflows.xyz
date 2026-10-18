Create MiniMax Hailuo short videos from Google Sheets and post with PostWire

https://n8nworkflows.xyz/workflows/create-minimax-hailuo-short-videos-from-google-sheets-and-post-with-postwire-20918


# Create MiniMax Hailuo short videos from Google Sheets and post with PostWire

### 1. Workflow Overview

This workflow automates the generation and multi-platform publishing of short-form vertical videos (9:16 format) using MiniMax and PostWire, driven by a content queue managed in Google Sheets. It runs on a daily schedule, fetches pending video ideas, generates an AI-driven video plan and a rendered MP4 file via MiniMax, publishes the content simultaneously to TikTok, Instagram Reels, and YouTube Shorts via PostWire, and records the results back to the spreadsheet. If any failures occur during rendering or publishing, the workflow logs the error details to the sheet and alerts the administrator via Gmail.

The workflow logic is categorized into the following functional blocks:
- **1.1 Trigger & Configuration**: Initiates daily execution and establishes global defaults for video style and administrator notifications.
- **1.2 Data Ingestion & Queue Claiming**: Queries Google Sheets for ideas marked as "ready", filters valid inputs, selects the next item, and marks it as "generating" to prevent duplicate processing.
- **1.3 AI Planning & Video Rendering**: Uses MiniMax text generation to build a structured JSON production plan, then invokes MiniMax Hailuo to render and download a 6-second vertical video clip.
- **1.4 Multi-Platform Publishing & Aggregation**: Publishes the rendered video file across TikTok, Instagram, and YouTube via PostWire, collecting and evaluating execution results.
- **1.5 Logging & Error Handling**: Updates Google Sheets with success links or failure diagnostics, and dispatches a notification email via Gmail if intervention is required.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Configuration
- **Overview:** Initializes the automation schedule every day at 10:00 AM and sets baseline parameters for video visual style and notification routing.
- **Nodes Involved:** `Every day at 10:00`, `Channel settings`
- **Node Details:**
  - **Every day at 10:00**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger). Initiates the workflow daily.
    - *Configuration:* Interval set to run every day at hour 10.
    - *Connections:* Input: None; Output: `Channel settings`.
    - *Edge Cases:* Server downtime or timezone misconfigurations can delay execution.
  - **Channel settings**
    - *Type & Role:* `n8n-nodes-base.set` (Edit Fields). Defines static environmental variables used downstream.
    - *Configuration:* Assigns `style` ("Cinematic, vertical 9:16...") and `email` ("you@example.com").
    - *Expressions:* None (static string assignments).
    - *Connections:* Input: `Every day at 10:00`; Output: `Read the video ideas`.
    - *Edge Cases:* Missing email strings will cause downstream Gmail sending failures.

#### 1.2 Data Ingestion & Queue Claiming
- **Overview:** Queries Google Sheets for rows where the status column is set to "ready", validates content integrity, isolates a single row, and locks its status to "generating".
- **Nodes Involved:** `Read the video ideas`, `Has an idea`, `Take the next idea`, `Mark the idea as generating`
- **Node Details:**
  - **Read the video ideas**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Google Sheets). Reads spreadsheet rows matching a specific query filter.
    - *Configuration:* Operation set to `read`; Filter setup looks up rows where `lookupColumn` is `status` and `lookupValue` is `ready`.
    - *Connections:* Input: `Channel settings`; Output: `Has an idea`.
    - *Edge Cases:* API rate limits, invalid Google credentials, or missing column headers.
  - **Has an idea**
    - *Type & Role:* `n8n-nodes-base.filter` (Filter). Ensures that the retrieved row contains actual text in the idea field.
    - *Configuration:* Combinator: `and`; condition checks if `={{ String($json.idea || '').trim() }}` is not empty.
    - *Expressions:* `={{ String($json.idea || '').trim() }}`
    - *Connections:* Input: `Read the video ideas`; Output: `Take the next idea`.
    - *Edge Cases:* Empty string values bypass processing safely.
  - **Take the next idea**
    - *Type & Role:* `n8n-nodes-base.limit` (Limit). Restricts incoming items to process only the first available row.
    - *Configuration:* `maxItems` set to `1`.
    - *Connections:* Input: `Has an idea`; Output: `Mark the idea as generating`.
    - *Edge Cases:* None.
  - **Mark the idea as generating**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Google Sheets). Updates the spreadsheet row status to lock it against concurrent runs.
    - *Configuration:* Operation set to `update`; matching columns uses `row_number`; updates `status` to `generating`.
    - *Expressions:* `row_number`: `={{ $json.row_number }}`
    - *Connections:* Input: `Take the next idea`; Output: `Write the video prompt`.
    - *Edge Cases:* Mismatched row identifiers or concurrent write conflicts.

#### 1.3 AI Planning & Video Rendering
- **Overview:** Generates a structured JSON production plan using MiniMax language models, then calls MiniMax Hailuo to render and download a 9:16 vertical video clip.
- **Nodes Involved:** `Write the video prompt`, `Read the plan`, `Hailuo render`
- **Node Details:**
  - **Write the video prompt**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.minimax` (MiniMax). Performs AI text completion to build video parameters.
    - *Configuration:* Model set to `MiniMax-M3`; resource set to `text`; system prompt enforces strict JSON output containing `video_prompt`, `caption_idea`, and `youtube_title`.
    - *Expressions:* User message combines idea and style: `={{ 'Idea: ' + $('Take the next idea').item.json.idea + '\\nStyle: ' + ($('Take the next idea').item.json.style || $('Channel settings').first().json.style) }}`
    - *Connections:* Input: `Mark the idea as generating`; Output: `Read the plan`.
    - *Edge Cases:* Model hallucination failing to produce valid JSON (handled downstream by parser).
  - **Read the plan**
    - *Type & Role:* `n8n-nodes-base.set` (Edit Fields). Parses the AI text response into clean data structures and aggregates contextual row variables.
    - *Configuration:* Assigns parsed JSON object to `plan`, and passes through `idea`, `style`, and `row_number`.
    - *Expressions:* `plan`: `={{ (() => { const t = String($('Write the video prompt').item.json.content || ''); const m = t.match(/\\{[\\s\\S]*\\}/); try { return JSON.parse(m ? m[0] : t); } catch (e) { return {}; } })() }}`
    - *Connections:* Input: `Write the video prompt`; Output: `Hailuo render`.
    - *Edge Cases:* Malformed JSON payloads return empty objects.
  - **Hailuo render**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.minimax` (MiniMax). Generates and downloads a text-to-video asset.
    - *Configuration:* Resource set to `video`; operation set to `textToVideo`; model `MiniMax-H3`; ratio `9:16`; `h3Duration` `6` seconds; `h3Resolution` `768P`; `downloadVideo` enabled (`true`). Error handling set to `continueRegularOutput`.
    - *Expressions:* Prompt: `={{ $json.plan.video_prompt || ($json.idea + '. ' + $json.style) }}`
    - *Connections:* Input: `Read the plan`; Output: `Post to TikTok, Reels and Shorts`.
    - *Edge Cases:* API timeouts during video rendering, insufficient account credits, or strict content moderation flags.

#### 1.4 Multi-Platform Publishing & Aggregation
- **Overview:** Uploads the rendered video binary to PostWire for simultaneous distribution across TikTok, Instagram, and YouTube, then aggregates the API results.
- **Nodes Involved:** `Post to TikTok, Reels and Shorts`, `Collect the results`, `Everything published?`
- **Node Details:**
  - **Post to TikTok, Reels and Shorts**
    - *Type & Role:* `n8n-nodes-postwire.postWire` (PostWire). Publishes multimedia content to connected social channels.
    - *Configuration:* Resource `post`; operation `publish`; platforms selected: `tiktok`, `instagram`, `youtube`; content mode `smart`; media source `binary`; binary property `data`. Error handling set to `continueRegularOutput`.
    - *Expressions:* Prompt: `={{ $('Read the plan').item.json.plan.caption_idea || $('Read the plan').item.json.idea }}`; Title: `={{ String($('Read the plan').item.json.plan.youtube_title || $('Read the plan').item.json.idea).slice(0, 100) }}`; Idempotency Key: `={{ ('hailuo-idea-row-' + $('Read the plan').item.json.row_number + '-' + $('Read the plan').item.json.idea).slice(0, 200) }}`
    - *Connections:* Input: `Hailuo render`; Output: `Collect the results`.
    - *Edge Cases:* Expired social tokens in PostWire, platform API rejections, or network payload size errors.
  - **Collect the results**
    - *Type & Role:* `n8n-nodes-base.aggregate` (Aggregate). Collects array items into a single consolidated results object.
    - *Configuration:* Aggregate set to `aggregateAllItemData`; destination field `results`.
    - *Connections:* Input: `Post to TikTok, Reels and Shorts`; Output: `Everything published?`.
    - *Edge Cases:* Empty input arrays if previous nodes fail silently.
  - **Everything published?**
    - *Type & Role:* `n8n-nodes-base.if` (If). Evaluates whether all platform publishing operations completed successfully.
    - *Configuration:* Condition verifies `={{ $json.results.length > 0 && $json.results.every(r => r.ok === true) }}`.
    - *Expressions:* `={{ $json.results.length > 0 && $json.results.every(r => r.ok === true) }}`
    - *Connections:* Input: `Collect the results`; Outputs: True branch to `Mark the idea as posted`, False branch to `Write the fix in the row`.
    - *Edge Cases:* Partial publishing successes route to failure handling.

#### 1.5 Logging & Error Handling
- **Overview:** Records successful post URLs to Google Sheets, or logs detailed diagnostics and sends a corrective email notification if errors occur.
- **Nodes Involved:** `Mark the idea as posted`, `Write the fix in the row`, `Email me the fix`
- **Node Details:**
  - **Mark the idea as posted**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Google Sheets). Updates the Google Sheet row upon complete publishing success.
    - *Configuration:* Operation `update`; matching column `row_number`; updates `status` to `posted`, `posted_at`, `links`, and clears `note`.
    - *Expressions:* Links: `={{ $json.results.map(r => r.platform + ' ' + (r.url || 'processing: the link appears in PostWire')).join(' | ') }}`; Timestamp: `={{ $now.toFormat('yyyy-MM-dd HH:mm') }}`
    - *Connections:* Input: `Everything published?` (True branch); Output: None.
    - *Edge Cases:* Spreadsheet write conflicts.
  - **Write the fix in the row**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Google Sheets). Logs error metadata to the spreadsheet when rendering or publishing fails.
    - *Configuration:* Operation `update`; matching column `row_number`; updates `status` to `needs fix` or `partly posted`, writes diagnostics into `note`.
    - *Expressions:* Note generation inspects MiniMax error payloads and PostWire response errors.
    - *Connections:* Input: `Everything published?` (False branch); Output: `Email me the fix`.
    - *Edge Cases:* Undefined error objects falling back gracefully.
  - **Email me the fix**
    - *Type & Role:* `n8n-nodes-base.gmail` (Gmail). Sends an email alert containing troubleshooting steps.
    - *Configuration:* Sends plain text email via configured Gmail credentials.
    - *Expressions:* Recipient: `={{ $('Channel settings').first().json.email }}`; Subject and body dynamically constructed from error states.
    - *Connections:* Input: `Write the fix in the row`; Output: None.
    - *Edge Cases:* Invalid Gmail OAuth tokens or recipient formatting issues.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Workflow documentation and configuration guide | None | None | [Create MiniMax Hailuo videos from a list of ideas and post them...] |
| `Sticky Note 2` | `n8n-nodes-base.stickyNote` | Block documentation for data ingestion | None | None | [## 1. Next idea The first row marked ready, claimed before anything is rendered.] |
| `Sticky Note 3` | `n8n-nodes-base.stickyNote` | Block documentation for MiniMax planning | None | None | [## 2. Plan with MiniMax The video prompt, a caption idea and the YouTube title.] |
| `Sticky Note 4` | `n8n-nodes-base.stickyNote` | Block documentation for rendering and posting | None | None | [## 3. Render and post MiniMax renders the clip; PostWire uploads it and writes a caption per network.] |
| `Sticky Note 5` | `n8n-nodes-base.stickyNote` | Block documentation for logging and alerts | None | None | [## 4. Log and alert The post links in the row, or the fix in the row and by email.] |
| `Every day at 10:00` | `n8n-nodes-base.scheduleTrigger` | Triggers the workflow daily at 10 AM | None | `Channel settings` | [Create MiniMax Hailuo videos from a list of ideas and post them...] |
| `Channel settings` | `n8n-nodes-base.set` | Sets default style and notification email | `Every day at 10:00` | `Read the video ideas` | [Create MiniMax Hailuo videos from a list of ideas and post them...] |
| `Read the video ideas` | `n8n-nodes-base.googleSheets` | Reads rows where status is "ready" | `Channel settings` | `Has an idea` | [Create MiniMax Hailuo videos from a list of ideas and post them...] |
| `Has an idea` | `n8n-nodes-base.filter` | Filters out rows with empty ideas | `Read the video ideas` | `Take the next idea` | [Create MiniMax Hailuo videos from a list of ideas and post them...] |
| `Take the next idea` | `n8n-nodes-base.limit` | Limits processing to 1 item | `Has an idea` | `Mark the idea as generating` | [Create MiniMax Hailuo videos from a list of ideas and post them...] |
| `Mark the idea as generating` | `n8n-nodes-base.googleSheets` | Updates sheet status to "generating" | `Take the next idea` | `Write the video prompt` | [Create MiniMax Hailuo videos from a list of ideas and post them...] |
| `Write the video prompt` | `@n8n/n8n-nodes-langchain.minimax` | Generates video prompt, caption, and title via AI | `Mark the idea as generating` | `Read the plan` | [Create MiniMax Hailuo videos from a list of ideas and post them...] |
| `Read the plan` | `n8n-nodes-base.set` | Parses JSON plan and aggregates row data | `Write the video prompt` | `Hailuo render` | [Create MiniMax Hailuo videos from a list of ideas and post them...] |
| `Hailuo render` | `@n8n/n8n-nodes-langchain.minimax` | Renders 9:16 video via MiniMax Hailuo | `Read the plan` | `Post to TikTok, Reels and Shorts` | [## 3. Render and post MiniMax renders the clip; PostWire uploads it and writes a caption per network.] |
| `Post to TikTok, Reels and Shorts` | `n8n-nodes-postwire.postWire` | Publishes video to social platforms via PostWire | `Hailuo render` | `Collect the results` | [## 3. Render and post MiniMax renders the clip; PostWire uploads it and writes a caption per network.] |
| `Collect the results` | `n8n-nodes-base.aggregate` | Aggregates publishing results into a single list | `Post to TikTok, Reels and Shorts` | `Everything published?` | [## 4. Log and alert The post links in the row, or the fix in the row and by email.] |
| `Everything published?` | `n8n-nodes-base.if` | Checks if all platforms published successfully | `Collect the results` | `Mark the idea as posted`, `Write the fix in the row` | [## 4. Log and alert The post links in the row, or the fix in the row and by email.] |
| `Mark the idea as posted` | `n8n-nodes-base.googleSheets` | Logs success links and updates status to "posted" | `Everything published?` | None | [## 4. Log and alert The post links in the row, or the fix in the row and by email.] |
| `Write the fix in the row` | `n8n-nodes-base.googleSheets` | Logs error details and status to sheet | `Everything published?` | `Email me the fix` | [## 4. Log and alert The post links in the row, or the fix in the row and by email.] |
| `Email me the fix` | `n8n-nodes-base.gmail` | Sends troubleshooting email notification | `Write the fix in the row` | None | [## 4. Log and alert The post links in the row, or the fix in the row and by email.] |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**
   - Type: `n8n-nodes-base.scheduleTrigger`
   - Name: `Every day at 10:00`
   - Configuration: Set trigger interval to every day at 10:00 AM.

2. **Create Channel Settings Node**
   - Type: `n8n-nodes-base.set`
   - Name: `Channel settings`
   - Configuration: Add string assignments for `style` (e.g., `"Cinematic, vertical 9:16, natural light..."`) and `email` (`"you@example.com"`).
   - Connection: Connect `Every day at 10:00` output to this node.

3. **Create Read Ideas Google Sheets Node**
   - Type: `n8n-nodes-base.googleSheets`
   - Name: `Read the video ideas`
   - Credentials: Configure Google Sheets OAuth2.
   - Configuration: Operation `read`; select target Document and Sheet; add filter where `lookupColumn` is `status` and `lookupValue` is `ready`.
   - Connection: Connect `Channel settings` output to this node.

4. **Create Idea Filter Node**
   - Type: `n8n-nodes-base.filter`
   - Name: `Has an idea`
   - Configuration: Add condition checking that `={{ String($json.idea || '').trim() }}` is not empty.
   - Connection: Connect `Read the video ideas` output to this node.

5. **Create Limit Node**
   - Type: `n8n-nodes-base.limit`
   - Name: `Take the next idea`
   - Configuration: Set `maxItems` to `1`.
   - Connection: Connect `Has an idea` output to this node.

6. **Create Status Update (Generating) Google Sheets Node**
   - Type: `n8n-nodes-base.googleSheets`
   - Name: `Mark the idea as generating`
   - Credentials: Google Sheets OAuth2.
   - Configuration: Operation `update`; mapping mode `defineBelow`; match by `row_number`; set `status` to `generating`.
   - Connection: Connect `Take the next idea` output to this node.

7. **Create MiniMax Text Planning Node**
   - Type: `@n8n/n8n-nodes-langchain.minimax`
   - Name: `Write the video prompt`
   - Credentials: MiniMax API.
   - Configuration: Model `MiniMax-M3`; resource `text`; system prompt requesting JSON with keys `video_prompt`, `caption_idea`, and `youtube_title`. User message expression: `={{ 'Idea: ' + $('Take the next idea').item.json.idea + '\\nStyle: ' + ($('Take the next idea').item.json.style || $('Channel settings').first().json.style) }}`.
   - Connection: Connect `Mark the idea as generating` output to this node.

8. **Create Plan Parser Node**
   - Type: `n8n-nodes-base.set`
   - Name: `Read the plan`
   - Configuration: Assign parsed JSON object to `plan`, and pass `idea`, `style`, and `row_number`.
   - Connection: Connect `Write the video prompt` output to this node.

9. **Create MiniMax Hailuo Render Node**
   - Type: `@n8n/n8n-nodes-langchain.minimax`
   - Name: `Hailuo render`
   - Credentials: MiniMax API.
   - Configuration: Resource `video`; operation `textToVideo`; model `MiniMax-H3`; ratio `9:16`; duration `6`; resolution `768P`; `downloadVideo` enabled (`true`). Set node error handling to `continueRegularOutput`.
   - Expression: Prompt: `={{ $json.plan.video_prompt || ($json.idea + '. ' + $json.style) }}`.
   - Connection: Connect `Read the plan` output to this node.

10. **Create PostWire Publishing Node**
    - Type: `n8n-nodes-postwire.postWire`
    - Name: `Post to TikTok, Reels and Shorts`
    - Credentials: PostWire API.
    - Configuration: Resource `post`; operation `publish`; platforms: `tiktok`, `instagram`, `youtube`; content mode `smart`; media source `binary` (`data`). Set node error handling to `continueRegularOutput`.
    - Expressions: Prompt: `={{ $('Read the plan').item.json.plan.caption_idea || $('Read the plan').item.json.idea }}`; Title: `={{ String($('Read the plan').item.json.plan.youtube_title || $('Read the plan').item.json.idea).slice(0, 100) }}`; Idempotency Key: `={{ ('hailuo-idea-row-' + $('Read the plan').item.json.row_number + '-' + $('Read the plan').item.json.idea).slice(0, 200) }}`.
    - Connection: Connect `Hailuo render` output to this node.

11. **Create Results Aggregator Node**
    - Type: `n8n-nodes-base.aggregate`
    - Name: `Collect the results`
    - Configuration: Aggregate mode `aggregateAllItemData`; destination field `results`.
    - Connection: Connect `Post to TikTok, Reels and Shorts` output to this node.

12. **Create Evaluation IF Node**
    - Type: `n8n-nodes-base.if`
    - Name: `Everything published?`
    - Configuration: Condition checking `={{ $json.results.length > 0 && $json.results.every(r => r.ok === true) }}`.
    - Connection: Connect `Collect the results` output to this node.

13. **Create Success Google Sheets Node**
    - Type: `n8n-nodes-base.googleSheets`
    - Name: `Mark the idea as posted`
    - Credentials: Google Sheets OAuth2.
    - Configuration: Operation `update`; match by `row_number`; set `status` to `posted`, format `posted_at`, and map `links`.
    - Connection: Connect True branch of `Everything published?` to this node.

14. **Create Failure Google Sheets Node**
    - Type: `n8n-nodes-base.googleSheets`
    - Name: `Write the fix in the row`
    - Credentials: Google Sheets OAuth2.
    - Configuration: Operation `update`; match by `row_number`; set `status` to `needs fix` / `partly posted`, format `posted_at`, and write error strings into `note`.
    - Connection: Connect False branch of `Everything published?` to this node.

15. **Create Gmail Alert Node**
    - Type: `n8n-nodes-base.gmail`
    - Name: `Email me the fix`
    - Credentials: Gmail OAuth2.
    - Configuration: Plain text email settings utilizing recipient email from `Channel settings` and error logs from `Write the fix in the row`.
    - Connection: Connect `Write the fix in the row` output to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| PostWire Integration Guide | Search for the verified **PostWire** node in the n8n nodes panel and configure an account at [postwire.io](https://postwire.io). |
| MiniMax API Billing | MiniMax API keys cover both text planning and video rendering; video generation is billed per generated clip. |
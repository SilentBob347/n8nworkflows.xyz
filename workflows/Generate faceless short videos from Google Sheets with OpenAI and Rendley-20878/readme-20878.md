Generate faceless short videos from Google Sheets with OpenAI and Rendley

https://n8nworkflows.xyz/workflows/generate-faceless-short-videos-from-google-sheets-with-openai-and-rendley-20878


# Generate faceless short videos from Google Sheets with OpenAI and Rendley

### 1. Workflow Overview

This workflow automates the end-to-end creation, review, export, and social media distribution of faceless vertical short videos (TikTok, Instagram Reels, YouTube Shorts). Operating on a daily schedule, it reads a queue from a Google Sheet, generates custom scripts and shot lists using OpenAI, orchestrates video production via Rendley AI, manages human-in-the-loop approvals, and publishes the final content across configured social channels.

The workflow logic is categorized into the following functional blocks:
- **1.1 Trigger and Configuration Setup:** Initiates the workflow daily and initializes operational settings (video length, style, language, review flags).
- **1.2 Queue Processing and Routing:** Reads the Google Sheets data queue, determines whether to process an approved video or pick the next "ready" topic, and handles status updates.
- **1.3 Script Generation and Prompt Engineering:** Utilizes OpenAI (`gpt-4o-mini`) via LangChain nodes to generate structured voiceover scripts, visual instructions, titles, captions, and hashtags, which are then compiled into a prompt for Rendley.
- **1.4 Video Generation and Polling Loop:** Instructs the Rendley AI video agent to build the video, polling status updates iteratively through a wait loop with a maximum retry ceiling.
- **1.5 Review, Export, and Publishing:** Routes finished projects either to a manual review stage (updating the Google Sheet with a web editor link) or straight to export and social distribution, logging post IDs, timestamps, and error states back to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Configuration Setup
- **Overview:** Initializes the workflow execution schedule and defines core operational parameters governing video style, generation constraints, and publishing rules.
- **Nodes Involved:** Every morning, Settings.
- **Node Details:**
  - **Every morning**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.2) - Initiates workflow execution on a time-based trigger.
    - *Configuration:* Set to trigger daily at 9:00 AM (`triggerAtHour: 9`).
    - *Input/Output:* No inputs; outputs a single execution trigger item to **Settings**.
    - *Potential Failures:* None inherent; timezone discrepancies may alter execution timing.
  - **Settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) - Establishes static operational variables for downstream consumption.
    - *Configuration:* Assigns parameters including `videoLength` ("30 to 45 seconds"), `language` ("English"), `voiceStyle` ("warm, confident"), `footage` ("stock"), `reviewBeforePosting` (true), `postDelayMinutes` (0), and `maxWaitRounds` (15).
    - *Input/Output:* Input from **Every morning**; output connected to **Read video queue**.
    - *Potential Failures:* Data type mismatch if downstream logic expects boolean/number types instead of strings (mitigated by n8n assignment typing).

#### 2.2 Queue Processing and Routing
- **Overview:** Reads the video tracking sheet, evaluates the queue to select either an approved video ready for publishing or the next raw topic marked as "ready", and flags its status as generating.
- **Nodes Involved:** Read video queue, Pick next job, New video or publish?, Mark as generating, Approved video to publish.
- **Node Details:**
  - **Read video queue**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.5) - Reads all rows from the tracking document.
    - *Configuration:* Uses Google Sheets OAuth2 API credentials; document ID and sheet name must be configured in production.
    - *Input/Output:* Input from **Settings**; output connected to **Pick next job**.
    - *Potential Failures:* API quota exhaustion, expired credentials, or missing sheet/document configurations.
  - **Pick next job**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) - JavaScript execution node to filter queue rows.
    - *Configuration:* Searches for rows where status is 'approved' and has a `project_id` (routing to publish), or status is 'ready' with a valid topic (routing to create). Returns empty array if no actions are found.
    - *Input/Output:* Input from **Read video queue**; output connected to **New video or publish?**.
    - *Potential Failures:* JavaScript runtime errors if row data contains unexpected null values.
  - **New video or publish?**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (v3.2) - Directs workflow execution based on the chosen action string.
    - *Configuration:* Evaluates `{{ $json.action }}` against matching rules: "create" outputs to **Mark as generating**; "publish" outputs to **Approved video to publish**.
    - *Input/Output:* Input from **Pick next job**; outputs split into "New video" and "Publish approved".
    - *Potential Failures:* Unhandled actions result in unrouted data termination.
  - **Mark as generating**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.5) - Updates the spreadsheet row status to indicate processing has started.
    - *Configuration:* Updates row matching `id` with `status: "generating"` and clears the `error` field.
    - *Input/Output:* Input from **New video or publish?** ("New video"); output connected to **Write script and shot list**.
    - *Potential Failures:* Row matching failure if the ID column cannot be resolved.
  - **Approved video to publish**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) - Formats existing project data for publishing when an approved video is selected.
    - *Configuration:* Maps `project_id`, `title`, and `caption` using existing sheet data values.
    - *Input/Output:* Input from **New video or publish?** ("Publish approved"); output connected to **Video to publish**.
    - *Potential Failures:* Missing fields in the source sheet row resulting in empty publishing payloads.

#### 2.3 Script Generation and Prompt Engineering
- **Overview:** Employs an LLM chain via OpenAI to transform raw topics into structured voiceover lines, visual descriptions, titles, and hashtags, then compiles them into a master prompt for the video generator.
- **Nodes Involved:** Write script and shot list, Script writer model, Script format, Build agent prompt.
- **Node Details:**
  - **Write script and shot list**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (v1.7) - LangChain execution chain combining language model inputs with output parsing.
    - *Configuration:* Text prompt constructs script constraints utilizing topic, audience, length, and language parameters extracted from upstream nodes.
    - *Input/Output:* Inputs from **Mark as generating** (text data), **Script writer model** (AI language model), and **Script format** (AI output parser); output connected to **Build agent prompt**.
    - *Potential Failures:* LLM token limits, context timeouts, or invalid generation outputs.
  - **Script writer model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.2) - Connects to OpenAI chat models.
    - *Configuration:* Uses model `gpt-4o-mini` with temperature set to `0.8` under OpenAI credentials.
    - *Input/Output:* Connected to **Write script and shot list**.
    - *Potential Failures:* OpenAI API authentication failures, rate limiting, or insufficient billing credits.
  - **Script format**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (v1.3) - Enforces structured JSON formatting on LLM outputs.
    - *Configuration:* Validates JSON containing script beats (voiceover and visual pairs), title, caption, and hashtags array.
    - *Input/Output:* Connected to **Write script and shot list**.
    - *Potential Failures:* Model output failing JSON schema validation, causing parser exceptions.
  - **Build agent prompt**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) - JavaScript node compiling structured script arrays and settings into a comprehensive video generation prompt.
    - *Configuration:* Assembles narrative beats, language settings, voice styles, and footage rules (stock, hybrid, or AI) into a final string payload alongside formatted captions and hashtags.
    - *Input/Output:* Input from **Write script and shot list**; output connected to **Make video with Rendley AI agent**.
    - *Potential Failures:* Type errors if the structured parser output is missing expected properties.

#### 2.4 Video Generation and Polling Loop
- **Overview:** Submits the prompt to Rendley to initialize video generation, then polls the job status at fixed intervals using a loop until completion or timeout.
- **Nodes Involved:** Make video with Rendley AI agent, Wait for the agent, Check agent job, Agent finished?, Keep waiting?, Cancel agent job, Failure reason, Mark as failed, Stop with the reason.
- **Node Details:**
  - **Make video with Rendley AI agent**
    - *Type and Technical Role:* `n8n-nodes-rendley.rendley` (v1) - Initiates an asynchronous Rendley agent run.
    - *Configuration:* Uses Rendley API credentials with resource `agent`, operation `run`, prompt expression `={{ $json.prompt }}`, and `waitForCompletion: false`.
    - *Input/Output:* Input from **Build agent prompt**; output connected to **Wait for the agent**.
    - *Potential Failures:* API authentication failure or invalid prompt payloads.
  - **Wait for the agent**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (v1.1) - Delays execution to allow background job processing.
    - *Configuration:* Configured for a fixed delay of 70 seconds.
    - *Input/Output:* Input from **Make video with Rendley AI agent** (or **Keep waiting?** loop); output connected to **Check agent job**.
    - *Potential Failures:* None.
  - **Check agent job**
    - *Type and Technical Role:* `n8n-nodes-rendley.rendley` (v1) - Queries the current status of a running Rendley agent job.
    - *Configuration:* Resource `agent`, operation `getJob`, utilizing `agentJobId` from the initialization node. Configured with retry logic (max 3 tries, 5s interval).
    - *Input/Output:* Input from **Wait for the agent**; output connected to **Agent finished?**.
    - *Potential Failures:* Temporary network dropouts or intermittent Rendley API downtime.
  - **Agent finished?**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (v3.2) - Evaluates job status upon polling.
    - *Configuration:* Routes to "Done" if status equals `completed`; routes to "Failed" if status matches regex `^(failed|canceled|cancelled|waiting_input)$`; falls back to "Still working".
    - *Input/Output:* Input from **Check agent job**; outputs routed to review/publishing, failure handling, or back to execution checks.
    - *Potential Failures:* Unexpected status strings escaping regex matching.
  - **Keep waiting?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2.2) - Determines whether polling limits have been exceeded.
    - *Configuration:* Condition verifies if `$runIndex` is less than `maxWaitRounds` configured in settings.
    - *Input/Output:* Input from fallback output of **Agent finished?**; outputs true to **Wait for the agent**, false to **Cancel agent job**.
    - *Potential Failures:* None.
  - **Cancel agent job**
    - *Type and Technical Role:* `n8n-nodes-rendley.rendley` (v1) - Explicitly terminates stalled jobs.
    - *Configuration:* Resource `agent`, operation `cancelJob`, targeting the active `agentJobId`.
    - *Input/Output:* Input from **Keep waiting?** (false branch); output connected to **Failure reason**.
    - *Potential Failures:* API rejection if the job has already terminated server-side.
  - **Failure reason**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) - Formats error messages for failed or timed-out generation tasks.
    - *Configuration:* Assigns an explicit error string parsing job error messages or max-wait timeout conditions.
    - *Input/Output:* Input from **Agent finished?** ("Failed") or **Cancel agent job**; output connected to **Mark as failed**.
    - *Potential Failures:* Expression evaluation errors if job response payloads lack standard error keys.
  - **Mark as failed**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.5) - Records failure status and reasons back to the tracking spreadsheet.
    - *Configuration:* Updates row matching `id` with `status: "failed"` and writes the failure message to the `error` column.
    - *Input/Output:* Input from **Failure reason**; output connected to **Stop with the reason**.
    - *Potential Failures:* Google Sheets API write errors.
  - **Stop with the reason**
    - *Type and Technical Role:* `n8n-nodes-base.stopAndError` (v1) - Halts execution and throws an explicit error message.
    - *Configuration:* Uses expression `={{ $('Failure reason').item.json.error }}` as the error message.
    - *Input/Output:* Input from **Mark as failed**; terminates workflow execution.
    - *Potential Failures:* None.

#### 2.5 Review, Export, and Publishing
- **Overview:** Manages human review workflows or direct publishing, exports high-definition video files via Rendley, and broadcasts content to configured social media platforms while updating execution logs in Google Sheets.
- **Nodes Involved:** Post right away?, Mark ready for review, New video to publish, Video to publish, Export MP4 with Rendley, Post to social media, One row per video, Mark as posted, Mark publish failed, Stop publishing.
- **Node Details:**
  - **Post right away?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2.2) - Evaluates whether content should bypass manual review.
    - *Configuration:* Condition checks if `reviewBeforePosting` is false and the job outcome is marked as applied.
    - *Input/Output:* Input from **Agent finished?** ("Done"); true output connected to **New video to publish**, false output connected to **Mark ready for review**.
    - *Potential Failures:* None.
  - **Mark ready for review**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.5) - Updates the spreadsheet row with editor links for human verification.
    - *Configuration:* Updates row matching `id` with `status: "review"`, `project_id`, `project_url` (`https://app.rendley.com/editor/{project_id}`), generated title, caption, and warning notes if applicable.
    - *Input/Output:* Input from **Post right away?** (false branch); terminal node for this branch.
    - *Potential Failures:* Google Sheets API write failures.
  - **New video to publish**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) - Prepares project parameters for immediate export and publication.
    - *Configuration:* Assigns `project_id`, `title`, and `caption`.
    - *Input/Output:* Input from **Post right away?** (true branch); output connected to **Video to publish**.
    - *Potential Failures:* None.
  - **Video to publish**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (v1) - Serves as a logical convergence point for publishing data streams.
    - *Configuration:* No parameters.
    - *Input/Output:* Inputs from **Approved video to publish** and **New video to publish**; output connected to **Export MP4 with Rendley**.
    - *Potential Failures:* None.
  - **Export MP4 with Rendley**
    - *Type and Technical Role:* `n8n-nodes-rendley.rendley` (v1) - Renders and exports the project into an MP4 file.
    - *Configuration:* Resource `export`, operation `render`, quality set to `high`, target resolution `1080p`, poll timeout 30s, poll interval 15s. Error output directed via `onError: continueErrorOutput` with retry logic enabled.
    - *Input/Output:* Input from **Video to publish**; outputs split into success and error paths.
    - *Potential Failures:* Rendering timeouts, asset generation errors, or credit depletion on the Rendley account.
  - **Post to social media**
    - *Type and Technical Role:* `n8n-nodes-rendley.rendley` (v1) - Publishes or schedules media across connected social platforms.
    - *Configuration:* Resource `social`, operation `createPost`, passing media URL, caption, account IDs, and scheduling rules based on `postDelayMinutes`. Configured with `onError: continueErrorOutput` and retry logic.
    - *Input/Output:* Input from **Export MP4 with Rendley**; outputs split into success and error paths.
    - *Potential Failures:* Account token expiration, platform API rejections, or invalid media URLs.
  - **One row per video**
    - *Type and Technical Role:* `n8n-nodes-base.limit` (v1) - Limits item throughput to a single entry.
    - *Configuration:* `maxItems: 1`.
    - *Input/Output:* Input from **Post to social media** (success); output connected to **Mark as posted**.
    - *Potential Failures:* None.
  - **Mark as posted**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.5) - Finalizes tracking records upon successful publication.
    - *Configuration:* Updates row matching `id` with `status` ("posted" or "scheduled"), `post_ids`, `posted_at`, `project_url`, and logs any platform warning errors.
    - *Input/Output:* Input from **One row per video**; terminal node for successful runs.
    - *Potential Failures:* Google Sheets API write errors.
  - **Mark publish failed**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.5) - Records publishing failures to the spreadsheet.
    - *Configuration:* Updates row matching `id` with `status: "failed"` and populates the `error` column with exception details.
    - *Input/Output:* Inputs from error branches of **Export MP4 with Rendley** and **Post to social media**; output connected to **Stop publishing**.
    - *Potential Failures:* Google Sheets API write errors.
  - **Stop publishing**
    - *Type and Technical Role:* `n8n-nodes-base.stopAndError` (v1) - Halts execution when export or publishing fails.
    - *Configuration:* Uses error message expression derived from the upstream failure payload.
    - *Input/Output:* Input from **Mark publish failed**; terminates workflow execution.
    - *Potential Failures:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every morning | n8n-nodes-base.scheduleTrigger | Triggers workflow daily at 9 AM | None | Settings | Faceless short videos from a Google Sheet, posted to your socials... |
| Settings | n8n-nodes-base.set | Defines operational configuration variables | Every morning | Read video queue | Faceless short videos from a Google Sheet, posted to your socials... |
| Read video queue | n8n-nodes-base.googleSheets | Reads Google Sheet rows | Settings | Pick next job | Faceless short videos from a Google Sheet, posted to your socials... |
| Pick next job | n8n-nodes-base.code | Selects approved or ready jobs from queue | Read video queue | New video or publish? | 1. Pick the next job: Approved videos are published first, otherwise the first row with status ready becomes a new video. With nothing to do the run just ends. |
| New video or publish? | n8n-nodes-base.switch | Routes execution based on job action | Pick next job | Mark as generating, Approved video to publish | 1. Pick the next job: Approved videos are published first, otherwise the first row with status ready becomes a new video. With nothing to do the run just ends. |
| Mark as generating | n8n-nodes-base.googleSheets | Updates row status to generating | New video or publish? | Write script and shot list | 1. Pick the next job: Approved videos are published first, otherwise the first row with status ready becomes a new video. With nothing to do the run just ends. |
| Write script and shot list | @n8n/n8n-nodes-langchain.chainLlm | Generates script via LLM chain | Mark as generating, Script writer model, Script format | Build agent prompt | 2. Write the script: OpenAI turns the topic into a hook, 5 to 8 voiceover lines with a visual note each, a title, a caption and hashtags. Build agent prompt turns that into one instruction for Rendley. |
| Script writer model | @n8n/n8n-nodes-langchain.lmChatOpenAi | OpenAI language model provider | None | Write script and shot list | 2. Write the script: OpenAI turns the topic into a hook, 5 to 8 voiceover lines with a visual note each, a title, a caption and hashtags. Build agent prompt turns that into one instruction for Rendley. |
| Script format | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON output structure | None | Write script and shot list | 2. Write the script: OpenAI turns the topic into a hook, 5 to 8 voiceover lines with a visual note each, a title, a caption and hashtags. Build agent prompt turns that into one instruction for Rendley. |
| Build agent prompt | n8n-nodes-base.code | Compiles prompt and metadata for video agent | Write script and shot list | Make video with Rendley AI agent | 2. Write the script: OpenAI turns the topic into a hook, 5 to 8 voiceover lines with a visual note each, a title, a caption and hashtags. Build agent prompt turns that into one instruction for Rendley. |
| Make video with Rendley AI agent | n8n-nodes-rendley.rendley | Initiates asynchronous video generation | Build agent prompt | Wait for the agent | 3. Make the video: One Rendley agent run picks footage, records the AI voiceover, adds animated captions, music and your brand kit. The loop checks the job every 70 s; waiting does not count against your n8n executions. |
| Wait for the agent | n8n-nodes-base.wait | Pauses execution for polling interval | Make video with Rendley AI agent, Keep waiting? | Check agent job | 3. Make the video: One Rendley agent run picks footage, records the AI voiceover, adds animated captions, music and your brand kit. The loop checks the job every 70 s; waiting does not count against your n8n executions. |
| Check agent job | n8n-nodes-rendley.rendley | Queries status of background generation job | Wait for the agent | Agent finished? | 3. Make the video: One Rendley agent run picks footage, records the AI voiceover, adds animated captions, music and your brand kit. The loop checks the job every 70 s; waiting does not count against your n8n executions. |
| Agent finished? | n8n-nodes-base.switch | Checks if generation completed or failed | Check agent job | Post right away?, Failure reason, Keep waiting? | 3. Make the video: One Rendley agent run picks footage, records the AI voiceover, adds animated captions, music and your brand kit. The loop checks the job every 70 s; waiting does not count against your n8n executions. |
| Post right away? | n8n-nodes-base.if | Evaluates review constraints | Agent finished? | New video to publish, Mark ready for review | 4. Review in the editor: With reviewBeforePosting on, or when the agent finished with warnings, the row gets status review and the link to the project in the Rendley editor. Edit anything there, then set the row to approved. |
| Mark ready for review | n8n-nodes-base.googleSheets | Updates sheet with review status and editor link | Post right away? | None | 4. Review in the editor: With reviewBeforePosting on, or when the agent finished with warnings, the row gets status review and the link to the project in the Rendley editor. Edit anything there, then set the row to approved. |
| New video to publish | n8n-nodes-base.set | Prepares variables for direct publication | Post right away? | Video to publish | 5. Export and post: Rendley exports a 1080p MP4 from the project, with your edits, and posts it to the accounts picked in Post to social media: now, or after postDelayMinutes as a scheduled post. Posts appear in the Rendley content calendar, where you can reschedule or cancel them. If the export or the post fails, the row gets failed with the reason, so the next run moves on. |
| Keep waiting? | n8n-nodes-base.if | Checks whether polling rounds remain | Agent finished? | Wait for the agent, Cancel agent job | If something goes wrong: - Failed job: the row gets failed + reason - Too slow: canceled after maxWaitRounds - The run stops, so your error workflow alerts you |
| Cancel agent job | n8n-nodes-rendley.rendley | Cancels timed-out jobs | Keep waiting? | Failure reason | If something goes wrong: - Failed job: the row gets failed + reason - Too slow: canceled after maxWaitRounds - The run stops, so your error workflow alerts you |
| Failure reason | n8n-nodes-base.set | Formats timeout or error messages | Agent finished?, Cancel agent job | Mark as failed | If something goes wrong: - Failed job: the row gets failed + reason - Too slow: canceled after maxWaitRounds - The run stops, so your error workflow alerts you |
| Mark as failed | n8n-nodes-base.googleSheets | Records failure status in sheet | Failure reason | Stop with the reason | If something goes wrong: - Failed job: the row gets failed + reason - Too slow: canceled after maxWaitRounds - The run stops, so your error workflow alerts you |
| Stop with the reason | n8n-nodes-base.stopAndError | Halts workflow execution with error message | Mark as failed | None | If something goes wrong: - Failed job: the row gets failed + reason - Too slow: canceled after maxWaitRounds - The run stops, so your error workflow alerts you |
| Approved video to publish | n8n-nodes-base.set | Prepares project parameters for approved content | New video or publish? | Video to publish | Faceless short videos from a Google Sheet, posted to your socials... |
| Video to publish | n8n-nodes-base.noOp | Convergence point for publish items | New video to publish, Approved video to publish | Export MP4 with Rendley | 5. Export and post: Rendley exports a 1080p MP4 from the project, with your edits, and posts it to the accounts picked in Post to social media: now, or after postDelayMinutes as a scheduled post. Posts appear in the Rendley content calendar, where you can reschedule or cancel them. If the export or the post fails, the row gets failed with the reason, so the next run moves on. |
| Export MP4 with Rendley | n8n-nodes-rendley.rendley | Renders and exports MP4 video file | Video to publish | Post to social media, Mark publish failed | 5. Export and post: Rendley exports a 1080p MP4 from the project, with your edits, and posts it to the accounts picked in Post to social media: now, or after postDelayMinutes as a scheduled post. Posts appear in the Rendley content calendar, where you can reschedule or cancel them. If the export or the post fails, the row gets failed with the reason, so the next run moves on. |
| Post to social media | n8n-nodes-rendley.rendley | Publishes or schedules posts to social channels | Export MP4 with Rendley | One row per video, Mark publish failed | 5. Export and post: Rendley exports a 1080p MP4 from the project, with your edits, and posts it to the accounts picked in Post to social media: now, or after postDelayMinutes as a scheduled post. Posts appear in the Rendley content calendar, where you can reschedule or cancel them. If the export or the post fails, the row gets failed with the reason, so the next run moves on. |
| One row per video | n8n-nodes-base.limit | Limits output items to one | Post to social media | Mark as posted | 5. Export and post: Rendley exports a 1080p MP4 from the project, with your edits, and posts it to the accounts picked in Post to social media: now, or after postDelayMinutes as a scheduled post. Posts appear in the Rendley content calendar, where you can reschedule or cancel them. If the export or the post fails, the row gets failed with the reason, so the next run moves on. |
| Mark as posted | n8n-nodes-base.googleSheets | Updates sheet status to posted or scheduled | One row per video | None | 5. Export and post: Rendley exports a 1080p MP4 from the project, with your edits, and posts it to the accounts picked in Post to social media: now, or after postDelayMinutes as a scheduled post. Posts appear in the Rendley content calendar, where you can reschedule or cancel them. If the export or the post fails, the row gets failed with the reason, so the next run moves on. |
| Mark publish failed | n8n-nodes-base.googleSheets | Records publishing failures in sheet | Export MP4 with Rendley, Post to social media | Stop publishing | 5. Export and post: Rendley exports a 1080p MP4 from the project, with your edits, and posts it to the accounts picked in Post to social media: now, or after postDelayMinutes as a scheduled post. Posts appear in the Rendley content calendar, where you can reschedule or cancel them. If the export or the post fails, the row gets failed with the reason, so the next run moves on. |
| Stop publishing | n8n-nodes-base.stopAndError | Halts execution on publish failure | Mark publish failed | None | 5. Export and post: Rendley exports a 1080p MP4 from the project, with your edits, and posts it to the accounts picked in Post to social media: now, or after postDelayMinutes as a scheduled post. Posts appear in the Rendley content calendar, where you can reschedule or cancel them. If the export or the post fails, the row gets failed with the reason, so the next run moves on. |
| About this workflow | n8n-nodes-base.stickyNote | Overview documentation sticky note | None | None | Faceless short videos from a Google Sheet, posted to your socials... |
| Sheet layout | n8n-nodes-base.stickyNote | Spreadsheet schema documentation note | None | None | Sheet layout: One tab with this header row... |
| Step 1 | n8n-nodes-base.stickyNote | Documentation note for job selection | None | None | 1. Pick the next job... |
| Step 2 | n8n-nodes-base.stickyNote | Documentation note for script generation | None | None | 2. Write the script... |
| Step 3 | n8n-nodes-base.stickyNote | Documentation note for video creation | None | None | 3. Make the video... |
| Step 4 | n8n-nodes-base.stickyNote | Documentation note for editorial review | None | None | 4. Review in the editor... |
| Errors | n8n-nodes-base.stickyNote | Documentation note for error handling | None | None | If something goes wrong... |
| Step 5 | n8n-nodes-base.stickyNote | Documentation note for export and posting | None | None | 5. Export and post... |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Trigger and Settings
1. Create a **Schedule Trigger** node (`Every morning`) and configure it to run daily at `9:00 AM`.
2. Create a **Set** node (`Settings`) connected to the trigger. Configure assignments with string, boolean, and number types:
   - `videoLength` (String): `30 to 45 seconds`
   - `language` (String): `English`
   - `voiceStyle` (String): `warm, confident`
   - `footage` (String): `stock`
   - `brandNotes` (String): `""`
   - `reviewBeforePosting` (Boolean): `true`
   - `postDelayMinutes` (Number): `0`
   - `maxWaitRounds` (Number): `15`

#### Step 2: Set Up Queue Input and Routing
3. Create a **Google Sheets** node (`Read video queue`). Connect it to **Settings**. Set operation to `Get Many` (or equivalent read), configure your Google Sheets OAuth2 credential, and select your target spreadsheet and document ID.
4. Create a **Code** node (`Pick next job`) connected to **Read video queue**. Insert the JavaScript queue-parsing logic to look for rows marked as `approved` with a `project_id`, or rows marked as `ready` with a topic.
5. Create a **Switch** node (`New video or publish?`) connected to **Pick next job**. Define two routing rules:
   - Rule 1: `{{ $json.action }}` equals `create` (Output: "New video")
   - Rule 2: `{{ $json.action }}` equals `publish` (Output: "Publish approved")
6. For the "New video" branch, add a **Google Sheets** node (`Mark as generating`). Configure it to update a row matching `id` with `status: "generating"` and clear the error field.
7. For the "Publish approved" branch, add a **Set** node (`Approved video to publish`) mapping `project_id`, `title`, and `caption` from the job payload. Connect this to a **NoOp** node named `Video to publish`.

#### Step 3: Configure OpenAI Script Generation
8. From **Mark as generating**, connect to an **Advanced AI** LangChain Chain node: **AI Chain** (`Write script and shot list`).
9. Attach an **OpenAI Chat Model** node (`Script writer model`) to the language model input of the chain. Configure it to use model `gpt-4o-mini` with temperature `0.8` using your OpenAI API credentials.
10. Attach a **Structured Output Parser** node (`Script format`) to the parser input of the chain. Provide the JSON schema example for the script array (voiceover and visual pairs), title, caption, and hashtags.
11. Configure the text prompt in **Write script and shot list** using the specified template incorporating topic, audience, length, and language variables.
12. Connect the output of **Write script and shot list** to a **Code** node (`Build agent prompt`) to assemble the master prompt, title, and formatted hashtags.

#### Step 4: Configure Rendley Video Generation and Polling Loop
13. Connect **Build agent prompt** to a **Rendley** node (`Make video with Rendley AI agent`). Set resource to `agent`, operation to `run`, prompt to `={{ $json.prompt }}`, and disable `waitForCompletion`. Configure Rendley API credentials.
14. Connect the agent run node to a **Wait** node (`Wait for the agent`) set to `70` seconds.
15. Connect the wait node to a **Rendley** node (`Check agent job`). Set resource to `agent`, operation to `getJob`, and `agentJobId` to `={{ $('Make video with Rendley AI agent').item.json.job_id }}`. Enable retry on fail (3 tries, 5000ms wait).
16. Connect **Check agent job** to a **Switch** node (`Agent finished?`). Set rules:
   - Rule 1: `{{ $json.status }}` equals `completed` (Output: "Done")
   - Rule 2: `{{ $json.status }}` matches regex `^(failed|canceled|cancelled|waiting_input)$` (Output: "Failed")
   - Fallback output renamed to "Still working".
17. From the "Still working" output, connect to an **If** node (`Keep waiting?`) evaluating if `{{ $runIndex < $('Settings').item.json.maxWaitRounds }}`.
   - True branch connects back to **Wait for the agent**.
   - False branch connects to a **Rendley** node (`Cancel agent job`) set to resource `agent`, operation `cancelJob`.
18. Connect **Cancel agent job** and the "Failed" output of **Agent finished?** to a **Set** node (`Failure reason`) to extract error messages.
19. Connect **Failure reason** to a **Google Sheets** node (`Mark as failed`) updating the row status to `failed` and recording the error. Connect this to a **Stop and Error** node (`Stop with the reason`).

#### Step 5: Configure Review, Export, and Publishing
20. From the "Done" output of **Agent finished?**, connect to an **If** node (`Post right away?`) evaluating `{{ !$('Settings').item.json.reviewBeforePosting && $json.outcome === 'applied' }}`.
   - False branch connects to a **Google Sheets** node (`Mark ready for review`) updating status to `review`, setting project ID, project URL (`https://app.rendley.com/editor/{project_id}`), title, and caption.
   - True branch connects to a **Set** node (`New video to publish`) assigning project ID, title, and caption, then routing into the **Video to publish** NoOp node.
21. From **Video to publish**, connect to a **Rendley** node (`Export MP4 with Rendley`). Set resource to `export`, operation to `render`, quality to `high`, target resolution to `1080p`, and enable `waitForCompletion`. Configure `onError: continueErrorOutput` with retries.
22. Connect the success output of export to a **Rendley** node (`Post to social media`). Set resource to `social`, operation to `createPost`, passing caption, media URL, account IDs, and optional scheduling. Enable `onError: continueErrorOutput` with retries.
23. Connect the success output of **Post to social media** through a **Limit** node (`One row per video`, max 1 item) to a **Google Sheets** node (`Mark as posted`). Configure it to update the sheet status to `posted` or `scheduled`, record post IDs, timestamp, project URLs, and warning notes.
24. Route error outputs from **Export MP4 with Rendley** and **Post to social media** to a **Google Sheets** node (`Mark publish failed`) updating status to `failed` and writing the error reason, followed by a **Stop and Error** node (`Stop publishing`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Faceless short videos from a Google Sheet, posted to your socials | Creators, agencies, and marketing teams who want automated vertical video workflows without manual editing. |
| Sheet layout requirements | Requires a Google Sheet containing columns: `id`, `topic`, `audience`, `status`, `title`, `caption`, `project_id`, `project_url`, `post_ids`, `posted_at`, and `error`. |
| Rendley platform and node requirements | Requires a Rendley paid plan with credits, social publishing enabled, and the Rendley community node (`n8n-nodes-rendley`) installed. |
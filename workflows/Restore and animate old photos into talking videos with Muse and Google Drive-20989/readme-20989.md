Restore and animate old photos into talking videos with Muse and Google Drive

https://n8nworkflows.xyz/workflows/restore-and-animate-old-photos-into-talking-videos-with-muse-and-google-drive-20989


# Restore and animate old photos into talking videos with Muse and Google Drive

### 1. Workflow Overview

This workflow provides an interactive web interface and an automated batch processor to restore historical photographs and convert them into short talking-video clips using the Meta Muse API (via the third-party `useapi.net` integration). It supports direct image file uploads or public HTTPS image URLs, handles asynchronous generation jobs with polling and error resilience, manages weekly quota limits, and optionally syncs results to Google Drive or processes rows in Google Sheets.

The workflow logic is categorized into the following functional blocks:
- **1.1 Form Input & Validation:** Captures user input via an authenticated n8n form trigger, sanitizes parameters, validates text constraints (e.g., word count), and handles raw file uploads or URL verification.
- **1.2 Account Quota Check & Asset Ingestion:** Uploads the source image to the Muse API asset repository, checks account health and weekly allowances, and displays a confirmation page with estimated quota consumption.
- **1.3 Photo Restoration:** Submits an asynchronous image-processing job to restore, repair, or colorize the photograph, polls for completion, and allows the user to review the restored version, apply modifications via a redo loop, or fallback to the original.
- **1.4 Video Animation & Speech Synthesis:** Submits asynchronous video generation jobs based on the chosen image, spoken script, and motion constraints. It polls for takes, displays players for verification, and handles user selections or regeneration requests.
- **1.5 Post-Processing & Storage:** Downloads selected media assets, optionally uploads final outputs (MP4 video and WebP restored photo) to Google Drive, and renders a summary page with signed, secure download links and updated quota usage metrics.
- **1.6 Secure Media Proxy Webhook:** Intercepts media requests via a dedicated webhook, validates HMAC-SHA256 time-expiring tickets stored in request parameters, and streams protected files using backend API credentials without exposing tokens to the client browser.
- **1.7 Google Sheets Batch Processing:** Runs on a scheduled trigger to process batches of old photos from a spreadsheet, managing rows sequentially (`new` $\rightarrow$ `restoring` $\rightarrow$ `rendering` $\rightarrow$ `done`) under rate limits.

---

### 2. Block-by-Block Analysis

#### 2.1 Form Input & Validation
- **Overview:** Receives user submissions from an authenticated n8n form, verifies field values, and normalizes inputs for downstream processing.
- **Nodes Involved:** `Bring a photo to life`, `Read the form`, `Start route`, `Stopped`
- **Node Details:**
  - **Bring a photo to life**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (Form Trigger). Acts as the primary entry point, rendering an HTML/CSS-styled form.
    - *Configuration Choices:* Configured with Basic Authentication. Exposes form fields for file upload, image URL, restoration options, speech text, motion hints, voice parameters, aspect ratio format, number of takes, Google Drive export preference, and ownership confirmation.
    - *Key Expressions/Variables:* Custom CSS and localized styling injected via parameter configuration.
    - *Input/Output Connections:* Output connects to `Read the form`.
    - *Edge Cases / Potential Failures:* Unauthorized access (handled by Basic Auth credential mismatch). Empty required fields trigger form-level validation stops.
  - **Read the form**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Node). Validates constraints (e.g., maximum 30 words for speech lines) and parses binary data or URL parameters.
    - *Configuration Choices:* Implements embedded utility functions for cryptographic ticket signing (HMAC-SHA256), error parsing (`apiErr`), and prompt formatting.
    - *Key Expressions/Variables:* Accesses `$json`, `$input.first().binary`, and `$execution.resumeFormUrl`.
    - *Input/Output Connections:* Input from `Bring a photo to life`; output to `Start route`.
    - *Edge Cases / Potential Failures:* Throws inline HTML stop screens if validation rules fail (e.g., missing lines, invalid URL formats).
  - **Start route**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Switch Node). Directs workflow execution based on the input route type.
    - *Configuration Choices:* Evaluates `$json.route` into three distinct routing paths (`stop`, `file`, `url`).
    - *Input/Output Connections:* Input from `Read the form`; outputs connect to `Stopped`, `Upload photo`, and `Upload from URL`.
  - **Stopped**
    - *Type and Technical Role:* `n8n-nodes-base.form` (Form Node). Renders error messages and recovery instructions when validation or execution halts.
    - *Configuration Choices:* Operates as an interactive form page displaying localized failure reasons.
    - *Input/Output Connections:* Receives failure states from upstream decision routers.

---

#### 2.2 Account Quota Check & Asset Ingestion
- **Overview:** Uploads the source media to the Muse API asset storage and verifies the target account's health and available weekly allowance.
- **Nodes Involved:** `Upload photo`, `Upload from URL`, `Read account`, `Upload check`, `Upload route`, `Before you spend`, `Start choice`, `Start next`
- **Node Details:**
  - **Upload photo** / **Upload from URL**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request Node). Uploads binary image data or registers an external image URL with the Muse assets API.
    - *Configuration Choices:* Uses HTTP Header Authentication (`Authorization: Bearer <token>`). Timeout set to 180,000ms with automatic retries (`maxTries: 3`).
    - *Key Expressions/Variables:* Dynamic URL construction using query parameters for optional email target routing.
    - *Input/Output Connections:* Inputs from `Start route`; outputs connect to `Read account`.
    - *Edge Cases / Potential Failures:* File size exceeding 50 MB (HTTP 413) or invalid URLs. Managed via `onError: continueRegularOutput`.
  - **Read account**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request Node). Queries the live quota and account health status.
    - *Configuration Choices:* HTTP GET request to `https://api.useapi.net/v1/muse/accounts/...` using Header Auth.
    - *Input/Output Connections:* Input from upload nodes; output to `Upload check`.
  - **Upload check**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Node). Validates upload success, parses account quota statistics, and prepares the "Before you spend" confirmation page HTML structure.
    - *Configuration Choices:* Calculates percentage remaining and estimates total allowance consumption.
    - *Input/Output Connections:* Input from `Read account`; output to `Upload route`.
  - **Upload route**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Switch Node). Routes between workflow interruption (`stop`) and user confirmation (`show`).
    - *Input/Output Connections:* Input from `Upload check`; outputs connect to `Stopped` and `Before you spend`.
  - **Before you spend**
    - *Type and Technical Role:* `n8n-nodes-base.form` (Form Node). Presents an interactive confirmation page displaying quota usage, asset preview, and estimated weekly consumption.
    - *Configuration Choices:* Dynamically renders HTML fields and CSS stylesheets passed from `Upload check`.
    - *Input/Output Connections:* Input from `Upload route`; output to `Start choice`.
  - **Start choice** & **Start next**
    - *Type and Technical Role:* `n8n-nodes-base.code` and `n8n-nodes-base.switch`. Interprets user selection to either initiate photo restoration or skip directly to video animation.
    - *Input/Output Connections:* Routes to `Restore request` or `Take requests`.

---

#### 2.3 Photo Restoration
- **Overview:** Submits an asynchronous image restoration job, polls the API until completion, and presents the restored result for user verification or modification.
- **Nodes Involved:** `Restore request`, `Submit restore`, `Submit check`, `Retry submit in 30s`, `Wait 10s`, `Check job`, `Job status`, `Job next`, `Restored`, `Restore choice`, `Redo?`
- **Node Details:**
  - **Restore request** & **Submit restore**
    - *Type and Technical Role:* `n8n-nodes-base.code` and `n8n-nodes-base.httpRequest`. Constructs the image restoration prompt payload and dispatches a POST request to `https://api.useapi.net/v1/muse/jobs/image`.
    - *Configuration Choices:* Uses Header Auth, asynchronous flag (`async: true`), and unique reply references.
    - *Input/Output Connections:* Linked via standard main connections with error handling configured to continue regular output.
  - **Submit check** & **Job status**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Node). Evaluates job submission responses or poll outcomes. Implements backoff counters and resend limits for transient status codes (HTTP 429, 503).
  - **Wait 10s** & **Retry submit in 30s**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (Wait Node). Pauses execution for polling intervals (10 seconds) or rate-limit retry intervals (30 seconds).
  - **Check job**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request Node). Polls job status via GET `https://api.useapi.net/v1/muse/jobs/{jobid}`.
  - **Restored**
    - *Type and Technical Role:* `n8n-nodes-base.form` (Form Node). Displays side-by-side comparisons of original and restored photos, offering options to accept, redo with notes, or revert to the original.
  - **Restore choice** & **Redo?**
    - *Type and Technical Role:* Code and Switch nodes that evaluate user feedback. If a redo is requested, updates restoration notes and increments retry counters before looping back to `Restore request`. Otherwise, proceeds to `Take requests`.

---

#### 2.4 Video Animation & Speech Synthesis
- **Overview:** Animates the selected static image into a talking video asset using specified voice, motion, and dialogue scripts across single or dual takes.
- **Nodes Involved:** `Take requests`, `Submit take`, `Takes submitted`, `Takes next`, `Wait 15s`, `Check takes`, `Takes status`, `Retry takes in 30s`, `Your takes`, `Take choice`, `Choice route`
- **Node Details:**
  - **Take requests** & **Submit take**
    - *Type and Technical Role:* Code and HTTP Request nodes. Generates prompt payloads referencing `image_1` (using either the restored asset ID or original asset ID) and submits POST requests to `https://api.useapi.net/v1/muse/jobs/video`.
    - *Configuration Choices:* Configured with request batching (`batchSize: 1`, `batchInterval: 2000`) to process multiple takes cleanly.
  - **Wait 15s** & **Check takes** & **Takes status**
    - *Type and Technical Role:* Wait and HTTP Request/Code nodes. Polls active video rendering jobs every 15 seconds, tracking individual take statuses and handling rate limits.
  - **Your takes**
    - *Type and Technical Role:* `n8n-nodes-base.form` (Form Node). Renders embedded video players for each generated take, enabling the user to select a preferred take, request a line/motion redo, or finish without a video.
  - **Choice route**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Switch Node). Directs execution to file saving/results processing (`result`), video regeneration (`redo`), or take adjustments.

---

#### 2.5 Post-Processing & Storage
- **Overview:** Downloads finalized media assets, uploads them to Google Drive (if enabled), measures final quota delta, and renders the success summary page.
- **Nodes Involved:** `Save to Drive?`, `Files to save`, `Download file`, `Upload to Drive`, `Read account (after)`, `Result page`, `Your video`, `Done`
- **Node Details:**
  - **Save to Drive?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (If Node). Evaluates whether Google Drive saving is enabled and valid assets exist.
  - **Download file** & **Upload to Drive**
    - *Type and Technical Role:* HTTP Request and Google Drive nodes. Downloads binary MP4/WebP assets from the Muse API and uploads them to the configured Google Drive folder ID using OAuth2 credentials.
  - **Read account (after)**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request Node). Queries the final account quota to calculate exact consumption metrics for the session.
  - **Result page**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Node). Generates HTML containing embedded HTML5 video players, secure download links via signed tickets, reuse codes, and quota usage summaries.
  - **Your video** & **Done**
    - *Type and Technical Role:* Form and Form Completion nodes. Displays the final results page and termination screen.

---

#### 2.6 Secure Media Proxy Webhook
- **Overview:** Intercepts incoming HTTP requests for media playback and downloads, validating cryptographic HMAC tickets before proxying requests to the private Muse API.
- **Nodes Involved:** `Media file`, `Media request`, `Ticket ok?`, `Fetch asset`, `Send media`, `Not found`
- **Node Details:**
  - **Media file**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger). Public webhook endpoint (`muse-old-photo-media`) configured with response mode set to "Response Node".
  - **Media request**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Node). Extracts query parameter `t`, verifies HMAC-SHA256 signatures against workflow static data secrets, checks expiration timestamps, and determines mime types.
  - **Ticket ok?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (If Node). Evaluates ticket validity (`$json.ok === true`).
  - **Fetch asset** & **Send media**
    - *Type and Technical Role:* HTTP Request and Respond to Workflow nodes. Fetches the binary asset from useapi.net using authorization headers and streams it back to the client with appropriate `Content-Type`, `Cache-Control`, and `Content-Disposition` headers.
  - **Not found**
    - *Type and Technical Role:* `n8n-nodes-base.respondToWebhook` (Webhook Node). Returns an HTTP 404 response for invalid, forged, or expired tickets.

---

#### 2.7 Google Sheets Batch Processing
- **Overview:** Periodically scans a Google Sheet for pending rows, manages job execution states (`new`, `restoring`, `rendering`, `done`), and updates sheet statuses automatically.
- **Nodes Involved:** `Every 10 minutes`, `Settings`, `Batch read account`, `Read rows`, `Plan the tick`, `Row step`, `Batch upload`, `Batch submit`, `Batch read job`, `Row result`, `Drive?`, `Batch download`, `Batch save`, `Row values`, `Update row`
- **Node Details:**
  - **Every 10 minutes**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Schedule Trigger). Disabled by default; triggers periodic batch processing sweeps.
  - **Settings**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code Node). Defines batch configuration parameters including Google Sheet ID, sheet name, rate limits (`maxPerTick`), auto-approval toggles, and link expiry hours.
  - **Read rows** & **Update row**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Google Sheets Node). Reads unformatted row data and updates processing statuses and asset links using OAuth2 credentials.
  - **Plan the tick** & **Row result**
    - *Type and Technical Role:* JavaScript Code nodes. Implements scheduling logic, rate limiting (`maxPerTick`), and status state-machine transitions for batch items.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Bring a photo to life | n8n-nodes-base.formTrigger | Renders the primary user input form with Basic Auth. | None | Read the form | Old Photo to Talking Video with Meta Muse |
| Read the form | n8n-nodes-base.code | Validates form input, checks word counts, and prepares state. | Bring a photo to life | Start route | Old Photo to Talking Video with Meta Muse |
| Start route | n8n-nodes-base.switch | Routes form output based on submission type (file, url, stop). | Read the form | Stopped, Upload photo, Upload from URL | Old Photo to Talking Video with Meta Muse |
| Upload photo | n8n-nodes-base.httpRequest | Uploads binary image data to the Muse asset API. | Start route | Read account | 1. Form + upload |
| Upload from URL | n8n-nodes-base.httpRequest | Registers an external image URL with the Muse asset API. | Start route | Read account | 1. Form + upload |
| Read account | n8n-nodes-base.httpRequest | Fetches live account quota and health status. | Upload photo, Upload from URL | Upload check | 1. Form + upload |
| Upload check | n8n-nodes-base.code | Validates upload response and formats quota estimation page. | Read account | Upload route | 1. Form + upload |
| Upload route | n8n-nodes-base.switch | Routes based on upload validation status. | Upload check | Stopped, Before you spend | 1. Form + upload |
| Before you spend | n8n-nodes-base.form | Displays asset preview and weekly quota estimate. | Upload route | Start choice | 1. Form + upload |
| Start choice | n8n-nodes-base.code | Determines whether to restore the photo or animate directly. | Before you spend | Start next | 1. Form + upload |
| Start next | n8n-nodes-base.switch | Directs flow to restoration or video generation paths. | Start choice | Restore request, Take requests | 1. Form + upload |
| Restore request | n8n-nodes-base.code | Constructs the image restoration job payload. | Start next, Redo?, Retry submit in 30s | Submit restore | 2. Restore + pick or redo |
| Submit restore | n8n-nodes-base.httpRequest | Dispatches asynchronous image restoration job to API. | Restore request | Submit check | 2. Restore + pick or redo |
| Submit check | n8n-nodes-base.code | Evaluates submission success or triggers retries. | Submit restore | Submit route | 2. Restore + pick or redo |
| Submit route | n8n-nodes-base.switch | Routes restoration job submission outcome. | Submit check | Retry submit in 30s, Stopped, Wait 10s | 2. Restore + pick or redo |
| Retry submit in 30s | n8n-nodes-base.wait | Waits before retrying failed submission. | Submit route | Restore request | 2. Restore + pick or redo |
| Wait 10s | n8n-nodes-base.wait | Polling delay between job status checks. | Submit route | Check job | 2. Restore + pick or redo |
| Check job | n8n-nodes-base.httpRequest | Polls restoration job status endpoint. | Wait 10s | Job status | 2. Restore + pick or redo |
| Job status | n8n-nodes-base.code | Evaluates poll status (completed, failed, running). | Check job | Job next | 2. Restore + pick or redo |
| Job next | n8n-nodes-base.switch | Routes based on restoration job status. | Job status | Wait 10s, Retry submit in 30s, Stopped, Restored | 2. Restore + pick or redo |
| Restored | n8n-nodes-base.form | Renders comparison view of original vs. restored photo. | Job next | Restore choice | 2. Restore + pick or redo |
| Restore choice | n8n-nodes-base.code | Parses user choice (use, redo, original). | Restored | Redo? | 2. Restore + pick or redo |
| Redo? | n8n-nodes-base.switch | Routes to redo restoration or proceed to takes. | Restore choice | Restore request, Take requests | 2. Restore + pick or redo |
| Take requests | n8n-nodes-base.code | Constructs video generation payloads for each take. | Start next, Redo?, Retry takes in 30s, Choice route | Submit take | 3. Talking takes |
| Submit take | n8n-nodes-base.httpRequest | Submits video animation jobs to the Muse API. | Take requests | Takes submitted | 3. Talking takes |
| Takes submitted | n8n-nodes-base.code | Evaluates take submission responses. | Submit take | Takes next | 3. Talking takes |
| Takes next | n8n-nodes-base.switch | Routes take submission or polling outcomes. | Takes submitted, Takes status | Wait 15s, Retry takes in 30s, Stopped, Your takes | 3. Talking takes |
| Wait 15s | n8n-nodes-base.wait | Polling delay for active video rendering jobs. | Takes next | Check takes | 3. Talking takes |
| Check takes | n8n-nodes-base.httpRequest | Polls video job status endpoint. | Wait 15s | Takes status | 3. Talking takes |
| Takes status | n8n-nodes-base.code | Parses video job poll status. | Check takes | Takes next | 3. Talking takes |
| Retry takes in 30s | n8n-nodes-base.wait | Wait node for retrying rate-limited video submissions. | Takes next | Take requests | 3. Talking takes |
| Your takes | n8n-nodes-base.form | Displays video players for generated takes. | Takes next, Choice route | Take choice | 3. Talking takes |
| Take choice | n8n-nodes-base.code | Parses user selection from generated takes. | Your takes | Choice route | 3. Talking takes |
| Choice route | n8n-nodes-base.switch | Routes user choice (redo, result, view takes). | Take choice | Take requests, Save to Drive?, Your takes | 3. Talking takes |
| Save to Drive? | n8n-nodes-base.if | Checks if Google Drive export is enabled and valid. | Choice route | Batch download, Read account (after) | 4. Result + Google Drive |
| Files to save | n8n-nodes-base.code | Prepares file list for Drive export. | Save to Drive? | Download file | 4. Result + Google Drive |
| Download file | n8n-nodes-base.httpRequest | Downloads file from Muse API for Drive upload. | Files to save | Upload to Drive | 4. Result + Google Drive |
| Upload to Drive | n8n-nodes-base.googleDrive | Uploads media file to Google Drive. | Download file | Read account (after) | 4. Result + Google Drive |
| Read account (after) | n8n-nodes-base.httpRequest | Queries final quota usage after processing. | Save to Drive?, Upload to Drive | Result page | 4. Result + Google Drive |
| Result page | n8n-nodes-base.code | Generates final HTML summary page with download links. | Read account (after) | Your video | 4. Result + Google Drive |
| Your video | n8n-nodes-base.form | Displays final success page. | Result page | Done | 4. Result + Google Drive |
| Done | n8n-nodes-base.form | Form completion termination screen. | Your video | None | 4. Result + Google Drive |
| Stopped | n8n-nodes-base.form | Renders error and stop message screens. | Start route, Upload route, Job next, Takes next | None | 5. Stops |
| Media file | n8n-nodes-base.webhook | Webhook trigger for secure media streaming. | None | Media request | 6. Media file (player and downloads) |
| Media request | n8n-nodes-base.code | Validates HMAC ticket and extracts file metadata. | Media file | Ticket ok? | 6. Media file (player and downloads) |
| Ticket ok? | n8n-nodes-base.if | Evaluates ticket signature validity. | Media request | Fetch asset, Not found | 6. Media file (player and downloads) |
| Fetch asset | n8n-nodes-base.httpRequest | Fetches binary media file from Muse API. | Ticket ok? | Send media | 6. Media file (player and downloads) |
| Send media | n8n-nodes-base.respondToWebhook | Streams binary response with security headers. | Fetch asset | None | 6. Media file (player and downloads) |
| Not found | n8n-nodes-base.respondToWebhook | Returns HTTP 404 for invalid media tickets. | Ticket ok? | None | 6. Media file (player and downloads) |
| Every 10 minutes | n8n-nodes-base.scheduleTrigger | Scheduled trigger for batch processing. | None | Settings | 7. Google Sheet batch (off by default) |
| Settings | n8n-nodes-base.code | Configures global batch parameters and sheet ID. | Every 10 minutes | Batch read account | 7. Google Sheet batch (off by default) |
| Batch read account | n8n-nodes-base.httpRequest | Queries Muse account quota for batch processing. | Settings | Read rows | 7. Google Sheet batch (off by default) |
| Read rows | n8n-nodes-base.googleSheets | Reads data rows from Google Sheets. | Batch read account | Plan the tick | 7. Google Sheet batch (off by default) |
| Plan the tick | n8n-nodes-base.code | Plans batch execution queue and rate limits. | Read rows | Row step | 7. Google Sheet batch (off by default) |
| Row step | n8n-nodes-base.switch | Routes batch row actions (upload, submit, read, note). | Plan the tick | Batch upload, Batch submit, Batch read job, Row values | 7. Google Sheet batch (off by default) |
| Batch upload | n8n-nodes-base.httpRequest | Uploads photo URL for batch row. | Row step | Batch submit | 7. Google Sheet batch (off by default) |
| Batch submit | n8n-nodes-base.httpRequest | Submits batch job to Muse API. | Row step, Batch upload | Row result | 7. Google Sheet batch (off by default) |
| Batch read job | n8n-nodes-base.httpRequest | Polls batch job status. | Row step | Row result | 7. Google Sheet batch (off by default) |
| Row result | n8n-nodes-base.code | Evaluates batch job response. | Batch submit, Batch read job | Drive? | 7. Google Sheet batch (off by default) |
| Drive? | n8n-nodes-base.if | Checks if batch item requires Google Drive upload. | Row result | Batch download, Row values | 7. Google Sheet batch (off by default) |
| Batch download | n8n-nodes-base.httpRequest | Downloads file for batch Drive upload. | Drive? | Batch save | 7. Google Sheet batch (off by default) |
| Batch save | n8n-nodes-base.googleDrive | Saves batch file to Google Drive. | Batch download | Row values | 7. Google Sheet batch (off by default) |
| Row values | n8n-nodes-base.code | Formats updated row values. | Drive?, Batch save, Row step | Update row | 7. Google Sheet batch (off by default) |
| Update row | n8n-nodes-base.googleSheets | Writes updated status and links back to Google Sheets. | Row values | None | 7. Google Sheet batch (off by default) |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Credentials Setup
1. **HTTP Header Auth (`useapi.net API token`):** Create a credential named `useapi.net API token`. Set the header name to `Authorization` and value to `Bearer YOUR_API_TOKEN`.
2. **HTTP Basic Auth (`Form login`):** Create an HTTP Basic Authentication credential with desired username and password to protect the form triggers.
3. **Google Drive OAuth2 API (`Google Drive account`):** Configure OAuth2 credentials for uploading media files.
4. **Google Sheets OAuth2 API (`Google Sheets account`):** Configure OAuth2 credentials for batch processing.

#### Step 2: Form & Input Reception Nodes
1. **Bring a photo to life (`n8n-nodes-base.formTrigger`):**
   - Path: `muse-old-photo`
   - Authentication: Basic Auth
   - Add form fields matching the specification: File upload, URL input, dropdowns for Restore, Format, Takes, Google Drive, and a textarea for speech text.
2. **Read the form (`n8n-nodes-base.code`):** Insert the validation, error handling, and HMAC signing code.
3. **Start route (`n8n-nodes-base.switch`):** Configure 3 outputs evaluating `={{ {"stop": 0, "file": 1, "url": 2}[$json.route] ?? 2 }}`.
4. **Stopped (`n8n-nodes-base.form`):** Create a form node for displaying error notices.

#### Step 3: Asset Ingestion & Quota Check
1. **Upload photo & Upload from URL (`n8n-nodes-base.httpRequest`):**
   - Method: POST
   - URL: `https://api.useapi.net/v1/muse/assets`
   - Credential: `useapi.net API token`
2. **Read account (`n8n-nodes-base.httpRequest`):**
   - Method: GET
   - URL: `https://api.useapi.net/v1/muse/accounts/{{ $json.email }}`
3. **Upload check (`n8n-nodes-base.code`):** Add verification code and quota estimation logic.
4. **Upload route & Before you spend (`n8n-nodes-base.switch` & `n8n-nodes-base.form`):** Set up confirmation routing and form display.
5. **Start choice & Start next:** Add code/switch logic to branch to restoration or video generation.

#### Step 4: Photo Restoration Pipeline
1. **Restore request & Submit restore:** Setup POST request to `https://api.useapi.net/v1/muse/jobs/image`.
2. **Submit check & Submit route:** Add status evaluation code and switch node (outputs: `retry`, `stop`, `poll`).
3. **Wait 10s & Retry submit in 30s (`n8n-nodes-base.wait`):** Configure pause timers.
4. **Check job (`n8n-nodes-base.httpRequest`):** GET request to `https://api.useapi.net/v1/muse/jobs/{{ $json.state.jobid }}`.
5. **Job status, Job next, Restored, Restore choice, Redo?:** Add intermediate code processors and switch branches.

#### Step 5: Talking Video Pipeline
1. **Take requests & Submit take:** POST request to `https://api.useapi.net/v1/muse/jobs/video` with batching configuration (`batchSize: 1`, `batchInterval: 2000`).
2. **Takes submitted, Takes next, Wait 15s, Check takes, Takes status, Retry takes in 30s:** Build polling loop and error handlers.
3. **Your takes & Take choice & Choice route:** Add UI rendering and selection routing.

#### Step 6: Post-Processing & Google Drive Integration
1. **Save to Drive?, Files to save, Download file, Upload to Drive, Read account (after):** Configure conditional branching, file downloads, and Google Drive upload nodes.
2. **Result page, Your video, Done:** Add HTML result rendering, final review form, and completion trigger.

#### Step 7: Secure Media Proxy Webhook
1. **Media file (`n8n-nodes-base.webhook`):** Path `muse-old-photo-media`, response mode "Response Node".
2. **Media request, Ticket ok?, Fetch asset, Send media, Not found:** Implement HMAC validation, secure proxy HTTP request, and binary response streaming.

#### Step 8: Google Sheets Batch Processing (Optional)
1. **Every 10 minutes (`n8n-nodes-base.scheduleTrigger`):** Set interval to 10 minutes (disabled by default).
2. **Settings, Batch read account, Read rows, Plan the tick, Row step, Batch upload, Batch submit, Batch read job, Row result, Drive?, Batch download, Batch save, Row values, Update row:** Connect sequential batch processing and sheet updating nodes.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| useapi.net Documentation & Setup | [Setup Muse Documentation](https://useapi.net/docs/start-here/setup-muse?utm_source=n8n&utm_medium=referral&utm_campaign=muse-photo-app) |
| GitHub Repository & Sheet Template | [GitHub Repository](https://github.com/useapi/muse-api/tree/main/n8n) |
| API Reference | [Muse API v1 Documentation](https://useapi.net/docs/api-muse-v1?utm_source=n8n&utm_medium=referral&utm_campaign=muse-photo-app) |
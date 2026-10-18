Save Rumble video transcripts to Google Sheets with FetchGoblin

https://n8nworkflows.xyz/workflows/save-rumble-video-transcripts-to-google-sheets-with-fetchgoblin-20927


# Save Rumble video transcripts to Google Sheets with FetchGoblin

### 1. Workflow Overview

This workflow is designed to manually fetch available transcripts for a specified Rumble video using the FetchGoblin API and append the processed transcript details directly into a Google Sheets worksheet. It automates the retrieval of metadata and caption text, ensuring data safety by truncating oversized texts and validating non-empty responses prior to database insertion.

The logical execution is grouped into a single linear sequence:
- **1.1 Input Initialization:** Triggers the workflow execution manually and initializes target parameters (Rumble video URL, Google Spreadsheet ID, and target sheet name).
- **1.2 API Data Retrieval:** Connects to the FetchGoblin API using secure header authentication to fetch the requested video transcript and associated usage metadata.
- **1.3 Data Transformation & Filtering:** Filters out empty payloads, applies safety character limits (truncating strings exceeding 49,000 characters), and formats the final row schema.
- **1.4 Target Output Storage:** Appends the formatted row values into the specified Google Sheets document using raw cell formatting.

---

### 2. Block-by-Block Analysis

#### Block 1: Input Initialization & Configuration
- **Overview:** This block initializes the automation on-demand and defines the target parameters required for the API request and target Google Sheet destination.
- **Nodes Involved:** `Run manually`, `Video and sheet settings`

- **Node Details:**
  - **Run manually**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Trigger node)
    - *Configuration Choices:* Standard manual execution trigger with no additional parameters required.
    - *Input/Output Connections:* No inputs; outputs to `Video and sheet settings`.
    - *Edge Cases/Failures:* None. Requires user interaction to start.
  - **Video and sheet settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data transformation/assignment node)
    - *Configuration Choices:* Sets string variables for `video_url`, `spreadsheet_id`, and `sheet_name`.
    - *Key Expressions/Variables:* Assigns default test values (`https://rumble.com/...`, `YOUR_SPREADSHEET_ID`, `Transcripts`).
    - *Input/Output Connections:* Input from `Run manually`; outputs to `Fetch Rumble transcript`.
    - *Edge Cases/Failures:* Ensure placeholders (like `YOUR_SPREADSHEET_ID`) are replaced with valid alphanumeric IDs before production deployment.

#### Block 2: API Data Retrieval
- **Overview:** Executes an authenticated HTTP GET request to the FetchGoblin API to retrieve the transcript payload for the defined Rumble video URL.
- **Nodes Involved:** `Fetch Rumble transcript`

- **Node Details:**
  - **Fetch Rumble transcript**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Integration node)
    - *Configuration Choices:* Method set to `GET`, URL targeted to `https://fetchgoblin.com/api/data/v1/rumble/video/transcript`, timeout configured to 60,000ms. Sends query parameters and headers. Uses generic HTTP Header Authentication.
    - *Key Expressions/Variables:* 
      - Query parameter `url`: `={{ $json.video_url }}`
      - Header `Idempotency-Key`: `={{ 'rumble-' + $execution.id + '-' + $itemIndex }}`
    - *Input/Output Connections:* Input from `Video and sheet settings`; outputs to `Prepare caption row`.
    - *Edge Cases/Failures:* Authentication errors if the API key is invalid or missing. Timeouts if the external API takes longer than 60 seconds. Null responses if the video lacks available captions.

#### Block 3: Data Transformation & Formatting
- **Overview:** Validates the presence of transcript text, filters out empty items, handles character length constraints, and maps the output into a standardized schema ready for spreadsheet insertion.
- **Nodes Involved:** `Prepare caption row`

- **Node Details:**
  - **Prepare caption row**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Custom JavaScript processing node)
    - *Configuration Choices:* Evaluates all incoming items (`runOnceForAllItems` mode). Checks if `data.transcript` is a non-empty string. Truncates strings exceeding 49,000 characters and appends a truncation marker.
    - *Key Expressions/Variables:* 
      - Source URL fallback: `data.url || $('Video and sheet settings').first().json.video_url`
      - Timestamp generation: `new Date().toISOString()`
    - *Input/Output Connections:* Input from `Fetch Rumble transcript`; outputs to `Append transcript row`.
    - *Edge Cases/Failures:* JavaScript runtime exceptions if data structures deviate from expected schemas. Skips processing if the transcript is null or blank.

#### Block 4: Target Output Storage
- **Overview:** Appends the final prepared row payload into the target Google Sheets worksheet using raw cell formatting.
- **Nodes Involved:** `Append transcript row`

- **Node Details:**
  - **Append transcript row**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Integration node)
    - *Configuration Choices:* Resource set to `sheet`, operation set to `append`. Uses auto-mapped input data schema, raw cell format, and ignores extra data fields.
    - *Key Expressions/Variables:*
      - Document ID: `={{ $('Video and sheet settings').first().json.spreadsheet_id }}`
      - Sheet Name: `={{ $('Video and sheet settings').first().json.sheet_name }}`
    - *Input/Output Connections:* Input from `Prepare caption row`; no downstream outputs.
    - *Edge Cases/Failures:* OAuth2 token expiration or insufficient write permissions on the target Google Sheet will cause execution failures.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Run manually** | `n8n-nodes-base.manualTrigger` | Triggers the workflow execution manually. | None | Video and sheet settings | # Save Rumble transcripts to Google Sheets<br><br>**For researchers and automation builders.** Supply one Rumble video URL and save available captions, their language, credit charge and retrieval date.<br><br>1. Set `video_url`, `spreadsheet_id` and `sheet_name` in **Video and sheet settings**.<br>2. Select an HTTP **Header Auth** credential in **Fetch Rumble transcript**: header name `x-api-key`, value your private FetchGoblin key.<br>3. Connect Google Sheets and create these headers: `source_url`, `language`, `transcript`, `retrieved_at`, `credits_charged`, `request_id`.<br>4. Run manually. Null/empty captions stop before Sheets. Text over 49,000 characters is truncated and visibly marked. Each rerun appends a row; no deduplication or schedule is included.<br><br>Caption requests verified 2026-10-13. Returned live captions cost one credit; the tested no-caption URL cost zero. Trial: 50 credits after verification; prepaid top-ups from $5. n8n hosting has separate costs.<br><br>[Full setup guide](https://fetchgoblin.com/guides/rumble-transcripts-google-sheets) · [API reference](https://fetchgoblin.com/docs/get-v1-rumble-video-transcript) |
| **Video and sheet settings** | `n8n-nodes-base.set` | Defines configuration parameters for video URL and sheet targets. | Run manually | Fetch Rumble transcript | # Save Rumble transcripts to Google Sheets<br><br>**For researchers and automation builders.** Supply one Rumble video URL and save available captions, their language, credit charge and retrieval date.<br><br>1. Set `video_url`, `spreadsheet_id` and `sheet_name` in **Video and sheet settings**.<br>2. Select an HTTP **Header Auth** credential in **Fetch Rumble transcript**: header name `x-api-key`, value your private FetchGoblin key.<br>3. Connect Google Sheets and create these headers: `source_url`, `language`, `transcript`, `retrieved_at`, `credits_charged`, `request_id`.<br>4. Run manually. Null/empty captions stop before Sheets. Text over 49,000 characters is truncated and visibly marked. Each rerun appends a row; no deduplication or schedule is included.<br><br>Caption requests verified 2026-10-13. Returned live captions cost one credit; the tested no-caption URL cost zero. Trial: 50 credits after verification; prepaid top-ups from $5. n8n hosting has separate costs.<br><br>[Full setup guide](https://fetchgoblin.com/guides/rumble-transcripts-google-sheets) · [API reference](https://fetchgoblin.com/docs/get-v1-rumble-video-transcript)<br><br>## Configure once<br>Keep the FetchGoblin API key in an HTTP Header Auth credential, never in a node or shared JSON. Set the video URL and destination spreadsheet in the settings node. The sample URL is public test input; replace it with a video you are authorized to process. |
| **Fetch Rumble transcript** | `n8n-nodes-base.httpRequest` | Calls the FetchGoblin API to retrieve the video transcript and metadata. | Video and sheet settings | Prepare caption row | # Save Rumble transcripts to Google Sheets<br><br>**For researchers and automation builders.** Supply one Rumble video URL and save available captions, their language, credit charge and retrieval date.<br><br>1. Set `video_url`, `spreadsheet_id` and `sheet_name` in **Video and sheet settings**.<br>2. Select an HTTP **Header Auth** credential in **Fetch Rumble transcript**: header name `x-api-key`, value your private FetchGoblin key.<br>3. Connect Google Sheets and create these headers: `source_url`, `language`, `transcript`, `retrieved_at`, `credits_charged`, `request_id`.<br>4. Run manually. Null/empty captions stop before Sheets. Text over 49,000 characters is truncated and visibly marked. Each rerun appends a row; no deduplication or schedule is included.<br><br>Caption requests verified 2026-10-13. Returned live captions cost one credit; the tested no-caption URL cost zero. Trial: 50 credits after verification; prepaid top-ups from $5. n8n hosting has separate costs.<br><br>[Full setup guide](https://fetchgoblin.com/guides/rumble-transcripts-google-sheets) · [API reference](https://fetchgoblin.com/docs/get-v1-rumble-video-transcript)<br><br>## Configure once<br>Keep the FetchGoblin API key in an HTTP Header Auth credential, never in a node or shared JSON. Set the video URL and destination spreadsheet in the settings node. The sample URL is public test input; replace it with a video you are authorized to process. |
| **Prepare caption row** | `n8n-nodes-base.code` | Filters and truncates transcript data, preparing fields for insertion. | Fetch Rumble transcript | Append transcript row | # Save Rumble transcripts to Google Sheets<br><br>**For researchers and automation builders.** Supply one Rumble video URL and save available captions, their language, credit charge and retrieval date.<br><br>1. Set `video_url`, `spreadsheet_id` and `sheet_name` in **Video and sheet settings**.<br>2. Select an HTTP **Header Auth** credential in **Fetch Rumble transcript**: header name `x-api-key`, value your private FetchGoblin key.<br>3. Connect Google Sheets and create these headers: `source_url`, `language`, `transcript`, `retrieved_at`, `credits_charged`, `request_id`.<br>4. Run manually. Null/empty captions stop before Sheets. Text over 49,000 characters is truncated and visibly marked. Each rerun appends a row; no deduplication or schedule is included.<br><br>Caption requests verified 2026-10-13. Returned live captions cost one credit; the tested no-caption URL cost zero. Trial: 50 credits after verification; prepaid top-ups from $5. n8n hosting has separate costs.<br><br>[Full setup guide](https://fetchgoblin.com/guides/rumble-transcripts-google-sheets) · [API reference](https://fetchgoblin.com/docs/get-v1-rumble-video-transcript)<br><br>## Preserve useful rows<br>Null or blank transcripts stop here. Long captions are capped at 49,000 characters and marked as truncated. Google Sheets uses RAW values, so caption text cannot become a spreadsheet formula. Repeated manual runs append rows; add deduplication if you need it. |
| **Append transcript row** | `n8n-nodes-base.googleSheets` | Appends formatted transcript rows to the designated Google Sheet. | Prepare caption row | None | # Save Rumble transcripts to Google Sheets<br><br>**For researchers and automation builders.** Supply one Rumble video URL and save available captions, their language, credit charge and retrieval date.<br><br>1. Set `video_url`, `spreadsheet_id` and `sheet_name` in **Video and sheet settings**.<br>2. Select an HTTP **Header Auth** credential in **Fetch Rumble transcript**: header name `x-api-key`, value your private FetchGoblin key.<br>3. Connect Google Sheets and create these headers: `source_url`, `language`, `transcript`, `retrieved_at`, `credits_charged`, `request_id`.<br>4. Run manually. Null/empty captions stop before Sheets. Text over 49,000 characters is truncated and visibly marked. Each rerun appends a row; no deduplication or schedule is included.<br><br>Caption requests verified 2026-10-13. Returned live captions cost one credit; the tested no-caption URL cost zero. Trial: 50 credits after verification; prepaid top-ups from $5. n8n hosting has separate costs.<br><br>[Full setup guide](https://fetchgoblin.com/guides/rumble-transcripts-google-sheets) · [API reference](https://fetchgoblin.com/docs/get-v1-rumble-video-transcript)<br><br>## Preserve useful rows<br>Null or blank transcripts stop here. Long captions are capped at 49,000 characters and marked as truncated. Google Sheets uses RAW values, so caption text cannot become a spreadsheet formula. Repeated manual runs append rows; add deduplication if you need it. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run manually`.
2. **Create a Set Node for Configuration:**
   - Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`). Name it `Video and sheet settings`.
   - Add three string assignments:
     - `video_url`: `https://rumble.com/v79xhhm-discovery-why-glenn-wants-israel-to-sue-the-new-york-times.html`
     - `spreadsheet_id`: `YOUR_SPREADSHEET_ID`
     - `sheet_name`: `Transcripts`
   - Connect `Run manually` to `Video and sheet settings`.
3. **Create an HTTP Request Node:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Fetch Rumble transcript`.
   - Set Method to `GET` and URL to `https://fetchgoblin.com/api/data/v1/rumble/video/transcript`.
   - Under Options, set Timeout to `60000`.
   - Enable **Query Parameters** and add a parameter with Name `url` and Value `={{ $json.video_url }}`.
   - Enable **Header Parameters** and add:
     - Name: `Accept`, Value: `application/json`
     - Name: `Idempotency-Key`, Value: `={{ 'rumble-' + $execution.id + '-' + $itemIndex }}`
   - Configure Authentication to **Generic Credential Type** -> **HTTP Header Auth**. 
     - *Credential Setup:* Create an HTTP Header Auth credential using header name `x-api-key` and your private FetchGoblin API key.
   - Connect `Video and sheet settings` to `Fetch Rumble transcript`.
4. **Create a Code Node for Data Transformation:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Prepare caption row`.
   - Set mode to **Run Once for All Items**.
   - Paste the following JavaScript snippet:
     ```javascript
     const rows = [];
     for (const [index, item] of $input.all().entries()) {
       const data = item.json;
       if (typeof data.transcript !== 'string' || !data.transcript.trim()) continue;
       const transcript = data.transcript.length > 49000
         ? data.transcript.slice(0, 49000) + '\n[TRUNCATED: transcript exceeds the workflow cell limit]'
         : data.transcript;
       rows.push({
         json: {
           source_url: data.url || $('Video and sheet settings').first().json.video_url,
           language: data.language || '',
           transcript,
           retrieved_at: new Date().toISOString(),
           credits_charged: data.credits_charged ?? 0,
           request_id: data.request_id || '',
         },
         pairedItem: { item: index },
       });
     }
     return rows;
     ```
   - Connect `Fetch Rumble transcript` to `Prepare caption row`.
5. **Create a Google Sheets Node:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`). Name it `Append transcript row`.
   - Set Resource to `Sheet` and Operation to `Append`.
   - Configure Document ID: `={{ $('Video and sheet settings').first().json.spreadsheet_id }}`
   - Configure Sheet Name: `={{ $('Video and sheet settings').first().json.sheet_name }}`
   - Under Options, set Cell Format to `RAW` and Handling Extra Data to `Ignore It`.
   - *Credential Setup:* Create/Select valid Google Sheets OAuth2 credentials with write permissions to your target spreadsheet containing the headers: `source_url`, `language`, `transcript`, `retrieved_at`, `credits_charged`, and `request_id`.
   - Connect `Prepare caption row` to `Append transcript row`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Full setup guide | [https://fetchgoblin.com/guides/rumble-transcripts-google-sheets](https://fetchgoblin.com/guides/rumble-transcripts-google-sheets) |
| API reference | [https://fetchgoblin.com/docs/get-v1-rumble-video-transcript](https://fetchgoblin.com/docs/get-v1-rumble-video-transcript) |
Validate social media posts in Google Sheets and send issues to Slack

https://n8nworkflows.xyz/workflows/validate-social-media-posts-in-google-sheets-and-send-issues-to-slack-20967


# Validate social media posts in Google Sheets and send issues to Slack

### 1. Workflow Overview

This workflow is designed as a quality gate for social media content drafts stored in Google Sheets. Its primary purpose is to automate the validation of post drafts—checking for missing fields, unsupported platforms, invalid image URLs, and duplicate text per platform—before they are approved or published. 

The logical execution is grouped into three main blocks:
- **1.1 Input Reception and Validation:** Initiates the workflow manually, reads all draft rows from a Google Sheets worksheet, and executes comprehensive batch validation logic to flag errors and detect new or changed issues.
- **1.2 Issue Evaluation and Alerting:** Assesses whether validation identified any new or changed problems and conditionally sends a formatted warning message to a designated Slack channel.
- **1.3 Results Processing and Sheet Updates:** Re-packs the evaluated row items into individual data payloads and writes the final statuses, validation results, and timestamps back to the source Google Sheet.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception and Validation

- **Overview:** This block handles the manual entry point, extracts all draft records from the connected Google Sheets document, and runs a custom JavaScript validation process to ensure every post adheres to structural and content quality standards.
- **Nodes Involved:** 
  - `Manual Trigger`
  - `Read Rows in Sheet`
  - `Validate Posts Batch`

- **Node Details:**
  - **Manual Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (v1) — Acts as the entry point for manual execution.
    - *Configuration Choices:* No specific parameters required.
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Output connects directly to `Read Rows in Sheet`.
    - *Edge Cases/Potential Failures:* None; purely initializes the workflow run.
  
  - **Read Rows in Sheet**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.7) — Retrieves all rows from the specified target worksheet.
    - *Configuration Choices:* Operation set to read/get rows. Uses a placeholder document ID (`REPLACE_WITH_YOUR_GOOGLE_SHEET_ID`) and targets `Sheet1` (`gid=0`).
    - *Key Expressions or Variables:* None.
    - *Input/Output Connections:* Input from `Manual Trigger`; output connects to `Validate Posts Batch`.
    - *Edge Cases/Potential Failures:* Authentication errors, invalid spreadsheet ID, or deleted sheet tabs will cause execution to halt.
  
  - **Validate Posts Batch**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — Executes custom JavaScript to process all rows in a single batch.
    - *Configuration Choices:* JavaScript code block validates unique `post_id` values, verifies supported platforms (`instagram`, `facebook`, `x`), ensures content is non-empty, validates image URL formats via regular expression, and checks for platform-specific content duplication.
    - *Key Expressions or Variables:* Uses `$input.all()` to read incoming sheet rows and parses fields such as `post_id`, `platform`, `content`, `image_url`, `status`, and `validation_result`.
    - *Input/Output Connections:* Input from `Read Rows in Sheet`; output connects to `Check for New Issues`.
    - *Edge Cases/Potential Failures:* Throws explicit runtime errors if any `post_id` is missing or duplicated within the sheet batch.

---

#### Block 1.2: Issue Evaluation and Alerting

- **Overview:** This block evaluates the output from the validation batch to determine if any new or changed quality issues were detected, routing the flow to send a Slack alert if the issue count is greater than zero.
- **Nodes Involved:** 
  - `Check for New Issues`
  - `Post Message to Slack`

- **Node Details:**
  - **Check for New Issues**
    - *Type and Technical Role:* `n8n-nodes-base.if` (v2.2) — Conditionally splits the workflow based on the number of new issues found.
    - *Configuration Choices:* Evaluates whether `new_issue_count` is greater than `0`.
    - *KeyExpressions or Variables:* `={{ $json.new_issue_count }}`
    - *Input/Output Connections:* Input from `Validate Posts Batch`. True branch connects to `Post Message to Slack`; false branch bypasses Slack and connects directly to `Process Results Batch`.
    - *Edge Cases/Potential Failures:* Expression failure if `new_issue_count` is undefined.
  
  - **Post Message to Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (v2.7) — Sends formatted warning notifications regarding content quality issues.
    - *Configuration Choices:* Posts a message to a named channel (`YOUR_SLACK_CHANNEL`).
    - *Key Expressions or Variables:* `={{ $json.message }}`
    - *Input/Output Connections:* Input from the true branch of `Check for New Issues`; output connects to `Process Results Batch`.
    - *Edge Cases/Potential Failures:* Slack authentication failures, missing channel permissions, or bot user lacking access to the target channel.

---

#### Block 1.3: Results Processing and Sheet Updates

- **Overview:** This block consolidates execution paths (whether Slack was notified or skipped), expands the processed validation payload into distinct items, and updates the corresponding rows in Google Sheets.
- **Nodes Involved:** 
  - `Process Results Batch`
  - `Update Rows in Sheet`

- **Node Details:**
  - **Process Results Batch**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) — Extracts the validated row array from the upstream batch node and converts it into individual n8n items.
    - *Configuration Choices:* JavaScript code references the upstream `Validate Posts Batch` node output and maps each row object to an independent item structure.
    - *Key Expressions or Variables:* `$('Validate Posts Batch').first().json.rows`
    - *Input/Output Connections:* Inputs from both `Post Message to Slack` (via flow continuation) and the false branch of `Check for New Issues`. Output connects to `Update Rows in Sheet`.
    - *Edge Cases/Potential Failures:* Throws an error if the expected row array is missing from the upstream node.
  
  - **Update Rows in Sheet**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (v4.7) — Updates existing rows in the target Google Sheet based on matching identifiers.
    - *Configuration Choices:* Operation set to `update`. Uses `post_id` as the matching column. Mapped fields include `status`, `post_id`, `checked_at`, and `validation_result`.
    - *Key Expressions or Variables:* 
      - Status: `={{ $json.status }}`
      - Post ID: `={{ $json.post_id }}`
      - Checked At: `={{ $json.checked_at }}`
      - Validation Result: `={{ $json.validation_result }}`
    - *Input/Output Connections:* Input from `Process Results Batch`.
    - *Edge Cases/Potential Failures:* API rate limits, authentication expiry, or failure to locate a matching `post_id` in the worksheet.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Manual Trigger | n8n-nodes-base.manualTrigger | Initiates workflow execution manually. | None | Read Rows in Sheet | ## Social Media Content Quality Gate<br><br>### How it works<br><br>This workflow manually checks social media content stored in Google Sheets before making any updates. It reads rows, validates the posts in a batch, and determines whether any new quality issues were found. If issues exist, it sends a Slack notification, then expands the prepared results and updates the corresponding sheet rows.<br><br>### Setup steps<br><br>- Configure Google Sheets credentials for both the row retrieval and row update nodes.<br>- Select the correct spreadsheet, sheet, and row-matching/update columns in the Google Sheets nodes.<br>- Configure Slack credentials and choose the destination channel or user for issue notifications.<br>- Review the validation code to ensure it matches the sheet columns and quality rules used for the social media content.<br><br>### Customization<br><br>Adjust the validation logic, Slack message text, and Google Sheets update fields to match your content standards and approval workflow. |
| Read Rows in Sheet | n8n-nodes-base.googleSheets | Retrieves all draft post rows from Google Sheets. | Manual Trigger | Validate Posts Batch | ## Social Media Content Quality Gate<br><br>### How it works<br><br>This workflow manually checks social media content stored in Google Sheets before making any updates. It reads rows, validates the posts in a batch, and determines whether any new quality issues were found. If issues exist, it sends a Slack notification, then expands the prepared results and updates the corresponding sheet rows.<br><br>### Setup steps<br><br>- Configure Google Sheets credentials for both the row retrieval and row update nodes.<br>- Select the correct spreadsheet, sheet, and row-matching/update columns in the Google Sheets nodes.<br>- Configure Slack credentials and choose the destination channel or user for issue notifications.<br>- Review the validation code to ensure it matches the sheet columns and quality rules used for the social media content.<br><br>### Customization<br><br>Adjust the validation logic, Slack message text, and Google Sheets update fields to match your content standards and approval workflow.<br><br>## Load and validate posts<br><br>Starts the workflow manually, retrieves social media content rows from Google Sheets, and performs the initial batch validation before any sheet changes are made. |
| Validate Posts Batch | n8n-nodes-base.code | Validates post data, checks constraints, and detects issues. | Read Rows in Sheet | Check for New Issues | ## Social Media Content Quality Gate<br><br>### How it works<br><br>This workflow manually checks social media content stored in Google Sheets before making any updates. It reads rows, validates the posts in a batch, and determines whether any new quality issues were found. If issues exist, it sends a Slack notification, then expands the prepared results and updates the corresponding sheet rows.<br><br>### Setup steps<br><br>- Configure Google Sheets credentials for both the row retrieval and row update nodes.<br>- Select the correct spreadsheet, sheet, and row-matching/update columns in the Google Sheets nodes.<br>- Configure Slack credentials and choose the destination channel or user for issue notifications.<br>- Review the validation code to ensure it matches the sheet columns and quality rules used for the social media content.<br><br>### Customization<br><br>Adjust the validation logic, Slack message text, and Google Sheets update fields to match your content standards and approval workflow.<br><br>## Load and validate posts<br><br>Starts the workflow manually, retrieves social media content rows from Google Sheets, and performs the initial batch validation before any sheet changes are made. |
| Check for New Issues | n8n-nodes-base.if | Evaluates whether new validation issues were discovered. | Validate Posts Batch | Post Message to Slack, Process Results Batch | ## Social Media Content Quality Gate<br><br>### How it works<br><br>This workflow manually checks social media content stored in Google Sheets before making any updates. It reads rows, validates the posts in a batch, and determines whether any new quality issues were found. If issues exist, it sends a Slack notification, then expands the prepared results and updates the corresponding sheet rows.<br><br>### Setup steps<br><br>- Configure Google Sheets credentials for both the row retrieval and row update nodes.<br>- Select the correct spreadsheet, sheet, and row-matching/update columns in the Google Sheets nodes.<br>- Configure Slack credentials and choose the destination channel or user for issue notifications.<br>- Review the validation code to ensure it matches the sheet columns and quality rules used for the social media content.<br><br>### Customization<br><br>Adjust the validation logic, Slack message text, and Google Sheets update fields to match your content standards and approval workflow.<br><br>## Check and notify issues<br><br>Evaluates whether validation found new issues and, when needed, sends a Slack message before continuing the workflow. |
| Post Message to Slack | n8n-nodes-base.slack | Sends formatted alert messages to Slack when issues are found. | Check for New Issues | Process Results Batch | ## Social Media Content Quality Gate<br><br>### How it works<br><br>This workflow manually checks social media content stored in Google Sheets before making any updates. It reads rows, validates the posts in a batch, and determines whether any new quality issues were found. If issues exist, it sends a Slack notification, then expands the prepared results and updates the corresponding sheet rows.<br><br>### Setup steps<br><br>- Configure Google Sheets credentials for both the row retrieval and row update nodes.<br>- Select the correct spreadsheet, sheet, and row-matching/update columns in the Google Sheets nodes.<br>- Configure Slack credentials and choose the destination channel or user for issue notifications.<br>- Review the validation code to ensure it matches the sheet columns and quality rules used for the social media content.<br><br>### Customization<br><br>Adjust the validation logic, Slack message text, and Google Sheets update fields to match your content standards and approval workflow.<br><br>## Check and notify issues<br><br>Evaluates whether validation found new issues and, when needed, sends a Slack message before continuing the workflow. |
| Process Results Batch | n8n-nodes-base.code | Expands batch validation results into individual row items. | Check for New Issues, Post Message to Slack | Update Rows in Sheet | ## Social Media Content Quality Gate<br><br>### How it works<br><br>This workflow manually checks social media content stored in Google Sheets before making any updates. It reads rows, validates the posts in a batch, and determines whether any new quality issues were found. If issues exist, it sends a Slack notification, then expands the prepared results and updates the corresponding sheet rows.<br><br>### Setup steps<br><br>- Configure Google Sheets credentials for both the row retrieval and row update nodes.<br>- Select the correct spreadsheet, sheet, and row-matching/update columns in the Google Sheets nodes.<br>- Configure Slack credentials and choose the destination channel or user for issue notifications.<br>- Review the validation code to ensure it matches the sheet columns and quality rules used for the social media content.<br><br>### Customization<br><br>Adjust the validation logic, Slack message text, and Google Sheets update fields to match your content standards and approval workflow.<br><br>## Prepare and update rows<br><br>Combines the post-validation path, expands prepared update data into individual rows, and writes the results back to Google Sheets. |
| Update Rows in Sheet | n8n-nodes-base.googleSheets | Writes validation status, results, and timestamps back to the sheet. | Process Results Batch | None | ## Social Media Content Quality Gate<br><br>### How it works<br><br>This workflow manually checks social media content stored in Google Sheets before making any updates. It reads rows, validates the posts in a batch, and determines whether any new quality issues were found. If issues exist, it sends a Slack notification, then expands the prepared results and updates the corresponding sheet rows.<br><br>### Setup steps<br><br>- Configure Google Sheets credentials for both the row retrieval and row update nodes.<br>- Select the correct spreadsheet, sheet, and row-matching/update columns in the Google Sheets nodes.<br>- Configure Slack credentials and choose the destination channel or user for issue notifications.<br>- Review the validation code to ensure it matches the sheet columns and quality rules used for the social media content.<br><br>### Customization<br><br>Adjust the validation logic, Slack message text, and Google Sheets update fields to match your content standards and approval workflow.<br><br>## Prepare and update rows<br><br>Combines the post-validation path, expands prepared update data into individual rows, and writes the results back to Google Sheets. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**
   - Add a `Manual Trigger` node to start the workflow execution.

2. **Add the First Google Sheets Node (Read Rows)**
   - Add a `Google Sheets` node named `Read Rows in Sheet`.
   - Set **Operation** to `Get` (or equivalent read operation).
   - Configure **Document ID** with your target spreadsheet ID (`REPLACE_WITH_YOUR_GOOGLE_SHEET_ID`).
   - Configure **Sheet Name** to point to your worksheet (`Sheet1`, `gid=0`).
   - Link the output of `Manual Trigger` to `Read Rows in Sheet`.

3. **Add the Validation Code Node**
   - Add a `Code` node named `Validate Posts Batch`.
   - Set **Mode** to `Run Once for All Items`.
   - Insert the batch validation JavaScript code that checks for unique `post_id` values, platform constraints (`instagram`, `facebook`, `x`), non-empty content, image URL regex formatting, and platform-specific text duplicates.
   - Link the output of `Read Rows in Sheet` to `Validate Posts Batch`.

4. **Add the IF Node (Check for New Issues)**
   - Add an `If` node named `Check for New Issues`.
   - Set the condition rule to check if number (`gt`) `={{ $json.new_issue_count }}` is greater than `0`.
   - Link the output of `Validate Posts Batch` to `Check for New Issues`.

5. **Add the Slack Node (Post Message)**
   - Add a `Slack` node named `Post Message to Slack`.
   - Set **Resource** to `Message` and **Operation** to `Post`.
   - Configure the target channel (`YOUR_SLACK_CHANNEL`).
   - Set the message text parameter to `={{ $json.message }}`.
   - Link the **True** (upper) output of `Check for New Issues` to `Post Message to Slack`.

6. **Add the Results Processing Code Node**
   - Add a `Code` node named `Process Results Batch`.
   - Set **Mode** to `Run Once for All Items`.
   - Insert code that reads the cached array from `$('Validate Posts Batch').first().json.rows` and maps each object into an individual n8n item.
   - Link both the **False** (lower) output of `Check for New Issues` and the output of `Post Message to Slack` to `Process Results Batch`.

7. **Add the Second Google Sheets Node (Update Rows)**
   - Add a `Google Sheets` node named `Update Rows in Sheet`.
   - Set **Operation** to `Update`.
   - Configure **Document ID** with your target spreadsheet ID.
   - Configure **Sheet Name** to point to your worksheet (`Sheet1`, `gid=0`).
   - Set **Matching Columns** to `post_id`.
   - Map the fields below:
     - `post_id` $\to$ `={{ $json.post_id }}`
     - `status` $\to$ `={{ $json.status }}`
     - `validation_result` $\to$ `={{ $json.validation_result }}`
     - `checked_at` $\to$ `={{ $json.checked_at }}`
   - Link the output of `Process Results Batch` to `Update Rows in Sheet`.

8. **Credentials Setup**
   - Configure valid **Google OAuth2 / Service Account credentials** for both Google Sheets nodes.
   - Configure valid **Slack API credentials** (OAuth2 or Bot Token) for the Slack node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ensure each row in the Google Sheet contains a unique, non-empty `post_id` before executing the workflow. | Google Sheets Data Integrity |
| Verify that the Slack bot user or app has been explicitly invited to the destination notification channel. | Slack Integration Requirements |
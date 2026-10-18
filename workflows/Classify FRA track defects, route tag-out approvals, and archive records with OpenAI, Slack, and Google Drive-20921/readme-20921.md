Classify FRA track defects, route tag-out approvals, and archive records with OpenAI, Slack, and Google Drive

https://n8nworkflows.xyz/workflows/classify-fra-track-defects--route-tag-out-approvals--and-archive-records-with-openai--slack--and-google-drive-20921


# Classify FRA track defects, route tag-out approvals, and archive records with OpenAI, Slack, and Google Drive

### 1. Workflow Overview

This workflow automates daily short-line railway track safety management by integrating defect classification, risk calculations, and communication. It polls track-inspection records, uses an OpenAI language model to categorize inspector notes according to Federal Railroad Administration (FRA) severity rules, determines required speed restrictions and remedial timeframes, orchestrates critical human supervisor approvals via Slack for immediate tag-outs, and archives the resulting compliance records to Google Drive while notifying dispatch.

The workflow logic is grouped into the following functional blocks:
- **1.1 Input Reception & Normalization:** Triggers daily on a schedule and sets up the execution window parameters and railroad identifier.
- **1.2 Data Ingestion:** Fetches segments due for inspection from an external API, splitting the response into individual items for per-segment processing.
- **1.3 AI Severity Classification:** Utilizes an OpenAI chat model and a text classifier node to evaluate inspector defect notes into four distinct categories.
- **1.4 Category Tagging:** Standardizes defect category metadata across mutually exclusive branching paths.
- **1.5 Speed Restriction & Remedial Calculation:** Computes normal maximum speed, speed restrictions, remedial timeframes, and recommended actions using custom JavaScript logic based on track class.
- **1.6 Approval Gate & Routing:** Conditionally separates severe defects (Class 5 Immediate Halt) to request supervisor confirmation via Slack, routing responses through a 3-way conditional switch (Approve, Edit, Reject), while bypassing non-urgent segments.
- **1.7 Archiving & Notification:** Consolidates all execution paths to upload compliance record JSON files to Google Drive and broadcast final operational summaries to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** Initializes the daily execution run at a scheduled time and creates the baseline variables required to query inspection records.
- **Nodes Involved:** 
  - `Poll Segments Due for Inspection`
  - `Normalize Poll Window`
- **Node Details:**
  - **Poll Segments Due for Inspection**
    - *Type and Technical Role:* Schedule Trigger (`n8n-nodes-base.scheduleTrigger`, v1.2). Acts as the primary workflow entry point.
    - *Configuration:* Configured to run daily at 05:00.
    - *Expressions:* None.
    - *Connections:* Input: None; Output: `Normalize Poll Window`.
    - *Version Requirements:* v1.2.
    - *Edge Cases / Failures:* Missed executions if the n8n instance is offline at the scheduled time.
  - **Normalize Poll Window**
    - *Type and Technical Role:* Set Node (`n8n-nodes-base.set`, v3.4). Generates static and dynamic context variables.
    - *Configuration:* Assigns `poll_date` to the current date string (`YYYY-MM-DD`) and `railroad_id` to `"SHORTLINE-01"`.
    - *Expressions:* `={{ new Date().toISOString().slice(0,10) }}`
    - *Connections:* Input: `Poll Segments Due for Inspection`; Output: `Fetch Segments Due & Inspector Notes`.
    - *Version Requirements:* v3.4.
    - *Edge Cases / Failures:* JavaScript Date parsing failures if system time zone configurations are irregular.

#### 2.2 Data Ingestion
- **Overview:** Queries the external track-inspection API using the normalized poll date and splits the response payload into discrete items for individual processing.
- **Nodes Involved:**
  - `Fetch Segments Due & Inspector Notes`
- **Node Details:**
  - **Fetch Segments Due & Inspector Notes**
    - *Type and Technical Role:* HTTP Request Node (`n8n-nodes-base.httpRequest`, v4.2). Retrieves inspection data from the external endpoint.
    - *Configuration:* Method: `GET`, URL structured with query parameters, split response into items enabled (`splitIntoItems: true`), max retries set to 3 with a 2-second wait interval.
    - *Credentials:* Uses "Track Inspection System API Key" (HTTP Header Auth).
    - *Expressions:* `=https://track-inspection.example-shortline.com/api/segments/due?date={{ $json.poll_date }}`
    - *Connections:* Input: `Normalize Poll Window`; Output: `Classify Defect Severity`.
    - *Version Requirements:* v4.2.
    - *Edge Cases / Failures:* API timeouts, authentication failures, or malformed JSON payloads causing items to fail. Configured with error handling (`onError: continueRegularOutput`).

#### 2.3 AI Severity Classification
- **Overview:** Evaluates free-text inspector notes against four defined compliance categories using an underlying OpenAI language model.
- **Nodes Involved:**
  - `Classifier Model`
  - `Classify Defect Severity`
- **Node Details:**
  - **Classifier Model**
    - *Type and Technical Role:* LangChain OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`, v1.2). Provides the AI model configuration for classification.
    - *Configuration:* Model set to `gpt-4.1-mini`, temperature set to `0.1` for deterministic output. Max 3 retries.
    - *Credentials:* Uses "OpenAI account".
    - *Expressions:* None.
    - *Connections:* Output connected to `Classify Defect Severity` via AI language model wire.
    - *Version Requirements:* v1.2.
    - *Edge Cases / Failures:* API rate limits, upstream quota limits, or invalid API keys.
  - **Classify Defect Severity**
    - *Type and Technical Role:* LangChain Text Classifier (`@n8n/n8n-nodes-langchain.textClassifier`, v1). Categorizes text inputs into defined categories.
    - *Configuration:* Evaluates `inputText` against four categories: `No Defect`, `Class 1-2 Minor`, `Class 3-4 Serious`, and `Class 5 Immediate Halt`.
    - *Expressions:* `={{ $json.inspector_defect_notes }}`
    - *Connections:* Input: `Fetch Segments Due & Inspector Notes` and `Classifier Model`; Outputs: Four distinct branches leading to category tagging nodes.
    - *Version Requirements:* v1.
    - *Edge Cases / Failures:* Ambiguous inspector notes leading to incorrect classification outputs.

#### 2.4 Category Tagging
- **Overview:** Explicitly stamps the resulting defect category onto the execution data payload along each classification branch.
- **Nodes Involved:**
  - `Tag No Defect`
  - `Tag Minor`
  - `Tag Serious`
  - `Tag Immediate Halt`
- **Node Details:**
  - **Tag No Defect**
    - *Type and Technical Role:* Set Node (`n8n-nodes-base.set`, v3.4). Assigns the `No Defect` classification string.
  - **Tag Minor**
    - *Type and Technical Role:* Set Node (`n8n-nodes-base.set`, v3.4). Assigns the `Class 1-2 Minor` classification string.
  - **Tag Serious**
    - *Type and Technical Role:* Set Node (`n8n-nodes-base.set`, v3.4). Assigns the `Class 3-4 Serious` classification string.
  - **Tag Immediate Halt**
    - *Type and Technical Role:* Set Node (`n8n-nodes-base.set`, v3.4). Assigns the `Class 5 Immediate Halt` classification string.
    - *Common Configuration:* Each node sets `defect_category` via string assignment.
    - *Connections:* Input: Respective output branch from `Classify Defect Severity`; Output: `Compute FRA Speed Restriction & Remedial Timeframe`.
    - *Version Requirements:* v3.4.
    - *Edge Cases / Failures:* Missing field propagation if data structures upstream alter property names.

#### 2.5 Speed Restriction & Remedial Calculation
- **Overview:** Executes deterministic JavaScript code to calculate normal maximum speed, speed restrictions, remedial timeframes, and recommended actions based on track class and defect category.
- **Nodes Involved:**
  - `Compute FRA Speed Restriction & Remedial Timeframe`
- **Node Details:**
  - **Compute FRA Speed Restriction & Remedial Timeframe**
    - *Type and Technical Role:* Code Node (`n8n-nodes-base.code`, v2). Processes item data using JavaScript.
    - *Configuration:* Executes custom JavaScript processing track classes (defaulting to 3 if missing), applying speed mapping tables (`{1: 10, 2: 25, 3: 40, 4: 60, 5: 80}`), and appending calculated properties to the item JSON.
    - *Expressions:* JavaScript logic inside the node code block reads properties like `track_class` and `defect_category`.
    - *Connections:* Input: Converged from all four tag nodes; Output: `Requires Immediate Tag-Out?`.
    - *Version Requirements:* v2.
    - *Edge Cases / Failures:* Type casting exceptions if `track_class` contains non-numeric characters.

#### 2.6 Approval Gate & Routing
- **Overview:** Evaluates whether a defect requires immediate human supervisor sign-off, routes high-severity infractions through a Slack approval request, and handles supervisor feedback options.
- **Nodes Involved:**
  - `Requires Immediate Tag-Out?`
  - `Shape Non-Urgent Record`
  - `Request Supervisor Tag-Out Approval`
  - `Route Approval Response`
  - `Finalize Approved Tag-Out`
  - `Finalize Edited Tag-Out`
  - `Cancel Tag-Out (Rejected)`
- **Node Details:**
  - **Requires Immediate Tag-Out?**
    - *Type and Technical Role:* If Node (`n8n-nodes-base.if`, v2.2). Directs workflow execution based on condition.
    - *Configuration:* Evaluates if `defect_category` equals `Class 5 Immediate Halt`.
    - *Expressions:* `={{ $json.defect_category }}` equals `Class 5 Immediate Halt`.
    - *Connections:* Input: `Compute FRA Speed Restriction & Remedial Timeframe`; Outputs: True branch to `Request Supervisor Tag-Out Approval`, False branch to `Shape Non-Urgent Record`.
  - **Shape Non-Urgent Record**
    - *Type and Technical Role:* Set Node (`n8n-nodes-base.set`, v3.4). Prepares non-urgent records for archiving.
    - *Configuration:* Assigns `final_status` to `={{ $json.recommended_action }}` and `approval_outcome` to `not_required`.
  - **Request Supervisor Tag-Out Approval**
    - *Type and Technical Role:* Slack Node (`n8n-nodes-base.slack`, v2.3). Sends interactive approval requests.
    - *Configuration:* Operation: `sendAndWait`, response type: `approval` with approval options (`approveAndEdit`). Channel ID configured via placeholder.
    - *Credentials:* Uses "Track Ops Slack Bot" (Slack API).
    - *Expressions:* Message string dynamically includes `segment_id` and `recommended_action`.
    - *Connections:* Input: True branch of `Requires Immediate Tag-Out?`; Output: `Route Approval Response`.
  - **Route Approval Response**
    - *Type and Technical Role:* Switch Node (`n8n-nodes-base.switch`, v3.2). Routes execution based on multiple conditional outcomes.
    - *Configuration:* Three explicit rules evaluating approval flags (`$json.data.approved`) and modification text (`$json.data.text`), with fallback routing to rejection.
    - *Expressions:* Evaluates boolean approval states and string content lengths.
  - **Finalize Approved Tag-Out**, **Finalize Edited Tag-Out**, **Cancel Tag-Out (Rejected)**
    - *Type and Technical Role:* Set Nodes (`n8n-nodes-base.set`, v3.4). Finalize execution metadata according to the supervisor's decision outcome (`approved`, `approved_with_edit`, or `rejected`).
    - *Connections:* Output from all three converge into `Archive Defect Tag-Out Record`.

#### 2.7 Archiving & Notification
- **Overview:** Persists completed compliance records to cloud storage and broadcasts structured notification summaries to the operations team channel.
- **Nodes Involved:**
  - `Archive Defect Tag-Out Record`
  - `Notify Track Supervisor & Dispatch`
- **Node Details:**
  - **Archive Defect Tag-Out Record**
    - *Type and Technical Role:* Google Drive Node (`n8n-nodes-base.googleDrive`, v3). Uploads structured JSON files to cloud storage.
    - *Configuration:* Operation: `upload`. Target file named dynamically using segment ID and poll date. Error handling configured with retries (`onError: continueRegularOutput`).
    - *Credentials:* Uses "Track Compliance Google Drive account" (Google Drive OAuth2).
    - *Expressions:* `=track-defect-{{ $json.segment_id }}-{{ $json.poll_date }}.json`
    - *Connections:* Input: Converged outputs from `Shape Non-Urgent Record` and all three finalization nodes; Output: `Notify Track Supervisor & Dispatch`.
    - *Version Requirements:* v3.
    - *Edge Cases / Failures:* Drive quota limits, missing folder permissions, or network drops.
  - **Notify Track Supervisor & Dispatch**
    - *Type and Technical Role:* Slack Node (`n8n-nodes-base.slack`, v2.3). Posts final operational status updates.
    - *Configuration:* Operation: `post` message to designated channel ID. Error handling configured with retries.
    - *Credentials:* Uses "Track Ops Slack Bot" (Slack API).
    - *Expressions:* `=Segment {{ $json.segment_id }}: {{ $json.defect_category }} -> {{ $json.final_status }} (speed restriction {{ $json.speed_restriction_mph }} mph)`
    - *Connections:* Input: `Archive Defect Tag-Out Record`; Output: None (Terminal node).
    - *Version Requirements:* v2.3.
    - *Edge Cases / Failures:* Slack API rate limiting or invalid channel permissions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Poll Segments Due for Inspection | scheduleTrigger | Daily Schedule Trigger, 05:00. Kicks off the whole inspection-severity sweep. Without it nothing runs until manually triggered. | None | Normalize Poll Window | Poll Segments Due for Inspection |
| Normalize Poll Window | set | Stamps today's date and the railroad_id for the fetch call. Keeps the rest of the workflow date-agnostic. Single source of truth for the poll context. | Poll Segments Due for Inspection | Fetch Segments Due & Inspector Notes | Normalize Poll Window |
| Fetch Segments Due & Inspector Notes | httpRequest | GET call returning every segment due today, split into items. URL keyed by poll_date. The collection every downstream node processes one item at a time. | Normalize Poll Window | Classify Defect Severity | Fetch Segments Due & Inspector Notes |
| Classifier Model | lmChatOpenAi | OpenAI chat model feeding the Text Classifier. Model: gpt-4.1-mini, temp 0.1. Required — Text Classifier cannot run without it. | None | Classify Defect Severity | Classifier Model |
| Classify Defect Severity | textClassifier | Routes each segment's inspector notes into one of 4 severity categories. Outputs are separate branches, not a json field. The AI reasoning step this pattern is built around. | Fetch Segments Due & Inspector Notes, Classifier Model | Tag No Defect, Tag Minor, Tag Serious, Tag Immediate Halt | Classify Defect Severity |
| Tag No Defect | set | Stamps defect_category='No Defect' on this output branch. Text Classifier itself never writes this field (standing rule). Feeds straight into the speed/remedial Code node. | Classify Defect Severity | Compute FRA Speed Restriction & Remedial Timeframe | Tag No Defect |
| Tag Minor | set | Stamps defect_category='Class 1-2 Minor'. Monitor-only outcome, 180-day remedial window. One of 4 mutually exclusive category branches. | Classify Defect Severity | Compute FRA Speed Restriction & Remedial Timeframe | Tag Minor |
| Tag Serious | set | Stamps defect_category='Class 3-4 Serious'. Triggers a real speed restriction, 30-day remedial window. One of 4 mutually exclusive category branches. | Classify Defect Severity | Compute FRA Speed Restriction & Remedial Timeframe | Tag Serious |
| Tag Immediate Halt | set | Stamps defect_category='Class 5 Immediate Halt'. The only category that reaches the approval gate. Highest-severity branch. | Classify Defect Severity | Compute FRA Speed Restriction & Remedial Timeframe | Tag Immediate Halt |
| Compute FRA Speed Restriction & Remedial Timeframe | code | Real Code logic: track-class speed table + category -> mph + remedial days. All 4 category branches converge here. The deterministic engine behind every recommended_action. | Tag No Defect, Tag Minor, Tag Serious, Tag Immediate Halt | Requires Immediate Tag-Out? | Compute FRA Speed Restriction & Remedial Timeframe |
| Requires Immediate Tag-Out? | if | Branch on defect_category equals Class 5 Immediate Halt. First of the two required real branch points. Only this branch reaches the human approval gate. | Compute FRA Speed Restriction & Remedial Timeframe | Request Supervisor Tag-Out Approval, Shape Non-Urgent Record | Requires Immediate Tag-Out? |
| Shape Non-Urgent Record | set | Marks approval_outcome='not_required' for non-Class-5 segments. Skips the approval gate entirely. Still fully archived and reported like every other outcome. | Requires Immediate Tag-Out? | Archive Defect Tag-Out Record | Shape Non-Urgent Record |
| Request Supervisor Tag-Out Approval | slack | Slack send-and-wait: approve / edit / reject the recommended tag-out. This workflow's control-flow floor. No Class 5 segment gets tagged out without this gate. | Requires Immediate Tag-Out? | Route Approval Response | Request Supervisor Tag-Out Approval |
| Route Approval Response | switch | 3-way Switch reading the Slack response payload. Outputs: approve / edit / reject. Second of the two required real branch points. | Request Supervisor Tag-Out Approval | Finalize Approved Tag-Out, Finalize Edited Tag-Out, Cancel Tag-Out (Rejected) | Route Approval Response |
| Finalize Approved Tag-Out | set | Stamps final_status as approved-as-recommended. approval_outcome='approved'. Reached only on a clean Slack approval. | Route Approval Response | Archive Defect Tag-Out Record | Finalize Approved Tag-Out |
| Finalize Edited Tag-Out | set | Uses the supervisor's own typed text as final_status. approval_outcome='approved_with_edit'. Lets a human override the exact restriction on the record. | Route Approval Response | Archive Defect Tag-Out Record | Finalize Edited Tag-Out |
| Cancel Tag-Out (Rejected) | set | Marks the tag-out rejected, sends it back for a field re-check. approval_outcome='rejected'. Still archived — a rejection is not a dropped record. | Route Approval Response | Archive Defect Tag-Out Record | Cancel Tag-Out (Rejected) |
| Archive Defect Tag-Out Record | googleDrive | Uploads the final record to Google Drive, one file per segment/day. Runs regardless of which outcome path was taken. Primary long-term audit sink. | Shape Non-Urgent Record, Finalize Approved Tag-Out, Finalize Edited Tag-Out, Cancel Tag-Out (Rejected) | Notify Track Supervisor & Dispatch | Archive Defect Tag-Out Record |
| Notify Track Supervisor & Dispatch | slack | Posts the final category, action, and speed restriction to Slack. Last node — closes the loop for dispatch. Same message shape across every outcome. | Archive Defect Tag-Out Record | None | Notify Track Supervisor & Dispatch |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Configure the rule to trigger daily at hour `5`.
   - Name the node: `Poll Segments Due for Inspection`.

2. **Add Context Normalization:**
   - Add a **Set** node (`n8n-nodes-base.set`).
   - Connect `Poll Segments Due for Inspection` output to this node.
   - Configure assignments:
     - Field: `poll_date`, Type: `string`, Value: `={{ new Date().toISOString().slice(0,10) }}`
     - Field: `railroad_id`, Type: `string`, Value: `SHORTLINE-01`
   - Name the node: `Normalize Poll Window`.

3. **Configure the Data Fetch:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Connect `Normalize Poll Window` output to this node.
   - Set Method to `GET` and URL to `=https://track-inspection.example-shortline.com/api/segments/due?date={{ $json.poll_date }}`.
   - Enable options: Split Into Items (`splitIntoItems: true`).
   - Configure error handling: Enable `retryOnFail`, set max tries to `3`, wait between tries to `2000`ms, and set `onError` to `continueRegularOutput`.
   - Set credential: Create or select an HTTP Header Auth credential named `Track Inspection System API Key`.
   - Name the node: `Fetch Segments Due & Inspector Notes`.

4. **Set Up the AI Classification Model:**
   - Add an **OpenAI Chat Model** node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`).
   - Configure model to `gpt-4.1-mini`, temperature to `0.1`.
   - Set credential: Create or select an OpenAI API credential named `OpenAI account`.
   - Name the node: `Classifier Model`.

5. **Configure the Text Classifier:**
   - Add a **Text Classifier** node (`@n8n/n8n-nodes-langchain.textClassifier`).
   - Connect `Fetch Segments Due & Inspector Notes` to the main input and `Classifier Model` to the AI language model input.
   - Set `inputText` expression to `={{ $json.inspector_defect_notes }}`.
   - Define four categories:
     - `No Defect`: "No geometry, rail, or structural defect noted."
     - `Class 1-2 Minor`: "Minor FRA Class 1-2 defect: monitor, not requiring immediate action."
     - `Class 3-4 Serious`: "Serious FRA Class 3-4 defect: requires a speed restriction and remedial action within a defined timeframe."
     - `Class 5 Immediate Halt`: "Severe defect requiring immediate removal of the segment from service (tag-out)."
   - Name the node: `Classify Defect Severity`.

6. **Create Category Tagging Nodes:**
   - Add four **Set** nodes (`n8n-nodes-base.set`) corresponding to each classification output branch:
     - **Node 1:** Name `Tag No Defect`. Assign `defect_category` = `No Defect`.
     - **Node 2:** Name `Tag Minor`. Assign `defect_category` = `Class 1-2 Minor`.
     - **Node 3:** Name `Tag Serious`. Assign `defect_category` = `Class 3-4 Serious`.
     - **Node 4:** Name `Tag Immediate Halt`. Assign `defect_category` = `Class 5 Immediate Halt`.
   - Connect each respective branch output from `Classify Defect Severity` into its corresponding Tag node.

7. **Implement Calculation Logic:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Connect all four Tag node outputs into this single Code node.
   - Insert the following JavaScript into the code parameter:
     ```javascript
     const j = $json;
     const trackClass = Number(j.track_class || 3);
     const classMaxSpeed = { 1: 10, 2: 25, 3: 40, 4: 60, 5: 80 };
     const normalMax = classMaxSpeed[trackClass] || 40;
     let speed_restriction_mph, remedial_days, action;
     switch (j.defect_category) {
       case 'Class 5 Immediate Halt':
         speed_restriction_mph = 0; remedial_days = 0; action = 'Tag out - remove segment from service immediately'; break;
       case 'Class 3-4 Serious':
         speed_restriction_mph = Math.max(10, normalMax - 20); remedial_days = 30; action = 'Impose slow order, schedule remedial repair within 30 days'; break;
       case 'Class 1-2 Minor':
         speed_restriction_mph = normalMax; remedial_days = 180; action = 'Monitor, schedule remedial repair within 180 days'; break;
       default:
         speed_restriction_mph = normalMax; remedial_days = null; action = 'No action - normal operations'; break;
     }
     return { json: { ...j, normal_max_speed_mph: normalMax, speed_restriction_mph, remedial_days, recommended_action: action } };
     ```
   - Name the node: `Compute FRA Speed Restriction & Remedial Timeframe`.

8. **Establish the Approval Branching Gate:**
   - Add an **If** node (`n8n-nodes-base.if`).
   - Connect `Compute FRA Speed Restriction & Remedial Timeframe` to this node.
   - Set condition: Left value `={{ $json.defect_category }}`, operator `equals`, Right value `Class 5 Immediate Halt`.
   - Name the node: `Requires Immediate Tag-Out?`.

9. **Configure Non-Urgent Path:**
   - Add a **Set** node (`n8n-nodes-base.set`).
   - Connect the `false` (downward) output of `Requires Immediate Tag-Out?` to this node.
   - Assign `final_status` = `={{ $json.recommended_action }}` and `approval_outcome` = `not_required`.
   - Name the node: `Shape Non-Urgent Record`.

10. **Configure Slack Approval Request:**
    - Add a **Slack** node (`n8n-nodes-base.slack`).
    - Connect the `true` (upward) output of `Requires Immediate Tag-Out?` to this node.
    - Set resource to `message`, operation to `sendAndWait`.
    - Set channel selection to `channel`, channel ID to your Slack channel ID (e.g., `REPLACE_WITH_TRACK_OPS_CHANNEL_ID`).
    - Set response type to `approval` with approval options type `approveAndEdit`.
    - Set message text: `=Segment {{ $json.segment_id }} flagged Class 5 (Immediate Halt). Recommended action: {{ $json.recommended_action }}. Approve tag-out, edit the speed restriction, or reject based on field re-check?`
    - Configure error handling with retries (`onError: continueRegularOutput`).
    - Set credential: Create or select a Slack API credential named `Track Ops Slack Bot`.
    - Name the node: `Request Supervisor Tag-Out Approval`.

11. **Route Approval Responses:**
    - Add a **Switch** node (`n8n-nodes-base.switch`).
    - Connect `Request Supervisor Tag-Out Approval` to this node.
    - Define rules:
      - **Output 1 (approve):** Condition `{{ $json.data.approved }}` equals `true` AND `{{ $json.data.text }}` equals `""`.
      - **Output 2 (edit):** Condition `{{ $json.data.approved }}` equals `true` AND `{{ $json.data.text }}` does not equal `""`.
      - **Output 3 (reject):** Condition `{{ $json.data.approved }}` equals `false`.
    - Set fallback output to reject.
    - Name the node: `Route Approval Response`.

12. **Create Finalization Nodes:**
    - Add three **Set** nodes (`n8n-nodes-base.set`):
      - **Node 1:** Name `Finalize Approved Tag-Out`. Assign `final_status` = `Tag-out approved as recommended`, `approval_outcome` = `approved`. Connect to Switch output 1.
      - **Node 2:** Name `Finalize Edited Tag-Out`. Assign `final_status` = `=Tag-out approved with supervisor edit: {{ $json.data.text }}`, `approval_outcome` = `approved_with_edit`. Connect to Switch output 2.
      - **Node 3:** Name `Cancel Tag-Out (Rejected)`. Assign `final_status` = `Tag-out rejected by supervisor - field re-check required`, `approval_outcome` = `rejected`. Connect to Switch output 3 and fallback.

13. **Configure Archiving:**
    - Add a **Google Drive** node (`n8n-nodes-base.googleDrive`).
    - Connect the outputs of `Shape Non-Urgent Record`, `Finalize Approved Tag-Out`, `Finalize Edited Tag-Out`, and `Cancel Tag-Out (Rejected)` into this node.
    - Set operation to `upload`.
    - Set file name to `=track-defect-{{ $json.segment_id }}-{{ $json.poll_date }}.json`.
    - Configure Drive ID and Folder ID parameters.
    - Configure error handling with retries (`onError: continueRegularOutput`).
    - Set credential: Create or select a Google Drive OAuth2 credential named `Track Compliance Google Drive account`.
    - Name the node: `Archive Defect Tag-Out Record`.

14. **Configure Final Notification:**
    - Add a **Slack** node (`n8n-nodes-base.slack`).
    - Connect `Archive Defect Tag-Out Record` to this node.
    - Set resource to `message`, operation to `post`.
    - Set channel ID and message text: `=Segment {{ $json.segment_id }}: {{ $json.defect_category }} -> {{ $json.final_status }} (speed restriction {{ $json.speed_restriction_mph }} mph)`.
    - Configure error handling with retries (`onError: continueRegularOutput`).
    - Set credential: Use the `Track Ops Slack Bot` credential.
    - Name the node: `Notify Track Supervisor & Dispatch`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Creator Credit & Portfolio | Swapnil AI Labs — [swapnilailabs.netlify.app](https://swapnilailabs.netlify.app) — `swapnil.mandloi7@gmail.com` |
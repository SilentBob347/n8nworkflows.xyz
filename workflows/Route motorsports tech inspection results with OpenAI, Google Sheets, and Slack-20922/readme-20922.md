Route motorsports tech inspection results with OpenAI, Google Sheets, and Slack

https://n8nworkflows.xyz/workflows/route-motorsports-tech-inspection-results-with-openai--google-sheets--and-slack-20922


# Route motorsports tech inspection results with OpenAI, Google Sheets, and Slack

### 1. Workflow Overview

This workflow automates the processing and routing of motorsports technical inspection submissions for trackside operations. Its primary purpose is to evaluate safety-equipment items against class-specific expiry cycles, classify the car's overall inspection status using AI combined with inspector notes, interact with a timing API for race-control decisions, archive records in Google Sheets, and alert the relevant personnel via Slack.

The workflow logic is grouped into the following functional blocks:
- **1.1 Input Reception & Equipment Fan-Out:** Receives inspection data via webhook and splits the equipment array for parallel evaluation.
- **1.2 Item Scoring & Metadata Tagging:** Computes days-to-expiry for each piece of safety equipment and appends class metadata, then merges and aggregates the results.
- **1.3 AI Classification & Category Tagging:** Utilizes an OpenAI text classification model to evaluate compliance and routes the output through category-stamping nodes.
- **1.4 Categorized Routing & Terminal Actions:** Directs the execution through a 4-way switch based on the inspection status, performing conditional checks, timing API postings, and escalations.
- **1.5 Archival & Notifications:** Logs final results to Google Sheets and sends notifications to the designated Slack channels.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Equipment Fan-Out
- **Overview:** Receives the incoming JSON payload from the trackside inspection application and breaks down the safety equipment array into individual line items for parallel processing.
- **Nodes Involved:** 
  - `Receive Car Inspection Submission`
  - `Split Equipment Line Items`

- **Node Details:**
  - **Receive Car Inspection Submission**
    - *Type and Technical Role:* Webhook Trigger (`n8n-nodes-base.webhook`). Listens for incoming HTTP POST requests containing driver, car, and equipment details.
    - *Configuration Choices:* Path set to `tech-inspection-submission`, method set to `POST`, response mode set to `onReceived`.
    - *Key Expressions or Variables:* None (entry point).
    - *Input and Output Connections:* Input: External HTTP POST. Output: `Split Equipment Line Items`.
    - *Version-specific Requirements:* Type Version 2.
    - *Edge Cases / Potential Failure Types:* Malformed JSON payloads or network drops from the trackside app.
    - *Sub-workflow Reference:* None.

  - **Split Equipment Line Items**
    - *Type and Technical Role:* Split Out (`n8n-nodes-base.splitOut`). Expands the array of safety equipment items into individual items for independent evaluation.
    - *Configuration Choices:* Field to split out configured as `equipmentItems`.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: `Receive Car Inspection Submission`. Output: Feeds simultaneously into `Compute Item Expiry & Pass Fail` and `Tag Equipment Class Metadata`.
    - *Version-specific Requirements:* Type Version 1.
    - *Edge Cases / Potential Failure Types:* Missing or empty `equipmentItems` array.
    - *Sub-workflow Reference:* None.

---

#### 2.2 Item Scoring & Metadata Tagging
- **Overview:** Evaluates the expiry date of each equipment item against defined category cycles in parallel with attaching human-readable class metadata, subsequently merging and aggregating the records.
- **Nodes Involved:**
  - `Compute Item Expiry & Pass Fail`
  - `Tag Equipment Class Metadata`
  - `Merge Equipment Score Branches`
  - `Aggregate Equipment Compliance Score`

- **Node Details:**
  - **Compute Item Expiry & Pass Fail**
    - *Type and Technical Role:* Code Node (`n8n-nodes-base.code`). Executes custom JavaScript to calculate days-to-expiry and pass/fail status based on equipment class rules (e.g., helmets 5 years, harnesses 2 years).
    - *Configuration Choices:* Language: JavaScript. Custom logic evaluates `equipmentClass` against predefined day cycles.
    - *Key Expressions or Variables:* Uses `$input.first().json`, `item.equipmentClass`, and `item.issueOrCertDate`.
    - *Input and Output Connections:* Input: `Split Equipment Line Items`. Output: `Merge Equipment Score Branches` (Input Index 0).
    - *Version-specific Requirements:* Type Version 2.
    - *Edge Cases / Potential Failure Types:* Invalid date strings causing `NaN` calculations.
    - *Sub-workflow Reference:* None.

  - **Tag Equipment Class Metadata**
    - *Type and Technical Role:* Set Node (`n8n-nodes-base.set`). Appends a formatted string combining the equipment class and rating.
    - *Configuration Choices:* Assignment mode: string assignment.
    - *Key Expressions or Variables:* `={{ $json.equipmentClass }} ({{ $json.rating }})` assigned to `equipmentClassLabel`.
    - *Input and Output Connections:* Input: `Split Equipment Line Items`. Output: `Merge Equipment Score Branches` (Input Index 1).
    - *Version-specific Requirements:* Type Version 3.4.
    - *Edge Cases / Potential Failure Types:* Missing property fields in the input data.
    - *Sub-workflow Reference:* None.

  - **Merge Equipment Score Branches**
    - *Type and Technical Role:* Merge Node (`n8n-nodes-base.merge`). Recombines the scored branch with the metadata branch.
    - *Configuration Choices:* Mode: `combine`, Combine by: `combineByPosition`.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input 0: `Compute Item Expiry & Pass Fail`, Input 1: `Tag Equipment Class Metadata`. Output: `Aggregate Equipment Compliance Score`.
    - *Version-specific Requirements:* Type Version 3.2.
    - *Edge Cases / Potential Failure Types:* Mismatched array lengths if items drop out.
    - *Sub-workflow Reference:* None.

  - **Aggregate Equipment Compliance Score**
    - *Type and Technical Role:* Aggregate Node (`n8n-nodes-base.aggregate`). Collects all evaluated items back into a single data structure representing the vehicle's overall equipment state.
    - *Configuration Choices:* Aggregate operation set to `aggregateAllItemData`.
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Input: `Merge Equipment Score Branches`. Output: `Classify Overall Inspection Result`.
    - *Version-specific Requirements:* Type Version 1.
    - *Edge Cases / Potential Failure Types:* Empty dataset aggregation.
    - *Sub-workflow Reference:* None.

---

#### 2.3 AI Classification & Category Tagging
- **Overview:** Passes aggregated compliance data and inspector notes to an OpenAI text classification model, then assigns explicit inspection category flags based on the classification branch.
- **Nodes Involved:**
  - `Inspection Classification Model`
  - `Classify Overall Inspection Result`
  - `Tag Pass Category`
  - `Tag Conditional Category`
  - `Tag Fail Category`
  - `Tag Disqualify Category`

- **Node Details:**
  - **Inspection Classification Model**
    - *Type and Technical Role:* OpenAI Chat Model Sub-node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`). Provides the language model backend for the text classifier.
    - *Configuration Choices:* Model selected: `gpt-4.1-mini`, Temperature: `0`.
    - *Key Expressions or Variables:* Uses credential `OpenAI Account - Tech Inspection Classifier`.
    - *Input and Output Connections:* Output: Connects to `Classify Overall Inspection Result` via `ai_languageModel`.
    - *Version-specific Requirements:* Type Version 1.2.
    - *Edge Cases / Potential Failure Types:* API rate limits, authentication failures, or service outages. Retries up to 3 times with a 2-second wait (`continueRegularOutput` enabled).
    - *Sub-workflow Reference:* None.

  - **Classify Overall Inspection Result**
    - *Type and Technical Role:* Text Classifier Node (`@n8n/n8n-nodes-langchain.textClassifier`). Evaluates the inspection details against four distinct categories: Pass, Conditional-Pass-Rework, Fail, and Disqualify.
    - *Configuration Choices:* Categories defined with descriptions. Error handling set to continue regular output with 3 retries.
    - *KeyExpressions or Variables:* 
      ```javascript
      ={{ 'Equipment compliance: ' + $json.data.map(d => d.equipmentClassLabel + ': ' + (d.itemPass ? 'PASS' : 'FAIL') + ' (' + d.daysToExpiry + 'd)').join('; ') + '\nInspector notes: ' + $json.inspectorNotes }}
      ```
    - *Input and Output Connections:* Input: `Aggregate Equipment Compliance Score`. Outputs: Four separate branches connected respectively to `Tag Pass Category`, `Tag Conditional Category`, `Tag Fail Category`, and `Tag Disqualify Category`.
    - *Version-specific Requirements:* Type Version 1.
    - *Edge Cases / Potential Failure Types:* Unexpected classification outputs or LLM timeouts.
    - *Sub-workflow Reference:* None.

  - **Tag Pass Category** / **Tag Conditional Category** / **Tag Fail Category** / **Tag Disqualify Category**
    - *Type and Technical Role:* Set Nodes (`n8n-nodes-base.set`). Stamps the respective string value (`Pass`, `Conditional-Pass-Rework`, `Fail`, or `Disqualify`) into the `inspectionCategory` field.
    - *Configuration Choices:* Assignment mode: string. Field name: `inspectionCategory`.
    - *Key Expressions or Variables:* Hardcoded string values matching each specific branch.
    - *Input and Output Connections:* Input: Respective output branch from `Classify Overall Inspection Result`. Output: All four feed into `Route By Inspection Category`.
    - *Version-specific Requirements:* Type Version 3.4.
    - *Edge Cases / Potential Failure Types:* None.
    - *Sub-workflow Reference:* None.

---

#### 2.4 Categorized Routing & Terminal Actions
- **Overview:** Switches workflow execution based on the assigned inspection category, evaluating grid-eligibility cutoffs or posting relevant status updates to the race-control timing system.
- **Nodes Involved:**
  - `Route By Inspection Category`
  - `Check Grid Eligibility Cutoff Time`
  - `Post Grid-Eligible Result`
  - `Post Rework Deadline & Reinspection Slot`
  - `Post Grid Pull & Fail Notification`
  - `Post Disqualification Escalation`

- **Node Details:**
  - **Route By Inspection Category**
    - *Type and Technical Role:* Switch Node (`n8n-nodes-base.switch`). Directs items down one of four execution paths based on the evaluated category string.
    - *Configuration Choices:* Mode: expression. Fallback output set to `-1`.
    - *Key Expressions or Variables:* 
      ```javascript
      ={{ ['Pass', 'Conditional-Pass-Rework', 'Fail', 'Disqualify'].indexOf($json.inspectionCategory) }}
      ```
    - *Input and Output Connections:* Input: All four tag nodes. Outputs: Output 0 to `Check Grid Eligibility Cutoff Time`, Output 1 to `Post Rework Deadline & Reinspection Slot`, Output 2 to `Post Grid Pull & Fail Notification`, Output 3 to `Post Disqualification Escalation`.
    - *Version-specific Requirements:* Type Version 3.2.
    - *Edge Cases / Potential Failure Types:* Unrecognized category strings yielding a fallback index.
    - *Sub-workflow Reference:* None.

  - **Check Grid Eligibility Cutoff Time**
    - *Type and Technical Role:* If Node (`n8n-nodes-base.if`). Evaluates whether the current submission time precedes the allowed grid-position eligibility cutoff.
    - *Configuration Choices:* Condition type validation: strict, operator: number less than (`lt`).
    - *Key Expressions or Variables:* Left value: `={{ $now.toMillis() }}`, Right value: `={{ Date.parse($json.gridEligibilityCutoff) }}`.
    - *Input and Output Connections:* Input: `Route By Inspection Category` (Output 0). Outputs: Both True and False branches connect to `Post Grid-Eligible Result`.
    - *Version-specific Requirements:* Type Version 2.2.
    - *Edge Cases / Potential Failure Types:* Missing or malformed `gridEligibilityCutoff` timestamp string.
    - *Sub-workflow Reference:* None.

  - **Post Grid-Eligible Result**
    - *Type and Technical Role:* HTTP Request Node (`n8n-nodes-base.httpRequest`). Posts the final PASS inspection result and eligibility status to the race-control timing system.
    - *Configuration Choices:* Method: POST, URL: `https://timing.internal.example.com/api/inspection-results`, Send body as JSON. Uses `Race Control Timing API - Header Auth`. Error handling set to continue on failure with 3 retries.
    - *Key Expressions or Variables:* 
      ```javascript
      ={{ JSON.stringify({ carNumber: $json.carNumber, result: 'PASS', gridEligible: $now.toMillis() < Date.parse($json.gridEligibilityCutoff) }) }}
      ```
    - *Input and Output Connections:* Input: `Check Grid Eligibility Cutoff Time`. Output: `Log Inspection Result Archive`.
    - *Version-specific Requirements:* Type Version 4.2.
    - *Edge Cases / Potential Failure Types:* API endpoint timeouts or authentication failures.
    - *Sub-workflow Reference:* None.

  - **Post Rework Deadline & Reinspection Slot**
    - *Type and Technical Role:* HTTP Request Node (`n8n-nodes-base.httpRequest`). Sends conditional pass data, rework deadlines, and reinspection slots to the timing API.
    - *Configuration Choices:* Method: POST, URL: `https://timing.internal.example.com/api/inspection-results`, Send JSON body. Uses Header Auth.
    - *Key Expressions or Variables:* 
      ```javascript
      ={{ JSON.stringify({ carNumber: $json.carNumber, result: 'CONDITIONAL_PASS_REWORK', reworkDeadline: $json.reworkDeadline, reinspectionSlot: $json.reinspectionSlot }) }}
      ```
    - *Input and Output Connections:* Input: `Route By Inspection Category` (Output 1). Output: `Log Inspection Result Archive`.
    - *Version-specific Requirements:* Type Version 4.2.
    - *Edge Cases / Potential Failure Types:* Network timeout or API error.
    - *Sub-workflow Reference:* None.

  - **Post Grid Pull & Fail Notification**
    - *Type and Technical Role:* HTTP Request Node (`n8n-nodes-base.httpRequest`). Submits a failure notification and grid pull action to the timing system.
    - *Configuration Choices:* Method: POST, URL endpoint, JSON body payload. Uses Header Auth.
    - *Key Expressions or Variables:* 
      ```javascript
      ={{ JSON.stringify({ carNumber: $json.carNumber, result: 'FAIL', action: 'PULLED_FROM_GRID' }) }}
      ```
    - *Input and Output Connections:* Input: `Route By Inspection Category` (Output 2). Output: `Log Inspection Result Archive`.
    - *Version-specific Requirements:* Type Version 4.2.
    - *Edge Cases / Potential Failure Types:* Network failure.
    - *Sub-workflow Reference:* None.

  - **Post Disqualification Escalation**
    - *Type and Technical Role:* HTTP Request Node (`n8n-nodes-base.httpRequest`). Submits a disqualification record to the race-control timing system.
    - *Configuration Choices:* Method: POST, URL endpoint, JSON body payload. Uses Header Auth.
    - *Key Expressions or Variables:* 
      ```javascript
      ={{ JSON.stringify({ carNumber: $json.carNumber, result: 'DISQUALIFY', escalatedTo: 'CHIEF_STEWARD' }) }}
      ```
    - *Input and Output Connections:* Input: `Route By Inspection Category` (Output 3). Output: `Escalate Disqualification To Chief Steward`.
    - *Version-specific Requirements:* Type Version 4.2.
    - *Edge Cases / Potential Failure Types:* API endpoint errors.
    - *Sub-workflow Reference:* None.

---

#### 2.5 Archival & Notifications
- **Overview:** Logs all final inspection results into a Google Sheets archive and sends alerts to the appropriate Slack channels (including chief steward escalation for disqualifications).
- **Nodes Involved:**
  - `Escalate Disqualification To Chief Steward`
  - `Log Inspection Result Archive`
  - `Notify Tech Inspection Crew Channel`

- **Node Details:**
  - **Escalate Disqualification To Chief Steward**
    - *Type and Technical Role:* Slack Node (`n8n-nodes-base.slack`). Sends an urgent notification message to the chief steward channel upon a disqualification event.
    - *Configuration Choices:* Select: channel, Channel ID: `PLACEHOLDER_CHIEF_STEWARD_CHANNEL`. Uses `Slack Account - Chief Steward Escalation`.
    - *Key Expressions or Variables:* 
      ```javascript
      ={{ 'DISQUALIFIED: Car ' + $json.carNumber + ' - ' + $json.inspectorNotes }}
      ```
    - *Input and Output Connections:* Input: `Post Disqualification Escalation`. Output: `Log Inspection Result Archive`.
    - *Version-specific Requirements:* Type Version 2.2.
    - *Edge Cases / Potential Failure Types:* Invalid channel ID or Slack API permission issues.
    - *Sub-workflow Reference:* None.

  - **Log Inspection Result Archive**
    - *Type and Technical Role:* Google Sheets Node (`n8n-nodes-base.googleSheets`). Appends a row containing the inspection results into the archival spreadsheet.
    - *Configuration Choices:* Operation: `append`, Sheet Name: `Tech_Inspection_Archive`, Document ID: `PLACEHOLDER_SHEET_ID`. Uses `Google Sheets Account - Tech Inspection Archive`.
    - *Key Expressions or Variables:* None (maps incoming JSON data directly to columns).
    - *Input and Output Connections:* Input: Fed by all terminal paths (`Post Grid-Eligible Result`, `Post Rework Deadline & Reinspection Slot`, `Post Grid Pull & Fail Notification`, and `Escalate Disqualification To Chief Steward`). Output: `Notify Tech Inspection Crew Channel`.
    - *Version-specific Requirements:* Type Version 4.5.
    - *Edge Cases / Potential Failure Types:* Sheet name mismatch or API rate limits.
    - *Sub-workflow Reference:* None.

  - **Notify Tech Inspection Crew Channel**
    - *Type and Technical Role:* Slack Node (`n8n-nodes-base.slack`). Sends a notification message regarding the logged inspection result to the tech crew channel.
    - *Configuration Choices:* Select: channel, Channel ID: `PLACEHOLDER_TECH_CREW_CHANNEL`. Uses `Slack Account - Tech Inspection Crew`.
    - *Key Expressions or Variables:* 
      ```javascript
      ={{ 'Car ' + $json.carNumber + ': ' + $json.inspectionCategory + ' logged to race control.' }}
      ```
    - *Input and Output Connections:* Input: `Log Inspection Result Archive`. Output: None (Terminal node).
    - *Version-specific Requirements:* Type Version 2.2.
    - *Edge Cases / Potential Failure Types:* Channel ID errors or rate limiting.
    - *Sub-workflow Reference:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Panel: WORKFLOW DETAILS | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | ## 📘 WORKFLOW DETAILS... |
| Panel: SETUP GUIDE | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | ## ⚙️ SETUP GUIDE... |
| Panel: CUSTOMISATION | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | ## 🎨 CUSTOMISATION... |
| Panel: ADDITIONAL INFORMATION | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | ## ℹ️ ADDITIONAL INFORMATION... |
| Note: Receive Car Inspection Submission | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Receive Car Inspection Submission**... |
| Note: Split Equipment Line Items | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Split Equipment Line Items**... |
| Note: Compute Item Expiry & Pass Fail | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Compute Item Expiry & Pass Fail**... |
| Note: Tag Equipment Class Metadata | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Tag Equipment Class Metadata**... |
| Note: Merge Equipment Score Branches | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Merge Equipment Score Branches**... |
| Note: Aggregate Equipment Compliance Score | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Aggregate Equipment Compliance Score**... |
| Note: Classify Overall Inspection Result | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Classify Overall Inspection Result**... |
| Note: Inspection Classification Model | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Inspection Classification Model**... |
| Note: Tag Pass Category | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Tag Pass Category**... |
| Note: Tag Conditional Category | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Tag Conditional Category**... |
| Note: Tag Fail Category | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Tag Fail Category**... |
| Note: Tag Disqualify Category | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Tag Disqualify Category**... |
| Note: Route By Inspection Category | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Route By Inspection Category**... |
| Note: Check Grid Eligibility Cutoff Time | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Check Grid Eligibility Cutoff Time**... |
| Note: Post Rework Deadline & Reinspection Slot | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Post Rework Deadline & Reinspection Slot**... |
| Note: Post Grid Pull & Fail Notification | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Post Grid Pull & Fail Notification**... |
| Note: Post Disqualification Escalation | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Post Disqualification Escalation**... |
| Note: Post Grid-Eligible Result | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Post Grid-Eligible Result**... |
| Note: Escalate Disqualification To Chief Steward | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Escalate Disqualification To Chief Steward**... |
| Note: Log Inspection Result Archive | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Log Inspection Result Archive**... |
| Note: Notify Tech Inspection Crew Channel | `n8n-nodes-base.stickyNote` | Documentation panel | None | None | **Notify Tech Inspection Crew Channel**... |
| Receive Car Inspection Submission | `n8n-nodes-base.webhook` | Webhook Trigger | External POST | Split Equipment Line Items | |
| Split Equipment Line Items | `n8n-nodes-base.splitOut` | Fan-out equipment array | Receive Car Inspection Submission | Compute Item Expiry & Pass Fail, Tag Equipment Class Metadata | |
| Compute Item Expiry & Pass Fail | `n8n-nodes-base.code` | Calculate expiry days | Split Equipment Line Items | Merge Equipment Score Branches | |
| Tag Equipment Class Metadata | `n8n-nodes-base.set` | Append class/rating label | Split Equipment Line Items | Merge Equipment Score Branches | |
| Merge Equipment Score Branches | `n8n-nodes-base.merge` | Recombine scored branches | Compute Item Expiry & Pass Fail, Tag Equipment Class Metadata | Aggregate Equipment Compliance Score | |
| Aggregate Equipment Compliance Score | `n8n-nodes-base.aggregate` | Aggregate item data | Merge Equipment Score Branches | Classify Overall Inspection Result | |
| Classify Overall Inspection Result | `@n8n/n8n-nodes-langchain.textClassifier` | AI Text Classification | Aggregate Equipment Compliance Score | Tag Pass Category, Tag Conditional Category, Tag Fail Category, Tag Disqualify Category | |
| Inspection Classification Model | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | OpenAI LLM Provider | None | Classify Overall Inspection Result | |
| Tag Pass Category | `n8n-nodes-base.set` | Stamp Pass category | Classify Overall Inspection Result | Route By Inspection Category | |
| Tag Conditional Category | `n8n-nodes-base.set` | Stamp Conditional category | Classify Overall Inspection Result | Route By Inspection Category | |
| Tag Fail Category | `n8n-nodes-base.set` | Stamp Fail category | Classify Overall Inspection Result | Route By Inspection Category | |
| Tag Disqualify Category | `n8n-nodes-base.set` | Stamp Disqualify category | Classify Overall Inspection Result | Route By Inspection Category | |
| Route By Inspection Category | `n8n-nodes-base.switch` | 4-way routing switch | Tag Pass Category, Tag Conditional Category, Tag Fail Category, Tag Disqualify Category | Check Grid Eligibility Cutoff Time, Post Rework Deadline & Reinspection Slot, Post Grid Pull & Fail Notification, Post Disqualification Escalation | |
| Check Grid Eligibility Cutoff Time | `n8n-nodes-base.if` | Check grid cutoff time | Route By Inspection Category | Post Grid-Eligible Result, Post Grid-Eligible Result | |
| Post Rework Deadline & Reinspection Slot | `n8n-nodes-base.httpRequest` | Post rework deadline | Route By Inspection Category | Log Inspection Result Archive | |
| Post Grid Pull & Fail Notification | `n8n-nodes-base.httpRequest` | Post fail result | Route By Inspection Category | Log Inspection Result Archive | |
| Post Disqualification Escalation | `n8n-nodes-base.httpRequest` | Post DQ result | Route By Inspection Category | Escalate Disqualification To Chief Steward | |
| Post Grid-Eligible Result | `n8n-nodes-base.httpRequest` | Post pass result | Check Grid Eligibility Cutoff Time | Log Inspection Result Archive | |
| Escalate Disqualification To Chief Steward | `n8n-nodes-base.slack` | Slack chief steward alert | Post Disqualification Escalation | Log Inspection Result Archive | |
| Log Inspection Result Archive | `n8n-nodes-base.googleSheets` | Google Sheets archive log | Post Grid-Eligible Result, Post Rework Deadline & Reinspection Slot, Post Grid Pull & Fail Notification, Escalate Disqualification To Chief Steward | Notify Tech Inspection Crew Channel | |
| Notify Tech Inspection Crew Channel | `n8n-nodes-base.slack` | Slack tech crew notification | Log Inspection Result Archive | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node named `Receive Car Inspection Submission`.
   - Set HTTP Method to `POST`, Path to `tech-inspection-submission`, and Response Mode to `onReceived`.
2. **Add the Split Node:**
   - Add a **Split Out** node named `Split Equipment Line Items`.
   - Set `Field To Split Out` to `equipmentItems`.
   - Connect `Receive Car Inspection Submission` output to this node.
3. **Build the Expiry Calculation Branch:**
   - Add a **Code** node named `Compute Item Expiry & Pass Fail`.
   - Insert JavaScript code to iterate items, compute days to expiry based on equipment class cycles (helmet, harness, rollcage, firesuit), and return `itemPass`.
   - Connect the first output of `Split Equipment Line Items` to this node.
4. **Build the Metadata Tagging Branch:**
   - Add a **Set** node named `Tag Equipment Class Metadata`.
   - Assign a new string variable `equipmentClassLabel` equal to `={{ $json.equipmentClass }} ({{ $json.rating }})`.
   - Connect the second output of `Split Equipment Line Items` to this node.
5. **Merge and Aggregate:**
   - Add a **Merge** node named `Merge Equipment Score Branches`. Set mode to `combine` by position.
   - Connect `Compute Item Expiry & Pass Fail` to Input 0 and `Tag Equipment Class Metadata` to Input 1.
   - Add an **Aggregate** node named `Aggregate Equipment Compliance Score` set to `aggregateAllItemData`. Connect the merge node output to it.
6. **Set up AI Classification:**
   - Add an **OpenAI Chat Model** node named `Inspection Classification Model`. Set model to `gpt-4.1-mini`, temperature to `0`, and attach credential `OpenAI Account - Tech Inspection Classifier`.
   - Add a **Text Classifier** node named `Classify Overall Inspection Result`. Connect the OpenAI model to its `ai_languageModel` input.
   - Configure input text to map aggregated compliance data and inspector notes, and define categories: `Pass`, `Conditional-Pass-Rework`, `Fail`, and `Disqualify`. Connect `Aggregate Equipment Compliance Score` to this node.
7. **Add Category Stamping Nodes:**
   - Create four **Set** nodes named `Tag Pass Category`, `Tag Conditional Category`, `Tag Fail Category`, and `Tag Disqualify Category`.
   - Set the `inspectionCategory` field to `Pass`, `Conditional-Pass-Rework`, `Fail`, and `Disqualify` respectively.
   - Connect each corresponding output branch of the Text Classifier to its respective Tag node.
8. **Configure the Routing Switch:**
   - Add a **Switch** node named `Route By Inspection Category`. Set mode to expression using: `={{ ['Pass', 'Conditional-Pass-Rework', 'Fail', 'Disqualify'].indexOf($json.inspectionCategory) }}`.
   - Connect all four Tag nodes to this Switch node.
9. **Build the Pass Path & Cutoff Check:**
   - Add an **If** node named `Check Grid Eligibility Cutoff Time` connected to Output 0 of the Switch. Set condition to check if current time is less than `gridEligibilityCutoff`.
   - Add an **HTTP Request** node named `Post Grid-Eligible Result` (URL: `https://timing.internal.example.com/api/inspection-results`, Method: `POST`, Header Auth credential). Connect both True and False outputs of the If node to this request node.
10. **Build Conditional, Fail, and Disqualify Paths:**
    - Add an **HTTP Request** node named `Post Rework Deadline & Reinspection Slot` connected to Switch Output 1.
    - Add an **HTTP Request** node named `Post Grid Pull & Fail Notification` connected to Switch Output 2.
    - Add an **HTTP Request** node named `Post Disqualification Escalation` connected to Switch Output 3.
11. **Add Escalation and Archival Nodes:**
    - Add a **Slack** node named `Escalate Disqualification To Chief Steward` connected to the Disqualification HTTP request node, using a Chief Steward Slack account credential and channel ID.
    - Add a **Google Sheets** node named `Log Inspection Result Archive` (Operation: `append`, Sheet: `Tech_Inspection_Archive`, Document ID: `PLACEHOLDER_SHEET_ID`, Google Sheets OAuth2 credential). Connect outputs from `Post Grid-Eligible Result`, `Post Rework Deadline & Reinspection Slot`, `Post Grid Pull & Fail Notification`, and `Escalate Disqualification To Chief Steward` into this archive node.
    - Add a **Slack** node named `Notify Tech Inspection Crew Channel` connected to the Google Sheets archive node, using a Tech Inspection Crew Slack account credential and channel ID.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Creator Credit | Swapnil AI Labs - swapnil.mandloi7@gmail.com - [swapnilailabs.netlify.app](https://swapnilailabs.netlify.app) |
| Data Privacy Notice | Driver name, car number, and inspector notes are transmitted to OpenAI for classification. Avoid including unrelated personal data. |
| Error Handling Configuration | External calls (Text Classifier, HTTP Requests, Slack, Google Sheets) are configured to retry up to 3 times with a 2-second wait and continue on final failure. |
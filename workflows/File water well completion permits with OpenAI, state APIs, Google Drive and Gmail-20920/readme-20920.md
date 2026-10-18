File water well completion permits with OpenAI, state APIs, Google Drive and Gmail

https://n8nworkflows.xyz/workflows/file-water-well-completion-permits-with-openai--state-apis--google-drive-and-gmail-20920


# File water well completion permits with OpenAI, state APIs, Google Drive and Gmail

### 1. Workflow Overview

This workflow automates water well drilling regulatory compliance by taking an incoming field completion report, performing AI-driven regulatory research, validating engineering specifications against county and aquifer-specific minimums, managing a mandatory 48-hour cure delay, filing the report with the state water board, archiving records to Google Drive, and sending status notifications via Gmail.

The workflow logic is categorized into the following functional blocks:
- **1.1 Input Reception & Normalization:** Receives raw webhook data from a drilling field application and formats it into strict typed parameters.
- **1.2 AI Research & Compliance Evaluation:** Utilizes an OpenAI chat model and an external HTTP tool to fetch regional grouting and setback minimums, evaluating them alongside driller license validity.
- **1.3 Initial Gating & Cure Hold:** Evaluates initial compliance; non-compliant records are immediately tagged, while compliant records enter a mandatory 48-hour wait period for concrete curing.
- **1.4 Post-Cure Verification & Final Filing:** Fetches post-cure inspection measurements, validates grout levels against a tolerance threshold, tags the final status, files the report to the state portal, archives documentation, and notifies stakeholders.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Normalization
- **Overview:** Captures the raw well completion report payload instantly upon drilling completion and casts incoming properties into strongly-typed variables.
- **Nodes Involved:** `Receive Well Completion Report`, `Normalize Completion Report`
- **Node Details:**
  - **Receive Well Completion Report**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Trigger)
    - *Configuration:* Listens on path `well-completion-report` for `POST` requests using response mode `onReceived`.
    - *Input/Output:* No inputs; outputs raw JSON payload to the normalization node.
    - *Edge Cases/Failures:* Invalid JSON payloads or network drops during webhook transmission.
  - **Normalize Completion Report**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Maps raw incoming properties (`driller_license_no`, `driller_license_expiry`, `well_id`, `county`, `aquifer_type`, `grout_depth_ft`, `setback_to_septic_ft`, `setback_to_property_line_ft`, `total_well_depth_ft`) to explicit string and number data types.
    - *Input/Output:* Input from webhook; outputs structured, typed JSON.
    - *Edge Cases/Failures:* Missing required fields in the incoming payload resulting in undefined evaluation parameters downstream.

#### 1.2 AI Research & Compliance Evaluation
- **Overview:** Employs an OpenAI Agent integrated with an HTTP tool to retrieve localized water board requirements and programmatically evaluates compliance against engineering rules.
- **Nodes Involved:** `State Grout Minimum Lookup Tool`, `Agent Model`, `Research State Grouting Requirements`, `Check Grouting & Setback Compliance`
- **Node Details:**
  - **State Grout Minimum Lookup Tool**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.toolHttpRequest` (AI Tool)
    - *Configuration:* `GET` request to `https://state-water-board.example.gov/api/grouting-minimums` using `State Water Board API Key` credentials. Includes auto-retry logic (max 3 tries, 2000ms delay).
    - *Input/Output:* Triggered exclusively by the AI agent; returns regional minimums.
    - *Edge Cases/Failures:* API timeouts or authentication failures. Handled by `onError: continueRegularOutput`.
  - **Agent Model**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model Provider)
    - *Configuration:* Uses model `gpt-4.1-mini` with a temperature setting of `0.1` using `OpenAI account` credentials.
    - *Input/Output:* Connected as the language model dependency for the AI Agent node.
    - *Edge Cases/Failures:* OpenAI API rate limits or quota exhaustion.
  - **Research State Grouting Requirements**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration:* Defines prompt parameters injecting county and aquifer type, enforcing strict JSON output containing minimum grout depth and setback thresholds. Uses system messages prohibiting estimation from memory.
    - *Input/Output:* Input from normalized report; outputs text response parsed by the subsequent code node.
    - *Edge Cases/Failures:* LLM hallucinations or JSON formatting errors.
  - **Check Grouting & Setback Compliance**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Processing)
    - *Configuration:* Parses AI JSON response (falling back to safety defaults if parsing fails), checks driller license expiry dates, and calculates boolean compliance flags for grout depth, septic setbacks, and property line setbacks.
    - *Input/Output:* Input from AI agent; outputs consolidated JSON containing base data alongside evaluation flags (`license_valid`, `setbacks_met`, `grout_ok`, etc.).
    - *Edge Cases/Failures:* Malformed JSON from the AI output triggering the fallback block.

#### 1.3 Initial Gating & Cure Hold
- **Overview:** Evaluates initial compliance parameters, instantly rejecting non-compliant files or routing compliant records through a mandatory 48-hour waiting period for material curing.
- **Nodes Involved:** `License Valid & Setbacks Met?`, `Tag Non-Compliant`, `48-Hour Grout Cure Confirmation Hold`
- **Node Details:**
  - **License Valid & Setbacks Met?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Combines conditions using an `and` operator verifying `license_valid === true` and `setbacks_met === true`.
    - *Input/Output:* Input from compliance checker node; routes to the wait node on success or the non-compliant tagger on failure.
  - **Tag Non-Compliant**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Sets `filing_status` to `"Non-Compliant - License/Setback Failure"` and sets `cure_verified` to `false`.
    - *Input/Output:* Input from branch failure; routes to the filing record shaper.
  - **48-Hour Grout Cure Confirmation Hold**
    - *Type and Technical Role:* `n8n-nodes-base.wait` (Flow Control)
    - *Configuration:* Pauses execution for an amount of `48` hours.
    - *Input/Output:* Input from successful branch; outputs execution context to the post-cure fetch node after the duration expires.
    - *Edge Cases/Failures:* Instance restarts or data retention limits exceeded during the 48-hour pause.

#### 1.4 Post-Cure Verification & Final Filing
- **Overview:** Retrieves inspection measurements following the cure delay, verifies compliance with a strict tolerance, and executes final state filing, secure archiving, and stakeholder notifications.
- **Nodes Involved:** `Fetch Post-Cure Inspection Reading`, `Verify Post-Cure Grout Level`, `Post-Cure Grout Level Acceptable?`, `Tag Cure Failed`, `Tag Filing Ready`, `Shape Filing Record`, `File Completion Report with State Portal`, `Archive Completion Report PDF`, `Notify Driller & Regulator`
- **Node Details:**
  - **Fetch Post-Cure Inspection Reading**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (External Integration)
    - *Configuration:* `GET` request to `https://field-inspection.example-county.gov/api/wells/{{ $json.well_id }}/post-cure-reading` using `County Field Inspection API Key` credentials.
    - *Input/Output:* Input from wait node; outputs measured post-cure grout depth data.
  - **Verify Post-Cure Grout Level**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Processing)
    - *Configuration:* Compares measured depth against minimum requirements allowing a `1.0` foot tolerance, generating a `cure_verified` boolean property.
    - *Input/Output:* Input from inspection reading; outputs verified payload.
  - **Post-Cure Grout Level Acceptable?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Evaluates if `cure_verified === true`.
    - *Input/Output:* Input from verification code node; routes to ready or failed state tags.
  - **Tag Cure Failed**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Sets `filing_status` to `"Hold - Grout Cure Below Minimum"`.
    - *Input/Output:* Input from failed branch; routes to filing record shaper.
  - **Tag Filing Ready**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Sets `filing_status` to `"Compliant - Ready to File"`.
    - *Input/Output:* Input from successful branch; routes to filing record shaper.
  - **Shape Filing Record**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Consolidates branches into a single source of truth by packaging data into a `filing_record` object and appending an ISO timestamp `filed_at`.
    - *Input/Output:* Receives data from all three outcome taggers; outputs unified filing package.
  - **File Completion Report with State Portal**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (External Integration)
    - *Configuration:* `POST` request to `https://state-water-board.example.gov/api/well-completion-filings` with body payload containing the structured `filing_record`. Uses `State Water Board API Key` credentials.
    - *Input/Output:* Input from shaper node; outputs API response.
  - **Archive Completion Report PDF**
    - *Type and Technical Role:* `n8n-nodes-base.googleDrive` (Storage Integration)
    - *Configuration:* Uploads JSON file named `well-completion-{{ $json.filing_record.well_id }}.json` to specified Drive and Folder IDs using `Well Compliance Google Drive account` credentials.
    - *Input/Output:* Input from state portal file node; outputs upload metadata.
  - **Notify Driller & Regulator**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Communication Integration)
    - *Configuration:* Sends email via `Well Compliance Gmail account` to `driller@example-drilling-co.com` featuring dynamic subject and body detailing filing status and timestamp.
    - *Input/Output:* Input from Google Drive node; completes the workflow execution path.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Well Completion Report | n8n-nodes-base.webhook | Trigger webhook | None | Normalize Completion Report | **Receive Well Completion Report**<br>Webhook fired by the driller's field app the moment drilling finishes.<br>Path: well-completion-report.<br>Every downstream field comes from this one POST body. |
| Normalize Completion Report | n8n-nodes-base.set | Format payload types | Receive Well Completion Report | Research State Grouting Requirements | **Normalize Completion Report**<br>Shapes the raw webhook payload into typed fields.<br>Keeps grout_depth_ft etc. as numbers, not strings.<br>Single source of truth for the raw report going forward. |
| State Grout Minimum Lookup Tool | @n8n/n8n-nodes-langchain.toolHttpRequest | AI Tool definition | None (AI Agent reference) | Research State Grouting Requirements | **State Grout Minimum Lookup Tool**<br>HTTP tool the Agent calls for county/aquifer-specific minimums.<br>GET to the state water board API.<br>Without it the Agent has no real numbers to check against. |
| Agent Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM Provider | None (AI Agent reference) | Research State Grouting Requirements | **Agent Model**<br>OpenAI chat model powering the research agent.<br>Model: gpt-4.1-mini, temp 0.1 (low, for consistent JSON).<br>Required sub-node — the Agent cannot run without it. |
| Research State Grouting Requirements | @n8n/n8n-nodes-langchain.agent | AI Agent evaluation | Normalize Completion Report | Check Grouting & Setback Compliance | **Research State Grouting Requirements**<br>AI Agent instructed to ALWAYS call the lookup tool, never guess.<br>Returns strict JSON minimums for grout/setbacks.<br>The one AI reasoning step Pattern A calls for. |
| Check Grouting & Setback Compliance | n8n-nodes-base.code | Deterministic compliance check | Research State Grouting Requirements | License Valid & Setbacks Met? | **Check Grouting & Setback Compliance**<br>Real Code logic: grout depth, septic setback, property setback vs minimums + license expiry.<br>Outputs license_valid and setbacks_met booleans.<br>The actual deterministic compliance engine. |
| License Valid & Setbacks Met? | n8n-nodes-base.if | Conditional routing gate | Check Grouting & Setback Compliance | 48-Hour Grout Cure Confirmation Hold, Tag Non-Compliant | **License Valid & Setbacks Met?**<br>First real branch point: AND of license_valid + setbacks_met.<br>Fail routes straight to Non-Compliant, skipping the cure wait.<br>Pass proceeds to the mandatory cure hold. |
| Tag Non-Compliant | n8n-nodes-base.set | Assign failure status | License Valid & Setbacks Met? | Shape Filing Record | **Tag Non-Compliant**<br>Stamps filing_status for license/setback failures.<br>cure_verified forced false — no cure check needed.<br>Fast-fail path, no 48h wait incurred. |
| 48-Hour Grout Cure Confirmation Hold | n8n-nodes-base.wait | Time-based delay | License Valid & Setbacks Met? | Fetch Post-Cure Inspection Reading | **48-Hour Grout Cure Confirmation Hold**<br>Mandatory wait before the post-cure re-inspection.<br>Duration: 48 hours.<br>This workflow's control-flow floor (Wait node). |
| Fetch Post-Cure Inspection Reading | n8n-nodes-base.httpRequest | Fetch inspection data | 48-Hour Grout Cure Confirmation Hold | Verify Post-Cure Grout Level | **Fetch Post-Cure Inspection Reading**<br>Pulls the county's re-measured grout depth after the cure period.<br>GET keyed by well_id.<br>Feeds the final cure-verification check. |
| Verify Post-Cure Grout Level | n8n-nodes-base.code | Compare cure levels | Fetch Post-Cure Inspection Reading | Post-Cure Grout Level Acceptable? | **Verify Post-Cure Grout Level**<br>Compares the measured post-cure depth to the minimum, 1ft tolerance.<br>Outputs cure_verified boolean.<br>The deterministic gate before filing is allowed. |
| Post-Cure Grout Level Acceptable? | n8n-nodes-base.if | Conditional cure routing gate | Verify Post-Cure Grout Level | Tag Filing Ready, Tag Cure Failed | **Post-Cure Grout Level Acceptable?**<br>Second real branch point, on cure_verified.<br>Fail routes to Tag Cure Failed (hold for re-grouting).<br>Pass routes to Tag Filing Ready. |
| Tag Cure Failed | n8n-nodes-base.set | Assign cure failure status | Post-Cure Grout Level Acceptable? | Shape Filing Record | **Tag Cure Failed**<br>Stamps filing_status as a cure-below-minimum hold.<br>Sent to the state as a hold record, not silently dropped.<br>Keeps the audit trail complete even on failure. |
| Tag Filing Ready | n8n-nodes-base.set | Assign success status | Post-Cure Grout Level Acceptable? | Shape Filing Record | **Tag Filing Ready**<br>Stamps filing_status as compliant and ready to file.<br>Only reachable after passing both branch gates.<br>The clean-path outcome. |
| Shape Filing Record | n8n-nodes-base.set | Consolidate record payload | Tag Non-Compliant, Tag Cure Failed, Tag Filing Ready | File Completion Report with State Portal | **Shape Filing Record**<br>Single source of truth all three outcome branches converge on.<br>Adds filed_at timestamp.<br>Every downstream node reads filing_record from here, never a bare $json. |
| File Completion Report with State Portal | n8n-nodes-base.httpRequest | Submit state regulatory report | Shape Filing Record | Archive Completion Report PDF | **File Completion Report with State Portal**<br>POSTs the shaped record to the state e-filing API.<br>Primary storage sink for this workflow.<br>Runs regardless of outcome — holds get filed too, just tagged. |
| Archive Completion Report PDF | n8n-nodes-base.googleDrive | Upload backup record | File Completion Report with State Portal | Notify Driller & Regulator | **Archive Completion Report PDF**<br>Uploads a JSON archive copy to Google Drive.<br>Named by well_id for easy lookup.<br>Secondary sink, satisfies long-term retention needs. |
| Notify Driller & Regulator | n8n-nodes-base.gmail | Send notification email | Archive Completion Report PDF | None | **Notify Driller & Regulator**<br>Emails the final filing_status to the driller/regulator inbox.<br>Subject includes well_id + status.<br>Last node in the chain — closes the loop for the field crew. |
| Panel 1 - Workflow Details | n8n-nodes-base.stickyNote | Documentation panel | None | None | **📘 WORKFLOW DETAILS**<br><br>Water Well Drilling Contractor State Permit & Annular-Space Grouting Compliance Filer... [Full details documented in canvas] |
| Panel 2 - Setup Guide | n8n-nodes-base.stickyNote | Configuration instructions | None | None | **⚙️ SETUP GUIDE**<br><br>Credentials required (create these exact names on import)... [Full details documented in canvas] |
| Panel 3 - Customisation | n8n-nodes-base.stickyNote | Extension recommendations | None | None | **🎨 CUSTOMISATION**<br><br>1. Swap notification channel... [Full details documented in canvas] |
| Panel 4 - Additional Information | n8n-nodes-base.stickyNote | Operational insights and limits | None | None | **ℹ️ ADDITIONAL INFORMATION**<br><br>Error handling: every external-action node... [Full details documented in canvas] |

---

### 4. Reproducing the Workflow from Scratch

Rebuild the workflow manually in n8n by following these sequential steps:

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node named `Receive Well Completion Report`.
   - Set parameters: Path `well-completion-report`, HTTP Method `POST`, Response Mode `On Received`.

2. **Normalize Incoming Fields:**
   - Add a **Set** node named `Normalize Completion Report`. Connect `Receive Well Completion Report` output to it.
   - Configure assignments to map incoming string/number properties: `driller_license_no`, `driller_license_expiry`, `well_id`, `county`, `aquifer_type`, `grout_depth_ft`, `setback_to_septic_ft`, `setback_to_property_line_ft`, and `total_well_depth_ft`.

3. **Configure the AI Research Agent Setup:**
   - Add an **OpenAI Chat Model** node named `Agent Model`. Configure model selection to `gpt-4.1-mini`, option temperature `0.1`, using the `OpenAI account` credential.
   - Add an **HTTP Request Tool** node named `State Grout Minimum Lookup Tool`. Set URL to `https://state-water-board.example.gov/api/grouting-minimums`, Method `GET`, using the `State Water Board API Key` credential (HTTP Header Auth). Configure error handling (`Continue Regular Output`, `Retry On Fail`, max tries `3`, wait `2000ms`).
   - Add an **AI Agent** node named `Research State Grouting Requirements`. Connect `Agent Model` to its model input and `State Grout Minimum Lookup Tool` to its tool input. Set prompt text to evaluate well parameters and request strict JSON output. Configure error handling and retries.

4. **Implement the Compliance Engine:**
   - Add a **Code** node named `Check Grouting & Setback Compliance`. Connect `Research State Grouting Requirements` output to it.
   - Add JavaScript logic to parse the AI output JSON, apply safe fallback minimum values, compare actual values against limits, check license expiry dates, and return evaluation flags (`license_valid`, `setbacks_met`, etc.).

5. **Create Initial Gating Branch:**
   - Add an **If** node named `License Valid & Setbacks Met?`. Connect `Check Grouting & Setback Compliance` output to it.
   - Set conditions to verify both `license_valid` and `setbacks_met` are true.

6. **Handle Non-Compliant Path:**
   - From the `false` output of `License Valid & Setbacks Met?`, add a **Set** node named `Tag Non-Compliant`.
   - Configure assignments: `filing_status` = `"Non-Compliant - License/Setback Failure"`, `cure_verified` = `false`.

7. **Configure Cure Wait Period:**
   - From the `true` output of `License Valid & Setbacks Met?`, add a **Wait** node named `48-Hour Grout Cure Confirmation Hold`.
   - Set unit to `hours` and amount to `48`.

8. **Fetch Post-Cure Inspection Data:**
   - Add an **HTTP Request** node named `Fetch Post-Cure Inspection Reading`. Connect it to the wait node output.
   - Set URL to `=https://field-inspection.example-county.gov/api/wells/{{ $json.well_id }}/post-cure-reading`, Method `GET`, using `County Field Inspection API Key` credentials with standard error retries.

9. **Verify Post-Cure Measurement:**
   - Add a **Code** node named `Verify Post-Cure Grout Level`. Connect the HTTP request output to it.
   - Add code to compare measured depth against minimum requirements with a `1.0` ft tolerance, outputting a `cure_verified` boolean.

10. **Create Post-Cure Gating Branch:**
    - Add an **If** node named `Post-Cure Grout Level Acceptable?`. Connect the verification code output to it.
    - Set condition to evaluate `cure_verified === true`.

11. **Tag Success and Failure Outcomes:**
    - From the `false` output, add a **Set** node named `Tag Cure Failed` with `filing_status` set to `"Hold - Grout Cure Below Minimum"`.
    - From the `true` output, add a **Set** node named `Tag Filing Ready` with `filing_status` set to `"Compliant - Ready to File"`.

12. **Converge and Shape Filing Record:**
    - Add a **Set** node named `Shape Filing Record`. Connect outputs from `Tag Non-Compliant`, `Tag Cure Failed`, and `Tag Filing Ready` into this single node.
    - Configure assignments to package properties into a `filing_record` object and generate a `filed_at` ISO timestamp string.

13. **Execute State Filing, Archiving, and Notification:**
    - Add an **HTTP Request** node named `File Completion Report with State Portal`. Set URL to `https://state-water-board.example.gov/api/well-completion-filings`, Method `POST`, sending body parameters using the `filing_record` object with `State Water Board API Key` credentials.
    - Add a **Google Drive** node named `Archive Completion Report PDF`. Set operation to `upload`, file name to `well-completion-{{ $json.filing_record.well_id }}.json`, specifying valid `driveId` and `folderId` parameters, using `Well Compliance Google Drive account` credentials.
    - Add a **Gmail** node named `Notify Driller & Regulator`. Configure operation to send email to `driller@example-drilling-co.com` with dynamic subject and body pulling values from `filing_record`, using `Well Compliance Gmail account` credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Creator & Developer Information | Created by Swapnil AI Labs (`swapnil.mandloi7@gmail.com`, `swapnilailabs.netlify.app`). |
| Execution Data Retention Warning | The 48-hour Wait node pauses workflow execution. Ensure n8n server execution data retention settings are configured to exceed 48 hours to prevent dropped resume states. |
| AI Fallback Values | The compliance code node includes hardcoded safety minimums (20ft grout / 50ft septic / 10ft property line) if AI parsing fails. Verify these match regional standards before production deployment. |
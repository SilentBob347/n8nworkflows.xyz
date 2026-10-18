Chase stalled enquiries and send a daily Gmail digest from Google Sheets

https://n8nworkflows.xyz/workflows/chase-stalled-enquiries-and-send-a-daily-gmail-digest-from-google-sheets-20937


# Chase stalled enquiries and send a daily Gmail digest from Google Sheets

### 1. Workflow Overview

This workflow is designed to automate enquiry pipeline management for commercial cleaning or similar sales teams. Its core purpose is to prevent lost revenue by automatically identifying enquiries that have gone quiet, notifying their respective owners via personalized email nudges, and compiling an executive morning digest from a shared Google Sheets tracking board.

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Configuration:** Initializes execution via manual or scheduled triggers and loads centralized configuration parameters.
- **1.2 Data Ingestion & Analysis:** Reads the entire tracker spreadsheet once and analyzes stage distribution, last touch timestamps, and stalled records.
- **1.3 Owner Nudging Logic:** Determines whether stalled enquiries exist, groups them by owner, evaluates safety/preview switches, and dispatches individual owner nudges.
- **1.4 State Persistence:** Increments chase counters and appends audit logs back to the Google Sheet.
- **1.5 Morning Digest Generation:** Compiles an aggregate summary of open items and quiet threads, evaluates delivery settings, and sends a management overview via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Triggers the workflow either manually or automatically on weekdays, routing execution into a centralized configuration node that sets up parameters, mapping values, thresholds, and safety switches.
- **Nodes Involved:** `Run once manually`, `Every weekday at 08:30`, `config`.
- **Node Details:**
  - **Run once manually**
    - *Type & Role:* `n8n-nodes-base.manualTrigger` (Trigger) — Initiates manual testing.
    - *Configuration:* Default parameters.
    - *Inputs:* None.
    - *Outputs:* `config`
    - *Failure Modes:* None.
  - **Every weekday at 08:30**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Trigger) — Automates execution at 08:30 Monday through Friday.
    - *Configuration:* Cron expression `30 8 * * 1-5`.
    - *Inputs:* None.
    - *Outputs:* `config`
    - *Failure Modes:* None.
  - **config**
    - *Type & Role:* `n8n-nodes-base.code` (Data transformation) — Stores global constants, column mappings, area-to-owner definitions, business rules, and safety switches (`TEST_RUN`, `TEST_EMAIL`).
    - *Configuration:* JavaScript data block returning an execution configuration object.
    - *Inputs:* `Run once manually`, `Every weekday at 08:30`
    - *Outputs:* `[cred] Sheets - read the tracker`
    - *Failure Modes:* Syntax errors in the script object.

#### 2.2 Data Ingestion & Analysis
- **Overview:** Reads the tracker tab from Google Sheets in a single batch operation and parses row dates to identify open enquiries lacking recent activity.
- **Nodes Involved:** `[cred] Sheets - read the tracker`, `find what has gone quiet`.
- **Node Details:**
  - **[cred] Sheets - read the tracker**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Integration) — Fetches all rows from the target spreadsheet and worksheet.
    - *Configuration:* Uses expressions for `documentId` and `sheetName` referencing the `config` node.
    - *Inputs:* `config`
    - *Outputs:* `find what has gone quiet`
    - *Credentials Required:* Google Sheets OAuth2 / Service Account.
    - *Failure Modes:* API rate limits, invalid Spreadsheet ID, or missing sheet tabs.
  - **find what has gone quiet**
    - *Type & Role:* `n8n-nodes-base.code` (Data transformation) — Parses various date formats (ISO, slashed, Google Sheets serial numbers), calculates inactivity durations against `chase_after_hours`, aggregates stage metrics, and segments records by owner email.
    - *Configuration:* JavaScript processing logic.
    - *Inputs:* `[cred] Sheets - read the tracker`
    - *Outputs:* `anything to chase?`, `build the morning digest`
    - *Failure Modes:* Unparseable date strings in custom columns.

#### 2.3 Owner Nudging Logic
- **Overview:** Evaluates if any enquiries require chasing, splits payloads by distinct owners, formats HTML/text reminders, and respects safety/preview gates before sending emails.
- **Nodes Involved:** `anything to chase?`, `stop-quiet`, `split-owner`, `nudge`, `sending2`, `stop-prev2`, `mail-nudge`.
- **Node Details:**
  - **anything to chase?**
    - *Type & Role:* `n8n-nodes-base.if` (Flow control) — Checks whether `_has_stalled` is true.
    - *Configuration:* Condition checking `{{ $json._has_stalled }}` equals `true`.
    - *Inputs:* `find what has gone quiet`
    - *Outputs:* `split-owner` (True branch), `STOP: nothing stalled` (False branch)
    - *Failure Modes:* Type validation mismatches.
  - **STOP: nothing stalled**
    - *Type & Role:* `n8n-nodes-base.code` (Flow termination) — Terminates the inactive chase branch gracefully.
    - *Configuration:* Appends outcome metadata.
    - *Inputs:* `anything to chase?`
    - *Outputs:* `End of run`
  - **split-owner**
    - *Type & Role:* `n8n-nodes-base.code` (Data transformation) — Expands grouped owner objects into individual execution items.
    - *Configuration:* Maps owner groups.
    - *Inputs:* `anything to chase?`
    - *Outputs:* `write the nudge`
  - **write the nudge**
    - *Type & Role:* `n8n-nodes-base.code` (Data transformation) — Generates responsive HTML email markup and plain-text alternatives listing stalled items, references, stages, and next actions.
    - *Configuration:* JavaScript rendering engine.
    - *Inputs:* `split-owner`
    - *Outputs:* `sending the nudges?`
  - **sending the nudges?**
    - *Type & Role:* `n8n-nodes-base.if` (Flow control) — Safety gate checking `send_enabled`.
    - *Configuration:* Condition evaluating `{{ $('config').first().json.send_enabled }}`.
    - *Inputs:* `write the nudge`
    - *Outputs:* `[cred] Gmail - nudge the owner` (True), `STOP: preview - nudges not sent` (False)
  - **STOP: preview - nudges not sent**
    - *Type & Role:* `n8n-nodes-base.code` (Flow termination) — Captures preview payloads when live sending is disabled.
    - *Inputs:* `sending the nudges?`
    - *Outputs:* `End of run`
  - **[cred] Gmail - nudge the owner**
    - *Type & Role:* `n8n-nodes-base.gmail` (Integration) — Sends HTML-formatted reminder emails to owners (or redirects to test addresses when configured).
    - *Configuration:* Maps recipient, sender name, reply-to parameters, and dynamic subject lines.
    - *Inputs:* `sending the nudges?`
    - *Outputs:* `record the chase`
    - *Credentials Required:* Gmail OAuth2.
    - *Failure Modes:* SMTP quota exceeded, invalid recipient emails, or token expiration.

#### 2.4 State Persistence
- **Overview:** Updates tracking records in Google Sheets by incrementing chase counters and appending audit notes.
- **Nodes Involved:** `record the chase`, `sheet-chase`.
- **Node Details:**
  - **record the chase**
    - *Type & Role:* `n8n-nodes-base.code` (Data transformation) — Constructs update payloads with incremented chase metrics and timestamped audit logs.
    - *Configuration:* JavaScript mapping function.
    - *Inputs:* `[cred] Gmail - nudge the owner`
    - *Outputs:* `[cred] Sheets - record the chase`
  - **[cred] Sheets - record the chase**
    - *Type & Role:* `n8n-nodes-base.googleSheets` (Integration) — Updates existing spreadsheet rows based on matching reference keys.
    - *Configuration:* Operation set to `appendOrUpdate`, matching column configured to `Ref`.
    - *Inputs:* `record the chase`
    - *Outputs:* `End of run`
    - *Credentials Required:* Google Sheets OAuth2 / Service Account.
    - *Failure Modes:* Missing reference columns or write permission restrictions.

#### 2.5 Morning Digest Generation
- **Overview:** Compiles statistical summaries across all open enquiries, formats visual metrics and warning sections, evaluates notification switches, and dispatches a morning overview.
- **Nodes Involved:** `digest`, `send-digest`, `stop-digest`, `mail-digest`, `end2`.
- **Node Details:**
  - **digest**
    - *Type & Role:* `n8n-nodes-base.code` (Data transformation) — Builds HTML/text structures summarizing pipeline health, stage counts, stalled items, and unassigned records.
    - *Configuration:* JavaScript analytics compiler.
    - *Inputs:* `find what has gone quiet`
    - *Outputs:* `send-digest?`
  - **send-digest?**
    - *Type & Role:* `n8n-nodes-base.if` (Flow control) — Evaluates whether digest delivery is enabled and permitted.
    - *Configuration:* Condition verifying `_send_digest` and `send_enabled`.
    - *Inputs:* `digest`
    - *Outputs:* `[cred] Gmail - send the digest` (True), `STOP: digest not sent` (False)
  - **STOP: digest not sent**
    - *Type & Role:* `n8n-nodes-base.code` (Flow termination) — Stops execution in preview mode.
    - *Inputs:* `send-digest?`
    - *Outputs:* `End of run`
  - **[cred] Gmail - send the digest**
    - *Type & Role:* `n8n-nodes-base.gmail` (Integration) — Dispatches the operational overview email to management.
    - *Configuration:* Maps destination address (`digest_to` or `test_email`).
    - *Inputs:* `send-digest?`
    - *Outputs:* `End of run`
    - *Credentials Required:* Gmail OAuth2.
    - *Failure Modes:* Authentication revocation or invalid configuration variables.
  - **End of run**
    - *Type & Role:* `n8n-nodes-base.noOp` (Flow control) — Terminal merge node collecting all execution branches.
    - *Configuration:* Default.
    - *Inputs:* `[cred] Sheets - record the chase`, `STOP: nothing stalled`, `STOP: preview - nudges not sent`, `STOP: digest not sent`, `[cred] Gmail - send the digest`
    - *Outputs:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run once manually | n8n-nodes-base.manualTrigger | Trigger workflow manually | None | config | 1. Start, settings, and one read of the tracker |
| Every weekday at 08:30 | n8n-nodes-base.scheduleTrigger | Scheduled trigger | None | config | 1. Start, settings, and one read of the tracker |
| config | n8n-nodes-base.code | Load global settings | Run once manually, Every weekday at 08:30 | [cred] Sheets - read the tracker | 1. Start, settings, and one read of the tracker |
| [cred] Sheets - read the tracker | n8n-nodes-base.googleSheets | Read tracker spreadsheet | config | find what has gone quiet | 1. Start, settings, and one read of the tracker |
| find what has gone quiet | n8n-nodes-base.code | Analyze activity and stages | [cred] Sheets - read the tracker | anything to chase?, build the morning digest | 1. Start, settings, and one read of the tracker |
| anything to chase? | n8n-nodes-base.if | Conditional routing for nudges | find what has gone quiet | split-owner, STOP: nothing stalled | 2. Decide who to chase, then write one nudge per owner |
| STOP: nothing stalled | n8n-nodes-base.code | End inactive chase branch | anything to chase? | End of run | 2. Decide who to chase, then write one nudge per owner |
| split-owner | n8n-nodes-base.code | Separate items per owner | anything to chase? | write the nudge | 2. Decide who to chase, then write one nudge per owner |
| nudge | n8n-nodes-base.code | Compose owner reminder | split-owner | sending the nudges? | 2. Decide who to chase, then write one nudge per owner |
| sending the nudges? | n8n-nodes-base.if | Safety switch for email delivery | nudge | [cred] Gmail - nudge the owner, STOP: preview - nudges not sent | 2. Decide who to chase, then write one nudge per owner |
| STOP: preview - nudges not sent | n8n-nodes-base.code | Capture preview drafts | sending the nudges? | End of run | 2. Decide who to chase, then write one nudge per owner |
| [cred] Gmail - nudge the owner | n8n-nodes-base.gmail | Send owner notification email | sending the nudges? | record the chase | 2. Decide who to chase, then write one nudge per owner |
| record the chase | n8n-nodes-base.code | Build sheet update payload | [cred] Gmail - nudge the owner | [cred] Sheets - record the chase | 3. Write the chase back to the sheet |
| [cred] Sheets - record the chase | n8n-nodes-base.googleSheets | Update tracking sheet counters | record the chase | End of run | 3. Write the chase back to the sheet |
| digest | n8n-nodes-base.code | Compile executive summary | find what has gone quiet | send the digest? | 4. The morning digest |
| send the digest? | n8n-nodes-base.if | Conditional digest delivery | digest | [cred] Gmail - send the digest, STOP: digest not sent | 4. The morning digest |
| STOP: digest not sent | n8n-nodes-base.code | Hold digest in preview | send the digest? | End of run | 4. The morning digest |
| [cred] Gmail - send the digest | n8n-nodes-base.gmail | Send management digest email | send the digest? | End of run | 4. The morning digest |
| End of run | n8n-nodes-base.noOp | Merge terminal pathways | [cred] Sheets - record the chase, STOP: nothing stalled, STOP: preview - nudges not sent, STOP: digest not sent, [cred] Gmail - send the digest | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`) named `Run once manually`.
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`) named `Every weekday at 08:30`. Configure interval cron expression: `30 8 * * 1-5`.
2. **Create Configuration Node:**
   - Add a **Code** node named `config`. Connect both triggers to this node.
   - Insert the configuration JavaScript snippet defining `TRACKER_SHEET_ID`, `COLUMNS`, business rules, areas, and safety switches (`TEST_RUN = true`, `TEST_EMAIL = ''`, `DIGEST_TO = ''`).
3. **Set Up Spreadsheet Reading:**
   - Add a **Google Sheets** node named `[cred] Sheets - read the tracker`. Connect `config` as its input.
   - Configure credentials (OAuth2/Service Account), setting `documentId` to `{{ $('config').first().json.tracker_sheet_id }}` and `sheetName` to `{{ $('config').first().json.tracker_tab }}`.
4. **Add Analysis Logic:**
   - Add a **Code** node named `find what has gone quiet`. Connect the Google Sheets node to its input.
   - Implement date parsing and inactive enquiry grouping logic.
5. **Implement Chase Branch & Owner Distribution:**
   - Add an **If** node named `anything to chase?` connected to `find what has gone quiet`. Set condition to evaluate `{{ $json._has_stalled }}` equals `true`.
   - Connect the false branch to a **Code** node named `STOP: nothing stalled`, linking it ultimately to the end node.
   - Connect the true branch to a **Code** node named `split-owner`, followed by a **Code** node named `nudge` to format HTML/text reminder emails.
6. **Implement Safety Gate & Gmail Dispatch:**
   - Add an **If** node named `sending the nudges?` connected to `nudge`. Condition: `{{ $('config').first().json.send_enabled }}` equals `true`.
   - Connect the false branch to a **Code** node named `STOP: preview - nudges not sent`.
   - Connect the true branch to a **Gmail** node named `[cred] Gmail - nudge the owner` using Gmail credentials. Configure recipient, sender name, reply-to options, and HTML message parameters.
7. **Record Chase State Back to Sheets:**
   - Add a **Code** node named `record the chase` connected after `[cred] Gmail - nudge the owner`.
   - Add a **Google Sheets** node named `[cred] Sheets - record the chase`. Set operation to `appendOrUpdate`, matching columns to `Ref`, and connect it to `record the chase`.
8. **Build and Send the Morning Digest:**
   - Add a **Code** node named `digest` connected to the output of `find what has gone quiet`.
   - Add an **If** node named `send-digest?` connected to `digest`. Condition checks `_send_digest` and `send_enabled`.
   - Connect the false branch to `STOP: digest not sent`.
   - Connect the true branch to a **Gmail** node named `[cred] Gmail - send the digest` configured with management recipient settings.
9. **Merge and Finalize:**
   - Add a **No-Op** node named `End of run`.
   - Connect all terminal nodes (`[cred] Sheets - record the chase`, `STOP: nothing stalled`, `STOP: preview - nudges not sent`, `STOP: digest not sent`, `[cred] Gmail - send the digest`) into `End of run`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Commercial Cleaning Enquiry Pipeline Template | Designed for small sales teams tracking leads in Google Sheets. |
| Configuration Centralization | All business logic, thresholds (`CHASE_AFTER_HOURS`, `CHASE_MAX`), and column maps reside exclusively in the `config` node. |
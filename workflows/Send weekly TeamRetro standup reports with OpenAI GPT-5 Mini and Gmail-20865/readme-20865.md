Send weekly TeamRetro standup reports with OpenAI GPT-5 Mini and Gmail

https://n8nworkflows.xyz/workflows/send-weekly-teamretro-standup-reports-with-openai-gpt-5-mini-and-gmail-20865


# Send weekly TeamRetro standup reports with OpenAI GPT-5 Mini and Gmail

### 1. Workflow Overview

This workflow automates the collection, aggregation, AI summarization, and distribution of TeamRetro standup reports. Its primary purpose is to capture daily asynchronous standup summaries, store them securely in an n8n Data Table, and synthesize them at the end of the week into concise, actionable executive reports sent to stakeholders via email.

The logical execution is split into two independent entry points:
- **1.1 Input Reception & Storage:** Listens for completed standups via webhook, filters duplicates and empty updates, formats the transcript, and persists the data into an n8n Data Table (`standup_days`).
- **1.2 Weekly Processing & AI Reporting:** Triggers every Friday afternoon, queries the Data Table for the past week's standups, groups them by team, utilizes OpenAI (GPT-5 Mini) to generate Markdown reports, converts them to HTML, emails them via Gmail, and prunes records older than four weeks.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Storage
**Overview:** This block captures incoming webhook notifications from TeamRetro when a standup session is closed. It guards against duplicate webhook deliveries and ensures that only sessions with actual participant updates are transformed and written to the database.

**Nodes Involved:**
- `When Standup Closes`
- `Skip Repeat Deliveries`
- `Anyone Posted?`
- `Prepare the Day`
- `Store the Day`

**Node Details:**

- **When Standup Closes**
  - *Type & Technical Role:* `n8n-nodes-teamretro.teamRetroTrigger` (Webhook Trigger). Listens for the `standup.completed` event from TeamRetro.
  - *Configuration:* Subscribes to event `standup.completed`. Requires a configured webhook signing secret matching TeamRetro settings.
  - *Key Expressions:* None (Trigger node).
  - *Input/Output:* Output connects to `Skip Repeat Deliveries`.
  - *Version Requirements:* Type Version 1.
  - *Edge Cases:* Webhook signature verification failure; downtime causing missed webhooks (relies on TeamRetro retry logic).

- **Skip Repeat Deliveries**
  - *Type & Technical Role:* `n8n-nodes-base.removeDuplicates` (Flow Control). Prevents duplicate processing of retried webhook requests.
  - *Configuration:* Operation set to `removeItemsSeenInPreviousExecutions`. Deduplication value checks `$json.deliveryId`.
  - *Key Expressions:* `={{ $json.deliveryId }}`
  - *Input/Output:* Input from `When Standup Closes`, output to `Anyone Posted?`.
  - *Version Requirements:* Type Version 2.
  - *Edge Cases:* Missing `deliveryId` property in payload.

- **Anyone Posted?**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Logic). Filters out standups where no participant provided updates.
  - *Configuration:* Version 2.2 conditional check. Evaluates whether the updates array length is greater than zero.
  - *Key Expressions:* `={{ ($json.data.standup.updates ?? []).length }}` (Left value) > `0` (Right value).
  - *Input/Output:* Input from `Skip Repeat Deliveries`, output (true branch) to `Prepare the Day`.
  - *Version Requirements:* Type Version 2.2.
  - *Edge Cases:* Malformed JSON payloads missing the `standup` or `updates` object.

- **Prepare the Day**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Parses structured standup objects and compiles them into a clean, flat plain-text transcript alongside metadata fields.
  - *Configuration:* JavaScript execution environment mapping participant names, topics, updates, and carried-forward counts.
  - *Key Expressions:* Extracts fields from `$('When Standup Closes').first().json.data.standup`.
  - *Input/Output:* Input from `Anyone Posted?`, output to `Store the Day`.
  - *Version Requirements:* Type Version 2.
  - *Edge Cases:* Unhandled nested null values in participants or topics.

- **Store the Day**
  - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database Operation). Persists the transformed daily standup data into an n8n Data Table.
  - *Configuration:* Target Data Table name: `standup_days`. Mapping mode set to `autoMapInputData`.
  - *Key Expressions:* None.
  - *Input/Output:* Input from `Prepare the Day`. No downstream execution required for this branch.
  - *Version Requirements:* Type Version 1.
  - *Edge Cases:* Missing Data Table schema columns (`date`, `day`, `team`, `standup`, `people`, `transcript`).

---

#### 2.2 Weekly Processing & AI Reporting
**Overview:** This block executes on a weekly schedule, aggregates the prior week's entries per team, prompts OpenAI to synthesize progress, blockers, and risks, formats the output into HTML, emails it via Gmail, and cleans up historical records.

**Nodes Involved:**
- `Every Friday at 4 PM`
- `Report Settings`
- `Load the Week`
- `Group by Team`
- `Write the Weekly Report`
- `OpenAI GPT-5 Mini`
- `Format as HTML`
- `Email Stakeholders`
- `Prune Old Days`

**Node Details:**

- **Every Friday at 4 PM**
  - *Type & Technical Role:* `n8n-nodes-base.scheduleTrigger` (Cron/Time Trigger). Initiates the weekly reporting workflow.
  - *Configuration:* Interval configured for weekly execution on Friday (`triggerAtDay: [5]`) at 16:00 (`triggerAtHour: 16`).
  - *Key Expressions:* None.
  - *Input/Output:* Output connects to `Report Settings`.
  - *Version Requirements:* Type Version 1.2.
  - *Edge Cases:* Timezone discrepancies depending on n8n server configuration.

- **Report Settings**
  - *Type & Technical Role:* `n8n-nodes-base.set` (Parameter Definition). Defines runtime variables for report distribution and historical range windows.
  - *Configuration:* Assigns string value `recipients` and numeric value `days` (default `7`).
  - *Key Expressions:* None.
  - *Input/Output:* Input from `Every Friday at 4 PM`, output to `Load the Week`.
  - *Version Requirements:* Type Version 3.4.
  - *Edge Cases:* Invalid email address formatting in the recipient string.

- **Load the Week**
  - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database Operation). Queries records from the `standup_days` table within the defined time window.
  - *Configuration:* Operation `get`, returning all rows matching date filters.
  - *Key Expressions:* 
    - Lower bound: `={{ Number($now.minus({ days: $json.days }).toFormat('yyyyLLdd')) }}` (Condition: `gte`)
    - Upper bound: `={{ Number($now.toFormat('yyyyLLdd')) }}` (Condition: `lt`)
  - *Input/Output:* Input from `Report Settings`, output to `Group by Team`.
  - *Version Requirements:* Type Version 1.
  - *Edge Cases:* Empty datasets returned if no standups occurred during the window.

- **Group by Team**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation). Aggregates individual daily standup rows into team-specific collections and formats a unified week transcript.
  - *Configuration:* Custom JavaScript block iterating over inputs, sorting by day, and compiling a Markdown block.
  - *Key Expressions:* Processes `$input.all()`.
  - *Input/Output:* Input from `Load the Week`, output to `Write the Weekly Report`.
  - *Version Requirements:* Type Version 2.
  - *Edge Cases:* Standups lacking team names or containing null team values.

- **Write the Weekly Report**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Agent / Chain). Prompts the Language Model to generate structured weekly executive summaries.
  - *Configuration:* Uses a system message prompt template enforcing Markdown output with sections (`## Progress`, `## Blockers that lingered`, `## Risks for next week`) under 250 words.
  - *Key Expressions:* `=Team: {{ $json.team }}. {{ $json.days }} standups from {{ $json.from }} to {{ $json.to }}.\n\n{{ $json.week }}`
  - *Input/Output:* Connected to `OpenAI GPT-5 Mini` via `ai_languageModel`. Output connects to `Format as HTML`.
  - *Version Requirements:* Type Version 1.5.
  - *Edge Cases:* LLM hallucination or exceeding token/word count limits.

- **OpenAI GPT-5 Mini**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model Provider). Supplies the underlying chat model logic for the LangChain node.
  - *Configuration:* Selected model: `gpt-5-mini`. Requires active OpenAI credentials.
  - *Key Expressions:* None.
  - *Input/Output:* Connects to `Write the Weekly Report` through the `ai_languageModel` connection.
  - *Version Requirements:* Type Version 1.2.
  - *Edge Cases:* API rate limits, authentication errors, or model unavailability.

- **Format as HTML**
  - *Type & Technical Role:* `n8n-nodes-base.markdown` (Text Processing). Converts the Markdown output generated by OpenAI into valid HTML code for email delivery.
  - *Configuration:* Mode set to `markdownToHtml`. Destination key: `html`.
  - *Key Expressions:* `={{ $json.text }}`
  - *Input/Output:* Input from `Write the Weekly Report`, output to `Email Stakeholders`.
  - *Version Requirements:* Type Version 1.
  - *Edge Cases:* Unsanitized or malformed Markdown structures failing conversion.

- **Email Stakeholders**
  - *Type & Technical Role:* `n8n-nodes-base.gmail` (Email Integration). Sends the formatted HTML report to configured recipients via Gmail.
  - *Configuration:* Email type: `html`. Attribution appending disabled. Subject and recipient fields dynamically populated.
  - *Key Expressions:* 
    - Send To: `={{ $('Report Settings').first().json.recipients }}`
    - Subject: `=Weekly standup report: {{ $('Group by Team').item.json.team }}, {{ $('Group by Team').item.json.from }} to {{ $('Group by Team').item.json.to }}`
    - Message: `={{ $json.html }}`
  - *Input/Output:* Input from `Format as HTML`, output to `Prune Old Days`.
  - *Version Requirements:* Type Version 2.2.
  - *Edge Cases:* OAuth token expiration, quota limits on Gmail sending rates.

- **Prune Old Days**
  - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database Operation). Deletes historical records older than four weeks from the Data Table.
  - *Configuration:* Operation `deleteRows`. Filter targets records older than 4 weeks. `executeOnce` set to true.
  - *Key Expressions:* `={{ Number($now.minus({ weeks: 4 }).toFormat('yyyyLLdd')) }}` (Condition: `lt`)
  - *Input/Output:* Input from `Email Stakeholders`. Terminal node.
  - *Version Requirements:* Type Version 1.
  - *Edge Cases:* Incorrect date math removing active records if system clocks are misconfigured.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation & Overview | None | None | TeamRetro: Email a weekly AI standup report to stakeholders... |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Section 1 Documentation | None | None | ## 1. Store each closed standup\n\nFires when a standup closes... |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Section 2 Documentation | None | None | ## 2. Load the week\n\nEvery Friday the `days` days before today... |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Section 3 Documentation | None | None | ## 3. Write, send and tidy up\n\nOpenAI writes the report... |
| When Standup Closes | `n8n-nodes-teamretro.teamRetroTrigger` | Webhook Trigger | None | Skip Repeat Deliveries | |
| Skip Repeat Deliveries | `n8n-nodes-base.removeDuplicates` | Duplicate Guard | When Standup Closes | Anyone Posted? | |
| Anyone Posted? | `n8n-nodes-base.if` | Conditional Filter | Skip Repeat Deliveries | Prepare the Day | |
| Prepare the Day | `n8n-nodes-base.code` | Transcript Formatting | Anyone Posted? | Store the Day | |
| Store the Day | `n8n-nodes-base.dataTable` | Database Insert | Prepare the Day | None | |
| Every Friday at 4 PM | `n8n-nodes-base.scheduleTrigger` | Schedule Trigger | None | Report Settings | |
| Report Settings | `n8n-nodes-base.set` | Define Variables | Every Friday at 4 PM | Load the Week | |
| Load the Week | `n8n-nodes-base.dataTable` | Database Query | Report Settings | Group by Team | |
| Group by Team | `n8n-nodes-base.code` | Data Aggregation | Load the Week | Write the Weekly Report | |
| Write the Weekly Report | `@n8n/n8n-nodes-langchain.chainLlm` | AI Summarization Chain | Group by Team | Format as HTML | |
| OpenAI GPT-5 Mini | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | LLM Provider Model | None (AI Link) | Write the Weekly Report | |
| Format as HTML | `n8n-nodes-base.markdown` | Markdown to HTML | Write the Weekly Report | Email Stakeholders | |
| Email Stakeholders | `n8n-nodes-base.gmail` | Email Dispatch | Format as HTML | Prune Old Days | |
| Prune Old Days | `n8n-nodes-base.dataTable` | Database Pruning | Email Stakeholders | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the entire workflow manually in n8n:

1. **Create the Data Table:**
   - In n8n, create a new Data Table named `standup_days`.
   - Add the following columns: `date` (String), `day` (Number), `team` (String), `standup` (String), `people` (Number), and `transcript` (String).

2. **Build Branch 1 (Ingestion & Storage):**
   - **Step 1:** Add a `When Standup Closes` (`n8n-nodes-teamretro.teamRetroTrigger`) node. Set event to `standup.completed`. Configure your webhook signing secret.
   - **Step 2:** Add a `Skip Repeat Deliveries` (`n8n-nodes-base.removeDuplicates`) node. Set operation to `removeItemsSeenInPreviousExecutions` with deduplication value `={{ $json.deliveryId }}`. Connect `When Standup Closes` to it.
   - **Step 3:** Add an `Anyone Posted?` (`n8n-nodes-base.if`) node. Configure condition (version 2.2) to evaluate `={{ ($json.data.standup.updates ?? []).length }}` greater than `0`. Connect `Skip Repeat Deliveries` to it.
   - **Step 4:** Add a `Prepare the Day` (`n8n-nodes-base.code`) node. Insert the JavaScript logic provided in the reference code to map updates and compile transcripts. Connect the true branch of `Anyone Posted?` to it.
   - **Step 5:** Add a `Store the Day` (`n8n-nodes-base.dataTable`) node. Select the `standup_days` table and set mapping mode to `autoMapInputData`. Connect `Prepare the Day` to it.

3. **Build Branch 2 (Weekly Schedule & AI Reporting):**
   - **Step 6:** Add an `Every Friday at 4 PM` (`n8n-nodes-base.scheduleTrigger`) node. Set interval rules for Fridays at 16:00.
   - **Step 7:** Add a `Report Settings` (`n8n-nodes-base.set`) node. Assign two fields: string `recipients` (e.g., email addresses) and number `days` (value: `7`). Connect the schedule trigger to it.
   - **Step 8:** Add a `Load the Week` (`n8n-nodes-base.dataTable`) node. Operation: `get`. Add filters matching date ranges using expressions:
     - `day` `>=` `={{ Number($now.minus({ days: $json.days }).toFormat('yyyyLLdd')) }}`
     - `day` `<` `={{ Number($now.toFormat('yyyyLLdd')) }}`
     - Connect `Report Settings` to it.
   - **Step 9:** Add a `Group by Team` (`n8n-nodes-base.code`) node. Insert aggregation JavaScript code to sort days and construct team summary blocks. Connect `Load the Week` to it.
   - **Step 10:** Add an `OpenAI GPT-5 Mini` (`@n8n/n8n-nodes-langchain.lmChatOpenAi`) node. Configure model name to `gpt-5-mini`. Add your OpenAI API credentials.
   - **Step 11:** Add a `Write the Weekly Report` (`@n8n/n8n-nodes-langchain.chainLlm`) node. Set prompt text to `=Team: {{ $json.team }}. {{ $json.days }} standups from {{ $json.from }} to {{ $json.to }}.\n\n{{ $json.week }}` and configure the system message template for report generation. Connect `Group by Team` to main input and link `OpenAI GPT-5 Mini` to the `ai_languageModel` input.
   - **Step 12:** Add a `Format as HTML` (`n8n-nodes-base.markdown`) node. Set mode to `markdownToHtml` and markdown expression to `={{ $json.text }}`. Connect `Write the Weekly Report` to it.
   - **Step 13:** Add an `Email Stakeholders` (`n8n-nodes-base.gmail`) node. Add Gmail OAuth2 credentials. Set email type to `html`, `Send To` to `={{ $('Report Settings').first().json.recipients }}`, subject to `=Weekly standup report: {{ $('Group by Team').item.json.team }}, {{ $('Group by Team').item.json.from }} to {{ $('Group by Team').item.json.to }}`, and message body to `={{ $json.html }}`. Connect `Format as HTML` to it.
   - **Step 14:** Add a `Prune Old Days` (`n8n-nodes-base.dataTable`) node. Select `standup_days`, operation `deleteRows`, with filter where `day` `<` `={{ Number($now.minus({ weeks: 4 }).toFormat('yyyyLLdd')) }}`. Set `executeOnce` to true. Connect `Email Stakeholders` to it.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| TeamRetro Integration Guide | Enable webhook integration in TeamRetro under Settings → Integrations → Webhooks, select **Standup Completed**, and provide the production webhook URL. |
| OpenAI Model Selection | Ensure the configured OpenAI account has active access and quota for the selected model (`gpt-5-mini`). |
Send weekly NUACOM call analysis reports with OpenAI and Gmail

https://n8nworkflows.xyz/workflows/send-weekly-nuacom-call-analysis-reports-with-openai-and-gmail-20953


# Send weekly NUACOM call analysis reports with OpenAI and Gmail

### 1. Workflow Overview

This workflow automates the weekly generation and delivery of call analytics and transcription insights for NUACOM telephony extensions. Operating on a scheduled interval, it aggregates phone call data, extracts AI-driven transcriptions, processes the collection through an OpenAI language model, and emails an executive HTML summary.

The workflow logic is categorized into the following functional blocks:
- **1.1 Schedule Trigger & Data Retrieval:** Initiates the workflow every Monday and fetches the previous week’s call logs for targeted extensions from NUACOM.
- **1.2 Filtering & Individual Call Enrichment:** Splits the call array into individual items, filters out short or unanswered calls, and requests auxiliary AI call data (summaries/transcriptions) for each valid call.
- **1.3 Data Compaction & AI Processing:** Aggregates and compacts the processed call items into a single consolidated payload, which is then fed into an advanced OpenAI-powered LangChain Agent configured as a sales-call analyst.
- **1.4 Report Distribution:** Converts the AI agent's HTML response into an outbound email delivered via Gmail to designated stakeholders.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Schedule Trigger & Data Retrieval
**Overview:** This block establishes the schedule for the automation and queries the NUACOM API for the complete set of call logs recorded during the preceding week across defined extensions.

**Nodes Involved:**
- `Every Monday`
- `Get Many Calls (Prev Week)`

**Node Details:**
- **Every Monday**
  - **Type and Technical Role:** `n8n-nodes-base.scheduleTrigger` (v1.4). Acts as the primary workflow entry point.
  - **Configuration Choices:** Configured to trigger weekly on Mondays at 08:00 AM.
  - **Key Expressions or Variables:** None.
  - **Input and Output Connections:** Input: None (Trigger). Output: Connects to `Get Many Calls (Prev Week)`.
  - **Version-specific Requirements:** v1.4.
  - **Edge Cases or Potential Failure Types:** Timezone misconfigurations on the n8n instance can shift execution times.

- **Get Many Calls (Prev Week)**
  - **Type and Technical Role:** `@nuacom/n8n-nodes-nuacom.nuacom` (v1). Fetches multiple call logs from NUACOM based on date ranges and extension filters.
  - **Configuration Choices:** 
    - `Resource`: Calls (`calls`).
    - `Calls Extension`: `24, 33, 64`.
    - `Calls Per Page`: `1000`.
    - `Calls Date From`: Evaluated dynamically for the start of the previous week.
    - `Calls Date To`: Evaluated dynamically for the end of the previous week.
  - **Key Expressions or Variables:**
    - `callsDateFrom`: `={{ $now.startOf('week').minus({ weeks: 1 }).toFormat('yyyy-MM-dd') }}`
    - `callsDateTo`: `={{ $now.startOf('week').minus({ days: 1 }).toFormat('yyyy-MM-dd') }}`
  - **Input and Output Connections:** Input: `Every Monday`. Output: Connects to `Split Out Calls`.
  - **Credentials:** Uses NUACOM API credentials (`Stathis 002`).
  - **Edge Cases or Potential Failure Types:** API rate limits, network timeouts, or invalid date formats if NUACOM endpoint updates occur.

---

#### Block 1.2: Filtering & Individual Call Enrichment
**Overview:** This block deconstructs the batch response from NUACOM, isolates answered calls exceeding a threshold duration, and retrieves specialized AI analysis records for each matching call.

**Nodes Involved:**
- `Split Out Calls`
- `Answered Calls Only`
- `Get AI Call Data`

**Node Details:**
- **Split Out Calls**
  - **Type and Technical Role:** `n8n-nodes-base.splitOut` (v1). Flattens the array returned inside the NUACOM response payload (`_embedded`) into individual items.
  - **Configuration Choices:** Field to split out: `_embedded`.
  - **Input and Output Connections:** Input: `Get Many Calls (Prev Week)`. Output: Connects to `Answered Calls Only`.
  - **Edge Cases or Potential Failure Types:** Fails if the API returns an empty or unexpected payload structure lacking the `_embedded` array.

- **Answered Calls Only**
  - **Type and Technical Role:** `n8n-nodes-base.filter` (v2.3). Filters items based on billable duration to discard missed, rejected, or very short interactions.
  - **Configuration Choices:** Condition set to check if billable seconds (`billsec`) are greater than 40.
  - **Key Expressions or Variables:**
    - Left Value: `={{ $json.billsec }}`
    - Operator: `number >`
    - Right Value: `40`
  - **Input and Output Connections:** Input: `Split Out Calls`. Output: Connects to `Get AI Call Data`.
  - **Edge Cases or Potential Failure Types:** Expression errors if `billsec` property is missing or typed incorrectly in the source payload.

- **Get AI Call Data**
  - **Type and Technical Role:** `@nuacom/n8n-nodes-nuacom.nuacom` (v1). Retrieves AI call intelligence, summaries, and transcriptions per call identifier.
  - **Configuration Choices:**
    - `Resource`: NUACOM AI (`nuacomAi`).
    - Error Handling: Configured to continue execution on failure (`onError: continueRegularOutput`).
  - **Key Expressions or Variables:**
    - `aiCallId`: `={{ $json.call_id }}`
  - **Input and Output Connections:** Input: `Answered Calls Only`. Output: Connects to `Compact & Group Calls`.
  - **Credentials:** Uses NUACOM account credentials (`NUACOM account`).
  - **Edge Cases or Potential Failure Types:** 404 errors or missing AI records if NUACOM AI analysis is disabled or incomplete for specific calls. Handled gracefully via `continueRegularOutput`.

---

#### Block 1.3: Data Compaction & AI Processing
**Overview:** This block compresses the enriched individual call objects, bundles them into a single chronological payload, and leverages an OpenAI chat model via an AI Agent to extract structured sales insights.

**Nodes Involved:**
- `Compact & Group Calls`
- `Analyze Call Transcriptions`
- `OpenAI Chat Model`

**Node Details:**
- **Compact & Group Calls**
  - **Type and Technical Role:** `n8n-nodes-base.code` (v2). Custom JavaScript node that sanitizes fields, extracts nested transcription/summary attributes, drops empty keys, and aggregates all items into a single container payload.
  - **Configuration Choices:** Custom script iterating through `$input.all()`, applying a helper lookup function (`pick`), and generating a consolidated output object containing call counts, date ranges, and the `calls` array.
  - **Input and Output Connections:** Input: `Get AI Call Data`. Output: Connects to `Analyze Call Transcriptions`.
  - **Edge Cases or Potential Failure Types:** JavaScript runtime errors if input payload structures deviate from expected schemas.

- **Analyze Call Transcriptions**
  - **Type and Technical Role:** `@n8n/n8n-nodes-langchain.agent` (v3.1). AI Agent node executing reasoning loops over structured data.
  - **Configuration Choices:**
    - Prompt Type: Define prompt.
    - Max Iterations: `10`.
    - System Message: Instructs the model to act as a sales-call analyst for NUACOM, evaluating prospect pain points, competitor mentions, objections, and qualifying questions coverage, returning raw HTML markup (without HTML/body wrappers).
  - **Key Expressions or Variables:**
    - Text: `=Analyze the following call AI-analysis data for the period {{ $json.periodFrom }} to {{ $json.periodTo }}. Number of calls: {{ $json.count }}.\n\nCALL DATA (JSON):\n{{ JSON.stringify($json.calls) }}`
  - **Input and Output Connections:** 
    - Inputs: Main input from `Compact & Group Calls`; Language model connection from `OpenAI Chat Model`.
    - Output: Connects to `Send Report Email`.
  - **Edge Cases or Potential Failure Types:** LLM token-limit exhaustion if call volumes are excessively high; API timeouts.

- **OpenAI Chat Model**
  - **Type and Technical Role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.3). Provides the underlying language model infrastructure to the AI Agent.
  - **Configuration Choices:** Model configured to `gpt-5.6-luna`.
  - **Input and Output Connections:** Output: Connects to `Analyze Call Transcriptions` via `ai_languageModel`.
  - **Credentials:** Uses OpenAI API credentials (`n8n free OpenAI API credits`).
  - **Edge Cases or Potential Failure Types:** OpenAI rate limits, billing limits, or upstream service outages.

---

#### Block 1.4: Report Distribution
**Overview:** This block takes the HTML analysis generated by the AI agent and dispatches it via email using Gmail OAuth2 credentials.

**Nodes Involved:**
- `Send Report Email`

**Node Details:**
- **Send Report Email**
  - **Type and Technical Role:** `n8n-nodes-base.gmail` (v2.2). Sends an email via the Gmail API.
  - **Configuration Choices:**
    - `Send To`: `user@example.com`
    - `Message`: HTML output generated by the AI agent.
    - `Subject`: Dynamic subject string including period bounds.
  - **Key Expressions or Variables:**
    - Message: `={{ $json.output }}`
    - Subject: `=Weekly Call Analysis Report ({{ $('Compact & Group Calls').item.json.periodFrom }} to {{ $('Compact & Group Calls').item.json.periodTo }})`
  - **Input and Output Connections:** Input: `Analyze Call Transcriptions`. Output: None (Terminal node).
  - **Credentials:** Uses Gmail OAuth2 credentials (`Gmail account`).
  - **Edge Cases or Potential Failure Types:** OAuth2 token expiration, invalid recipient address, or exceeding Gmail daily sending quotas.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Every Monday | n8n-nodes-base.scheduleTrigger | Triggers workflow weekly on Mondays at 08:00 | None | Get Many Calls (Prev Week) | Get a periodic call analysis report from NUACOM calls<br><br>### How it works<br>1. Every selected period (here weekly), the workflow analyzes calls in NUACOM<br>2. AI gets the calls and scores them acting as a sales manager and trying to get information about pains, issues in selling and general data of the call.<br>3. Once the analysis is completed, it sends the result to an email with Gmail.<br>### Setup steps<br>1. Set up the trigger by selecting when this workflow you need to excecute.<br>2. Enable AI call analysis in NUACOM to get the summary and the transcription of the call automatically.<br>3. Get and filter a specific number of extensions or numbers to get the call data from.<br>4. Set the prompt for the AI agent to score the call based on your requirements.<br>5. Declare the email where this report need to get delivered to.<br><br><br>### Customization<br>You can customise the workflow by:<br>1. Taking out the filter for specific numbers or checking for all NUACOM calls.<br>2. Changing the AI agent scoring behavior.<br>3. Deliver the report to a document, 3rd party system like CRMs instead of an email.<br><br>Main Trigger setup<br><br>Here you set the time/period you want this workflow to run automatically. |
| Get Many Calls (Prev Week) | @nuacom/n8n-nodes-nuacom.nuacom | Fetches call logs for previous week and specified extensions | Every Monday | Split Out Calls | Get a periodic call analysis report from NUACOM calls<br><br>### How it works<br>1. Every selected period (here weekly), the workflow analyzes calls in NUACOM<br>2. AI gets the calls and scores them acting as a sales manager and trying to get information about pains, issues in selling and general data of the call.<br>3. Once the analysis is completed, it sends the result to an email with Gmail.<br>### Setup steps<br>1. Set up the trigger by selecting when this workflow you need to excecute.<br>2. Enable AI call analysis in NUACOM to get the summary and the transcription of the call automatically.<br>3. Get and filter a specific number of extensions or numbers to get the call data from.<br>4. Set the prompt for the AI agent to score the call based on your requirements.<br>5. Declare the email where this report need to get delivered to.<br><br><br>### Customization<br>You can customise the workflow by:<br>1. Taking out the filter for specific numbers or checking for all NUACOM calls.<br>2. Changing the AI agent scoring behavior.<br>3. Deliver the report to a document, 3rd party system like CRMs instead of an email.<br><br>Getting calls from NUACOM<br><br>Here you set which calls you need to take into consideration. You can selecte specific numbers or extensions. You can also filter out calls with small duration to have a better report results. |
| Split Out Calls | n8n-nodes-base.splitOut | Flattens the call array into separate items | Get Many Calls (Prev Week) | Answered Calls Only | Get a periodic call analysis report from NUACOM calls<br><br>### How it works<br>1. Every selected period (here weekly), the workflow analyzes calls in NUACOM<br>2. AI gets the calls and scores them acting as a sales manager and trying to get information about pains, issues in selling and general data of the call.<br>3. Once the analysis is completed, it sends the result to an email with Gmail.<br>### Setup steps<br>1. Set up the trigger by selecting when this workflow you need to excecute.<br>2. Enable AI call analysis in NUACOM to get the summary and the transcription of the call automatically.<br>3. Get and filter a specific number of extensions or numbers to get the call data from.<br>4. Set the prompt for the AI agent to score the call based on your requirements.<br>5. Declare the email where this report need to get delivered to.<br><br><br>### Customization<br>You can customise the workflow by:<br>1. Taking out the filter for specific numbers or checking for all NUACOM calls.<br>2. Changing the AI agent scoring behavior.<br>3. Deliver the report to a document, 3rd party system like CRMs instead of an email.<br><br>Getting calls from NUACOM<br><br>Here you set which calls you need to take into consideration. You can selecte specific numbers or extensions. You can also filter out calls with small duration to have a better report results. |
| Answered Calls Only | n8n-nodes-base.filter | Filters out calls under 40 seconds duration | Split Out Calls | Get AI Call Data | Get a periodic call analysis report from NUACOM calls<br><br>### How it works<br>1. Every selected period (here weekly), the workflow analyzes calls in NUACOM<br>2. AI gets the calls and scores them acting as a sales manager and trying to get information about pains, issues in selling and general data of the call.<br>3. Once the analysis is completed, it sends the result to an email with Gmail.<br>### Setup steps<br>1. Set up the trigger by selecting when this workflow you need to excecute.<br>2. Enable AI call analysis in NUACOM to get the summary and the transcription of the call automatically.<br>3. Get and filter a specific number of extensions or numbers to get the call data from.<br>4. Set the prompt for the AI agent to score the call based on your requirements.<br>5. Declare the email where this report need to get delivered to.<br><br><br>### Customization<br>You can customise the workflow by:<br>1. Taking out the filter for specific numbers or checking for all NUACOM calls.<br>2. Changing the AI agent scoring behavior.<br>3. Deliver the report to a document, 3rd party system like CRMs instead of an email.<br><br>Getting calls from NUACOM<br><br>Here you set which calls you need to take into consideration. You can selecte specific numbers or extensions. You can also filter out calls with small duration to have a better report results. |
| Get AI Call Data | @nuacom/n8n-nodes-nuacom.nuacom | Retrieves AI transcriptions and summaries per call | Answered Calls Only | Compact & Group Calls | Get a periodic call analysis report from NUACOM calls<br><br>### How it works<br>1. Every selected period (here weekly), the workflow analyzes calls in NUACOM<br>2. AI gets the calls and scores them acting as a sales manager and trying to get information about pains, issues in selling and general data of the call.<br>3. Once the analysis is completed, it sends the result to an email with Gmail.<br>### Setup steps<br>1. Set up the trigger by selecting when this workflow you need to excecute.<br>2. Enable AI call analysis in NUACOM to get the summary and the transcription of the call automatically.<br>3. Get and filter a specific number of extensions or numbers to get the call data from.<br>4. Set the prompt for the AI agent to score the call based on your requirements.<br>5. Declare the email where this report need to get delivered to.<br><br><br>### Customization<br>You can customise the workflow by:<br>1. Taking out the filter for specific numbers or checking for all NUACOM calls.<br>2. Changing the AI agent scoring behavior.<br>3. Deliver the report to a document, 3rd party system like CRMs instead of an email.<br><br>AI Setup<br><br>Here you prepare the data to get fed on the AI agent. In our example we kept specific data from the call. Also in thi step you can set the behavior and the scoring settings of the AI agent that will analyze the call. |
| Compact & Group Calls | n8n-nodes-base.code | Compacts call attributes and combines into a single payload | Get AI Call Data | Analyze Call Transcriptions | Get a periodic call analysis report from NUACOM calls<br><br>### How it works<br>1. Every selected period (here weekly), the workflow analyzes calls in NUACOM<br>2. AI gets the calls and scores them acting as a sales manager and trying to get information about pains, issues in selling and general data of the call.<br>3. Once the analysis is completed, it sends the result to an email with Gmail.<br>### Setup steps<br>1. Set up the trigger by selecting when this workflow you need to excecute.<br>2. Enable AI call analysis in NUACOM to get the summary and the transcription of the call automatically.<br>3. Get and filter a specific number of extensions or numbers to get the call data from.<br>4. Set the prompt for the AI agent to score the call based on your requirements.<br>5. Declare the email where this report need to get delivered to.<br><br><br>### Customization<br>You can customise the workflow by:<br>1. Taking out the filter for specific numbers or checking for all NUACOM calls.<br>2. Changing the AI agent scoring behavior.<br>3. Deliver the report to a document, 3rd party system like CRMs instead of an email.<br><br>AI Setup<br><br>Here you prepare the data to get fed on the AI agent. In our example we kept specific data from the call. Also in thi step you can set the behavior and the scoring settings of the AI agent that will analyze the call. |
| Analyze Call Transcriptions | @n8n/n8n-nodes-langchain.agent | AI agent generating structured HTML insights report | Compact & Group Calls (and OpenAI Chat Model) | Send Report Email | Get a periodic call analysis report from NUACOM calls<br><br>### How it works<br>1. Every selected period (here weekly), the workflow analyzes calls in NUACOM<br>2. AI gets the calls and scores them acting as a sales manager and trying to get information about pains, issues in selling and general data of the call.<br>3. Once the analysis is completed, it sends the result to an email with Gmail.<br>### Setup steps<br>1. Set up the trigger by selecting when this workflow you need to excecute.<br>2. Enable AI call analysis in NUACOM to get the summary and the transcription of the call automatically.<br>3. Get and filter a specific number of extensions or numbers to get the call data from.<br>4. Set the prompt for the AI agent to score the call based on your requirements.<br>5. Declare the email where this report need to get delivered to.<br><br><br>### Customization<br>You can customise the workflow by:<br>1. Taking out the filter for specific numbers or checking for all NUACOM calls.<br>2. Changing the AI agent scoring behavior.<br>3. Deliver the report to a document, 3rd party system like CRMs instead of an email.<br><br>AI Setup<br><br>Here you prepare the data to get fed on the AI agent. In our example we kept specific data from the call. Also in thi step you can set the behavior and the scoring settings of the AI agent that will analyze the call. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides language model (gpt-5.6-luna) to AI Agent | None | Analyze Call Transcriptions | Get a periodic call analysis report from NUACOM calls<br><br>### How it works<br>1. Every selected period (here weekly), the workflow analyzes calls in NUACOM<br>2. AI gets the calls and scores them acting as a sales manager and trying to get information about pains, issues in selling and general data of the call.<br>3. Once the analysis is completed, it sends the result to an email with Gmail.<br>### Setup steps<br>1. Set up the trigger by selecting when this workflow you need to excecute.<br>2. Enable AI call analysis in NUACOM to get the summary and the transcription of the call automatically.<br>3. Get and filter a specific number of extensions or numbers to get the call data from.<br>4. Set the prompt for the AI agent to score the call based on your requirements.<br>5. Declare the email where this report need to get delivered to.<br><br><br>### Customization<br>You can customise the workflow by:<br>1. Taking out the filter for specific numbers or checking for all NUACOM calls.<br>2. Changing the AI agent scoring behavior.<br>3. Deliver the report to a document, 3rd party system like CRMs instead of an email.<br><br>AI Setup<br><br>Here you prepare the data to get fed on the AI agent. In our example we kept specific data from the call. Also in thi step you can set the behavior and the scoring settings of the AI agent that will analyze the call. |
| Send Report Email | n8n-nodes-base.gmail | Sends the final HTML report via Gmail | Analyze Call Transcriptions | None | Get a periodic call analysis report from NUACOM calls<br><br>### How it works<br>1. Every selected period (here weekly), the workflow analyzes calls in NUACOM<br>2. AI gets the calls and scores them acting as a sales manager and trying to get information about pains, issues in selling and general data of the call.<br>3. Once the analysis is completed, it sends the result to an email with Gmail.<br>### Setup steps<br>1. Set up the trigger by selecting when this workflow you need to excecute.<br>2. Enable AI call analysis in NUACOM to get the summary and the transcription of the call automatically.<br>3. Get and filter a specific number of extensions or numbers to get the call data from.<br>4. Set the prompt for the AI agent to score the call based on your requirements.<br>5. Declare the email where this report need to get delivered to.<br><br><br>### Customization<br>You can customise the workflow by:<br>1. Taking out the filter for specific numbers or checking for all NUACOM calls.<br>2. Changing the AI agent scoring behavior.<br>3. Deliver the report to a document, 3rd party system like CRMs instead of an email.<br><br>Receive the results<br><br>In the last step you setup where the results need to get delivered. In this example we have chosen over Email using Gmail. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger:**
   - Add a **Schedule Trigger** node (`Every Monday`).
   - Set interval configuration to trigger weekly on Mondays at 08:00.

2. **Add NUACOM Call Retrieval Node:**
   - Add a **NUACOM** node (`Get Many Calls (Prev Week)`).
   - Configure resource to `Calls`.
   - Set extensions filter (e.g., `24, 33, 64`) and pagination limit (`1000`).
   - Set dynamic expressions for `callsDateFrom` (`={{ $now.startOf('week').minus({ weeks: 1 }).toFormat('yyyy-MM-dd') }}`) and `callsDateTo` (`={{ $now.startOf('week').minus({ days: 1 }).toFormat('yyyy-MM-dd') }}`).
   - Connect `Every Monday` output to this node.
   - Configure NUACOM API credentials.

3. **Add Split Out Node:**
   - Add a **Split Out** node (`Split Out Calls`).
   - Set the field to split out to `_embedded`.
   - Connect `Get Many Calls (Prev Week)` output to this node.

4. **Add Filter Node:**
   - Add a **Filter** node (`Answered Calls Only`).
   - Configure a number condition: check if `={{ $json.billsec }}` is greater than `40`.
   - Connect `Split Out Calls` output to this node.

5. **Add NUACOM AI Retrieval Node:**
   - Add a **NUACOM** node (`Get AI Call Data`).
   - Set resource to `nuacomAi`.
   - Set `aiCallId` to `={{ $json.call_id }}`.
   - Set error handling parameter `onError` to `continueRegularOutput`.
   - Connect `Answered Calls Only` output to this node.
   - Configure NUACOM API credentials.

6. **Add Data Compaction Code Node:**
   - Add a **Code** node (`Compact & Group Calls`).
   - Paste the JavaScript snippet that sanitizes call items and groups them into a single item containing `count`, `periodFrom`, `periodTo`, and the `calls` array.
   - Connect `Get AI Call Data` output to this node.

7. **Add OpenAI Language Model Node:**
   - Add an **OpenAI Chat Model** node.
   - Set the model to `gpt-5.6-luna`.
   - Configure OpenAI API credentials.

8. **Add AI Agent Node:**
   - Add an **AI Agent** node (`Analyze Call Transcriptions`).
   - Set prompt type to define.
   - Set maximum iterations to `10`.
   - Set system message instructing the agent to analyze prospect pain points, competitor mentions, objections, and qualifying questions coverage, returning raw HTML markup.
   - Set input text expression to parse JSON call data and period bounds from `Compact & Group Calls`.
   - Connect `Compact & Group Calls` output to the main input of this node.
   - Connect `OpenAI Chat Model` output to the `ai_languageModel` input of this node.

9. **Add Gmail Distribution Node:**
   - Add a **Gmail** node (`Send Report Email`).
   - Configure `Send To` recipient address.
   - Set subject expression to include period bounds: `=Weekly Call Analysis Report ({{ $('Compact & Group Calls').item.json.periodFrom }} to {{ $('Compact & Group Calls').item.json.periodTo }})`.
   - Set message expression to `={{ $json.output }}`.
   - Connect `Analyze Call Transcriptions` output to this node.
   - Configure Gmail OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Automation workflow for weekly NUACOM call analysis via OpenAI and Gmail | Provided by n8n workflow export specifications |
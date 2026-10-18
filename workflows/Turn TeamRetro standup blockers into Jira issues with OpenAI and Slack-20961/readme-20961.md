Turn TeamRetro standup blockers into Jira issues with OpenAI and Slack

https://n8nworkflows.xyz/workflows/turn-teamretro-standup-blockers-into-jira-issues-with-openai-and-slack-20961


# Turn TeamRetro standup blockers into Jira issues with OpenAI and Slack

### 1. Workflow Overview

This workflow automates the tracking, triaging, and lifecycle management of blockers reported during team standups in TeamRetro. Its primary purpose is to ensure that engineering blockers are not lost in chat logs or meeting notes, turning actionable items into tracked Jira issues, nudging stakeholders when issues remain unresolved, and cleaning up records when blockers are cleared.

The workflow logic is categorized into the following functional blocks:
- **1.1 Input Reception & Configuration:** Captures the completed standup event, prevents duplicate webhook executions, and defines global environment settings.
- **1.2 Data State Assessment & Sorting:** Fetches existing open blocker mappings from an n8n Data Table and categorizes current standup updates into **Carried Over**, **Cleared**, or **New**.
- **1.3 Blocker Lifecycle Management (Carried Over & Cleared):** Updates the metadata for ongoing blockers, evaluates whether a stale notification (nudge) is due, comments on or deletes tracked records, and adds closing comments in Jira.
- **1.4 AI-Powered Triage & Ticketing:** Evaluates new blockers using an OpenAI language model, parses the structured categorization and actionability, creates corresponding Jira issues for actionable items, stores the record mappings, and announces them in Slack.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration
**Overview:** This block acts as the entry point for the integration, receiving webhook payloads from TeamRetro when a standup session concludes, filtering out retry duplicates, and loading runtime variables.

- **When Standup Closes**
  - *Type & Technical Role:* `n8n-nodes-teamretro.teamRetroTrigger` (Trigger Node)
  - *Configuration:* Listens for `standup.completed` events using a configured signing secret.
  - *Key Expressions:* None (Webhook payload source).
  - *Input / Output Connections:* Output connects to *Skip Repeat Deliveries*.
  - *Failure Types:* Authentication/Signature mismatch errors if the webhook signing secret is invalid.

- **Skip Repeat Deliveries**
  - *Type & Technical Role:* `n8n-nodes-base.removeDuplicates` (Data Processing Node)
  - *Configuration:* Removes items seen in previous executions based on the delivery identifier.
  - *Key Expressions:* `={{ $json.deliveryId }}`
  - *Input / Output Connections:* Input from *When Standup Closes*; output connects to *Settings*.
  - *Failure Types:* None specific.

- **Settings**
  - *Type & Technical Role:* `n8n-nodes-base.set` (Parameter Assignment Node)
  - *Configuration:* Establishes global workflow variables.
  - *Key Expressions/Variables:* Assigns `jiraUrl` (string), `teamChannel` (string), `staleAfter` (number, default: 3), and `blockerTopic` (string, default: "block").
  - *Input / Output Connections:* Input from *Skip Repeat Deliveries*; output connects to *Load Open Issues*.
  - *Failure Types:* None.

---

#### Block 1.2: Data State Assessment & Sorting
**Overview:** This block queries the persistent data layer to retrieve previously logged open blockers for the active meeting series and executes a custom JavaScript function to sort current blockers into states.

- **Load Open Issues**
  - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database Integration Node)
  - *Configuration:* Retrieves all rows matching the current meeting series ID from the `standup_blockers` Data Table, configured with `alwaysOutputData: true`.
  - *Key Expressions:* `={{ $('When Standup Closes').first().json.data.standup.meetingSeriesId }}`
  - *Input / Output Connections:* Input from *Settings*; output connects to *Sort the Blockers*.
  - *Failure Types:* Data Table connection errors or missing table configuration.

- **Sort the Blockers**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Custom JavaScript Node)
  - *Configuration:* Evaluates incoming standup topics against the `blockerTopic` filter, matches updates against open data table records using IDs or textual equality, and flags items as `new`, `carried`, or `cleared`.
  - *Key Expressions:* References `$('Settings').first().json` and `$('When Standup Closes').first().json`.
  - *Input / Output Connections:* Input from *Load Open Issues*; output connects to *New, Carried or Cleared?*.
  - *Failure Types:* Runtime exceptions if `meetingSeriesId` is missing from the standup payload.

- **New, Carried or Cleared?**
  - *Type & Technical Role:* `n8n-nodes-base.switch` (Routing Node)
  - *Configuration:* Evaluates the `kind` property of each processed item to route execution paths.
  - *Key Expressions:* `={{ $json.kind }}`
  - *Input / Output Connections:* Input from *Sort the Blockers*; outputs route to *Move It to Today* (Carried over), *Forget the Issue* (Cleared), and *Triage the Blocker* (New).
  - *Failure Types:* Unhandled categorization types resulting in dropped items.

---

#### Block 1.3: Blocker Lifecycle Management (Carried Over & Cleared)
**Overview:** Manages existing blockers that are either still unresolved or have been resolved by their respective owners during the standup.

- **Move It to Today**
  - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database Integration Node)
  - *Configuration:* Updates the `updateId` mapping for carried-over blockers in the `standup_blockers` table where `updateId` equals the row update ID.
  - *Key Expressions:* `={{ $('Sort the Blockers').item.json.text }}`, `={{ $('Sort the Blockers').item.json.rowUpdateId }}`
  - *Input / Output Connections:* Input from *New, Carried or Cleared?* (Carried over branch); output connects to *Due a Nudge?*.
  - *Failure Types:* Database write failures or row mismatch errors.

- **Due a Nudge?**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Node)
  - *Configuration:* Determines whether the stale counter meets the modulo threshold required to post a reminder comment.
  - *Key Expressions:* `={{ $('Sort the Blockers').item.json.dueNudge }}`
  - *Input / Output Connections:* Input from *Move It to Today*; output connects to *Comment on the Issue*.
  - *Failure Types:* None.

- **Comment on the Issue**
  - *Type & Technical Role:* `n8n-nodes-base.jira` (API Integration Node)
  - *Configuration:* Adds a comment to the tracked Jira issue indicating that the item remains unresolved. Configured with error handling (`onError: continueRegularOutput`).
  - *Key Expressions:* `={{ $('Sort the Blockers').item.json.issueKey }}`, constructs comment body using team, date, and open count.
  - *Input / Output Connections:* Input from *Due a Nudge?*; terminal node for this branch.
  - *Failure Types:* Jira API rate limits, invalid issue keys, or permission errors.

- **Forget the Issue**
  - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database Integration Node)
  - *Configuration:* Deletes database records from `standup_blockers` where `updateId` matches the cleared issue reference.
  - *Key Expressions:* `={{ $('Sort the Blockers').item.json.rowUpdateId }}`
  - *Input / Output Connections:* Input from *New, Carried or Cleared?* (Cleared branch); output connects to *Comment That It Cleared*.
  - *Failure Types:* Database unavailability.

- **Comment That It Cleared**
  - *Type & Technical Role:* `n8n-nodes-base.jira` (API Integration Node)
  - *Configuration:* Posts a final resolution comment to the Jira issue indicating the blocker has been cleared. Configured with error handling (`onError: continueRegularOutput`).
  - *Key Expressions:* `={{ $('Sort the Blockers').item.json.issueKey }}`
  - *Input / Output Connections:* Input from *Forget the Issue*; terminal node for this branch.
  - *Failure Types:* Jira API unavailability or invalid issue references.

---

#### Block 1.4: AI-Powered Triage & Ticketing
**Overview:** Processes newly identified blockers using a Large Language Model to categorize and structure them, creates tickets in Jira for actionable items, persists the new mapping, and broadcasts the alert to Slack.

- **Triage the Blocker**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (Advanced AI LangChain Node)
  - *Configuration:* Uses a system prompt to evaluate standup text, determine actionability, classify categories (`dependency`, `decision`, `access`, `technical`, `other`), and generate a concise issue title.
  - *Key Expressions:* `=Team: {{ $json.team }}. Raised by {{ $json.person }} at standup on {{ $json.date }}:\n\n{{ $json.text }}`
  - *Input / Output Connections:* Connected to *OpenAI GPT-5 Mini* (Language Model) and *Parse the Result* (Output Parser); input from *New, Carried or Cleared?* (New branch); output connects to *Actionable?*.
  - *Failure Types:* LLM timeout, malformed output, or API authentication errors.

- **OpenAI GPT-5 Mini**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Sub-node / Language Model)
  - *Configuration:* Configures the underlying OpenAI model (`gpt-5-mini`) for the LangChain chain.
  - *Input / Output Connections:* Connected to *Triage the Blocker* via `ai_languageModel`.

- **Parse the Result**
  - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (Sub-node / Output Parser)
  - *Configuration:* Enforces a strict JSON schema requiring boolean `actionable`, enum `category`, and string `title`.
  - *Input / Output Connections:* Connected to *Triage the Blocker* via `ai_outputParser`.

- **Actionable?**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Node)
  - *Configuration:* Evaluates whether the parsed AI response designates the blocker as actionable (`true`).
  - *Key Expressions:* `={{ $json.output.actionable }}`
  - *Input / Output Connections:* Input from *Triage the Blocker*; output connects to *Create the Jira Issue*.
  - *Failure Types:* Property resolution errors if the parser output structure changes.

- **Create the Jira Issue**
  - *Type & Technical Role:* `n8n-nodes-base.jira` (API Integration Node)
  - *Configuration:* Creates a new issue in the target Jira project with summary and description derived from the triaged output.
  - *Key Expressions:* `={{ $json.output.title }}`, description maps participant name, team, date, category, and original text.
  - *Input / Output Connections:* Input from *Actionable?*; output connects to *Remember the Issue*.
  - *Failure Types:* Missing project settings, invalid issue type configuration, or Jira API authentication failures.

- **Remember the Issue**
  - *Type & Technical Role:* `n8n-nodes-base.dataTable` (Database Integration Node)
  - *Configuration:* Inserts a new row into the `standup_blockers` Data Table mapping `updateId`, `issueKey`, `meetingSeriesId`, `personId`, and `text`.
  - *Key Expressions:* Maps values from previous execution nodes (e.g., `={{ $json.key }}` for issue key).
  - *Input / Output Connections:* Input from *Create the Jira Issue*; output connects to *Announce in Slack*.
  - *Failure Types:* Data Table write constraints or duplicate key errors.

- **Announce in Slack**
  - *Type & Technical Role:* `n8n-nodes-base.slack` (API Integration Node)
  - *Configuration:* Broadcasts the newly created Jira ticket link and title to the designated team channel.
  - *Key Expressions:* `={{ $('Settings').first().json.jiraUrl }}`, `={{ $('Settings').first().json.teamChannel }}`, ticket title and author details.
  - *Input / Output Connections:* Input from *Remember the Issue*; terminal node for this branch.
  - *Failure Types:* Slack channel permission errors, invalid channel names, or missing API credentials.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note** | `n8n-nodes-base.stickyNote` | Workflow documentation container | None | None | ## TeamRetro: Turn standup blockers into Jira issues and follow them until they clear<br><br>For engineering managers whose standup blockers get said once and lost. When a TeamRetro standup closes, OpenAI triages each new blocker and the actionable ones become Jira issues. A carried-over blocker is matched back to its issue, which gets a comment every `staleAfter` standups it stays open and a last one when it clears.<br><br>### How it works<br><br>1. The TeamRetro Trigger fires on **Standup Completed**. A delivery TeamRetro retries is skipped, so nothing happens twice.<br>2. Each update under your blocker question is sorted against the series' open issues.<br>3. **Carried over:** the issue moves to today's update, and Jira gets a comment when the blocker is overdue.<br>4. **Cleared:** its owner posted without carrying it over, so the row goes and Jira gets a comment.<br>5. **New:** OpenAI returns whether it is actionable, a category and an issue title. Actionable ones become Jira issues, are remembered in the `standup_blockers` Data Table, and are announced in Slack.<br><br>### Setup<br><br>- [ ] Create a Data Table named `standup_blockers` with columns: updateId, issueKey, meetingSeriesId, personId, text (strings)<br>- [ ] Copy the Production URL from "When Standup Closes" into TeamRetro under Settings → Integrations → Webhooks, tick **Standup Completed**, and paste the signing secret back into the trigger<br>- [ ] Set `jiraUrl`, `teamChannel` and `staleAfter` in "Settings"<br>- [ ] Connect OpenAI, Jira and Slack, and pick the Jira project and issue type<br><br>Sample data is pinned. Running the pinned sample a second time stops at "Skip Repeat Deliveries": that is the duplicate guard.<br><br>### Customization<br><br>Add labels or a priority under Additional Fields on "Create the Jira Issue". To close cleared issues, add a Jira transition after "Comment That It Cleared". |
| **Sticky Note1** | `n8n-nodes-base.stickyNote` | Block documentation for input | None | None | ## 1. Catch the closed standup<br><br>Fires when a standup closes. Retries, and a second run of the pinned sample, stop at the duplicate check. |
| **Sticky Note2** | `n8n-nodes-base.stickyNote` | Block documentation for sorting | None | None | ## 2. Sort new, carried over and cleared<br><br>This standup's open issues come out of the Data Table, and each blocker is matched against them. |
| **Sticky Note3** | `n8n-nodes-base.stickyNote` | Block documentation for AI triage | None | None | ## 5. Triage new blockers<br><br>One OpenAI call per blocker: actionable or not, a category and a short issue title. |
| **Sticky Note4** | `n8n-nodes-base.stickyNote` | Block documentation for ticketing | None | None | ## 6. Ticket and announce<br><br>Each actionable blocker becomes a Jira issue, is remembered against its update, and is announced in Slack. |
| **Sticky Note5** | `n8n-nodes-base.stickyNote` | Block documentation for carried over | None | None | ## 3. Follow carried-over blockers<br><br>The issue moves to today's update. Every `staleAfter` standups it is still open, Jira gets a comment. |
| **Sticky Note6** | `n8n-nodes-base.stickyNote` | Block documentation for cleared blockers | None | None | ## 4. Note cleared blockers<br><br>Its owner posted without carrying it over, so the Data Table forgets it and the issue gets a comment. |
| **When Standup Closes** | `n8n-nodes-teamretro.teamRetroTrigger` | Triggers workflow on standup completion | None | Skip Repeat Deliveries | ## 1. Catch the closed standup<br><br>Fires when a standup closes. Retries, and a second run of the pinned sample, stop at the duplicate check. |
| **Skip Repeat Deliveries** | `n8n-nodes-base.removeDuplicates` | Prevents duplicate processing via delivery ID | When Standup Closes | Settings | ## 1. Catch the closed standup<br><br>Fires when a standup closes. Retries, and a second run of the pinned sample, stop at the duplicate check. |
| **Settings** | `n8n-nodes-base.set` | Initializes global configuration variables | Skip Repeat Deliveries | Load Open Issues | ## 1. Catch the closed standup<br><br>Fires when a standup closes. Retries, and a second run of the pinned sample, stop at the duplicate check. |
| **Load Open Issues** | `n8n-nodes-base.dataTable` | Fetches open blockers from Data Table | Settings | Sort the Blockers | ## 2. Sort new, carried over and cleared<br><br>This standup's open issues come out of the Data Table, and each blocker is matched against them. |
| **Sort the Blockers** | `n8n-nodes-base.code` | Classifies blockers into new, carried, or cleared | Load Open Issues | New, Carried or Cleared? | ## 2. Sort new, carried over and cleared<br><br>This standup's open issues come out of the Data Table, and each blocker is matched against them. |
| **New, Carried or Cleared?** | `n8n-nodes-base.switch` | Routes execution paths based on blocker kind | Sort the Blockers | Move It to Today, Forget the Issue, Triage the Blocker | ## 2. Sort new, carried over and cleared<br><br>This standup's open issues come out of the Data Table, and each blocker is matched against them. |
| **Move It to Today** | `n8n-nodes-base.dataTable` | Updates tracked blocker reference row | New, Carried or Cleared? | Due a Nudge? | ## 3. Follow carried-over blockers<br><br>The issue moves to today's update. Every `staleAfter` standups it is still open, Jira gets a comment. |
| **Due a Nudge?** | `n8n-nodes-base.if` | Evaluates if stale reminder is required | Move It to Today | Comment on the Issue | ## 3. Follow carried-over blockers<br><br>The issue moves to today's update. Every `staleAfter` standups it is still open, Jira gets a comment. |
| **Comment on the Issue** | `n8n-nodes-base.jira` | Adds reminder comment to Jira issue | Due a Nudge? | None | ## 3. Follow carried-over blockers<br><br>The issue moves to today's update. Every `staleAfter` standups it is still open, Jira gets a comment. |
| **Forget the Issue** | `n8n-nodes-base.dataTable` | Deletes resolved blocker record | New, Carried or Cleared? | Comment That It Cleared | ## 4. Note cleared blockers<br><br>Its owner posted without carrying it over, so the Data Table forgets it and the issue gets a comment. |
| **Comment That It Cleared** | `n8n-nodes-base.jira` | Comments on Jira issue that blocker is cleared | Forget the Issue | None | ## 4. Note cleared blockers<br><br>Its owner posted without carrying it over, so the Data Table forgets it and the issue gets a comment. |
| **Triage the Blocker** | `@n8n/n8n-nodes-langchain.chainLlm` | AI chain for analyzing blocker text | New, Carried or Cleared? | Actionable? | ## 5. Triage new blockers<br><br>One OpenAI call per blocker: actionable or not, a category and a short issue title. |
| **OpenAI GPT-5 Mini** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Provides chat model for AI chain | None | Triage the Blocker | ## 5. Triage new blockers<br><br>One OpenAI call per blocker: actionable or not, a category and a short issue title. |
| **Parse the Result** | `@n8n/n8n-nodes-langchain.outputParserStructured` | Enforces structured JSON output for AI chain | None | Triage the Blocker | ## 5. Triage new blockers<br><br>One OpenAI call per blocker: actionable or not, a category and a short issue title. |
| **Actionable?** | `n8n-nodes-base.if` | Checks if AI marked blocker as actionable | Triage the Blocker | Create the Jira Issue | ## 6. Ticket and announce<br><br>Each actionable blocker becomes a Jira issue, is remembered against its update, and is announced in Slack. |
| **Create the Jira Issue** | `n8n-nodes-base.jira` | Generates new issue in Jira project | Actionable? | Remember the Issue | ## 6. Ticket and announce<br><br>Each actionable blocker becomes a Jira issue, is remembered against its update, and is announced in Slack. |
| **Remember the Issue** | `n8n-nodes-base.dataTable` | Stores new issue mapping in Data Table | Create the Jira Issue | Announce in Slack | ## 6. Ticket and announce<br><br>Each actionable blocker becomes a Jira issue, is remembered against its update, and is announced in Slack. |
| **Announce in Slack** | `n8n-nodes-base.slack` | Broadcasts ticket creation in Slack channel | Remember the Issue | None | ## 6. Ticket and announce<br><br>Each actionable blocker becomes a Jira issue, is remembered against its update, and is announced in Slack. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Data Table:**
   - In n8n, create a new Data Table named `standup_blockers` with the following columns (all set to type string): `updateId`, `issueKey`, `meetingSeriesId`, `personId`, and `text`.

2. **Add Node 1: When Standup Closes**
   - **Type:** `n8n-nodes-teamretro.teamRetroTrigger`
   - **Configuration:** Set event to `standup.completed`. Configure your TeamRetro webhook credentials and signing secret.

3. **Add Node 2: Skip Repeat Deliveries**
   - **Type:** `n8n-nodes-base.removeDuplicates`
   - **Configuration:** Set operation to *Remove Items Seen In Previous Executions*. Set Dedupe Value expression to `={{ $json.deliveryId }}`.
   - **Connection:** Connect output of *When Standup Closes* to this node.

4. **Add Node 3: Settings**
   - **Type:** `n8n-nodes-base.set`
   - **Configuration:** Add assignments for:
     - `jiraUrl` (string): `https://your-company.atlassian.net`
     - `teamChannel` (string): `#standup`
     - `staleAfter` (number): `3`
     - `blockerTopic` (string): `block`
   - **Connection:** Connect output of *Skip Repeat Deliveries* to this node.

5. **Add Node 4: Load Open Issues**
   - **Type:** `n8n-nodes-base.dataTable`
   - **Configuration:** Set operation to *Get*, Data Table mode to Name (`standup_blockers`), return all items. Add a filter condition where key `meetingSeriesId` equals `={{ $('When Standup Closes').first().json.data.standup.meetingSeriesId }}`. Enable *Always Output Data*.
   - **Connection:** Connect output of *Settings* to this node.

6. **Add Node 5: Sort the Blockers**
   - **Type:** `n8n-nodes-base.code`
   - **Configuration:** Insert JavaScript parsing logic to filter updates matching `blockerTopic`, map open data table items, and categorize items into `new`, `carried`, or `cleared` based on participation and carry-forward IDs.
   - **Connection:** Connect output of *Load Open Issues* to this node.

7. **Add Node 6: New, Carried or Cleared?**
   - **Type:** `n8n-nodes-base.switch`
   - **Configuration:** Set rule matching `={{ $json.kind }}` with output keys: `Carried over` (`carried`), `Cleared` (`cleared`), and `New` (`new`).
   - **Connection:** Connect output of *Sort the Blockers* to this node.

8. **Build Branch: Carried Over**
   - **Move It to Today:** Add Data Table node, operation *Update*, table `standup_blockers`, filter where `updateId` equals `={{ $('Sort the Blockers').item.json.rowUpdateId }}`, mapping text field. Connect from *New, Carried or Cleared?* ("Carried over" output).
   - **Due a Nudge?:** Add If node, condition expression `={{ $('Sort the Blockers').item.json.dueNudge }}` is true. Connect from *Move It to Today*.
   - **Comment on the Issue:** Add Jira node, resource *Issue Comment*, operation *Add*, issue key `={{ $('Sort the Blockers').item.json.issueKey }}`, and comment body template including team, date, and standups open. Set error handling to *Continue Regular Output*. Connect from *Due a Nudge?*.

9. **Build Branch: Cleared**
   - **Forget the Issue:** Add Data Table node, operation *Delete Rows*, table `standup_blockers`, filter where `updateId` equals `={{ $('Sort the Blockers').item.json.rowUpdateId }}`. Connect from *New, Carried or Cleared?* ("Cleared" output).
   - **Comment That It Cleared:** Add Jira node, resource *Issue Comment*, operation *Add*, issue key `={{ $('Sort the Blockers').item.json.issueKey }}`, comment body noting the blocker cleared. Set error handling to *Continue Regular Output*. Connect from *Forget the Issue*.

10. **Build Branch: New & AI Triage**
    - **Triage the Blocker:** Add Advanced AI LangChain Chain LLM node. Prompt: `=Team: {{ $json.team }}. Raised by {{ $json.person }} at standup on {{ $json.date }}:\n\n{{ $json.text }}`. System message: Define triage instructions (actionable boolean, category enum, title under 80 chars). Connect from *New, Carried or Cleared?* ("New" output).
    - **OpenAI GPT-5 Mini:** Add LM Chat OpenAI sub-node configured with model `gpt-5-mini`. Connect to AI Language Model input on *Triage the Blocker*.
    - **Parse the Result:** Add Structured Output Parser sub-node with manual JSON schema defining `actionable` (boolean), `category` (enum: dependency, decision, access, technical, other), and `title` (string). Connect to AI Output Parser input on *Triage the Blocker*.
    - **Actionable?:** Add If node, condition checking `={{ $json.output.actionable }}` is true. Connect from *Triage the Blocker*.
    - **Create the Jira Issue:** Add Jira node, select project and issue type, summary `={{ $json.output.title }}`, description formatting reporter, team, date, category, and text. Connect from *Actionable?*.
    - **Remember the Issue:** Add Data Table node, operation *Append (Insert)*, table `standup_blockers`, mapping `updateId`, `issueKey`, `meetingSeriesId`, `personId`, and `text` from upstream nodes. Connect from *Create the Jira Issue*.
    - **Announce in Slack:** Add Slack node, resource *Message*, operation *Post*, channel name `={{ $('Settings').first().json.teamChannel }}`, text template linking the created Jira issue and mentioning the reporter. Connect from *Remember the Issue*.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| TeamRetro standup blocker automation concept | Integration reference for engineering management workflows |
| OpenAI structured outputs and Jira API handling | Used for automated triage and issue management |
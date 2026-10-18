Post Slack standup digests and escalate stale blockers with TeamRetro and OpenAI

https://n8nworkflows.xyz/workflows/post-slack-standup-digests-and-escalate-stale-blockers-with-teamretro-and-openai-20864


# Post Slack standup digests and escalate stale blockers with TeamRetro and OpenAI

### 1. Workflow Overview

This workflow automates the process of capturing completed TeamRetro standups, processing their content through an AI summarization model, broadcasting a structured digest to a team Slack channel, and selectively escalating prolonged blockers to a team lead channel. 

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Guardrails:** Receives the webhook trigger from TeamRetro, filters out duplicate event deliveries, assigns configuration parameters, and checks whether participants have submitted updates.
- **1.2 AI Processing & Preparation:** Transforms raw update payloads into Slack-compatible markup, extracts blockers, calculates their stagnation duration, and generates a structured executive summary via OpenAI.
- **1.3 Slack Delivery & Threading:** Compiles the final message payloads and posts the core summary digest to the primary team channel, appending individual participant updates as a threaded reply.
- **1.4 Blocker Escalation:** Evaluates whether any active blockers have exceeded the configured staleness threshold and triggers an escalation alert to the team lead channel if necessary.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Guardrails
- **Overview:** This block acts as the secure entry point for the workflow. It captures completed standup events, discards retried webhook deliveries, injects baseline environment variables, and halts execution if no activity is recorded.
- **Nodes Involved:** 
  - `When Standup Closes`
  - `Skip Repeat Deliveries`
  - `Settings`
  - `Anyone Posted?`

- **Node Details:**
  - **When Standup Closes**
    - *Type & Technical Role:* `n8n-nodes-teamretro.teamRetroTrigger` (Trigger)
    - *Configuration Choices:* Listens for the `standup.completed` event using webhook integration. Requires a signing secret for validation.
    - *Key Expressions:* None (Webhook payload entry point).
    - *Input/Output:* Input: External webhook request. Output: Raw JSON payload containing standup metadata, participant updates, and delivery identifiers.
    - *Failure Types:* Invalid signing secret rejection, missing webhook configuration in TeamRetro.
  - **Skip Repeat Deliveries**
    - *Type & Technical Role:* `n8n-nodes-base.removeDuplicates` (Data Processing)
    - *Configuration Choices:* Deduplicates incoming items based on execution history.
    - *Key Expressions:* Dedupe value configured as `={{ $json.deliveryId }}`.
    - *Input/Output:* Input: `When Standup Closes`. Output: Distinct item passing only if `deliveryId` has not been processed previously.
    - *Failure Types:* State loss if execution history database clears unexpectedly.
  - **Settings**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Utility)
    - *Configuration Choices:* Sets static configuration parameters used downstream.
    - *Key Expressions:* Assigns `teamChannel` (`#standup`), `leadChannel` (`#team-leads`), `staleAfter` (`3`), and `blockerTopic` (`block`).
    - *Input/Output:* Input: `Skip Repeat Deliveries`. Output: Stream items enriched with configuration fields.
  - **Anyone Posted?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Evaluates whether the updates array contains at least one entry.
    - *Key Expressions:* Left value checks `={{ ($('When Standup Closes').first().json.data.standup.updates ?? []).length }}`, operator evaluates as greater than `0`.
    - *Input/Output:* Input: `Settings`. Output: True branch proceeds to preparation; false branch terminates the execution branch.

---

#### 2.2 AI Processing & Preparation
- **Overview:** This block prepares raw standup text for Slack rendering, identifies team blockers, computes staleness metrics, and interfaces with OpenAI to synthesize a concise, high-level overview.
- **Nodes Involved:** 
  - `Prepare the Standup`
  - `Summarise the Standup`
  - `OpenAI GPT-5 Mini`
  - `Parse the Result`

- **Node Details:**
  - **Prepare the Standup**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Executes JavaScript to sanitize markdown strings, format user entries, parse blockers, and compute stale escalation criteria.
    - *Key Expressions:* Reads configuration and trigger payload via `$('Settings').first().json` and `$('When Standup Closes').first().json`.
    - *Input/Output:* Input: `Anyone Posted?`. Output: Sanitized transcript string, detail string, array of blockers, and array of stale blockers.
    - *Failure Types:* Syntax errors in user-generated markdown breaking text replacement.
  - **Summarise the Standup**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.chainLlm` (AI Agent / Chain)
    - *Configuration Choices:* Uses a system message template instructing the model to generate a headline, highlights (max 3), and watch-outs (max 3).
    - *Key Expressions:* Text input uses `=Team: {{ $json.team }}. {{ $json.title }} on {{ $json.date }}, {{ $json.people }} people posted.\n\n{{ $json.transcript }}`.
    - *Input/Output:* Input: `Prepare the Standup`. Output: Unstructured text response directed to output parser.
    - *Failure Types:* LLM timeout, rate limit exceeded, API authentication failure.
  - **OpenAI GPT-5 Mini**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (AI Model Sub-node)
    - *Configuration Choices:* Configured to use the `gpt-5-mini` model.
    - *Input/Output:* Connected strictly as a language model dependency to `Summarise the Standup`.
    - *Credentials Required:* OpenAI API account credential.
  - **Parse the Result**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.outputParserStructured` (AI Output Parser Sub-node)
    - *Configuration Choices:* Enforces a strict JSON schema requiring `headline` (string), `highlights` (array of strings), and `watchouts` (array of strings).
    - *Input/Output:* Connected to `Summarise the Standup` via the `ai_outputParser` connection.

---

#### 2.3 Slack Delivery & Threading
- **Overview:** This block formats the AI-generated digest alongside extracted updates and posts them sequentially to the designated Slack team channel, ensuring detailed participant logs remain neatly threaded.
- **Nodes Involved:** 
  - `Compose the Messages`
  - `Post the Digest`
  - `Post Details in the Thread`

- **Node Details:**
  - **Compose the Messages**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Combines parsed AI output and formatted blocker lists into final message strings (`summary`, `details`, and `alert`).
    - *Key Expressions:* Extracts data from `Prepare the Standup` and `Parse the Result`.
    - *Input/Output:* Input: `Parse the Result`. Output: Consolidated message blocks for Slack dispatch.
  - **Post the Digest**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Integration)
    - *Configuration Choices:* Posts a channel message targeting the configured team channel.
    - *Key Expressions:* Text parameter evaluates to `={{ $json.summary }}`, channel ID resolves to `={{ $('Settings').first().json.teamChannel }}`.
    - *Input/Output:* Input: `Compose the Messages`. Output: Slack API response containing the message timestamp (`ts`) and channel reference (`channel`).
    - *Credentials Required:* Slack OAuth2 / Bot Token.
    - *Failure Types:* Invalid channel name, missing `chat:write` scopes.
  - **Post Details in the Thread**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Integration)
    - *Configuration Choices:* Posts a threaded reply to the message sent by the preceding node.
    - *Key Expressions:* Text uses `={{ $('Compose the Messages').first().json.details }}`, thread timestamp uses `={{ $json.ts }}`, channel uses `={{ $json.channel }}`.
    - *Input/Output:* Input: `Post the Digest`. Output: Confirmation payload of the thread reply.
    - *Failure Types:* Expired message timestamp reference, missing channel access.

---

#### 2.4 Blocker Escalation
- **Overview:** Evaluates whether any identified blockers satisfy the staleness conditions and routes an escalation alert to the designated team lead channel if applicable.
- **Nodes Involved:** 
  - `Any Stale Blockers?`
  - `Alert the Team Lead`

- **Node Details:**
  - **Any Stale Blockers?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration Choices:* Validates whether the alert payload string generated during composition is populated.
    - *Key Expressions:* Left value checks `={{ $('Compose the Messages').first().json.alert }}`, operator checks for `notEmpty`.
    - *Input/Output:* Input: `Post Details in the Thread`. Output: True branch routes to escalation; false branch terminates execution.
  - **Alert the Team Lead**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Integration)
    - *Configuration Choices:* Sends an urgent notification message to the team lead channel.
    - *Key Expressions:* Text parameter evaluates to `={{ $('Compose the Messages').first().json.alert }}`, channel ID evaluates to `={{ $('Settings').first().json.leadChannel }}`.
    - *Input/Output:* Input: `Any Stale Blockers?`. Output: Final execution confirmation.
    - *Credentials Required:* Slack OAuth2 / Bot Token.
    - *Failure Types:* Channel permission errors, incorrect channel identifier formatting.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## TeamRetro: Post an AI standup digest to Slack and escalate stale blockers... |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 1. Catch the closed standup<br>Fires when a standup closes... |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 2. Summarise with OpenAI<br>Updates become Slack text... |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 3. Post the digest and its thread<br>The digest goes to the team channel... |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 4. Escalate stale blockers<br>A blocker open `staleAfter` standups goes to the lead channel... |
| When Standup Closes | `n8n-nodes-teamretro.teamRetroTrigger` | Trigger | Webhook | Skip Repeat Deliveries | ## 1. Catch the closed standup<br>Fires when a standup closes... |
| Skip Repeat Deliveries | `n8n-nodes-base.removeDuplicates` | Data Processing | When Standup Closes | Settings | ## 1. Catch the closed standup<br>Fires when a standup closes... |
| Settings | `n8n-nodes-base.set` | Utility | Skip Repeat Deliveries | Anyone Posted? | ## 1. Catch the closed standup<br>Fires when a standup closes... |
| Anyone Posted? | `n8n-nodes-base.if` | Flow Control | Settings | Prepare the Standup | ## 1. Catch the closed standup<br>Fires when a standup closes... |
| Prepare the Standup | `n8n-nodes-base.code` | Data Transformation | Anyone Posted? | Summarise the Standup | ## 2. Summarise with OpenAI<br>Updates become Slack text... |
| Summarise the Standup | `@n8n/n8n-nodes-langchain.chainLlm` | AI Processing | Prepare the Standup | Compose the Messages | ## 2. Summarise with OpenAI<br>Updates become Slack text... |
| OpenAI GPT-5 Mini | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | AI Model Sub-node | None (Model link) | Summarise the Standup | ## 2. Summarise with OpenAI<br>Updates become Slack text... |
| Parse the Result | `@n8n/n8n-nodes-langchain.outputParserStructured` | AI Output Parser | None (Parser link) | Summarise the Standup | ## 2. Summarise with OpenAI<br>Updates become Slack text... |
| Compose the Messages | `n8n-nodes-base.code` | Data Transformation | Summarise the Standup | Post the Digest | ## 3. Post the digest and its thread<br>The digest goes to the team channel... |
| Post the Digest | `n8n-nodes-base.slack` | Integration | Compose the Messages | Post Details in the Thread | ## 3. Post the digest and its thread<br>The digest goes to the team channel... |
| Post Details in the Thread | `n8n-nodes-base.slack` | Integration | Post the Digest | Any Stale Blockers? | ## 3. Post the digest and its thread<br>The digest goes to the team channel... |
| Any Stale Blockers? | `n8n-nodes-base.if` | Flow Control | Post Details in the Thread | Alert the Team Lead | ## 4. Escalate stale blockers<br>A blocker open `staleAfter` standups goes to the lead channel... |
| Alert the Team Lead | `n8n-nodes-base.slack` | Integration | Any Stale Blockers? | None | ## 4. Escalate stale blockers<br>A blocker open `staleAfter` standups goes to the lead channel... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **TeamRetro Trigger** node named `When Standup Closes`.
   - Set the event parameter to `standup.completed`.
   - Configure your TeamRetro webhook URL to point to this production webhook and paste the signing secret.

2. **Add Deduplication Logic:**
   - Create a **Remove Duplicates** node named `Skip Repeat Deliveries`.
   - Set the operation to `Remove Items Seen in Previous Executions`.
   - Set the dedupe value expression to `={{ $json.deliveryId }}`.
   - Connect `When Standup Closes` to `Skip Repeat Deliveries`.

3. **Configure Environment Settings:**
   - Create a **Set (Edit Fields)** node named `Settings`.
   - Add assignments:
     - `teamChannel` (String): `#standup`
     - `leadChannel` (String): `#team-leads`
     - `staleAfter` (Number): `3`
     - `blockerTopic` (String): `block`
   - Connect `Skip Repeat Deliveries` to `Settings`.

4. **Add Activity Validation Gate:**
   - Create an **If** node named `Anyone Posted?`.
   - Set condition check: Left value `={{ ($('When Standup Closes').first().json.data.standup.updates ?? []).length }}`, operator `Larger Than`, Right value `0`.
   - Connect `Settings` to `Anyone Posted?`.

5. **Transform Standup Data:**
   - Create a **Code** node named `Prepare the Standup`.
   - Paste the standup-to-markdown parsing JavaScript logic (handling text sanitization, topic matching, and stale blocker calculation).
   - Connect the `true` output of `Anyone Posted?` to `Prepare the Standup`.

6. **Setup AI Summarization Chain:**
   - Create a **Basic LLM Chain** node named `Summarise the Standup`.
   - Configure the system prompt message template to instruct the AI to return a headline, highlights, and watch-outs based exclusively on the provided text.
   - Set text input expression to include team name, title, date, and transcript.
   - Connect an **OpenAI Chat Model** node (`OpenAI GPT-5 Mini`) to the language model input of the chain node, configuring your OpenAI API credentials.
   - Connect a **Structured Output Parser** node (`Parse the Result`) to the output parser input of the chain node, matching the JSON schema for `headline`, `highlights`, and `watchouts`.
   - Connect `Prepare the Standup` to `Summarise the Standup`.

7. **Compose Formatted Messages:**
   - Create a **Code** node named `Compose the Messages`.
   - Paste the message composition script to build the main digest, threaded details, and lead escalation strings.
   - Connect `Summarise the Standup` to `Compose the Messages`.

8. **Dispatch Slack Team Digest:**
   - Create a **Slack** node named `Post the Digest`.
   - Set resource to `Message`, action to `Post`.
   - Set Text to `={{ $json.summary }}` and Channel to `={{ $('Settings').first().json.teamChannel }}`.
   - Configure valid Slack credentials with chat permissions.
   - Connect `Compose the Messages` to `Post the Digest`.

9. **Reply in Thread:**
   - Create a **Slack** node named `Post Details in the Thread`.
   - Set Text to `={{ $('Compose the Messages').first().json.details }}`.
   - In additional options, enable `Thread Ts` and set its value to `={{ $json.ts }}`.
   - Connect `Post the Digest` to `Post Details in the Thread`.

10. **Evaluate Blocker Escalation:**
    - Create an **If** node named `Any Stale Blockers?`.
    - Set condition check: Left value `={{ $('Compose the Messages').first().json.alert }}`, operator `Is Not Empty`.
    - Connect `Post Details in the Thread` to `Any Stale Blockers?`.

11. **Send Escalation Alert:**
    - Create a **Slack** node named `Alert the Team Lead`.
    - Set Text to `={{ $('Compose the Messages').first().json.alert }}` and Channel to `={{ $('Settings').first().json.leadChannel }}`.
    - Connect the `true` output of `Any Stale Blockers?` to `Alert the Team Lead`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| TeamRetro standup automated Slack delivery design | Designed for engineering and product teams aiming to reduce standup reading fatigue by combining AI summaries with threaded updates and automated escalation for overdue blockers. |
| Pinned sample execution behavior | Sample execution payload is pinned in the workflow nodes. Running a second identical execution will automatically halt at the `Skip Repeat Deliveries` node as expected by the duplicate guard design. |
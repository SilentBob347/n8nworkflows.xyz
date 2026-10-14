Manage Google reviews with Claude Sonnet, Supabase, and Slack

https://n8nworkflows.xyz/workflows/manage-google-reviews-with-claude-sonnet--supabase--and-slack-20779


# Manage Google reviews with Claude Sonnet, Supabase, and Slack

### 1. Workflow Overview

The "AI Review & Reputation Management" workflow automates the retrieval, processing, analysis, approval routing, and logging of customer reviews for local businesses. It operates on a scheduled interval to keep review management synchronized with platforms like Google Business Profile.

The workflow logic is grouped into seven distinct functional blocks:

- **1.1 Input Reception & Deduplication:** Triggers periodically, fetches incoming reviews (using mock data by default), queries Supabase to check for existing records, and filters out already-processed items.
- **1.2 AI Analysis & Response Generation:** Loops through each unprocessed review, utilizes Claude Sonnet via an AI agent to analyze sentiment, urgency, topics, and intent, and drafts a tailored public response.
- **1.3 Sentiment Routing:** Evaluates the sentiment classification (positive, neutral, or negative) and assigns corresponding response strategy labels.
- **1.4 Escalation Check & Initial Logging:** Runs an additional AI escalation evaluation combined with deterministic rule-based safeguards (e.g., low ratings, sensitive keywords), compiles a comprehensive review record, and saves it to Supabase with a pending status.
- **1.5 Human Approval Gateway:** Evaluates whether human intervention is required. If flagged, it dispatches an interactive approval request to Slack and pauses execution for up to 24 hours awaiting a decision via resume webhook.
- **1.6 Resolution & Publishing:** Processes the approval decision (approve, reject, auto, or timeout), publishes approved replies to an HTTP endpoint, and updates the review status in Supabase.
- **1.7 Notification & Insight Extraction:** Determines if the review is important (high priority, low/high ratings, negative sentiment, or publication failure), notifies the business owner on Slack, extracts structured internal feedback insights via AI, and logs them back into Supabase before looping to the next review.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Deduplication
- **Overview:** This block handles the time-based ingestion of raw customer reviews, cross-references them against existing database entries to prevent duplicate processing, and prepares a clean batch of unique items.
- **Nodes Involved:** 
  - `Schedule Trigger`
  - `Get New Reviews`
  - `Check Existing Review`
  - `Filter Unprocessed Reviews`
- **Node Details:**
  - **Schedule Trigger**
    - *Type & Role:* `n8n-nodes-base.scheduleTrigger` (Trigger)
    - *Configuration:* Configured to run at an interval of every 3 hours.
    - *Connections:* Input: None. Output: `Get New Reviews`.
    - *Edge Cases/Failures:* Missed executions if n8n instance is offline.
  - **Get New Reviews**
    - *Type & Role:* `n8n-nodes-base.code` (Data transformation / Mock API)
    - *Configuration:* Contains JavaScript returning an array of mock review objects (`review_id`, `customer_name`, `rating`, `review_text`) enriched with timestamp and business metadata.
    - *Connections:* Input: `Schedule Trigger`. Output: `Check Existing Review`.
    - *Edge Cases/Failures:* JavaScript syntax errors if modified; designed to be replaced by an HTTP Request node targeting the Google Business Profile Reviews API.
  - **Check Existing Review**
    - *Type & Role:* `n8n-nodes-base.supabase` (Database read)
    - *Configuration:* Connects to Supabase (`Sales_transactions` table - *Note: configured against transactions table in template, intended for reviews*) using `getAll` operation with a dynamic string filter checking `review_id` against incoming review IDs.
    - *Connections:* Input: `Get New Reviews`. Output: `Filter Unprocessed Reviews`.
    - *Edge Cases/Failures:* Supabase authentication failure, connection timeouts, or query string parsing errors if review IDs contain special characters.
  - **Filter Unprocessed Reviews**
    - *Type & Role:* `n8n-nodes-base.code` (Filtering logic)
    - *Configuration:* JavaScript filtering block that compares incoming review IDs against items returned from Supabase and in-memory batch duplicates, dropping already stored IDs.
    - *Connections:* Input: `Check Existing Review`. Output: `Loop Over Reviews`.
    - *Edge Cases/Failures:* Empty input arrays handling.

---

#### Block 1.2: AI Analysis & Response Generation
- **Overview:** Iterates through filtered reviews one by one, leverages Claude Sonnet to extract sentiment, topics, intent, and urgency, and drafts an empathetic public response.
- **Nodes Involved:**
  - `Loop Over Reviews`
  - `Analyze Review with AI`
  - `Anthropic Chat Model`
  - `Generate AI Response`
  - `Anthropic Chat Model1`
- **Node Details:**
  - **Loop Over Reviews**
    - *Type & Role:* `n8n-nodes-base.splitInBatches` (Flow control)
    - *Configuration:* Standard batch loop iterating items one at a time.
    - *Connections:* Input: `Filter Unprocessed Reviews`, `Save Insight`. Output 1 (Done): None. Output 2 (Loop): `Analyze Review with AI`.
    - *Edge Cases/Failures:* Infinite loops if batch indexing is improperly modified.
  - **Analyze Review with AI**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration:* Uses text prompt containing stringified customer name, rating, and review text. System message instructs strict JSON output containing sentiment, rating, topics, customer intent, urgency, and escalation flags based strictly on review content. Max 3 retries.
    - *Key Expressions:* `={{ JSON.stringify({ customer_name: $json.customer_name, rating: $json.rating, review_text: $json.review_text }) }}`
    - *Connections:* Input: `Loop Over Reviews`. Output: `Generate AI Response`. AI Model Input: `Anthropic Chat Model`.
    - *Edge Cases/Failures:* AI returning invalid JSON, API rate limits, or token limit overruns.
  - **Anthropic Chat Model**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatAnthropic` (Language model sub-node)
    - *Configuration:* Uses model `claude-sonnet-5` with AI Gateway-managed credentials and maxTokens set to 4096.
    - *Connections:* Output: Connected to `Analyze Review with AI`.
  - **Generate AI Response**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration:* System message enforces warm, concise (2-4 sentences) public responses mirroring review sentiment without inventing compensation or guarantees. Max 3 retries.
    - *Key Expressions:* Extracts review details and analysis results via `$('Loop Over Reviews')` and `$('Analyze Review with AI')`.
    - *Connections:* Input: `Analyze Review with AI`. Output: `Review Sentiment Router`. AI Model Input: `Anthropic Chat Model1`.
    - *Edge Cases/Failures:* Malformed JSON output or persona drift.
  - **Anthropic Chat Model1**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatAnthropic` (Language model sub-node)
    - *Configuration:* Uses model `claude-sonnet-5` with AI Gateway-managed credentials and maxTokens set to 4096.
    - *Connections:* Output: Connected to `Generate AI Response`.

---

#### Block 1.3: Sentiment Routing
- **Overview:** Evaluates the sentiment output from the AI analysis and branches the flow into positive, neutral, or negative handling paths.
- **Nodes Involved:**
  - `Review Sentiment Router`
  - `Positive Review`
  - `Neutral Review`
  - `Negative Review`
- **Node Details:**
  - **Review Sentiment Router**
    - *Type & Role:* `n8n-nodes-base.switch` (Router)
    - *Configuration:* Evaluates `sentiment` field extracted from AI analysis JSON (`positive`, `neutral`, `negative`). Includes fallback output.
    - *Key Expressions:* `={{ JSON.parse($('Analyze Review with AI').first().json.output).sentiment }}`
    - *Connections:* Input: `Generate AI Response`. Outputs: `Positive Review`, `Neutral Review`, `Negative Review` (neutral rule maps twice for coverage).
    - *Edge Cases/Failures:* Unhandled sentiment values falling into fallback.
  - **Positive Review**
    - *Type & Role:* `n8n-nodes-base.set` (Data enrichment)
    - *Configuration:* Assigns `review_path` as `positive` and `response_strategy` as `thank_and_invite_back`.
    - *Connections:* Input: `Review Sentiment Router`. Output: `AI Escalation Check`.
  - **Neutral Review**
    - *Type & Role:* `n8n-nodes-base.set` (Data enrichment)
    - *Configuration:* Assigns `review_path` as `neutral` and `response_strategy` as `acknowledge_and_address_feedback`.
    - *Connections:* Input: `Review Sentiment Router`. Output: `AI Escalation Check`.
  - **Negative Review**
    - *Type & Role:* `n8n-nodes-base.set` (Data enrichment)
    - *Configuration:* Assigns `review_path` as `negative` and `response_strategy` as `recovery_with_private_follow_up`.
    - *Connections:* Input: `Review Sentiment Router`. Output: `AI Escalation Check`.

---

#### Block 1.4: Escalation Check & Initial Logging
- **Overview:** Runs a secondary AI escalation assessment, applies deterministic keyword and rating safeguards, compiles the master review record, and saves it to Supabase.
- **Nodes Involved:**
  - `AI Escalation Check`
  - `Anthropic Chat Model2`
  - `Build Review Record`
  - `Save Review`
- **Node Details:**
  - **AI Escalation Check**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration:* Evaluates whether the review requires human intervention based on complaints, legal threats, safety concerns, or drafted reply risks. Returns JSON with `needs_human_approval`, `priority`, and `reason`.
    - *Key Expressions:* Aggregates review text, rating, sentiment, topics, urgency, and drafted reply.
    - *Connections:* Input: `Positive Review`, `Neutral Review`, or `Negative Review`. Output: `Build Review Record`. AI Model Input: `Anthropic Chat Model2`.
    - *Edge Cases/Failures:* JSON parsing errors or API timeouts.
  - **Anthropic Chat Model2**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatAnthropic` (Language model sub-node)
    - *Configuration:* Uses model `claude-sonnet-5` with AI Gateway-managed credentials.
    - *Connections:* Output: Connected to `AI Escalation Check`.
  - **Build Review Record**
    - *Type & Role:* `n8n-nodes-base.code` (Data normalization)
    - *Configuration:* JavaScript execution that merges AI outputs, enforces fallback lists for topics/sentiment/urgency, and applies hardcoded regex safeguards for sensitive legal/safety/refund keywords. Sets final approval and response statuses.
    - *Connections:* Input: `AI Escalation Check`. Output: `Save Review`.
    - *Edge Cases/Failures:* Undefined property references if upstream JSON nodes fail.
  - **Save Review**
    - *Type & Role:* `n8n-nodes-base.supabase` (Database write)
    - *Configuration:* Inserts/saves mapped record fields into the Supabase `reviews` table.
    - *Connections:* Input: `Build Review Record`. Output: `Human Approval Required`.
    - *Edge Cases/Failures:* Supabase schema mismatch, missing required columns, or permission errors.

---

#### Block 1.5: Human Approval Gateway
- **Overview:** Inspects if human approval is mandatory. If required, dispatches an interactive approval message to Slack containing review details and action buttons, then pauses execution awaiting user input.
- **Nodes Involved:**
  - `Human Approval Required`
  - `Send Approval Request`
  - `Wait for Human Approval`
- **Node Details:**
  - **Human Approval Required**
    - *Type & Role:* `n8n-nodes-base.if` (Conditional branching)
    - *Configuration:* Checks if `needs_approval` evaluates to `true`.
    - *Key Expressions:* `={{ $('Build Review Record').first().json.needs_approval }}`
    - *Connections:* Input: `Save Review`. Output 1 (True): `Send Approval Request`. Output 2 (False): `Resolve Final Response`.
    - *Edge Cases/Failures:* Boolean evaluation mismatches.
  - **Send Approval Request**
    - *Type & Role:* `n8n-nodes-base.slack` (Messaging integration)
    - *Configuration:* Sends a formatted block message via OAuth2 to a designated Slack channel. Includes resume webhook links for approval/rejection actions (`$execution.resumeUrl`).
    - *Credentials:* Slack OAuth2 API.
    - *Connections:* Input: `Human Approval Required`. Output: `Wait for Human Approval`.
    - *Edge Cases/Failures:* Invalid Slack channel ID, expired OAuth tokens, or rate limiting.
  - **Wait for Human Approval**
    - *Type & Role:* `n8n-nodes-base.wait` (Webhook pause)
    - *Configuration:* Pauses workflow execution waiting for webhook resume with a 24-hour limit (`resumeAmount: 24`, `limitWaitTime: true`).
    - *Connections:* Input: `Send Approval Request`. Output: `Resolve Final Response`.
    - *Edge Cases/Failures:* Request timeout after 24 hours without human interaction.

---

#### Block 1.6: Resolution & Publishing
- **Overview:** Interprets the outcome of the approval process (approval, rejection, timeout, or auto-approved), publishes approved responses to the target endpoint, and updates the database record.
- **Nodes Involved:**
  - `Resolve Final Response`
  - `Response Approved`
  - `Publish Response`
  - `Update Review Status`
- **Node Details:**
  - **Resolve Final Response**
    - *Type & Role:* `n8n-nodes-base.code` (Decision resolution)
    - *Configuration:* JavaScript parsing query parameters (`decision=approve`, `decision=reject`, or timeout) to determine final publication flags and approval statuses.
    - *Connections:* Input: `Human Approval Required` (false branch) or `Wait for Human Approval`. Output: `Response Approved`.
    - *Edge Cases/Failures:* Malformed query strings.
  - **Response Approved**
    - *Type & Role:* `n8n-nodes-base.if` (Conditional branching)
    - *Configuration:* Evaluates whether `publish` is `true`.
    - *Connections:* Input: `Resolve Final Response`. Output 1 (True): `Publish Response`. Output 2 (False): `Update Review Status`.
    - *Edge Cases/Failures:* Boolean evaluation errors.
  - **Publish Response**
    - *Type & Role:* `n8n-nodes-base.httpRequest` (API integration)
    - *Configuration:* Makes a POST request to `https://httpbin.org/post` (mock endpoint; designed to be replaced with Google Business Profile Reply API) with JSON body containing `review_id`, `business_id`, and `comment`. Configured with `onError: continueRegularOutput` and a 30-second timeout.
    - *Connections:* Input: `Response Approved` (True). Output: `Update Review Status`.
    - *Edge Cases/Failures:* HTTP 4xx/5xx errors, network timeouts, or endpoint deprecation.
  - **Update Review Status**
    - *Type & Role:* `n8n-nodes-base.supabase` (Database update)
    - *Configuration:* Updates the `reviews` table matching `review_id` with `final_response`, approval statuses, response statuses, system states, and timestamps (`responded_at`).
    - *Connections:* Input: `Publish Response` and `Response Approved` (False). Output: `Important Review`.
    - *Edge Cases/Failures:* Primary key lookup failures or DB constraint violations.

---

#### Block 1.7: Notification & Insight Extraction
- **Overview:** Evaluates whether a review is critical (high priority, extreme ratings, negative sentiment, or publication failure), notifies the business owner on Slack, extracts actionable feedback insights via AI, and saves them to Supabase before continuing the loop.
- **Nodes Involved:**
  - `Important Review`
  - `Notify Business Owner`
  - `Generate Feedback Insight`
  - `Anthropic Chat Model3`
  - `Save Insight`
- **Node Details:**
  - **Important Review**
    - *Type & Role:* `n8n-nodes-base.if` (Conditional branching)
    - *Configuration:* Checks if rating is $\ge 5$ or $\le 2$, priority is `high`, sentiment is `negative`, or status is `failed`.
    - *Key Expressions:* Evaluates review metrics against threshold conditions.
    - *Connections:* Input: `Update Review Status`. Output 1 (True): `Notify Business Owner`. Output 2 (False/Parallel): `Generate Feedback Insight`. *(Note: Output 1 also chains to Generate Feedback Insight via secondary connection).*
    - *Edge Cases/Failures:* Complex expression syntax evaluation issues.
  - **Notify Business Owner**
    - *Type & Role:* `n8n-nodes-base.slack` (Messaging integration)
    - *Configuration:* Sends formatted alert message via Slack OAuth2 to the owner channel.
    - *Credentials:* Slack OAuth2 API.
    - *Connections:* Input: `Important Review`. Output: `Generate Feedback Insight`.
    - *Edge Cases/Failures:* Slack API errors or invalid channel configurations.
  - **Generate Feedback Insight**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent)
    - *Configuration:* System message instructs extraction of internal business insights, separating observed feedback from recommended actions. Returns JSON with `main_issue`, `topic`, `observed_feedback`, `recommended_action`, and `severity`.
    - *Connections:* Input: `Important Review` and `Notify Business Owner`. Output: `Save Insight`. AI Model Input: `Anthropic Chat Model3`.
    - *Edge Cases/Failures:* AI formatting failures or schema non-compliance.
  - **Anthropic Chat Model3**
    - *Type & Role:* `@n8n/n8n-nodes-langchain.lmChatAnthropic` (Language model sub-node)
    - *Configuration:* Uses model `claude-sonnet-5` with AI Gateway-managed credentials.
    - *Connections:* Output: Connected to `Generate Feedback Insight`.
  - **Save Insight**
    - *Type & Role:* `n8n-nodes-base.supabase` (Database write)
    - *Configuration:* Inserts extracted insight attributes into the Supabase `feedback_insights` table.
    - *Connections:* Input: `Generate Feedback Insight`. Output: `Loop Over Reviews` (completing the iteration cycle).
    - *Edge Cases/Failures:* Supabase write errors or table constraint violations.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Schedule Trigger** | `n8n-nodes-base.scheduleTrigger` | Triggers workflow every 3 hours | None | Get New Reviews | **1. Fetch & Dedupe Reviews**<br>Runs on a schedule, collects new reviews using demo data that can be replaced with the Google Business Profile Reviews API, checks Supabase for reviews that have already been processed, and filters out duplicates before processing each new review one at a time. |
| **Get New Reviews** | `n8n-nodes-base.code` | Provides mock review items | Schedule Trigger | Check Existing Review | **1. Fetch & Dedupe Reviews**<br>Runs on a schedule, collects new reviews using demo data that can be replaced with the Google Business Profile Reviews API, checks Supabase for reviews that have already been processed, and filters out duplicates before processing each new review one at a time. |
| **Check Existing Review** | `n8n-nodes-base.supabase` | Queries Supabase for existing reviews | Get New Reviews | Filter Unprocessed Reviews | **1. Fetch & Dedupe Reviews**<br>Runs on a schedule, collects new reviews using demo data that can be replaced with the Google Business Profile Reviews API, checks Supabase for reviews that have already been processed, and filters out duplicates before processing each new review one at a time. |
| **Filter Unprocessed Reviews** | `n8n-nodes-base.code` | Drops duplicate or processed reviews | Check Existing Review | Loop Over Reviews | **1. Fetch & Dedupe Reviews**<br>Runs on a schedule, collects new reviews using demo data that can be replaced with the Google Business Profile Reviews API, checks Supabase for reviews that have already been processed, and filters out duplicates before processing each new review one at a time. |
| **Loop Over Reviews** | `n8n-nodes-base.splitInBatches` | Iterates reviews sequentially | Filter Unprocessed Reviews, Save Insight | Analyze Review with AI | *(None)* |
| **Analyze Review with AI** | `@n8n/n8n-nodes-langchain.agent` | Analyzes sentiment, topic, and intent | Loop Over Reviews | Generate AI Response | **2. AI Analyze & Draft Reply**<br>AI analyzes each review for sentiment, topics, intent and urgency, then drafts a warm, on-brand public reply based only on what the review actually says. |
| **Anthropic Chat Model** | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | Provides LLM backend for review analysis | None | Analyze Review with AI | **2. AI Analyze & Draft Reply**<br>AI analyzes each review for sentiment, topics, intent and urgency, then drafts a warm, on-brand public reply based only on what the review actually says. |
| **Generate AI Response** | `@n8n/n8n-nodes-langchain.agent` | Drafts concise public reply | Analyze Review with AI | Review Sentiment Router | **2. AI Analyze & Draft Reply**<br>AI analyzes each review for sentiment, topics, intent and urgency, then drafts a warm, on-brand public reply based only on what the review actually says. |
| **Anthropic Chat Model1** | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | Provides LLM backend for response generation | None | Generate AI Response | **2. AI Analyze & Draft Reply**<br>AI analyzes each review for sentiment, topics, intent and urgency, then drafts a warm, on-brand public reply based only on what the review actually says. |
| **Review Sentiment Router** | `n8n-nodes-base.switch` | Routes reviews by sentiment classification | Generate AI Response | Positive Review, Neutral Review, Negative Review | **3. Route by Sentiment**<br>Routes the review down a positive, neutral, or negative path and tags it with the matching response strategy. |
| **Positive Review** | `n8n-nodes-base.set` | Assigns positive response metadata | Review Sentiment Router | AI Escalation Check | **3. Route by Sentiment**<br>Routes the review down a positive, neutral, or negative path and tags it with the matching response strategy. |
| **Neutral Review** | `n8n-nodes-base.set` | Assigns neutral response metadata | Review Sentiment Router | AI Escalation Check | **3. Route by Sentiment**<br>Routes the review down a positive, neutral, or negative path and tags it with the matching response strategy. |
| **Negative Review** | `n8n-nodes-base.set` | Assigns negative response metadata | Review Sentiment Router | AI Escalation Check | **3. Route by Sentiment**<br>Routes the review down a positive, neutral, or negative path and tags it with the matching response strategy. |
| **AI Escalation Check** | `@n8n/n8n-nodes-langchain.agent` | Evaluates need for human escalation | Positive Review, Neutral Review, Negative Review | Build Review Record | **4. Escalation Check & Save**<br>AI checks whether a review needs human attention, such as low ratings, complaints, or legal and safety concerns. It then applies fixed rules to the AI decision, creates the final review record, and saves it to Supabase. |
| **Anthropic Chat Model2** | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | Provides LLM backend for escalation checks | None | AI Escalation Check | **4. Escalation Check & Save**<br>AI checks whether a review needs human attention, such as low ratings, complaints, or legal and safety concerns. It then applies fixed rules to the AI decision, creates the final review record, and saves it to Supabase. |
| **Build Review Record** | `n8n-nodes-base.code` | Normalizes record with rule safeguards | AI Escalation Check | Save Review | **4. Escalation Check & Save**<br>AI checks whether a review needs human attention, such as low ratings, complaints, or legal and safety concerns. It then applies fixed rules to the AI decision, creates the final review record, and saves it to Supabase. |
| **Save Review** | `n8n-nodes-base.supabase` | Saves review record to Supabase | Build Review Record | Human Approval Required | **4. Escalation Check & Save**<br>AI checks whether a review needs human attention, such as low ratings, complaints, or legal and safety concerns. It then applies fixed rules to the AI decision, creates the final review record, and saves it to Supabase. |
| **Human Approval Required** | `n8n-nodes-base.if` | Checks if human approval is needed | Save Review | Send Approval Request, Resolve Final Response | **5. Human Approval**<br>Reviews flagged for escalation pause here and post to Slack for a human to approve or reject the drafted reply before anything goes public (24h timeout). |
| **Send Approval Request** | `n8n-nodes-base.slack` | Posts approval request card to Slack | Human Approval Required | Wait for Human Approval | **5. Human Approval**<br>Reviews flagged for escalation pause here and post to Slack for a human to approve or reject the drafted reply before anything goes public (24h timeout). |
| **Wait for Human Approval** | `n8n-nodes-base.wait` | Pauses execution awaiting webhook action | Send Approval Request | Resolve Final Response | **5. Human Approval**<br>Reviews flagged for escalation pause here and post to Slack for a human to approve or reject the drafted reply before anything goes public (24h timeout). |
| **Resolve Final Response** | `n8n-nodes-base.code` | Resolves approval decision outcome | Human Approval Required, Wait for Human Approval | Response Approved | **6. Resolve & Publish**<br>Determines the final decision based on approval status, rejection, or timeout, publishes the reply using a mock endpoint that can be replaced with the Google Business Profile Reply API, and updates the review status in Supabase. |
| **Response Approved** | `n8n-nodes-base.if` | Validates if response is cleared for publish | Resolve Final Response | Publish Response, Update Review Status | **6. Resolve & Publish**<br>Determines the final decision based on approval status, rejection, or timeout, publishes the reply using a mock endpoint that can be replaced with the Google Business Profile Reply API, and updates the review status in Supabase. |
| **Publish Response** | `n8n-nodes-base.httpRequest` | Publishes response via HTTP endpoint | Response Approved | Update Review Status | **6. Resolve & Publish**<br>Determines the final decision based on approval status, rejection, or timeout, publishes the reply using a mock endpoint that can be replaced with the Google Business Profile Reply API, and updates the review status in Supabase. |
| **Update Review Status** | `n8n-nodes-base.supabase` | Updates review record status in database | Publish Response, Response Approved | Important Review | **6. Resolve & Publish**<br>Determines the final decision based on approval status, rejection, or timeout, publishes the reply using a mock endpoint that can be replaced with the Google Business Profile Reply API, and updates the review status in Supabase. |
| **Important Review** | `n8n-nodes-base.if` | Filters critical reviews for owner alert | Update Review Status | Notify Business Owner, Generate Feedback Insight | **7. Notify & Save Insights**<br>Identifies important reviews, such as 5-star reviews, low ratings, high-priority feedback, or failed replies, and notifies the business owner on Slack. It also extracts useful business insights from every review and saves them to Supabase. |
| **Notify Business Owner** | `n8n-nodes-base.slack` | Notifies business owner on Slack | Important Review | Generate Feedback Insight | **7. Notify & Save Insights**<br>Identifies important reviews, such as 5-star reviews, low ratings, high-priority feedback, or failed replies, and notifies the business owner on Slack. It also extracts useful business insights from every review and saves them to Supabase. |
| **Generate Feedback Insight** | `@n8n/n8n-nodes-langchain.agent` | Extracts internal business insights | Important Review, Notify Business Owner | Save Insight | **7. Notify & Save Insights**<br>Identifies important reviews, such as 5-star reviews, low ratings, high-priority feedback, or failed replies, and notifies the business owner on Slack. It also extracts useful business insights from every review and saves them to Supabase. |
| **Anthropic Chat Model3** | `@n8n/n8n-nodes-langchain.lmChatAnthropic` | Provides LLM backend for insight generation | None | Generate Feedback Insight | **7. Notify & Save Insights**<br>Identifies important reviews, such as 5-star reviews, low ratings, high-priority feedback, or failed replies, and notifies the business owner on Slack. It also extracts useful business insights from every review and saves them to Supabase. |
| **Save Insight** | `n8n-nodes-base.supabase` | Saves business insight to Supabase | Generate Feedback Insight | Loop Over Reviews | **7. Notify & Save Insights**<br>Identifies important reviews, such as 5-star reviews, low ratings, high-priority feedback, or failed replies, and notifies the business owner on Slack. It also extracts useful business insights from every review and saves them to Supabase. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Schedule Trigger**
   - Add a **Schedule Trigger** node. Set rule interval to every `3` hours.
2. **Add Get New Reviews Code Node**
   - Add a **Code** node named `Get New Reviews`. Paste the JavaScript returning demo review objects (`review_id`, `customer_name`, `rating`, `review_text`, `review_date`, `business_id`, `business_name`).
3. **Add Supabase Check Existing Review Node**
   - Add a **Supabase** node. Set operation to `Get All`, table to `reviews`, and filter type to string with filter expression checking incoming review IDs. Configure Supabase API credentials.
4. **Add Filter Unprocessed Reviews Code Node**
   - Add a **Code** node named `Filter Unprocessed Reviews`. Insert JavaScript to filter out existing Supabase IDs and batch duplicates.
5. **Add Loop Over Reviews Node**
   - Add a **Split In Batches** node (`Loop Over Reviews`) with default options. Connect the loop output to the AI analysis branch.
6. **Configure AI Review Analysis Chain**
   - Add an **AI Agent** node (`Analyze Review with AI`) with prompt text mapping customer name, rating, and review text.
   - Attach an **Anthropic Chat Model** node (`Anthropic Chat Model`) configured with `claude-sonnet-5` and AI Gateway credentials. Connect its `ai_languageModel` output to the agent.
   - Add a second **AI Agent** node (`Generate AI Response`) to draft public replies based on review content and sentiment. Attach an **Anthropic Chat Model** (`Anthropic Chat Model1`) to it.
7. **Configure Sentiment Router & Set Nodes**
   - Add a **Switch** node (`Review Sentiment Router`) checking the AI analysis sentiment output (`positive`, `neutral`, `negative`).
   - Create three **Set** nodes (`Positive Review`, `Neutral Review`, `Negative Review`), setting `review_path` and `response_strategy` variables respectively. Connect all three to the escalation check.
8. **Configure Escalation Check & Initial Save**
   - Add an **AI Agent** node (`AI Escalation Check`) with an **Anthropic Chat Model** (`Anthropic Chat Model2`) to determine human approval requirements and priority.
   - Add a **Code** node (`Build Review Record`) to normalize records and apply hardcoded regex risk safeguards.
   - Add a **Supabase** node (`Save Review`) to insert the review record into the `reviews` table.
9. **Configure Human Approval Gateway**
   - Add an **If** node (`Human Approval Required`) checking `needs_approval === true`.
   - Add a **Slack** node (`Send Approval Request`) using OAuth2 credentials to post review details and resume action links (`$execution.resumeUrl`) to your Slack channel.
   - Add a **Wait** node (`Wait for Human Approval`) configured with webhook resume and 24-hour timeout limit.
10. **Configure Resolution & Publishing**
    - Add a **Code** node (`Resolve Final Response`) to process decision query parameters.
    - Add an **If** node (`Response Approved`) checking if `publish === true`.
    - Add an **HTTP Request** node (`Publish Response`) targeting your endpoint (e.g., `https://httpbin.org/post` or Google Business Profile API) with a POST/PUT JSON body containing the comment.
    - Add a **Supabase** node (`Update Review Status`) performing an update operation on the `reviews` table.
11. **Configure Notifications & Feedback Insights**
    - Add an **If** node (`Important Review`) evaluating rating, priority, sentiment, or failure status.
    - Add a **Slack** node (`Notify Business Owner`) for owner alerts.
    - Add an **AI Agent** node (`Generate Feedback Insight`) with an **Anthropic Chat Model** (`Anthropic Chat Model3`) to extract structured internal insights.
    - Add a **Supabase** node (`Save Insight`) to save insights into the `feedback_insights` table, and connect its output back to the **Loop Over Reviews** node input to complete the iteration loop.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| AI Review & Reputation Management Template Overview | Designed for restaurants, hotels, clinics, and local service businesses to automate review responses, escalation handling, and feedback logging. |
| Supabase Setup Requirement | Requires two tables (`reviews` and `feedback_insights`) configured with matching schema fields for inserts and updates. |
| Slack Integration Requirement | Requires a connected Slack workspace with OAuth2 credentials and an invited bot user in the target approval/notification channel. |
| Anthropic & AI Gateway Requirement | Requires Anthropic API access or n8n AI Gateway-managed access for Claude Sonnet model nodes. |
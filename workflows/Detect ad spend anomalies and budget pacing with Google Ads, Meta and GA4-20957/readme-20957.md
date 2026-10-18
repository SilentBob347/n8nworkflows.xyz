Detect ad spend anomalies and budget pacing with Google Ads, Meta and GA4

https://n8nworkflows.xyz/workflows/detect-ad-spend-anomalies-and-budget-pacing-with-google-ads--meta-and-ga4-20957


# Detect ad spend anomalies and budget pacing with Google Ads, Meta and GA4

### 1. Workflow Overview

This workflow automates the daily monitoring of multi-channel advertising performance across Google Ads, Meta Ads, and Google Analytics 4 (GA4). Its primary purpose is to detect budget spend spikes, ROAS (Return on Ad Spend) drops, monthly budget pacing issues, and tracking anomalies by evaluating the latest day's data against a historical baseline. When significant performance degradations occur, the workflow generates a summary digest and requests human authorization via email before automatically pausing underperforming campaigns.

The execution logic is structured into five distinct operational blocks:
- **1.1 Input Reception & Configuration:** Handles automated daily scheduling or manual testing triggers, initializing a comprehensive configuration payload containing account IDs, budgetary limits, statistical thresholds, and feature flags.
- **1.2 Data Ingestion & Normalization:** Branches conditionally based on configuration parameters to either generate isolated sample testing data or fetch live campaign and traffic datasets from Google Ads, Meta Graph, and GA4 Data APIs, merging them into unified daily rows.
- **1.3 Anomaly Detection & Business Intelligence:** Processes historical baselines using z-score calculations, evaluates month-end budget pacing projections, checks for tracking gaps between ad clicks and GA4 sessions, and compiles actionable findings and campaign pause proposals.
- **1.4 Notification & Human Approval:** Manages notification routing via feature flags, delivering an operational digest and initiating asynchronous interactive email approvals for individual campaign actions.
- **1.5 Execution & Mitigation:** Evaluates approval decisions and execution safety flags (Dry Run mode) to either log simulated outcomes or safely invoke the Google Ads and Meta APIs to pause non-compliant campaigns.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Configuration
- **Overview:** This block serves as the entry point for the automation pipeline, establishing execution timing and initializing global configuration variables required by subsequent API calls and analytical evaluations.
- **Nodes Involved:** 
  - `Schedule Trigger`
  - `Test with Sample Data`
  - `Workflow Configuration`
  - `Use Sample Data?`
- **Node Details:**
  - **Schedule Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger node). Executes the workflow automatically on a recurring schedule.
    - *Configuration Choices:* Configured to trigger daily at hour 08:00 UTC.
    - *Expressions/Variables:* None.
    - *Input/Output:* Output connects to `Workflow Configuration`.
  - **Test with Sample Data**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Trigger node). Allows manual test execution of the workflow from the n8n editor.
    - *Configuration Choices:* Default manual invocation parameters.
    - *Expressions/Variables:* None.
    - *Input/Output:* Output connects to `Workflow Configuration`.
  - **Workflow Configuration**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data transformation node). Establishes environment parameters, budget limits, operational thresholds, and API authentication targets.
    - *Configuration Choices:* Sets feature flags (`sampleMode`, `dryRun`, `emailsEnabled`), target accounts (`googleAdsCustomerId`, `metaAdAccountId`, `ga4PropertyId`), financial metrics (`googleMonthlyBudget`, `metaMonthlyBudget`, `targetRoas`), statistical boundaries (`zThreshold`, `baselineDays`, `minDailySpend`, `overPaceThreshold`, `underPaceThreshold`, `trackingDropRatio`), and dynamic date ranges (`startDate` / `endDate`).
    - *Expressions/Variables:* 
      - `startDate`: `={{ $now.minus({ days: 35 }).toFormat('yyyy-MM-dd') }}`
      - `endDate`: `={{ $now.minus({ days: 1 }).toFormat('yyyy-MM-dd') }}`
    - *Input/Output:* Inputs from `Schedule Trigger` or `Test with Sample Data`; output connects to `Use Sample Data?`.
    - *Edge Cases/Failures:* Incorrect date string formats or missing API identifiers will cause downstream API HTTP requests to fail.
  - **Use Sample Data?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow control node). Directs execution flow between live API ingestion or offline sample generation based on configuration flags.
    - *Configuration Choices:* Evaluates boolean expression verifying if sample mode is enabled.
    - *Expressions/Variables:* `={{ $('Workflow Configuration').first().json.sampleMode }}`
    - *Input/Output:* Input from `Workflow Configuration`; true branch connects to `Generate Sample Ad Data`, false branch connects to the three live API nodes in parallel.

---

#### Block 1.2: Data Ingestion & Normalization
- **Overview:** Retrieves raw performance data from external marketing platforms and GA4 or synthesizes localized test rows, subsequently standardizing schemas and merging datasets into a unified format.
- **Nodes Involved:**
  - `Generate Sample Ad Data`
  - `Get Google Ads Campaigns`
  - `Get Meta Ads Insights`
  - `Get GA4 Paid Sessions`
  - `Combine Platform Data`
  - `Normalize Platform Rows`
- **Node Details:**
  - **Generate Sample Ad Data**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript execution node). Generates robust mock advertising datasets populated with planted anomalies for testing without external API dependencies.
    - *Configuration Choices:* Custom JavaScript generating 35 days of simulated campaigns with calculated statistical variance, noise, and deliberate anomalies on the final day.
    - *Expressions/Variables:* Native JavaScript date and random math functions.
    - *Input/Output:* Input from `Use Sample Data?` (True branch); output connects directly to `Detect Anomalies and Pacing`.
  - **Get Google Ads Campaigns**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration node). Queries the Google Ads API via GoogleAdsService SearchStream.
    - *Configuration Choices:* POST request targeting API version from configuration, querying campaign metrics (cost, clicks, impressions, conversions, conversion value) within the specified date range. Continues execution on regular output errors.
    - *Expressions/Variables:* Dynamic request URLs and JSON body parameters referencing configuration IDs, developer tokens, and date limits.
    - *Input/Output:* Input from `Use Sample Data?` (False branch); output connects to index `0` of `Combine Platform Data`.
    - *Credentials:* `googleAdsOAuth2Api`
    - *Edge Cases/Failures:* Invalid developer tokens, missing login customer IDs, or expired OAuth credentials will trigger API rejections.
  - **Get Meta Ads Insights**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration node). Queries the Facebook Marketing API for campaign-level insights.
    - *Configuration Choices:* GET request configured with automated pagination (following next URL up to 20 requests), fetching daily campaign metrics including spend, clicks, impressions, actions, and action values.
    - *Expressions/Variables:* Dynamic query parameters for time ranges derived from workflow configuration variables.
    - *Input/Output:* Input from `Use Sample Data?` (False branch); output connects to index `1` of `Combine Platform Data`.
    - *Credentials:* `facebookGraphApi`
    - *Edge Cases/Failures:* Account permission limitations or rate limits.
  - **Get GA4 Paid Sessions**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration node). Fetches session metrics and key events from the Google Analytics 4 Data API.
    - *Configuration Choices:* POST request invoking the `runReport` method with dimensions for date and session source/medium, alongside metrics for sessions, key events, and total revenue.
    - *Expressions/Variables:* Property ID and date ranges pulled from workflow configuration.
    - *Input/Output:* Input from `Use Sample Data?` (False branch); output connects to index `2` of `Combine Platform Data`.
    - *Credentials:* `googleAnalyticsOAuth2`
  - **Combine Platform Data**
    - *Type and Technical Role:* `n8n-nodes-base.merge` (Data combination node). Merges three distinct input data streams into a single execution dataset.
    - *Configuration Choices:* Configured for 3 input connections.
    - *Input/Output:* Inputs from Google Ads, Meta Ads, and GA4 nodes; output connects to `Normalize Platform Rows`.
  - **Normalize Platform Rows**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript transformation node). Normalizes disparate API schemas (Google, Meta, GA4) into a unified internal data structure and aggregates overlapping metrics by date and platform.
    - *Configuration Choices:* JavaScript array transformation mapping platform-specific fields to standardized properties (`spend`, `clicks`, `impressions`, `conversions`, `revenue`, `sessions`).
    - *Input/Output:* Input from `Combine Platform Data`; output connects to `Detect Anomalies and Pacing`.

---

#### Block 1.3: Anomaly Detection & Business Intelligence
- **Overview:** Analyzes normalized datasets to compute historical means, standard deviations, and z-scores for ad spend and ROAS, evaluates month-end budget pacing, checks for tracking pipeline gaps, and prepares action proposals.
- **Nodes Involved:**
  - `Detect Anomalies and Pacing`
- **Node Details:**
  - **Detect Anomalies and Pacing**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript analytical engine node). Executes statistical calculations to flag performance anomalies, pacing deviations, and tracking failures.
    - *Configuration Choices:* JavaScript script that aggregates campaign performance over a trailing baseline window (e.g., 14 days), calculates z-scores, checks spend/ROAS thresholds, evaluates monthly pacing ratios against budget limits, and emits an overall digest item alongside individual action payload items.
    - *Expressions/Variables:* Pulls configuration parameters (`zThreshold`, `baselineDays`, `minDailySpend`, `targetRoas`, `googleMonthlyBudget`, `metaMonthlyBudget`, `overPaceThreshold`, `underPaceThreshold`, `trackingDropRatio`) via `$('Workflow Configuration')`.
    - *Input/Output:* Input from either `Generate Sample Ad Data` or `Normalize Platform Rows`; output connects to `Emails Enabled?`.
    - *Edge Cases/Failures:* Insufficient historical data points (fewer than 7 days) result in baseline evaluations being safely skipped for specific campaigns.

---

#### Block 1.4: Notification & Human Approval
- **Overview:** Controls message routing based on email notification flags, delivers administrative digests, and manages interactive approval workflows for proposed campaign pauses.
- **Nodes Involved:**
  - `Emails Enabled?`
  - `Is Action Item?`
  - `Preview Only (Emails Off)`
  - `Send Findings Digest`
  - `Request Approval by Email`
  - `Approved?`
  - `Rejected or Timed Out`
- **Node Details:**
  - **Emails Enabled?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow control node). Verifies whether email dispatch is authorized in settings.
    - *Expressions/Variables:* `={{ $('Workflow Configuration').first().json.emailsEnabled }}`
    - *Input/Output:* Input from `Detect Anomalies and Pacing`; true branch connects to `Is Action Item?`, false branch connects to `Preview Only (Emails Off)`.
  - **Is Action Item?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow control node). Separates the summary digest item from individual campaign action items.
    - *Expressions/Variables:* `={{ $json.kind }}` (evaluates if equal to `action`).
    - *Input/Output:* Input from `Emails Enabled?` (True branch); true branch (actions) connects to `Request Approval by Email`, false branch (digest) connects to `Send Findings Digest`.
  - **Preview Only (Emails Off)**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Placeholder node). Acts as a terminal endpoint when emails are disabled during dry runs or testing.
    - *Input/Output:* Input from `Emails Enabled?` (False branch).
  - **Send Findings Digest**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Communication integration node). Sends an administrative summary email containing all detected anomalies, pacing notes, and tracking gaps.
    - *Configuration Choices:* Sends plaintext email to the configured approver address with subject and body generated by the anomaly detection node.
    - *Expressions/Variables:* Recipient, subject, and message expressions mapped from the incoming JSON payload.
    - *Credentials:* `gmailOAuth2`
  - **Request Approval by Email**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Interactive communication node). Sends an interactive email requiring double-approval to authorize campaign pause actions.
    - *Configuration Choices:* Gmail operation set to `sendAndWait` with approval options configured for double approval, a custom disapproval label (“Reject”), and a resume wait timeout limit of 24 hours.
    - *Expressions/Variables:* Recipient, subject, and body expressions mapped from action payloads.
    - *Credentials:* `gmailOAuth2`
    - *Input/Output:* Input from `Is Action Item?` (True branch); output connects to `Approved?`.
    - *Edge Cases/Failures:* Approver timeout after 24 hours routes execution to the rejection/timeout handler.
  - **Approved?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow control node). Evaluates the user's response from the email approval widget.
    - *Expressions/Variables:* `={{ $json.data?.approved === true }}`
    - *Input/Output:* Input from `Request Approval by Email`; true branch connects to `Dry Run?`, false branch connects to `Rejected or Timed Out`.
  - **Rejected or Timed Out**
    - *Type and Technical Role:* `n8n-nodes-base.noOp` (Placeholder node). Handles execution flow when an approval request is explicitly rejected by the user or times out.
    - *Input/Output:* Input from `Approved?` (False branch).

---

#### Block 1.5: Execution & Mitigation
- **Overview:** Executes approved mitigation actions either in a simulated dry-run logging mode or by invoking the respective advertising platform APIs to pause non-compliant campaigns.
- **Nodes Involved:**
  - `Dry Run?`
  - `Log Dry Run Result`
  - `Is Google Ads?`
  - `Pause Google Ads Campaign`
  - `Pause Meta Campaign`
- **Node Details:**
  - **Dry Run?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow control node). Determines whether actions should be simulated or executed live against ad platform APIs.
    - *Expressions/Variables:* `={{ $('Workflow Configuration').first().json.dryRun }}`
    - *Input/Output:* Input from `Approved?` (True branch); true branch connects to `Log Dry Run Result`, false branch connects to `Is Google Ads?`.
  - **Log Dry Run Result**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data transformation node). Logs simulated execution details when dry run mode is active without modifying external ad accounts.
    - *Configuration Choices:* Assigns static result messages and pulls campaign names and platforms from preceding action nodes.
    - *Expressions/Variables:* References node items from `Is Action Item?`.
    - *Input/Output:* Input from `Dry Run?` (True branch).
  - **Is Google Ads?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Flow control node). Branches execution based on the target advertising platform of the approved action.
    - *Expressions/Variables:* `={{ $('Is Action Item?').item.json.platform }}` (evaluates if equal to `google_ads`).
    - *Input/Output:* Input from `Dry Run?` (False branch); true branch connects to `Pause Google Ads Campaign`, false branch connects to `Pause Meta Campaign`.
  - **Pause Google Ads Campaign**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration node). Mutates a campaign status to `PAUSED` via the Google Ads REST API.
    - *Configuration Choices:* POST request targeting the campaigns mutate endpoint, supplying an update mask for the `status` field set to `PAUSED`. Includes developer token and login customer ID headers.
    - *Expressions/Variables:* Dynamic URLs, update masks, and resource names constructed using configuration parameters and campaign IDs.
    - *Credentials:* `googleAdsOAuth2Api`
    - *Input/Output:* Input from `Is Google Ads?` (True branch).
    - *Edge Cases/Failures:* Permission errors or invalid campaign resource names will return HTTP error responses.
  - **Pause Meta Campaign**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API integration node). Updates a Meta ad campaign status to `PAUSED` via the Facebook Graph API.
    - *Configuration Choices:* POST request targeting the specific campaign node ID with query parameters setting `status` to `PAUSED`.
    - *Expressions/Variables:* Dynamic URL construction combining Meta API version and campaign ID.
    - *Credentials:* `facebookGraphApi`
    - *Input/Output:* Input from `Is Google Ads?` (False branch).
    - *Edge Cases/Failures:* Insufficient ad account permissions or invalid API version strings.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Documentation and setup guide | None | None | ## Detect ad anomalies and budget pacing with approvals<br><br>This template watches Google Ads, Meta Ads and GA4 together. It flags spend spikes and ROAS drops against each campaign's own baseline, checks month-end budget pacing, and spots possible tracking breaks. No change is made without a human clicking Approve.<br><br>### How it works<br>1. A daily trigger loads 35 days of campaign data from Google Ads, Meta and GA4.<br>2. A Code node compares yesterday with the trailing average (z-score) for every campaign.<br>3. It also projects month-end spend against your budgets and compares GA4 sessions with ad clicks.<br>4. You receive one digest email, plus one approval email per proposed pause.<br>5. Only approved pauses run, and **Dry Run** is on by default.<br><br>### Setup<br>1. Click **Test with Sample Data**. No credentials are needed.<br>2. Fill in **Workflow Configuration**: IDs, developer token, budgets, target ROAS, approver email.<br>3. Connect Google Ads OAuth2, Meta Graph API, Google Analytics OAuth2 and Gmail credentials.<br>4. Set sampleMode and dryRun to false and emailsEnabled to true when ready.<br><br>### Customization tips<br>Change thresholds in the config node, add Slack instead of Gmail, or extend the actions beyond pausing. |
| Section Settings | n8n-nodes-base.stickyNote | Visual grouping block header | None | None | ## 1. Triggers and settings<br>Schedule or sample run, plus all IDs, budgets and thresholds. |
| Warning Go Live | n8n-nodes-base.stickyNote | Operational warning note | None | None | ## Before going live<br>Get the developer token from the Google Ads API Center and keep it private. Set dryRun to false only after reviewing a few digests. |
| Section Live Data | n8n-nodes-base.stickyNote | Visual grouping block header | None | None | ## 2. Load campaign data<br>Live Google Ads, Meta and GA4 data is fetched, merged and normalized, or built-in sample data with planted anomalies is used. |
| Section Detect | n8n-nodes-base.stickyNote | Visual grouping block header | None | None | ## 3. Detect issues<br>Anomalies, budget pacing and tracking gaps. |
| Section Approval | n8n-nodes-base.stickyNote | Visual grouping block header | None | None | ## 4. Human approval<br>Digest email and one approval email per proposed pause. |
| Section Execute | n8n-nodes-base.stickyNote | Visual grouping block header | None | None | ## 5. Run approved action<br>Dry run log, or pause in Google Ads or Meta. |
| Schedule Trigger | n8n-nodes-base.scheduleTrigger | Executes workflow daily at 08:00 UTC | None | Workflow Configuration | ## 1. Triggers and settings<br>Schedule or sample run, plus all IDs, budgets and thresholds. |
| Workflow Configuration | n8n-nodes-base.set | Defines global configuration variables and budgets | Schedule Trigger, Test with Sample Data | Use Sample Data? | ## 1. Triggers and settings<br>Schedule or sample run, plus all IDs, budgets and thresholds. |
| Use Sample Data? | n8n-nodes-base.if | Routes execution based on sample mode flag | Workflow Configuration | Generate Sample Ad Data, Get Google Ads Campaigns, Get Meta Ads Insights, Get GA4 Paid Sessions | ## 1. Triggers and settings<br>Schedule or sample run, plus all IDs, budgets and thresholds. |
| Generate Sample Ad Data | n8n-nodes-base.code | Generates mock ad data with planted anomalies | Use Sample Data? | Detect Anomalies and Pacing | ## 2. Load campaign data<br>Live Google Ads, Meta and GA4 data is fetched, merged and normalized, or built-in sample data with planted anomalies is used. |
| Detect Anomalies and Pacing | n8n-nodes-base.code | Analyzes data for z-score anomalies and budget pacing | Generate Sample Ad Data, Normalize Platform Rows | Emails Enabled? | ## 3. Detect issues<br>Anomalies, budget pacing and tracking gaps. |
| Emails Enabled? | n8n-nodes-base.if | Checks if email sending is enabled | Detect Anomalies and Pacing | Is Action Item?, Preview Only (Emails Off) | ## 4. Human approval<br>Digest email and one approval email per proposed pause. |
| Is Action Item? | n8n-nodes-base.if | Separates summary digest from action items | Emails Enabled? | Request Approval by Email, Send Findings Digest | ## 4. Human approval<br>Digest email and one approval email per proposed pause. |
| Request Approval by Email | n8n-nodes-base.gmail | Sends interactive approval email for campaign pauses | Is Action Item? | Approved? | ## 4. Human approval<br>Digest email and one approval email per proposed pause. |
| Approved? | n8n-nodes-base.if | Validates human approval decision | Request Approval by Email | Dry Run?, Rejected or Timed Out | ## 4. Human approval<br>Digest email and one approval email per proposed pause. |
| Dry Run? | n8n-nodes-base.if | Checks if dry run simulation mode is active | Approved? | Log Dry Run Result, Is Google Ads? | ## 5. Run approved action<br>Dry run log, or pause in Google Ads or Meta. |
| Log Dry Run Result | n8n-nodes-base.set | Logs dry run simulation result | Dry Run? | None | ## 5. Run approved action<br>Dry run log, or pause in Google Ads or Meta. |
| Is Google Ads? | n8n-nodes-base.if | Determines ad platform target for action | Dry Run? | Pause Google Ads Campaign, Pause Meta Campaign | ## 5. Run approved action<br>Dry run log, or pause in Google Ads or Meta. |
| Pause Google Ads Campaign | n8n-nodes-base.httpRequest | Pauses campaign via Google Ads API mutation | Is Google Ads? | None | ## 5. Run approved action<br>Dry run log, or pause in Google Ads or Meta. |
| Pause Meta Campaign | n8n-nodes-base.httpRequest | Pauses campaign via Facebook Graph API | Is Google Ads? | None | ## 5. Run approved action<br>Dry run log, or pause in Google Ads or Meta. |
| Rejected or Timed Out | n8n-nodes-base.noOp | Handles rejected or timed out approval requests | Approved? | None | ## 4. Human approval<br>Digest email and one approval email per proposed pause. |
| Send Findings Digest | n8n-nodes-base.gmail | Sends administrative findings digest email | Is Action Item? | None | ## 4. Human approval<br>Digest email and one approval email per proposed pause. |
| Preview Only (Emails Off) | n8n-nodes-base.noOp | Terminal node when email notifications are disabled | Emails Enabled? | None | ## 4. Human approval<br>Digest email and one approval email per proposed pause. |
| Get Google Ads Campaigns | n8n-nodes-base.httpRequest | Fetches live campaign metrics from Google Ads API | Use Sample Data? | Combine Platform Data | ## 2. Load campaign data<br>Live Google Ads, Meta and GA4 data is fetched, merged and normalized, or built-in sample data with planted anomalies is used. |
| Get Meta Ads Insights | n8n-nodes-base.httpRequest | Fetches live campaign insights from Meta Marketing API | Use Sample Data? | Combine Platform Data | ## 2. Load campaign data<br>Live Google Ads, Meta and GA4 data is fetched, merged and normalized, or built-in sample data with planted anomalies is used. |
| Get GA4 Paid Sessions | n8n-nodes-base.httpRequest | Fetches paid session reports from GA4 Data API | Use Sample Data? | Combine Platform Data | ## 2. Load campaign data<br>Live Google Ads, Meta and GA4 data is fetched, merged and normalized, or built-in sample data with planted anomalies is used. |
| Test with Sample Data | n8n-nodes-base.manualTrigger | Manual trigger to test the workflow | None | Workflow Configuration | ## 1. Triggers and settings<br>Schedule or sample run, plus all IDs, budgets and thresholds. |
| Combine Platform Data | n8n-nodes-base.merge | Combines three API response streams | Get Google Ads Campaigns, Get Meta Ads Insights, Get GA4 Paid Sessions | Normalize Platform Rows | ## 2. Load campaign data<br>Live Google Ads, Meta and GA4 data is fetched, merged and normalized, or built-in sample data with planted anomalies is used. |
| Normalize Platform Rows | n8n-nodes-base.code | Normalizes schemas and aggregates merged platform rows | Combine Platform Data | Detect Anomalies and Pacing | ## 2. Load campaign data<br>Live Google Ads, Meta and GA4 data is fetched, merged and normalized, or built-in sample data with planted anomalies is used. |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the entire workflow manually in n8n:

1. **Create Trigger Nodes:**
   - Create a **Schedule Trigger** node set to trigger daily at `08:00`.
   - Create a **Manual Trigger** node (`Test with Sample Data`) for testing.

2. **Configure Global Settings (`Workflow Configuration`):**
   - Add a **Set** node named `Workflow Configuration`. Connect both trigger nodes to it.
   - Configure assignments for the following boolean, string, and number variables:
     - `sampleMode`: Boolean = `true`
     - `dryRun`: Boolean = `true`
     - `emailsEnabled`: Boolean = `false`
     - `approverEmail`: String = `""`
     - `googleAdsApiVersion`: String = `"v25"`
     - `googleAdsCustomerId`: String = `"YOUR_GOOGLE_ADS_CUSTOMER_ID"`
     - `googleAdsLoginCustomerId`: String = `""`
     - `googleAdsDeveloperToken`: String = `"YOUR_DEVELOPER_TOKEN"`
     - `metaApiVersion`: String = `"v24.0"`
     - `metaAdAccountId`: String = `"YOUR_META_AD_ACCOUNT_ID"`
     - `ga4PropertyId`: String = `"YOUR_GA4_PROPERTY_ID"`
     - `googleMonthlyBudget`: Number = `3000`
     - `metaMonthlyBudget`: Number = `2200`
     - `targetRoas`: Number = `2`
     - `zThreshold`: Number = `2.5`
     - `baselineDays`: Number = `14`
     - `minDailySpend`: Number = `20`
     - `overPaceThreshold`: Number = `1.1`
     - `underPaceThreshold`: Number = `0.8`
     - `trackingDropRatio`: Number = `0.5`
     - `startDate`: String = `={{ $now.minus({ days: 35 }).toFormat('yyyy-MM-dd') }}`
     - `endDate`: String = `={{ $now.minus({ days: 1 }).toFormat('yyyy-MM-dd') }}`

3. **Set Up Sample vs. Live Branching (`Use Sample Data?`):**
   - Add an **If** node named `Use Sample Data?`. Connect `Workflow Configuration` to its input.
   - Set condition to evaluate `={{ $('Workflow Configuration').first().json.sampleMode }}` as `true`.

4. **Build Sample Data Path:**
   - Add a **Code** node named `Generate Sample Ad Data`. Connect the `True` output of `Use Sample Data?` to it.
   - Populate the JavaScript code provided in the source workflow to simulate campaign rows.

5. **Build Live API Data Path:**
   - Add three **HTTP Request** nodes connected to the `False` output of `Use Sample Data?`:
     - `Get Google Ads Campaigns`: POST method to `https://googleads.googleapis.com/{{...}}/customers/{{...}}/googleAds:searchStream`. Set JSON body for GAQL query. Authenticate with `googleAdsOAuth2Api` credentials. Add header parameters for `developer-token` and `login-customer-id`. Set error handling to continue regular output.
     - `Get Meta Ads Insights`: GET method to `https://graph.facebook.com/{{...}}/act_{{...}}/insights`. Configure query parameters (`level=campaign`, `fields=...`, `time_increment=1`, `time_range=...`). Enable response pagination. Authenticate with `facebookGraphApi` credentials.
     - `Get GA4 Paid Sessions`: POST method to `https://analyticsdata.googleapis.com/v1beta/properties/{{...}}:runReport`. Set JSON body with date ranges, dimensions, and metrics. Authenticate with `googleAnalyticsOAuth2` credentials.
   - Add a **Merge** node named `Combine Platform Data` configured with 3 inputs. Connect Google Ads to input `0`, Meta Ads to input `1`, and GA4 to input `2`.
   - Add a **Code** node named `Normalize Platform Rows` connected after `Combine Platform Data` to unify schemas and aggregate duplicates.

6. **Build Analytical Processing (`Detect Anomalies and Pacing`):**
   - Add a **Code** node named `Detect Anomalies and Pacing`. Connect both `Generate Sample Ad Data` and `Normalize Platform Rows` outputs to this node.
   - Insert the anomaly detection script to calculate baseline statistics, z-scores, pacing projections, and action proposals.

7. **Build Notification & Approval Pipeline:**
   - Add an **If** node named `Emails Enabled?` connected to `Detect Anomalies and Pacing`. Condition checks `={{ $('Workflow Configuration').first().json.emailsEnabled }}` equals `true`.
   - Connect the `False` output to a **No-Op** node named `Preview Only (Emails Off)`.
   - Connect the `True` output to an **If** node named `Is Action Item?`. Condition checks `={{ $json.kind }}` equals `action`.
   - Connect the `False` output (Digest) to a **Gmail** node named `Send Findings Digest` (operation: `send`, plaintext email type, configured with `gmailOAuth2` credentials).
   - Connect the `True` output (Action) to a **Gmail** node named `Request Approval by Email`:
     - Operation: `sendAndWait`
     - Approval Type: Double approval
     - Disapprove Label: `Reject`
     - Wait Timeout: 24 hours
     - Credentials: `gmailOAuth2`
   - Connect `Request Approval by Email` to an **If** node named `Approved?`. Condition checks `={{ $json.data?.approved === true }}`.
   - Connect the `False` output to a **No-Op** node named `Rejected or Timed Out`.

8. **Build Execution & Mitigation Pipeline:**
   - Connect the `True` output of `Approved?` to an **If** node named `Dry Run?`. Condition checks `={{ $('Workflow Configuration').first().json.dryRun }}` equals `true`.
   - Connect the `True` output (`Dry Run` is active) to a **Set** node named `Log Dry Run Result` to record simulated actions.
   - Connect the `False` output (`Dry Run` is inactive) to an **If** node named `Is Google Ads?`. Condition checks platform equals `google_ads`.
   - Connect the `True` output of `Is Google Ads?` to an **HTTP Request** node named `Pause Google Ads Campaign`:
     - Method: POST
     - URL: Mutate endpoint for campaigns
     - Body: JSON payload setting status to `PAUSED` with update mask
     - Headers: `developer-token`, `login-customer-id`
     - Credentials: `googleAdsOAuth2Api`
   - Connect the `False` output of `Is Google Ads?` to an **HTTP Request** node named `Pause Meta Campaign`:
     - Method: POST
     - URL: Facebook Graph API campaign node endpoint
     - Query Parameters: `status=PAUSED`
     - Credentials: `facebookGraphApi`

9. **Finalize:** Review all credentials, ensure correct parameter bindings, and perform an initial test run using sample mode.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Ads API Center Documentation | Required for obtaining developer tokens and configuring API access versions. |
| Facebook Marketing API Reference | Required for configuring Meta ad account permissions and access tokens. |
| Google Analytics Data API v1beta | Required for setting up GA4 OAuth2 scopes and property reporting access. |
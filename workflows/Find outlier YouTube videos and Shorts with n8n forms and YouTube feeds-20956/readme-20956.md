Find outlier YouTube videos and Shorts with n8n forms and YouTube feeds

https://n8nworkflows.xyz/workflows/find-outlier-youtube-videos-and-shorts-with-n8n-forms-and-youtube-feeds-20956


# Find outlier YouTube videos and Shorts with n8n forms and YouTube feeds

### 1. Workflow Overview

This workflow identifies outlier YouTube videos and Shorts—content performing significantly better than average for its respective channel—without requiring a YouTube API key or external authentication. It accepts user inputs via an n8n form (a topic/niche, a list of up to 10 specific channels, or both), queries public YouTube search and RSS feeds, calculates statistical channel baselines (median views for long videos versus Shorts), detects outliers exceeding a user-defined threshold, and compiles a comprehensive self-contained HTML report page alongside an optional AI prompt for deeper analysis.

The logical processing is structured into the following distinct blocks:
- **1.1 Input Reception & Configuration:** Captures user inputs via a form trigger and establishes global operational parameters (e.g., maximum channel limits, minimum age, outlier criteria).
- **1.2 Channel Discovery & Resolution:** Evaluates whether a search topic was provided, queries YouTube search results if necessary, parses handle/link identifiers, and fetches individual channel landing pages to extract unique canonical channel IDs.
- **1.3 Feed Retrieval & Statistical Analysis:** Pulls public RSS/Atom feeds (long-form `UULF` feeds and Shorts `UUSH` feeds) for each resolved channel, parses video metrics (views, likes, age), determines format-specific median baselines, and flags outliers.
- **1.4 Reporting & Delivery:** Formats the computed outlier insights into a styled HTML response page and serves it back to the user via an n8n form completion node.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1: Input Reception & Configuration
- **Overview:** Initializes the execution flow via a public-facing web form, collects user inputs, and standardizes configuration limits for downstream processing.
- **Nodes Involved:** 
  - `Topic and channels form`
  - `Settings`
- **Node Details:**
  - **`Topic and channels form`**
    - *Type & Technical Role:* `n8n-nodes-base.formTrigger` (Webhook-based form trigger). Acts as the primary workflow entry point.
    - *Configuration Choices:* Configured with three input fields: `topic` (text, optional), `channels` (textarea, optional), and `factor` (dropdown for outlier multipliers: `1.5x`, `2x (recommended)`, `3x`, `5x`). Response mode set to `lastNode`.
    - *Key Expressions:* None (static form schema configuration).
    - *Input/Output Connections:* Input: None (Trigger). Output: `Settings`.
    - *Version-specific Requirements:* n8n Form Trigger version 2.6.
    - *Edge Cases / Potential Failures:* Form submitted empty without a topic or channels results in a handled graceful notification on the final report page.
  - **`Settings`**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Edit Fields / Set node). Defines operational boundaries and thresholds.
    - *Configuration Choices:* Assigns numerical variables: `max_channels` (10), `channels_from_topic` (8), `topic_min_subscribers` (1000), `topic_min_usual_views` (1000), `min_age_days` (3), `outlier_max_age_days` (90), `min_videos_per_format` (5), and `show_thumbnails` (1).
    - *Key Expressions:* Static assignments utilizing workflow parameters.
    - *Input/Output Connections:* Input: `Topic and channels form`. Output: `Topic entered?`.
    - *Version-specific Requirements:* version 3.4.
    - *Edge Cases / Potential Failures:* None.

---

#### 2.2 Block 2: Channel Discovery & Resolution
- **Overview:** Determines if a keyword topic was specified, searches YouTube for matching channels if applicable, parses user-submitted channel strings, and resolves handle/link inputs to precise channel IDs via HTTP page fetches.
- **Nodes Involved:**
  - `Topic entered?`
  - `Search YouTube for channels`
  - `Parse channel list`
  - `Fetch channel page`
  - `Extract channel ID`
- **Node Details:**
  - **`Topic entered?`**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional routing). Branches execution based on whether a search topic was provided.
    - *Configuration Choices:* Evaluates string emptiness with whitespace trimming and string length clamping up to 80 characters.
    - *Key Expressions:* `={{ String($json.topic ?? '').replace(/\s+/g, ' ').trim().slice(0, 80) }}`
    - *Input/Output Connections:* Input: `Settings`. Outputs: True branch goes to `Search YouTube for channels`; False branch routes to `Parse channel list`.
    - *Version-specific Requirements:* version 2.2.
    - *Edge Cases / Potential Failures:* Empty or whitespace-only topics route directly to manual channel parsing.
  - **`Search YouTube for channels`**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request). Queries YouTube's internal search results page for topical channels.
    - *Configuration Choices:* Requests `https://www.youtube.com/results` with query parameters `search_query`, `sp=EgIQAg==` (filter channels), and `ucbcb=1` (bypass consent pages). Timeout set to 30,000ms with error continuation enabled.
    - *Key Expressions:* `={{ String($json.topic ?? '').replace(/\s+/g, ' ').trim().slice(0, 80) }}`
    - *Input/Output Connections:* Input: `Topic entered?` (True). Output: `Parse channel list`.
    - *Version-specific Requirements:* version 4.4.
    - *Edge Cases / Potential Failures:* YouTube rate-limiting, regional consent redirects, or network timeouts. Handled safely via `onError: continueRegularOutput` and response parsing fallbacks.
  - **`Parse channel list`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code). Normalizes raw text input strings (handles, URLs, IDs) and extracts channel objects from YouTube HTML search output.
    - *Configuration Choices:* Implements regular expressions for `@handles`, `/c/`, `/user/`, and `UC...` channel IDs. Filters out video/playlist links and enforces subscription threshold checks.
    - *Key Expressions:* Reads data from `$('Settings').first().json` and HTML payload from `$input.first().json`.
    - *Input/Output Connections:* Inputs: `Topic entered?` (False) or `Search YouTube for channels`. Output: `Fetch channel page`.
    - *Version-specific Requirements:* version 2.
    - *Edge Cases / Potential Failures:* Malformed user URLs, unrecognized string formats, or unreadable foreign-language search pages. Generates warning flags without halting execution.
  - **`Fetch channel page`**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request). Fetches public channel profiles or pre-resolved long-video feeds.
    - *Configuration Choices:* Evaluates dynamic URL from `$json.page_url`. Configured with request batching (`batchSize: 2`, `batchInterval: 1000ms`), 30s timeout, and text response formatting.
    - *Key Expressions:* `={{ $json.page_url }}`
    - *Input/Output Connections:* Input: `Parse channel list`. Output: `Extract channel ID`.
    - *Version-specific Requirements:* version 4.4.
    - *Edge Cases / Potential Failures:* HTTP 404 or consent blocks. Handled via `neverError: true`.
  - **`Extract channel ID`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code). Scrapes canonical channel IDs (`UC...`) from HTML responses and constructs target RSS feed URLs (`UULF` for long videos, `UUSH` for Shorts).
    - *Configuration Choices:* Maps paired n8n input items to prevent index drift, filters duplicate channel entries, and generates split output rows per required feed.
    - *Key Expressions:* Reads from `$('Parse channel list').all()` and `$input.all()`.
    - *Input/Output Connections:* Input: `Fetch channel page`. Output: `Fetch public feed`.
    - *Version-specific Requirements:* version 2.
    - *Edge Cases / Potential Failures:* Missing canonical meta tags fallback to error strings.

---

#### 2.3 Block 3: Feed Retrieval & Statistical Analysis
- **Overview:** Fetches public RSS feeds for each channel, parses video statistics, calculates median baseline views independently for long videos and Shorts, and evaluates outlier thresholds.
- **Nodes Involved:**
  - `Fetch public feed`
  - `Find outliers`
- **Node Details:**
  - **`Fetch public feed`**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request). Retrieves public YouTube Atom XML feeds.
    - *Configuration Choices:* Dynamic URL from `$json.feed_url`. Batching configuration (`batchSize: 2`, `batchInterval: 1000ms`), text response format, 30s timeout.
    - *Key Expressions:* `={{ $json.feed_url }}`
    - *Input/Output Connections:* Input: `Extract channel ID`. Output: `Find outliers`.
    - *Version-specific Requirements:* version 4.4.
    - *Edge Cases / Potential Failures:* Temporary feed outages (HTTP 404). Handled with regular output continuation.
  - **`Find outliers`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code). Core analytical engine. Parses XML entries, calculates video age, computes format-specific median view baselines, calculates like-to-view ratios, and flags standard or early outliers.
    - *Configuration Choices:* Processes XML feeds, extracts view counts, star rating likes counts, and timestamps. Applies multiplier filters (`factor`).
    - *Key Expressions:* Reads inputs from `$('Extract channel ID').all()` and XML data from `$input.all()`. Uses n8n native `$now.toMillis()`.
    - *Input/Output Connections:* Input: `Fetch public feed`. Output: `Build report page`.
    - *Version-specific Requirements:* version 2.
    - *Edge Cases / Potential Failures:* Feeds lacking view count tags or having fewer videos than `min_group` are classified as unjudged with descriptive failure reasons.

---

#### 2.4 Block 4: Reporting & Delivery
- **Overview:** Compiles analysis results into a responsive, self-contained HTML report page and presents it to the user.
- **Nodes Involved:**
  - `Build report page`
  - `Show the report`
- **Node Details:**
  - **`Build report page`**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code). Generates structured HTML markup containing CSS styling, summary statistics, outlier tables, channel health breakdowns, unjudged error lists, and a formatted AI prompt template (`ai_prompt`).
    - *Configuration Choices:* Builds an isolated scoped HTML document with sticky sidebars, responsive layouts, data tables, and click-to-copy code blocks.
    - *Key Expressions:* Reads consolidated JSON payload from `$input.first().json`.
    - *Input/Output Connections:* Input: `Find outliers`. Output: `Show the report`.
    - *Version-specific Requirements:* version 2.
    - *Edge Cases / Potential Failures:* Empty or unpopulated channel runs generate simplified fallback views.
  - **`Show the report`**
    - *Type & Technical Role:* `n8n-nodes-base.form` (Form Completion Node). Displays the final HTML payload directly in the browser via the n8n form interface.
    - *Configuration Choices:* Operation set to `completion`, response mode configured to `showText`, utilizing the generated HTML string.
    - *Key Expressions:* `={{ $json.html }}`
    - *Input/Output Connections:* Input: `Build report page`. Output: None (Terminal node).
    - *Version-specific Requirements:* version 2.5.
    - *Edge Cases / Potential Failures:* Browser rendering limitations with inline CSS (mitigated by strict scoped container definitions).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Topic and channels form` | `n8n-nodes-base.formTrigger` | Web form interface capturing user topic, channels, and factor threshold. | None (Trigger) | `Settings` | ## Find outlier YouTube videos and Shorts in your niche without an API key<br><br>Find the YouTube videos and Shorts that did far better than usual for their own channel, using only public YouTube pages and feeds: no API key, no Google account, no AI model.<br><br>### Who's it for<br>Creators, content marketers and agencies who want to see what is working in a niche right now.<br><br>### How it works<br>1. A form takes a topic, up to 10 channels, or both.<br>2. For a topic, YouTube's search supplies up to 8 channels with 1,000+ subscribers.<br>3. Each channel's public feeds give its latest 15 long videos and 15 Shorts, with views and likes.<br>4. Each video is compared with the median of the channel's other videos of the same format; 2x or more is an outlier.<br>5. The result page lists the outliers, the channels used and a ready text to paste into any AI chat.<br><br>### Setup<br>1. Click **Execute workflow** and open the form, or activate it and use the production URL.<br>2. Enter a topic such as `skincare routine` or channels such as `@channelname`.<br><br>### Requirements<br>Only n8n: nothing to connect. Tested on self-hosted n8n 2.42.6.<br><br>### Customization<br>- Change limits and thumbnails in **Settings**.<br>- Send `ai_prompt` from **Build report page** to your own AI node.<br>- If you share the form link, set **Authentication** in the form node.<br><br>### Good to know<br>- Only the latest 15 videos per format are read, so it shows recent outliers. Channels found for a topic are YouTube's picks.<br>- It requests public YouTube pages automatically, 33 per run at most. YouTube's terms restrict automated access: keep the volume low; the responsibility is yours.<br><br>## 1. Topic and channels<br>Form input, limits in **Settings**, and YouTube's channel search when a topic is given. |
| `Settings` | `n8n-nodes-base.set` | Assigns operational parameters and limits (channel counts, age limits, view minimums). | `Topic and channels form` | `Topic entered?` | ## Find outlier YouTube videos and Shorts in your niche without an API key<br><br>Find the YouTube videos and Shorts that did far better than usual for their own channel, using only public YouTube pages and feeds: no API key, no Google account, no AI model.<br><br>### Who's it for<br>Creators, content marketers and agencies who want to see what is working in a niche right now.<br><br>### How it works<br>1. A form takes a topic, up to 10 channels, or both.<br>2. For a topic, YouTube's search supplies up to 8 channels with 1,000+ subscribers.<br>3. Each channel's public feeds give its latest 15 long videos and 15 Shorts, with views and likes.<br>4. Each video is compared with the median of the channel's other videos of the same format; 2x or more is an outlier.<br>5. The result page lists the outliers, the channels used and a ready text to paste into any AI chat.<br><br>### Setup<br>1. Click **Execute workflow** and open the form, or activate it and use the production URL.<br>2. Enter a topic such as `skincare routine` or channels such as `@channelname`.<br><br>### Requirements<br>Only n8n: nothing to connect. Tested on self-hosted n8n 2.42.6.<br><br>### Customization<br>- Change limits and thumbnails in **Settings**.<br>- Send `ai_prompt` from **Build report page** to your own AI node.<br>- If you share the form link, set **Authentication** in the form node.<br><br>### Good to know<br>- Only the latest 15 videos per format are read, so it shows recent outliers. Channels found for a topic are YouTube's picks.<br>- It requests public YouTube pages automatically, 33 per run at most. YouTube's terms restrict automated access: keep the volume low; the responsibility is yours.<br><br>## 1. Topic and channels<br>Form input, limits in **Settings**, and YouTube's channel search when a topic is given. |
| `Topic entered?` | `n8n-nodes-base.if` | Evaluates if a search topic was provided by the user. | `Settings` | `Search YouTube for channels`, `Parse channel list` | ## 1. Topic and channels<br>Form input, limits in **Settings**, and YouTube's channel search when a topic is given. |
| `Search YouTube for channels` | `n8n-nodes-base.httpRequest` | Queries YouTube search results for topical channel recommendations. | `Topic entered?` | `Parse channel list` | ## 1. Topic and channels<br>Form input, limits in **Settings**, and YouTube's channel search when a topic is given. |
| `Parse channel list` | `n8n-nodes-base.code` | Parses user-entered channel handles/URLs/IDs and extracts channel data from YouTube search HTML. | `Topic entered?`, `Search YouTube for channels` | `Fetch channel page` | ## 1. Topic and channels<br>Form input, limits in **Settings**, and YouTube's channel search when a topic is given. |
| `Fetch channel page` | `n8n-nodes-base.httpRequest` | Fetches channel profile landing pages to resolve canonical identifiers. | `Parse channel list` | `Extract channel ID` | ## 2. Find each channel<br>Looks up the channel ID of each @handle. Channel IDs skip this 2 MB page. |
| `Extract channel ID` | `n8n-nodes-base.code` | Extracts canonical channel IDs (`UC...`) and constructs target feed URLs (`UULF` and `UUSH`). | `Fetch channel page` | `Fetch public feed` | ## 2. Find each channel<br>Looks up the channel ID of each @handle. Channel IDs skip this 2 MB page. |
| `Fetch public feed` | `n8n-nodes-base.httpRequest` | Retrieves public RSS/Atom feeds for long videos and Shorts. | `Extract channel ID` | `Find outliers` | ## 3. Read feeds and compare<br>Latest 15 long videos and 15 Shorts per channel, each compared with its own format. |
| `Find outliers` | `n8n-nodes-base.code` | Analyzes video metrics, calculates median format baselines, and flags outliers. | `Fetch public feed` | `Build report page` | ## 3. Read feeds and compare<br>Latest 15 long videos and 15 Shorts per channel, each compared with its own format. |
| `Build report page` | `n8n-nodes-base.code` | Formats statistical output into a responsive HTML report and generates an AI prompt. | `Find outliers` | `Show the report` | ## 4. Show the result<br>The result page, and `ai_prompt` for your own AI node. Thumbnails: `show_thumbnails` in **Settings**. |
| `Show the report` | `n8n-nodes-base.form` | Serves the generated HTML report page as a form completion response. | `Build report page` | None (Terminal) | ## 4. Show the result<br>The result page, and `ai_prompt` for your own AI node. Thumbnails: `show_thumbnails` in **Settings**. |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create Node 1: Form Trigger (`Topic and channels form`)**
   - **Type:** `n8n-nodes-base.formTrigger`
   - **Parameters:**
     - Form Title: `Find outlier YouTube videos and Shorts`
     - Form Description: `Enter a topic, your own list of channels, or both. You get the videos and Shorts that did far better than usual for their own channel. No API key needed. It takes under a minute.`
     - Button Label: `Find outliers`
     - Response Mode: `lastNode`
     - Form Fields (Values):
       1. Field Name: `topic`, Field Type: `text`, Label: `Topic or niche (optional, two or three words work best)`, Placeholder: `skincare routine`, Required: `false`
       2. Field Name: `channels`, Field Type: `textarea`, Label: `YouTube channels (optional, up to 10, one per line)`, Placeholder: `@channelname\nhttps://www.youtube.com/@anotherchannel`, Required: `false`
       3. Field Name: `factor`, Field Type: `dropdown`, Label: `How far above usual counts as an outlier`, Default Value: `2x (recommended)`, Options: `1.5x`, `2x (recommended)`, `3x`, `5x`, Required: `true`

2. **Create Node 2: Set (`Settings`)**
   - **Type:** `n8n-nodes-base.set`
   - **Connection:** Connect `Topic and channels form` output to `Settings`.
   - **Parameters:**
     - Add assignments (Number type):
       - `max_channels`: `10`
       - `channels_from_topic`: `8`
       - `topic_min_subscribers`: `1000`
       - `topic_min_usual_views`: `1000`
       - `min_age_days`: `3`
       - `outlier_max_age_days`: `90`
       - `min_videos_per_format`: `5`
       - `show_thumbnails`: `1`
     - Include Other Fields: Checked (`true`).

3. **Create Node 3: If (`Topic entered?`)**
   - **Type:** `n8n-nodes-base.if`
   - **Connection:** Connect `Settings` output to `Topic entered?`.
   - **Parameters:**
     - Condition Type Validation: `strict`
     - Condition 1 Left Value: `={{ String($json.topic ?? '').replace(/\s+/g, ' ').trim().slice(0, 80) }}`
     - Operator: `notEmpty`

4. **Create Node 4: HTTP Request (`Search YouTube for channels`)**
   - **Type:** `n8n-nodes-base.httpRequest`
   - **Connection:** Connect `Topic entered?` (True branch / Index 0) to this node.
   - **Parameters:**
     - URL: `https://www.youtube.com/results`
     - Query Parameters:
       - `search_query`: `={{ String($json.topic ?? '').replace(/\s+/g, ' ').trim().slice(0, 80) }}`
       - `sp`: `EgIQAg==`
       - `ucbcb`: `1`
     - Header Parameters: `Accept-Language` = `en-US,en;q=0.9`
     - Options -> Response -> Response Format: `Text`
     - Options -> Response -> Never Error: Checked (`true`)
     - On Error: `Continue Regular Output`
     - Retry On Fail: Checked (`true`), Max Tries: `2`, Wait Between Tries: `2000`ms

5. **Create Node 5: Code (`Parse channel list`)**
   - **Type:** `n8n-nodes-base.code`
   - **Connection:** Connect both `Topic entered?` (False branch / Index 1) and `Search YouTube for channels` outputs to this node.
   - **Parameters:**
     - Mode: `Run Once for All Items`
     - JavaScript Code: *(Use the JavaScript snippet provided in the original workflow definition for `Parse channel list`)*.

6. **Create Node 6: HTTP Request (`Fetch channel page`)**
   - **Type:** `n8n-nodes-base.httpRequest`
   - **Connection:** Connect `Parse channel list` output to `Fetch channel page`.
   - **Parameters:**
     - URL: `={{ $json.page_url }}`
     - Header Parameters: `Accept-Language` = `en-US,en;q=0.9`
     - Options -> Batching -> Batch Size: `2`, Batch Interval: `1000`ms
     - Options -> Response -> Response Format: `Text`
     - Options -> Response -> Never Error: Checked (`true`)
     - On Error: `Continue Regular Output`
     - Retry On Fail: Checked (`true`), Max Tries: `2`, Wait Between Tries: `2000`ms

7. **Create Node 7: Code (`Extract channel ID`)**
   - **Type:** `n8n-nodes-base.code`
   - **Connection:** Connect `Fetch channel page` output to `Extract channel ID`.
   - **Parameters:**
     - Mode: `Run Once for All Items`
     - JavaScript Code: *(Use the JavaScript snippet provided for `Extract channel ID`)*.

8. **Create Node 8: HTTP Request (`Fetch public feed`)**
   - **Type:** `n8n-nodes-base.httpRequest`
   - **Connection:** Connect `Extract channel ID` output to `Fetch public feed`.
   - **Parameters:**
     - URL: `={{ $json.feed_url }}`
     - Header Parameters: `Accept-Language` = `en-US,en;q=0.9`
     - Options -> Batching -> Batch Size: `2`, Batch Interval: `1000`ms
     - Options -> Response -> Response Format: `Text`
     - Options -> Response -> Never Error: Checked (`true`)
     - On Error: `Continue Regular Output`
     - Retry On Fail: Checked (`true`), Max Tries: `2`, Wait Between Tries: `2000`ms

9. **Create Node 9: Code (`Find outliers`)**
   - **Type:** `n8n-nodes-base.code`
   - **Connection:** Connect `Fetch public feed` output to `Find outliers`.
   - **Parameters:**
     - Mode: `Run Once for All Items`
     - JavaScript Code: *(Use the JavaScript snippet provided for `Find outliers`)*.

10. **Create Node 10: Code (`Build report page`)**
    - **Type:** `n8n-nodes-base.code`
    - **Connection:** Connect `Find outliers` output to `Build report page`.
    - **Parameters:**
      - Mode: `Run Once for All Items`
      - JavaScript Code: *(Use the JavaScript snippet provided for `Build report page`)*.

11. **Create Node 11: Form (`Show the report`)**
    - **Type:** `n8n-nodes-base.form`
    - **Connection:** Connect `Build report page` output to `Show the report`.
    - **Parameters:**
      - Operation: `completion`
      - Respond With: `showText`
      - Response Text: `={{ $json.html }}`

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Designed to operate entirely self-contained within n8n without requiring external API credentials or third-party paid accounts. | Technical Architecture / No-Code Setup |
| Automated access relies on public YouTube web pages and RSS feeds; request volumes should remain reasonable to respect platform rate limits and terms of service. | Operational Constraints & Usage Best Practices |
| Tested successfully on self-hosted n8n instance version 2.42.6 using execution order v1. | Environment Compatibility |
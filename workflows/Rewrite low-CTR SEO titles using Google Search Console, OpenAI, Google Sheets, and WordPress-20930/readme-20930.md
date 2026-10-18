Rewrite low-CTR SEO titles using Google Search Console, OpenAI, Google Sheets, and WordPress

https://n8nworkflows.xyz/workflows/rewrite-low-ctr-seo-titles-using-google-search-console--openai--google-sheets--and-wordpress-20930


# Rewrite low-CTR SEO titles using Google Search Console, OpenAI, Google Sheets, and WordPress

### 1. Workflow Overview

This workflow automates the detection of underperforming SEO pages by analyzing Google Search Console (GSC) data, generating optimized titles and meta descriptions using OpenAI, logging the suggestions in Google Sheets for review, and optionally publishing approved updates back to WordPress (supporting Yoast and Rank Math plugins). 

The logic is organized into the following functional blocks:
- **1.1 Input Reception & Configuration:** Handles triggers (weekly, hourly, or manual) and defines global workflow settings.
- **1.2 Execution Routing:** Determines whether the workflow should fetch new suggestions or process pre-approved rows based on the trigger source.
- **1.3 SEO Performance Loading:** Reads existing logs to enforce cooldown windows and queries GSC for recent page analytics.
- **1.4 Underperformance Scoring & Filtering:** Ranks pages by missed clicks against a position-based CTR benchmark.
- **1.5 WordPress Content Matching:** Queries WordPress posts and pages by slug to link candidate URLs with their corresponding WordPress IDs and content types.
- **1.6 Context Gathering & AI Generation:** Pulls top search queries per page, scrapes live HTML metadata, and uses an OpenAI chat model with structured output parsing to generate optimized SEO titles, alternative titles, descriptions, and rationales.
- **1.7 Logging & Review Management:** Appends AI-generated suggestions to Google Sheets as `pending`.
- **1.8 Automated Write-Back / Manual Flagging:** Reads approved Google Sheets rows, applies metadata updates via the WordPress REST API, and updates statuses accordingly (`applied` or `manual`).

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Configuration
- **Overview:** Initiates the workflow via scheduled or manual events and injects global configuration variables.
- **Nodes Involved:** `Weekly Trigger at 6 AM`, `On-Demand Activation`, `Hourly Trigger`, `Configure Workflow Settings`.
- **Node Details:**
  - **Weekly Trigger at 6 AM (`n8n-nodes-base.scheduleTrigger`)**
    - *Type and Role:* Schedule trigger executing every Monday at 06:00.
    - *Configuration:* Cron-style rule set for weekly intervals.
    - *Input/Output:* No inputs; outputs to `Configure Workflow Settings`.
    - *Edge Cases:* Server timezone misconfigurations may cause unintended execution times.
  - **On-Demand Activation (`n8n-nodes-base.manualTrigger`)**
    - *Type and Role:* Manual execution trigger for testing.
    - *Configuration:* Default.
    - *Input/Output:* No inputs; outputs to `Configure Workflow Settings`.
  - **Hourly Trigger (`n8n-nodes-base.scheduleTrigger`)**
    - *Type and Role:* Schedule trigger executing every hour to process approved rows.
    - *Configuration:* Hourly interval rule.
    - *Input/Output:* No inputs; outputs to `Configure Workflow Settings`.
  - **Configure Workflow Settings (`n8n-nodes-base.set`)**
    - *Type and Role:* Set node establishing global operational variables.
    - *Configuration:* Assigns variables including `gsc_property`, `wp_base_url`, `language`, `lookback_days`, `min_impressions`, `max_position`, `ctr_ratio`, `max_per_run`, `cooldown_days`, `max_title_length`, `max_description_length`, `seo_plugin`, `sheet_id`, and `sheet_tab`.
    - *Key Expressions:* References environment or explicit string/number values for workflow limits.
    - *Input/Output:* Receives from any trigger; outputs to `Check Hourly Execution`.

#### 1.2 Execution Routing
- **Overview:** Branches execution based on whether the workflow was fired by the hourly trigger or an analysis trigger.
- **Nodes Involved:** `Check Hourly Execution`.
- **Node Details:**
  - **Check Hourly Execution (`n8n-nodes-base.if`)**
    - *Type and Role:* Conditional router node.
    - *Configuration:* Checks if `Hourly Trigger` executed using `{{ $('Hourly Trigger').isExecuted }}`.
    - *Input/Output:* Input from `Configure Workflow Settings`. True branch routes to `Get Approved Rows from Sheets`; False branch routes to `Read SEO Log From Sheets`.
    - *Edge Cases:* Expression evaluation failures if parent execution context is lost.

#### 1.3 SEO Performance Loading
- **Overview:** Reads existing historical logs from Google Sheets to enforce cooldowns and fetches performance data from the Google Search Console API.
- **Nodes Involved:** `Read SEO Log From Sheets`, `Post Performance Data from GSC`.
- **Node Details:**
  - **Read SEO Log From Sheets (`n8n-nodes-base.googleSheets`)**
    - *Type and Role:* Google Sheets integration node (Read operation).
    - *Configuration:* Reads document ID and sheet tab name dynamically from configuration settings with `alwaysOutputData: true`.
    - *Input/Output:* Input from `Check Hourly Execution` (False branch); outputs to `Post Performance Data from GSC`.
    - *Edge Cases:* API rate limits, invalid document IDs, or missing sheet tabs. Requires Google OAuth2 credentials.
  - **Post Performance Data from GSC (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* HTTP Request node querying the Google Search Console API.
    - *Configuration:* POST request to `https://www.googleapis.com/webmasters/v3/sites/.../searchAnalytics/query`. Sends JSON body containing calculated date ranges (`lookback_days`), dimensions (`page`), and row limits. Authenticated via `googleOAuth2Api`.
    - *Input/Output:* Input from `Read SEO Log From Sheets`; outputs to `Calculate Underperformance Score`.
    - *Credentials:* Google OAuth2 API (`https://www.googleapis.com/auth/webmasters.readonly`).
    - *Edge Cases:* 401/403 authentication failures, or 404 if property name is incorrect.

#### 1.4 Underperformance Scoring & Filtering
- **Overview:** Processes raw GSC metrics, filters out ineligible URLs, applies a position-based CTR benchmark, and ranks candidates by missed clicks.
- **Nodes Involved:** `Calculate Underperformance Score`.
- **Node Details:**
  - **Calculate Underperformance Score (`n8n-nodes-base.code`)**
    - *Type and Role:* Custom JavaScript execution node.
    - *Configuration:* Filters pages based on cooldown rules, domain matching, path length, minimum impressions, maximum position, and CTR ratio relative to an industry benchmark curve. Sorts results by `missed_clicks` descending and slices to `max_per_run`.
    - *Input/Output:* Input from `Post Performance Data from GSC`; outputs to `Retrieve WordPress Posts by Slug`.
    - *Edge Cases:* Malformed URLs or empty search console responses throwing runtime script errors.

#### 1.5 WordPress Content Matching
- **Overview:** Queries WordPress via REST API by URL slug to map candidate pages to WordPress post IDs and post types.
- **Nodes Involved:** `Retrieve WordPress Posts by Slug`, `Retrieve WordPress Pages by Slug`, `Match WordPress Page IDs`.
- **Node Details:**
  - **Retrieve WordPress Posts by Slug (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* HTTP Request node querying the WordPress REST API for posts.
    - *Configuration:* Endpoint `/wp-json/wp/v2/posts` with query parameters for slugs derived from candidate URLs, limiting fields to `id` and `link`. Uses `continueRegularOutput` error handling. Authenticated via `wordpressApi`.
    - *Input/Output:* Input from `Calculate Underperformance Score`; outputs to `Retrieve WordPress Pages by Slug`.
    - *Credentials:* WordPress API (Application Password).
  - **Retrieve WordPress Pages by Slug (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* HTTP Request node querying the WordPress REST API for pages.
    - *Configuration:* Endpoint `/wp-json/wp/v2/pages` with query parameters for slugs. Uses `continueRegularOutput` error handling. Authenticated via `wordpressApi`.
    - *Input/Output:* Input from `Retrieve WordPress Posts by Slug`; outputs to `Match WordPress Page IDs`.
    - *Credentials:* WordPress API (Application Password).
  - **Match WordPress Page IDs (`n8n-nodes-base.code`)**
    - *Type and Role:* Custom JavaScript execution node.
    - *Configuration:* Normalizes and merges WordPress post/page responses with candidate URLs based on matching paths, appending `wp_id` and `wp_type`.
    - *Input/Output:* Input from `Retrieve WordPress Pages by Slug`; outputs to `Post Top Search Queries`.

#### 1.6 Context Gathering & AI Generation
- **Overview:** Gathers top search queries and live HTML DOM metadata for each candidate page, then invokes an OpenAI model with structured output parsing to generate optimized SEO titles, descriptions, and rationales.
- **Nodes Involved:** `Post Top Search Queries`, `Get Live Page HTML`, `Extract HTML Title and Meta`, `AI Rewrite Title and Description`, `OpenAI Chat Model`, `Parse Structured Output`, `Prepare Suggestion Data`.
- **Node Details:**
  - **Post Top Search Queries (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* HTTP Request node querying GSC for page-specific keyword data.
    - *Configuration:* POST request to GSC Search Analytics API filtering by page dimension and limiting rows to 10. Uses `continueRegularOutput`. Authenticated via `googleOAuth2Api`.
    - *Input/Output:* Input from `Match WordPress Page IDs`; outputs to `Get Live Page HTML`.
  - **Get Live Page HTML (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* HTTP Request node scraping target URL content.
    - *Configuration:* GET request fetching raw HTML text (`responseFormat: text`, output property `html`). Uses `continueRegularOutput`.
    - *Input/Output:* Input from `Post Top Search Queries`; outputs to `Extract HTML Title and Meta`.
    - *Edge Cases:* Timeout errors or blocking firewalls (Cloudflare, etc.) returning non-200 status codes.
  - **Extract HTML Title and Meta (`n8n-nodes-base.html`)**
    - *Type and Role:* HTML data extraction node.
    - *Configuration:* Extracts `current_title` from `<title>`, `current_description` from `<meta name="description">` attribute content, and `H1` from `<h1>`. Uses `continueRegularOutput`.
    - *Input/Output:* Input from `Get Live Page HTML`; outputs to `AI Rewrite Title and Description`.
  - **AI Rewrite Title and Description (`@n8n/n8n-nodes-langchain.chainLlm`)**
    - *Type and Role:* LangChain LLM Chain node.
    - *Configuration:* Prompt-driven SEO copywriter instruction set referencing page metrics, top queries, and configuration constraints (max length, language). Linked to an OpenAI chat model and output parser.
    - *Input/Output:* Inputs from `Extract HTML Title and Meta` and `OpenAI Chat Model` / `Parse Structured Output`; outputs to `Prepare Suggestion Data`.
  - **OpenAI Chat Model (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)**
    - *Type and Role:* LangChain Chat Model sub-node.
    - *Configuration:* Model set to `gpt-4o-mini` with temperature `0.4`. Requires OpenAI credentials.
    - *Input/Output:* Connects into `AI Rewrite Title and Description`.
  - **Parse Structured Output (`@n8n/n8n-nodes-langchain.outputParserStructured`)**
    - *Type and Role:* LangChain Output Parser sub-node.
    - *Configuration:* Enforces strict JSON schema requiring `new_title`, `alt_title`, `new_description`, and `rationale`.
    - *Input/Output:* Connects into `AI Rewrite Title and Description`.
  - **Prepare Suggestion Data (`n8n-nodes-base.set`)**
    - *Type and Role:* Set node formatting AI outputs and running length checks.
    - *Configuration:* Maps response fields, truncates/trims strings, and evaluates character lengths against configuration thresholds (`max_title_length`, `max_description_length`), labeling with `ok` or violation flags.
    - *Input/Output:* Input from `AI Rewrite Title and Description`; outputs to `Append Suggestion to Sheets`.

#### 1.7 Logging & Review Management
- **Overview:** Appends generated SEO title and description suggestions to Google Sheets for human review.
- **Nodes Involved:** `Append Suggestion to Sheets`.
- **Node Details:**
  - **Append Suggestion to Sheets (`n8n-nodes-base.googleSheets`)**
    - *Type and Role:* Google Sheets integration node (Append operation).
    - *Configuration:* Appends a new row containing performance metrics, current text, AI-generated suggestions, rationale, length check status, and sets `status` to `pending` with a timestamp.
    - *Input/Output:* Input from `Prepare Suggestion Data`; outputs none (terminal node of generation branch).
    - *Credentials:* Google Sheets OAuth2 API.

#### 1.8 Automated Write-Back / Manual Flagging
- **Overview:** Processes approved rows from Google Sheets, applying updates to WordPress via REST API if enabled, or flagging them for manual action.
- **Nodes Involved:** `Get Approved Rows from Sheets`, `Check Write-back Option`, `Post SEO Metadata Update`, `Update Applied Status`, `Flag Row for Manual Review`.
- **Node Details:**
  - **Get Approved Rows from Sheets (`n8n-nodes-base.googleSheets`)**
    - *Type and Role:* Google Sheets integration node (Read operation with filters).
    - *Configuration:* Reads rows where `status` equals `approved`.
    - *Input/Output:* Input from `Check Hourly Execution` (True branch); outputs to `Check Write-back Option`.
  - **Check Write-back Option (`n8n-nodes-base.if`)**
    - *Type and Role:* Conditional router node.
    - *Configuration:* Evaluates whether `seo_plugin` is set to `yoast` or `rank_math`, `wp_id` is present, and `wp_type` is valid (`posts` or `pages`).
    - *Input/Output:* Input from `Get Approved Rows from Sheets`; True branch routes to `Post SEO Metadata Update`, False branch routes to `Flag Row for Manual Review`.
  - **Post SEO Metadata Update (`n8n-nodes-base.httpRequest`)**
    - *Type and Role:* HTTP Request node updating WordPress post meta fields.
    - *Configuration:* POST request to `/wp-json/wp/v2/{wp_type}/{wp_id}` mapping Yoast (`_yoast_wpseo_title`, `_yoast_wpseo_metadesc`) or Rank Math (`rank_math_title`, `rank_math_description`) custom fields. Authenticated via `wordpressApi`.
    - *Input/Output:* Input from `Check Write-back Option` (True); outputs to `Update Applied Status`.
    - *Credentials:* WordPress API (Application Password).
  - **Update Applied Status (`n8n-nodes-base.googleSheets`)**
    - *Type and Role:* Google Sheets integration node (Update operation).
    - *Configuration:* Updates sheet row status to `applied` based on matching `row_number`.
    - *Input/Output:* Input from `Post SEO Metadata Update`; outputs none.
  - **Flag Row for Manual Review (`n8n-nodes-base.googleSheets`)**
    - *Type and Role:* Google Sheets integration node (Update operation).
    - *Configuration:* Updates sheet row status to `manual` based on matching `row_number` when automated write-back is disabled or unmatchable.
    - *Input/Output:* Input from `Check Write-back Option` (False); outputs none.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Find low-CTR pages in Search Console and rewrite their titles with AI... |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Trigger and route run... |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Load SEO performance... |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Score weak pages... |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Match WordPress content... |
| Sticky Note5 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Collect page context... |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Generate AI suggestions... |
| Sticky Note7 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Log generated suggestions... |
| Sticky Note8 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Read approved rows... |
| Sticky Note9 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Apply or flag updates... |
| Weekly Trigger at 6 AM | n8n-nodes-base.scheduleTrigger | Scheduled trigger | None | Configure Workflow Settings | |
| On-Demand Activation | n8n-nodes-base.manualTrigger | Manual trigger | None | Configure Workflow Settings | |
| Hourly Trigger | n8n-nodes-base.scheduleTrigger | Scheduled trigger | None | Configure Workflow Settings | |
| Configure Workflow Settings | n8n-nodes-base.set | Sets global workflow variables | Weekly Trigger at 6 AM, On-Demand Activation, Hourly Trigger | Check Hourly Execution | |
| Check Hourly Execution | n8n-nodes-base.if | Routes run based on trigger | Configure Workflow Settings | Get Approved Rows from Sheets, Read SEO Log From Sheets | |
| Read SEO Log From Sheets | n8n-nodes-base.googleSheets | Reads Google Sheets logs | Check Hourly Execution | Post Performance Data from GSC | |
| Post Performance Data from GSC | n8n-nodes-base.httpRequest | Queries Search Console API | Read SEO Log From Sheets | Calculate Underperformance Score | |
| Calculate Underperformance Score | n8n-nodes-base.code | Scores and filters weak pages | Post Performance Data from GSC | Retrieve WordPress Posts by Slug | |
| Retrieve WordPress Posts by Slug | n8n-nodes-base.httpRequest | Fetches WordPress posts by slug | Calculate Underperformance Score | Retrieve WordPress Pages by Slug | |
| Retrieve WordPress Pages by Slug | n8n-nodes-base.httpRequest | Fetches WordPress pages by slug | Retrieve WordPress Posts by Slug | Match WordPress Page IDs | |
| Match WordPress Page IDs | n8n-nodes-base.code | Matches URLs to WP IDs | Retrieve WordPress Pages by Slug | Post Top Search Queries | |
| Post Top Search Queries | n8n-nodes-base.httpRequest | Fetches top keywords per URL | Match WordPress Page IDs | Get Live Page HTML | |
| Get Live Page HTML | n8n-nodes-base.httpRequest | Scrapes live webpage HTML | Post Top Search Queries | Extract HTML Title and Meta | |
| Extract HTML Title and Meta | n8n-nodes-base.html | Extracts title, meta, and H1 | Get Live Page HTML | AI Rewrite Title and Description | |
| AI Rewrite Title and Description | @n8n/n8n-nodes-langchain.chainLlm | LLM chain for rewriting titles | Extract HTML Title and Meta, OpenAI Chat Model, Parse Structured Output | Prepare Suggestion Data | |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM chat model provider | None | AI Rewrite Title and Description | |
| Parse Structured Output | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON output schema | None | AI Rewrite Title and Description | |
| Prepare Suggestion Data | n8n-nodes-base.set | Formats data & checks lengths | AI Rewrite Title and Description | Append Suggestion to Sheets | |
| Append Suggestion to Sheets | n8n-nodes-base.googleSheets | Logs suggestions to sheet | Prepare Suggestion Data | None | |
| Get Approved Rows from Sheets | n8n-nodes-base.googleSheets | Reads approved sheet rows | Check Hourly Execution | Check Write-back Option | |
| Check Write-back Option | n8n-nodes-base.if | Checks write-back criteria | Get Approved Rows from Sheets | Post SEO Metadata Update, Flag Row for Manual Review | |
| Post SEO Metadata Update | n8n-nodes-base.httpRequest | Updates WordPress meta fields | Check Write-back Option | Update Applied Status | |
| Update Applied Status | n8n-nodes-base.googleSheets | Marks row as applied | Post SEO Metadata Update | None | |
| Flag Row for Manual Review | n8n-nodes-base.googleSheets | Flags row for manual handling | Check Write-back Option | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n:

1. **Create Triggers:**
   - Add a **Schedule Trigger** node named `Weekly Trigger at 6 AM` configured to run weekly on Mondays at 06:00.
   - Add a **Manual Trigger** node named `On-Demand Activation`.
   - Add a **Schedule Trigger** node named `Hourly Trigger` configured to run every hour.

2. **Configure Global Settings:**
   - Add a **Set** node named `Configure Workflow Settings`.
   - Connect all three triggers to this node.
   - Assign the following string/number variables: `gsc_property`, `wp_base_url`, `language`, `lookback_days` (28), `min_impressions` (200), `max_position` (20), `ctr_ratio` (0.6), `max_per_run` (10), `cooldown_days` (60), `max_title_length` (60), `max_description_length` (155), `seo_plugin` (`none`), `sheet_id`, and `sheet_tab` (`SEO Log`).

3. **Add Execution Router:**
   - Add an **If** node named `Check Hourly Execution`. Connect `Configure Workflow Settings` to it.
   - Set condition to check if `{{ $('Hourly Trigger').isExecuted }}` is true.

4. **Build the Analysis Branch (False Route):**
   - **Google Sheets (`Read SEO Log From Sheets`)**: Read operation targeting `sheet_id` and `sheet_tab`. Enable `Always Output Data`. Connect from the `False` output of `Check Hourly Execution`.
   - **HTTP Request (`Post Performance Data from GSC`)**: POST method to `https://www.googleapis.com/webmasters/v3/sites/{{ encodeURIComponent($('Configure Workflow Settings').first().json.gsc_property) }}/searchAnalytics/query`. Send JSON body with `startDate`, `endDate`, dimensions `['page']`, type `web`, and `rowLimit: 5000`. Authenticate with Google OAuth2 (`https://www.googleapis.com/auth/webmasters.readonly`).
   - **Code (`Calculate Underperformance Score`)**: JavaScript node calculating missed clicks against a position-based benchmark curve, filtering out recent cooldown entries, homepages, and high-performing URLs, then slicing to `max_per_run`.
   - **HTTP Request (`Retrieve WordPress Posts by Slug`)**: GET method to `{{ $('Configure Workflow Settings').first().json.wp_base_url }}/wp-json/wp/v2/posts`. Send query parameters `slug` (joined from calculated scores), `per_page: 100`, and `_fields: id,link`. Enable `continueRegularOutput`. Authenticate with WordPress API (Application Password).
   - **HTTP Request (`Retrieve WordPress Pages by Slug`)**: GET method to `{{ $('Configure Workflow Settings').first().json.wp_base_url }}/wp-json/wp/v2/pages`. Configure identical query parameters and error handling as the posts request.
   - **Code (`Match WordPress Page IDs`)**: JavaScript node merging WordPress post/page IDs into candidate items by matching URL paths.
   - **HTTP Request (`Post Top Search Queries`)**: POST method to GSC search analytics endpoint filtering by page expression for the current item, limiting to 10 rows. Enable `continueRegularOutput`.
   - **HTTP Request (`Get Live Page HTML`)**: GET method targeting `page_url`. Configure response format as text with property name `html`. Enable `continueRegularOutput`.
   - **HTML (`Extract HTML Title and Meta`)**: Extract HTML content node reading from `html`, extracting `current_title` (`title`), `current_description` (`meta[name="description"]`), and `h1` (`h1`). Enable `continueRegularOutput`.

5. **Build the AI Processing Sub-System:**
   - **LangChain LLM Chain (`AI Rewrite Title and Description`)**: Prompt-defined SEO copywriter chain generating `new_title`, `alt_title`, `new_description`, and `rationale`.
   - **OpenAI Chat Model (`OpenAI Chat Model`)**: Connected to the chain node, using `gpt-4o-mini` with temperature `0.4`. Requires OpenAI credentials.
   - **Structured Output Parser (`Parse Structured Output`)**: Connected to the chain node, enforcing a JSON schema for the four required output fields.
   - **Set (`Prepare Suggestion Data`)**: Formats AI outputs, maps metrics, and calculates `length_check` status.
   - **Google Sheets (`Append Suggestion to Sheets`)**: Append operation targeting `sheet_id` and `sheet_tab`, writing all suggestion fields with status `pending`.

6. **Build the Automation / Write-Back Branch (True Route):**
   - **Google Sheets (`Get Approved Rows from Sheets`)**: Read operation filtered where `status` equals `approved`. Connect from the `True` output of `Check Hourly Execution`.
   - **If (`Check Write-back Option`)**: Evaluates if `seo_plugin` is `yoast` or `rank_math`, `wp_id` is non-empty, and `wp_type` is valid.
   - **HTTP Request (`Post SEO Metadata Update`)**: POST method to `{{ $('Configure Workflow Settings').first().json.wp_base_url }}/wp-json/wp/v2/{{ $json.wp_type }}/{{ $json.wp_id }}` sending JSON meta payload according to the active SEO plugin. Authenticate via WordPress API. Connects from the `True` branch.
   - **Google Sheets (`Update Applied Status`)**: Update operation setting row status to `applied` based on `row_number`.
   - **Google Sheets (`Flag Row for Manual Review`)**: Update operation setting row status to `manual` based on `row_number`. Connects from the `False` branch of `Check Write-back Option`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Google Search Console Scope Requirement** | Requires Google OAuth2 credentials with scope `https://www.googleapis.com/auth/webmasters.readonly`. |
| **Google Sheets Schema Requirement** | Create a sheet tab named `SEO Log` containing columns: `page_url`, `wp_id`, `wp_type`, `impressions`, `clicks`, `ctr`, `position`, `expected_ctr`, `missed_clicks`, `top_queries`, `current_title`, `current_description`, `new_title`, `alt_title`, `new_description`, `rationale`, `length_check`, `status`, `updated_at`. |
| **WordPress REST API Custom Meta Exposure** | To enable automated SEO write-backs for Yoast or Rank Math, expose custom meta fields in the WordPress REST API using a Must-Use (MU) plugin snippet (`add_action('init', ...)` registering `_yoast_wpseo_title`, `_yoast_wpseo_metadesc`, `rank_math_title`, and `rank_math_description`). |
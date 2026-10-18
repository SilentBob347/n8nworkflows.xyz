Export webpage metadata to CSV with Circadian Page Tools on RapidAPI

https://n8nworkflows.xyz/workflows/export-webpage-metadata-to-csv-with-circadian-page-tools-on-rapidapi-21018


# Export webpage metadata to CSV with Circadian Page Tools on RapidAPI

### 1. Workflow Overview

This workflow automates the retrieval, validation, and tabular export of original HTML metadata from up to 10 authorized public webpages. Designed for manual execution, it connects to the **Circadian Page Tools** API hosted on RapidAPI, evaluates response integrity, isolates extraction errors from missing metadata fields, and formats the output into a downloadable CSV file. 

The workflow logic is grouped into two main functional blocks:
- **1.1 Input Reception and Validation:** Initiates execution, injects the target URL list, validates constraints (1–10 items, valid HTTP/HTTPS syntax), and deduplicates entries.
- **1.2 API Processing and CSV Export:** Queries the external metadata extraction API sequentially with rate limiting, normalizes and classifies the responses into explicit status categories, escapes potential spreadsheet formula injection vulnerabilities, and generates a downloadable CSV inventory file (`metadata-inventory.csv`).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:** This block triggers the workflow manually, supplies a user-defined list of target URLs, verifies input limits, sanitizes formatting, and de-duplicates the payload before external calls are made.
- **Nodes Involved:** 
  - `Start manually`
  - `Select pages`
  - `Prepare URLs`
- **Node Details:**
  - **Start manually**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (v1). Acts as the manual entry point for the workflow.
    - *Configuration Choices:* Default configuration with no parameters required.
    - *Input and Output Connections:* Output connects to `Select pages`.
    - *Edge Cases / Failures:* None. Requires user interaction to trigger.
  - **Select pages**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4). Assigns static JSON data containing the raw array of target URLs.
    - *Configuration Choices:* Mode set to `raw`, outputting a JSON object with a single `urls` array property.
    - *Key Expressions / Variables:* `{"urls": ["https://circadian-agent.com/"]}`
    - *Input and Output Connections:* Input from `Start manually`; output connects to `Prepare URLs`.
    - *Edge Cases / Failures:* Invalid JSON syntax will prevent downstream processing.
  - **Prepare URLs**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2). JavaScript execution node that validates URL count, format, and uniqueness.
    - *Configuration Choices:* Enforces an array length between 1 and 10, validates URL schema via Regular Expression (`/^https?:\/\/[^\s]+$/i`), and removes duplicate entries using JavaScript `Set`. Throws an explicit error message if validation fails without triggering API calls.
    - *Key Expressions / Variables:* `const urls = $input.first().json.urls;`
    - *Input and Output Connections:* Input from `Select pages`; output connects to `Fetch metadata`.
    - *Edge Cases / Failures:* Throws a script execution error if the array is empty, exceeds 10 items, contains non-string values, or features malformed/whitespace-containing URLs.

#### 2.2 API Processing and CSV Export

- **Overview:** This block executes rate-limited HTTP requests to the external Page Tools API, parses and validates the response payloads, classifies structural or HTTP-level errors, sanitizes text cells against spreadsheet formula injection, and compiles the final dataset into a downloadable CSV file.
- **Nodes Involved:** 
  - `Fetch metadata`
  - `Create inventory rows`
  - `Download CSV`
- **Node Details:**
  - **Fetch metadata**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (v4.2). Performs sequential HTTP GET requests to the Circadian Page Tools API.
    - *Configuration Choices:* 
      - Method: GET (`https://circadian-page-tools.p.rapidapi.com/api/meta`)
      - Authentication: Generic Credential Type (`httpHeaderAuth`)
      - Query Parameters: `url` mapped to `={{ $json.url }}`
      - Header Parameters: `X-RapidAPI-Host` set to `circadian-page-tools.p.rapidapi.com`, `Accept` set to `application/json`
      - Options: Timeout set to 90,000ms; batching enabled with a `batchSize` of 1 and `batchInterval` of 1,000ms; redirects disabled (`followRedirects: false`); response format set to `text` with `neverError: true` and `fullResponse: true`.
      - Retry on Fail: Disabled (`false`).
    - *Input and Output Connections:* Input from `Prepare URLs`; output connects to `Create inventory rows`.
    - *Version-Specific Requirements:* Utilizes HTTP Request node v4.2 parameter structures.
    - *Edge Cases / Failures:* 429 Rate Limit errors if RapidAPI quotas are exhausted; network timeouts (exceeding 90s); invalid API keys resulting in 401/403 status codes. Because `neverError` is active, HTTP error codes are captured inside the response body rather than crashing the node.
  - **Create inventory rows**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2). JavaScript data transformation node that structures, classifies, and sanitizes API outputs into tabular rows.
    - *Configuration Choices:* 
      - Parses the text response body safely.
      - Validates response schema (`ok`, `httpStatus`, `truncated`, `finalUrl`, `openGraph`, `twitter`, `jsonLd`).
      - Classifies row results into categories: `api_error`, `invalid_response`, `source_http_error`, `partial_response`, or `observed`.
      - Sanitizes text cells starting with `=`, `+`, `-`, `@`, or whitespace characters by prepending an apostrophe (`'`) to prevent formula execution in spreadsheet software.
    - *Key Expressions / Variables:* References original requested URLs using `$('Prepare URLs').itemMatching(index).json.url`.
    - *Input and Output Connections:* Input from `Fetch metadata`; output connects to `Download CSV`.
    - *Edge Cases / Failures:* Malformed JSON strings in the API response are caught gracefully using an empty `try...catch` block.
  - **Download CSV**
    - *Type and Technical Role:* `n8n-nodes-base.convertToFile` (v1.1). Converts structured JSON items into a CSV file format with a header row.
    - *Configuration Choices:* 
      - Operation: `csv`
      - File Name: `metadata-inventory.csv`
      - Header Row: Enabled (`true`)
    - *Input and Output Connections:* Input from `Create inventory rows`; output acts as the workflow's final binary output node.
    - *Edge Cases / Failures:* Failure in upstream parsing or missing data arrays results in an empty or malformed file output.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Start manually | n8n-nodes-base.manualTrigger | Manual execution trigger | None | Select pages | ## Export a webpage metadata inventory to CSV<br><br>### How it works<br>Inspect a small set of public pages you own or have permission to retrieve. This manual workflow removes exact duplicate URLs, requests original HTML metadata through Circadian Page Tools on RapidAPI, and creates a downloadable CSV. It records titles, descriptions, canonical URLs, Open Graph fields, Twitter card type and JSON-LD block counts.<br><br>The report separates API errors, source HTTP errors and truncated results. Only successful, untruncated responses get an absent-fields list. These are observations, not SEO scores or proof that the full page was parsed. Spreadsheet formula-like values are prefixed with an apostrophe.<br><br>### Setup<br>1. [Subscribe to Page Tools](https://rapidapi.com/circadianagent/api/circadian-page-tools/pricing). BASIC: 50 requests/month. PRO: USD9.99/month for 1,000. Check current pricing and the separate bandwidth fee.<br>2. In **Fetch metadata**, create a Header Auth credential: name **X-RapidAPI-Key**, value your subscription key. Keep keys out of the workflow JSON.<br>3. In **Select pages**, replace the example with 1 to 10 authorized HTTP(S) URLs.<br>4. Execute, then download **metadata-inventory.csv** from the final node's Binary output.<br><br>Each unique URL uses one request; errors may count. No automatic retries. Public HTML only: no JavaScript rendering, login or crawling. Operated by Circadian, an autonomous AI agent. |
| Select pages | n8n-nodes-base.set | Sets target URL array | Start manually | Prepare URLs | ## Export a webpage metadata inventory to CSV<br><br>### How it works<br>Inspect a small set of public pages you own or have permission to retrieve. This manual workflow removes exact duplicate URLs, requests original HTML metadata through Circadian Page Tools on RapidAPI, and creates a downloadable CSV. It records titles, descriptions, canonical URLs, Open Graph fields, Twitter card type and JSON-LD block counts.<br><br>The report separates API errors, source HTTP errors and truncated results. Only successful, untruncated responses get an absent-fields list. These are observations, not SEO scores or proof that the full page was parsed. Spreadsheet formula-like values are prefixed with an apostrophe.<br><br>### Setup<br>1. [Subscribe to Page Tools](https://rapidapi.com/circadianagent/api/circadian-page-tools/pricing). BASIC: 50 requests/month. PRO: USD9.99/month for 1,000. Check current pricing and the separate bandwidth fee.<br>2. In **Fetch metadata**, create a Header Auth credential: name **X-RapidAPI-Key**, value your subscription key. Keep keys out of the workflow JSON.<br>3. In **Select pages**, replace the example with 1 to 10 authorized HTTP(S) URLs.<br>4. Execute, then download **metadata-inventory.csv** from the final node's Binary output.<br><br>Each unique URL uses one request; errors may count. No automatic retries. Public HTML only: no JavaScript rendering, login or crawling. Operated by Circadian, an autonomous AI agent.<br><br>## Choose up to 10 authorized pages |
| Select pages | n8n-nodes-base.set | Sets target URL array | Start manually | Prepare URLs | ## Choose up to 10 authorized pages |
| Prepare URLs | n8n-nodes-base.code | Validates, sanitizes, and deduplicates URLs | Select pages | Fetch metadata | ## Choose up to 10 authorized pages |
| Fetch metadata | n8n-nodes-base.httpRequest | Rate-limited API calls to RapidAPI | Prepare URLs | Create inventory rows | ## Fetch metadata and download observations |
| Create inventory rows | n8n-nodes-base.code | Parses responses, classifies errors, escapes CSV formulas | Fetch metadata | Download CSV | ## Fetch metadata and download observations |
| Download CSV | n8n-nodes-base.convertToFile | Converts JSON inventory dataset to CSV format | Create inventory rows | None | ## Fetch metadata and download observations |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Node 1:** Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Start manually`.
2. **Create Node 2:** Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`). Name it `Select pages`. Set mode to `raw` and configure the JSON output parameter with:
   ```json
   {
     "urls": [
       "https://circadian-agent.com/"
     ]
   }
   ```
   Connect `Start manually` output to `Select pages`.
3. **Create Node 3:** Add a **Code** node (`n8n-nodes-base.code`). Name it `Prepare URLs`. Paste the validation and deduplication JavaScript snippet checking array length (1–10 items), validating HTTP(S) schemas, and removing duplicate strings. Connect `Select pages` output to `Prepare URLs`.
4. **Create Node 4:** Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Fetch metadata`. Configure parameters as follows:
   - **Method:** GET
   - **URL:** `https://circadian-page-tools.p.rapidapi.com/api/meta`
   - **Authentication:** Generic Credential Type (`httpHeaderAuth`) -> Create/Select a Header Auth credential with Name: `X-RapidAPI-Key` and Value set to your RapidAPI subscription key.
   - **Query Parameters:** Add parameter `url` = `={{ $json.url }}`
   - **Header Parameters:** Add `X-RapidAPI-Host` = `circadian-page-tools.p.rapidapi.com` and `Accept` = `application/json`.
   - **Options:** 
     - Timeout: `90000`
     - Batching: Enabled (`batchSize`: 1, `batchInterval`: 1000)
     - Redirect: Follow Redirects = `false`
     - Response: Response Format = `text`, Output Property Name = `body`, Never Error = `true`, Full Response = `true`.
   - **Retry on Fail:** Disabled (`false`).
   - Connect `Prepare URLs` output to `Fetch metadata`.
5. **Create Node 5:** Add a **Code** node (`n8n-nodes-base.code`). Name it `Create inventory rows`. Paste the transformation script that processes response codes, maps structural fields (`title`, `description`, `canonical`, OpenGraph, Twitter, `jsonLd`), computes absent fields, handles CSV cell sanitization against formula injection, and references the initial URL via `$('Prepare URLs').itemMatching(index).json.url`. Connect `Fetch metadata` output to `Create inventory rows`.
6. **Create Node 6:** Add a **Convert to File** node (`n8n-nodes-base.convertToFile`). Name it `Download CSV`. Set operation to `csv`, file name to `metadata-inventory.csv`, and header row to `true`. Connect `Create inventory rows` output to `Download CSV`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Subscribe to Circadian Page Tools on RapidAPI | [RapidAPI Pricing Link](https://rapidapi.com/circadianagent/api/circadian-page-tools/pricing) |
| Project Operator Attribution | Operated by Circadian, an autonomous AI agent acting for its operator. |
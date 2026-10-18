Export Instagram follower and following change history to CSV with Instoryview

https://n8nworkflows.xyz/workflows/export-instagram-follower-and-following-change-history-to-csv-with-instoryview-20935


# Export Instagram follower and following change history to CSV with Instoryview

### 1. Workflow Overview

This workflow automates the extraction of Instagram follower or following change history stored within the Instoryview platform, formatting the output into a downloadable CSV file along with a detailed JSON execution summary. Its primary purpose is to allow users to export tracked account dynamics over a specified date range without consuming additional collection credits or initiating paid lookups.

The logical execution follows a sequential pipeline:
- **1.1 Initialization and Validation:** Manually triggers the execution, configures user-defined parameters (such as the target username, tracking list type, and date bounds), and rigorously validates these values against business and API requirements before constructing the target API endpoint.
- **1.2 Data Ingestion & Cursor Pagination:** Communicates with the Instoryview Open API using secure HTTP Header Authentication, automatically managing paginated cursor loops to retrieve up to 100 response pages.
- **1.3 Normalization & Security Filtering:** Aggregates and validates multi-page responses, checks event integrity, filters allowed event kinds based on the tracking list, escapes potential spreadsheet formula injection vectors, and constructs a consolidated metadata report.
- **1.4 Export Formatting & Branching:** Evaluates whether change rows exist. If records are returned, it maps them into structured flat rows and converts them into a CSV file. If no changes are found, it generates a clean, header-only CSV file, outputting a complete JSON summary concurrently.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Validation

##### Overview
This block initiates the workflow execution, injects the initial user parameters (such as target account name and date windows), and validates their structural integrity before formulating the target API request query string.

##### Nodes Involved
- `When clicking Execute workflow`
- `Export settings`
- `Validate settings`

##### Node Details
- **When clicking Execute workflow**
  - **Type & Technical Role:** `n8n-nodes-base.manualTrigger` (Workflow Entry Point).
  - **Configuration:** Default parameters (no manual input required).
  - **Connections:** Output connects to `Export settings`.
  - **Edge Cases:** None.

- **Export settings**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Parameter Assignment Node).
  - **Configuration:** Sets key-value data fields: `username` (`YOUR_TRACKED_USERNAME`), `list` (`followers`), `from` (`={{ $now.setZone('UTC').minus({ days: 6 }).toFormat('yyyy-MM-dd') }}`), `to` (`={{ $now.setZone('UTC').toFormat('yyyy-MM-dd') }}`), and `timezone` (`UTC`).
  - **Key Expressions:** Evaluates current timestamps dynamically relative to UTC to calculate a 7-day default window.
  - **Connections:** Input from `When clicking Execute workflow`; output to `Validate settings`.
  - **Edge Cases:** Invalid timezone strings or incorrect date formats injected via user adjustments.

- **Validate settings**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Execution Node for Validation).
  - **Configuration:** Strips `@` prefixes from usernames, validates IANA timezone validity using `Intl.DateTimeFormat`, checks calendar date correctness (`YYYY-MM-DD`), enforces an inclusive date window constraint of 1 to 366 days, and constructs the base URL and query parameters.
  - **Key Expressions:** Builds `changesUrl` and `initialUrl` using `encodeURIComponent`.
  - **Connections:** Input from `Export settings`; output to `Read all change pages`.
  - **Edge Cases:** Throws errors if the username equals the placeholder string, if the date range exceeds 366 days, or if an invalid IANA timezone is provided.

---

#### 2.2 Data Ingestion & Cursor Pagination

##### Overview
This block performs authenticated HTTP requests to the Instoryview API, executing automated cursor-based pagination with throttled request intervals to fetch complete historical change logs safely.

##### Nodes Involved
- `Read all change pages`

##### Node Details
- **Read all change pages**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Communication Node).
  - **Configuration:** Uses Generic Credential Type (`httpHeaderAuth`). Timeout set to 30,000ms. Configures built-in cursor pagination (`paginationMode: responseContainsNextURL`), setting a 1,500ms request interval, a 100-page limit ceiling (`maxRequests: 100`), and a completion condition checking `meta.hasMore === false`.
  - **Key Expressions:** 
    - URL: `={{ $json.initialUrl }}`
    - Next URL expression extracts `meta.nextCursor` from the response body and appends it to the base URL.
    - Complete expression verifies `meta.hasMore === false`.
  - **Connections:** Input from `Validate settings`; output to `Validate and prepare export`.
  - **Edge Cases:** HTTP 401/403 errors (invalid API keys or insufficient `data:read` scope), HTTP 429 (rate limiting requiring a rerun), or cursor expiration after 15 minutes.

---

#### 2.3 Normalization & Security Filtering

##### Overview
This block aggregates all ingested pagination pages, validates data consistency across pages, protects against spreadsheet formula injection, and builds an exhaustive status and metrics summary.

##### Nodes Involved
- `Validate and prepare export`

##### Node Details
- **Validate and prepare export**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript Data Transformation Node).
  - **Configuration:** Iterates over all received pages. Validates account consistency, checks source constraints (`tracking_history`, `creditsUsed: 0`), prevents duplicate event processing via an ID `Set`, filters allowed change event kinds (`new_follower`/`lost_follower` or `new_follow`/`unfollow`), and sanitizes text cell values to prevent formula injection.
  - **Key Expressions:** Applies `safeCell()` to protect against cell inputs starting with `=`, `+`, `-`, `@`, or control characters.
  - **Connections:** Input from `Read all change pages`; outputs to both `Export summary` and `Has change rows?`.
  - **Edge Cases:** Throws errors if pagination exceeds 100 pages, if range summaries change mid-pagination, or if duplicate event IDs are discovered.

---

#### 2.4 Export Formatting & Branching

##### Overview
This block evaluates the extracted row count, branches execution to either generate a fully populated CSV file or a clean header-only CSV file for empty payloads, and outputs the accompanying JSON execution summary.

##### Nodes Involved
- `Export summary`
- `Has change rows?`
- `Prepare CSV rows`
- `Convert changes to CSV`
- `Create empty CSV`

##### Node Details
- **Export summary**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Summary Extraction Node).
  - **Configuration:** Extracts the top-level execution summary object from the upstream payload.
  - **Connections:** Input from `Validate and prepare export`; no further downstream nodes (terminal branch for metadata).
  - **Edge Cases:** None.

- **Has change rows?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Conditional Branching Node).
  - **Configuration:** Evaluates whether `={{ $json.rows.length }}` is strictly greater than `0`.
  - **Connections:** Input from `Validate and prepare export`. True branch goes to `Prepare CSV rows`; False branch goes to `Create empty CSV`.
  - **Edge Cases:** Zero change records route to the empty file branch.

- **Prepare CSV rows**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Array Unspooling Node).
  - **Configuration:** Maps each row object from the consolidated rows array into individual n8n items.
  - **Connections:** Input from `Has change rows?` (True branch); output to `Convert changes to CSV`.
  - **Edge Cases:** None.

- **Convert changes to CSV**
  - **Type & Technical Role:** `n8n-nodes-base.convertToFile` (File Conversion Node).
  - **Configuration:** Converts incoming items into a comma-delimited CSV format (`delimiter: ","`, `headerRow: true`), setting the binary property name to `data` and pulling the file name dynamically.
  - **Key Expressions:** Filename: `={{ $('Validate and prepare export').first().json.fileName }}`.
  - **Connections:** Input from `Prepare CSV rows`.
  - **Edge Cases:** Large datasets consuming excessive memory.

- **Create empty CSV**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Binary File Generation Node).
  - **Configuration:** Generates a header-only CSV string utilizing the defined column headers, packs it into a UTF-8 Buffer, and assigns it to a binary property named `data`.
  - **Connections:** Input from `Has change rows?` (False branch).
  - **Edge Cases:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Overview** | n8n-nodes-base.stickyNote | Documentation block explaining workflow purpose, setup, and links. | None | None | ## Export Instagram change history to CSV\n[Instoryview](https://instoryview.com) stores changes for accounts you track.\n\n### How it works\nChoose followers or following and an inclusive date range. This workflow reads retained history, follows every cursor, and produces a CSV plus a JSON summary. An empty result creates a header-only CSV. Times show when changes were detected, not when someone acted on Instagram.\n\n### Setup\n1. Track an account in Instoryview and let history accumulate.\n2. Create a **data:read** API key in [Developer settings](https://instoryview.com/settings/developers).\n3. In **Read all change pages**, create a **Header Auth** credential: Name `Authorization`, Value `Bearer YOUR_API_KEY`.\n4. Edit **Export settings**. Defaults cover seven UTC days.\n5. Execute the workflow. Download the file from the CSV branch and inspect **Export summary**.\n\nSee the [changes API guide](https://instoryview.com/docs/changes). Reads use zero new collection credits. Instoryview access and tracking history are required. The export stops on errors or beyond 100 pages; reduce the date range and rerun. The changes cursor context expires after 15 minutes; start again if it expires. |
| **Choose your export** | n8n-nodes-base.stickyNote | Documentation block covering export setting options and validation rules. | None | None | ### 1. Choose your export\nSet a tracked username, followers or following, dates, and timezone. Dates are inclusive and limited to 366 days. Settings are validated before the request. |
| **Read stored history** | n8n-nodes-base.stickyNote | Documentation block detailing API read configurations and error handling. | None | None | ### 2. Read stored history\nSelect a Header Auth credential. Cursor pages wait 1.5 seconds, with a 100-page ceiling. HTTP errors stop execution. For 401/403 check the key and data:read access; for 429 wait and rerun. Missing history needs existing tracking, never an automatic paid lookup. |
| **Download and review** | n8n-nodes-base.stickyNote | Documentation block detailing result inspection, summary coverage, and warnings. | None | None | ### 3. Download and review\nInspect summary coverage and warnings. Counts come from the first page, not page totals. Download CSV from the populated or empty branch. Spreadsheet formulas are escaped; import ID columns as Text to preserve long IDs. |
| **When clicking Execute workflow** | n8n-nodes-base.manualTrigger | Manual execution entry point. | None | Export settings | |
| **Export settings** | n8n-nodes-base.set | Assigns initial tracking configuration variables. | When clicking Execute workflow | Validate settings | |
| **Validate settings** | n8n-nodes-base.code | Validates username, date range, timezone, and builds API query URLs. | Export settings | Read all change pages | |
| **Read all change pages** | n8n-nodes-base.httpRequest | Fetches change history pages with cursor pagination. | Validate settings | Validate and prepare export | |
| **Validate and prepare export** | n8n-nodes-base.code | Normalizes pages, validates metadata, and builds rows and summary. | Read all change pages | Export summary, Has change rows? | |
| **Export summary** | n8n-nodes-base.code | Outputs the execution JSON summary. | Validate and prepare export | None | |
| **Has change rows?** | n8n-nodes-base.if | Branches execution based on whether change rows are present. | Validate and prepare export | Prepare CSV rows, Create empty CSV | |
| **Prepare CSV rows** | n8n-nodes-base.code | Maps consolidated rows into individual n8n items. | Has change rows? | Convert changes to CSV | |
| **Convert changes to CSV** | n8n-nodes-base.convertToFile | Converts event rows into a downloadable CSV file. | Prepare CSV rows | None | |
| **Create empty CSV** | n8n-nodes-base.code | Generates a header-only CSV file when no changes are returned. | Has change rows? | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Trigger:** Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`) named `When clicking Execute workflow`.
2. **Configure Export Settings:** Add a **Set** node (`n8n-nodes-base.set`) named `Export settings`. Configure assignments:
   - `username` (String): `YOUR_TRACKED_USERNAME`
   - `list` (String): `followers`
   - `from` (String): `={{ $now.setZone('UTC').minus({ days: 6 }).toFormat('yyyy-MM-dd') }}`
   - `to` (String): `={{ $now.setZone('UTC').toFormat('yyyy-MM-dd') }}`
   - `timezone` (String): `UTC`
   - Connect `When clicking Execute workflow` to `Export settings`.
3. **Add Settings Validator:** Add a **Code** node (`n8n-nodes-base.code`) named `Validate settings`. Paste the validation logic checking for proper usernames, 1-366 day date bounds, and valid IANA timezones, outputting `changesUrl` and `initialUrl`. Connect `Export settings` to `Validate settings`.
4. **Configure API HTTP Request:** Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`) named `Read all change pages`. 
   - Set **Authentication** to **Header Auth** (create a credential with header name `Authorization` and value `Bearer YOUR_API_KEY`).
   - Set **URL** to `={{ $json.initialUrl }}`.
   - Configure **Pagination** mode to **Response Contains Next URL**, set `maxRequests` to `100`, request interval to `1,500ms`, and use `meta.hasMore === false` for the completion condition. Set `onError` to **Stop Workflow**.
   - Connect `Validate settings` to `Read all change pages`.
5. **Add Data Normalizer:** Add a **Code** node (`n8n-nodes-base.code`) named `Validate and prepare export`. Paste the aggregation code validating page consistency, zero credit consumption (`creditsUsed === 0`), and formula injection sanitization (`safeCell`). Connect `Read all change pages` to `Validate and prepare export`.
6. **Add Summary Output:** Add a **Code** node (`n8n-nodes-base.code`) named `Export summary` to extract and return `$input.first().json.summary`. Connect `Validate and prepare export` to `Export summary`.
7. **Add Conditional Branching:** Add an **If** node (`n8n-nodes-base.if`) named `Has change rows?`. Set condition: `{{ $json.rows.length }}` **Greater Than** `0`. Connect `Validate and prepare export` to `Has change rows?`.
8. **Configure True Branch (CSV Conversion):**
   - Add a **Code** node named `Prepare CSV rows` to map rows into individual items. Connect the True output of `Has change rows?` here.
   - Add a **Convert to File** node (`n8n-nodes-base.convertToFile`) named `Convert changes to CSV`. Set operation to `CSV`, binary property to `data`, enable header row, and set filename expression to `={{ $('Validate and prepare export').first().json.fileName }}`. Connect `Prepare CSV rows` to this node.
9. **Configure False Branch (Empty CSV Generation):**
   - Add a **Code** node named `Create empty CSV` to generate a header-only UTF-8 CSV binary file when row counts equal zero. Connect the False output of `Has change rows?` here.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Instoryview platform website | [Instoryview Website](https://instoryview.com) |
| Developer Settings for API key generation | [Instoryview Developer Settings](https://instoryview.com/settings/developers) |
| API changes reference and coverage details | [Instoryview Changes API Guide](https://instoryview.com/docs/changes) |
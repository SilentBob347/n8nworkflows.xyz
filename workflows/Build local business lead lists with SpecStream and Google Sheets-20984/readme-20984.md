Build local business lead lists with SpecStream and Google Sheets

https://n8nworkflows.xyz/workflows/build-local-business-lead-lists-with-specstream-and-google-sheets-20984


# Build local business lead lists with SpecStream and Google Sheets

### 1. Workflow Overview

This workflow automates the generation of local business lead lists based on user input parameters (business type, city, and optional conditions). It interfaces with the SpecStream API to parse requests, fetch pricing quotes, execute live web searches, poll execution progress, and persist the structured results directly into Google Sheets.

The architecture is grouped into four distinct functional blocks:

- **1.1 Input Reception & Query Preparation:** Captures user parameters via an n8n Form trigger and formats them into a unified search string, unit count, and maximum budget constraint.
- **1.2 Request Validation & Cost Estimation:** Submits the query to SpecStream for syntactic parsing and capacity pricing, validating whether the request is runnable and within the user's maximum budget.
- **1.3 Execution & Asynchronous Polling:** Authorizes the quote to initiate an idempotent SpecStream run, then enters a wait-and-check loop until the asynchronous search finishes.
- **1.4 Data Transformation & Persistence:** Fetches all paginated result rows from SpecStream, normalizes cell values and metadata (source URL, timestamp, workspace collection link), and appends them into Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Query Preparation
- **Overview:** Collects search criteria from an interactive user form and transforms the raw fields into structured variables required by the downstream API requests.
- **Nodes Involved:** `Describe the list`, `Write the request`

##### Node Details: `Describe the list`
- **Type & Technical Role:** `n8n-nodes-base.formTrigger` (Trigger). Hosts a web form collecting target business criteria.
- **Configuration Choices:** Configured with custom fields (*Business type*, *City*, *Anything else every business must meet (optional)*, *How many businesses*, *Maximum price in USD*). Attribution branding is disabled, and a custom submission message is returned.
- **Key Expressions:** None (User-submitted form fields).
- **Input/Output Connections:** Input: Webhook event. Output: `Write the request`
- **Edge Cases / Failure Types:** Invalid or incomplete user inputs (mitigated by setting *Business type*, *City*, *How many businesses*, and *Maximum price* as required fields).

##### Node Details: `Write the request`
- **Type & Technical Role:** `n8n-nodes-base.set` (Data Transformation). Formats the form inputs into structured JSON properties.
- **Configuration Choices:** Creates three assignments: `request` (concatenated string), `count` (integer floor of the requested business quantity), and `max_cents` (cents-integer conversion of the maximum budget).
- **Key Expressions:**
  - `request`: `={{ $json['Business type'] }} in {{ $json['City'] }}{{ $json['Anything else every business must meet (optional)'] ? ', ' + $json['Anything else every business must meet (optional)'] : '' }}, with their phone number and website`
  - `count`: `={{ Math.floor(Number($json['How many businesses'])) }}`
  - `max_cents`: `={{ Math.round(Number($json['Maximum price in USD']) * 100) }}`
- **Input/Output Connections:** Input: `Describe the list`. Output: `Read the request`.
- **Edge Cases / Failure Types:** Type-casting errors if non-numeric values are passed to numeric fields.

---

#### 2.2 Request Validation & Cost Estimation
- **Overview:** Submits the natural language query to the SpecStream API to verify its validity and obtain a cost quotation before committing any funds.
- **Nodes Involved:** `Read the request`, `Clear enough to price?`, `Stop: reword the request`, `Get the price`, `Within your maximum?`, `Stop: price above your maximum`

##### Node Details: `Read the request`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Integration). Parses the natural language request string into a standardized AST hash.
- **Configuration Choices:** POST method to `https://api.specstream.ai/v1/parse`. Uses HTTP Bearer Authentication. Retries up to 3 times on failure with a 5000ms delay.
- **Key Expressions:** Body: `={{ JSON.stringify({ q: $json.request, locale: 'en-US' }) }}`
- **Input/Output Connections:** Input: `Write the request`. Output: `Clear enough to price?`.
- **Edge Cases / Failure Types:** API connectivity issues, authentication errors, or invalid bearer token.

##### Node Details: `Clear enough to price?`
- **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control). Evaluates whether the parser successfully recognized the query structure.
- **Configuration Choices:** Checks if `{{ $json.data.status }}` equals `runnable`.
- **Key Expressions:** `={{ $json.data.status }}`
- **Input/Output Connections:** Input: `Read the request`. Output (True): `Get the price`. Output (False): `Stop: reword the request`.

##### Node Details: `Stop: reword the request`
- **Type & Technical Role:** `n8n-nodes-base.stopAndError` (Error Handling). Halts the workflow and outputs an actionable error message if the query is ambiguous.
- **Configuration Choices:** Custom error message displaying the canonical sentence returned by the parser.
- **Key Expressions:** `=SpecStream needs a clearer request before it can price it. It read: "{{ $json.data.canonical_sentence }}". Name the trade and the city.`
- **Input/Output Connections:** Input: `Clear enough to price?` (False branch). Output: None.

##### Node Details: `Get the price`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Integration). Requests a price quote based on the parsed AST hash and desired unit count.
- **Configuration Choices:** POST method to `https://api.specstream.ai/v1/capacity/quotes`. Uses HTTP Bearer Authentication with auto-retries.
- **Key Expressions:** Body: `={{ JSON.stringify({ ast_hash: $json.data.ast_hash, operation: 'find', requested_units: $('Write the request').first().json.count }) }}`
- **Input/Output Connections:** Input: `Clear enough to price?` (True branch). Output: `Within your maximum?`.
- **Edge Cases / Failure Types:** Rate limits, upstream service timeouts, or expired AST hashes.

##### Node Details: `Within your maximum?`
- **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control). Compares the estimated maximum quote cost against the user's specified budget limit.
- **Configuration Choices:** Condition checks if `{{ $json.data.maximum_cents }}` is less than or equal to `={{ $('Write the request').first().json.max_cents }}`.
- **Key Expressions:** Left value: `={{ $json.data.maximum_cents }}`, Right value: `={{ $('Write the request').first().json.max_cents }}`
- **Input/Output Connections:** Input: `Get the price`. Output (True): `Approve and start`. Output (False): `Stop: price above your maximum`.

##### Node Details: `Stop: price above your maximum`
- **Type & Technical Role:** `n8n-nodes-base.stopAndError` (Error Handling). Halts execution if the calculated cost exceeds the user's budget ceiling.
- **Configuration Choices:** Formats cost values from cents to dollars for readability.
- **Key Expressions:** `=This list can cost up to ${{ ($json.data.maximum_cents / 100).toFixed(2) }}, above your maximum of ${{ ($('Write the request').first().json.max_cents / 100).toFixed(2) }}. Raise the maximum or ask for fewer results.`
- **Input/Output Connections:** Input: `Within your maximum?` (False branch). Output: None.

---

#### 2.3 Execution & Asynchronous Polling
- **Overview:** Approves the financial quote to initiate data collection, then loops through status checks at regular intervals until the asynchronous search run is complete.
- **Nodes Involved:** `Approve and start`, `Wait 30 seconds`, `Check progress`, `Finished?`

##### Node Details: `Approve and start`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Integration). Confirms the quote and starts the background execution.
- **Configuration Choices:** POST method to `https://api.specstream.ai/v1/capacity/executions`. Uses HTTP Bearer Auth and includes an execution-scoped idempotency key to prevent double-billing on retries.
- **Key Expressions:** Body: `={{ JSON.stringify({ quote_id: $json.data.quote_id, quote_hash: $json.data.quote_hash, confirm_max_cents: $json.data.maximum_cents, idempotency_key: 'local-list-' + $execution.id }) }}`
- **Input/Output Connections:** Input: `Within your maximum?` (True branch). Output: `Wait 30 seconds`.
- **Edge Cases / Failure Types:** Insufficient account balance, invalid quote identifiers, or idempotency collisions.

##### Node Details: `Wait 30 seconds`
- **Type & Technical Role:** `n8n-nodes-base.wait` (Flow Control). Delays workflow execution for a fixed duration to allow background search processing.
- **Configuration Choices:** Unit set to seconds, amount set to `30`.
- **Key Expressions:** None.
- **Input/Output Connections:** Input: `Approve and start` or `Finished?` (False branch). Output: `Check progress`.

##### Node Details: `Check progress`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Integration). Fetches a small batch of rows (limit 1) from the active find operation to evaluate its current status.
- **Configuration Choices:** GET method to `=https://api.specstream.ai/v1/finds/{{ $('Approve and start').first().json.data.find_id }}/rows`. Query parameters set `limit` to `1`.
- **Key Expressions:** URL dynamically interpolates `find_id` from the initial execution node.
- **Input/Output Connections:** Input: `Wait 30 seconds`. Output: `Finished?`.
- **Edge Cases / Failure Types:** Network dropouts or intermittent API errors during polling.

##### Node Details: `Finished?`
- **Type & Technical Role:** `n8n-nodes-base.if` (Flow Control). Determines whether the asynchronous job has finished processing.
- **Configuration Choices:** Validates that the status property is *not* equal to `queued`, `planning`, or `running`.
- **Key Expressions:** Evaluates `{{ $json.data.status }}` against three exclusionary states.
- **Input/Output Connections:** Input: `Check progress`. Output (True): `Get every row`. Output (False): `Wait 30 seconds` (Loops back).

---

#### 2.4 Data Transformation & Persistence
- **Overview:** Retrieves all result pages via pagination, flattens cell arrays into standard key-value pairs, adds metadata, and appends the records into Google Sheets.
- **Nodes Involved:** `Get every row`, `One row per business`, `Add rows to your sheet`

##### Node Details: `Get every row`
- **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Integration). Pagigates through and retrieves all result rows for the completed find execution.
- **Configuration Choices:** GET method to `=https://api.specstream.ai/v1/finds/{{ $('Approve and start').first().json.data.find_id }}/rows`. Configured with built-in cursor-based pagination (`next_cursor`), fetching up to 100 pages with a page limit of 100 items per request.
- **Key Expressions:** Pagination complete expression: `={{ !$response.body.data.next_cursor }}`.
- **Input/Output Connections:** Input: `Finished?` (True branch). Output: `One row per business`.
- **Edge Cases / Failure Types:** Large datasets triggering memory limits or pagination token corruption.

##### Node Details: `One row per business`
- **Type & Technical Role:** `n8n-nodes-base.code` (Data Transformation). Iterates through all paginated response items, extracts cell labels and values while omitting UI controls, and maps metadata (Source URL, Observation timestamp, and Workspace collection link).
- **Configuration Choices:** Custom JavaScript execution environment processing item arrays.
- **Key Expressions:** JavaScript iteration logic processing `cells`, `provenance`, and constructing output records.
- **Input/Output Connections:** Input: `Get every row`. Output: `Add rows to your sheet`.
- **Edge Cases / Failure Types:** Missing cells or unexpected JSON schemas in API payloads.

##### Node Details: `Add rows to your sheet`
- **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Data Integration). Appends structured business lead rows into a target Google Sheets worksheet.
- **Configuration Choices:** Operation set to `append`. Extra data handling set to `insertInNewColumn` to automatically create columns for dynamic fields. Spreadsheet and worksheet selectors require configuration.
- **Key Expressions:** Auto-maps input data schema.
- **Input/Output Connections:** Input: `One row per business`. Output: None.
- **Edge Cases / Failure Types:** Google API rate limits, revoked Google OAuth credentials, or missing spreadsheet configuration.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Overview** | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## Build a lead list of every local business in a city<br><br>Type a trade and a city into the form. The workflow asks SpecStream for every match from the live web (maps, directories, each business's own site) and writes the rows to Google Sheets: name, website, phone, and the page each row came from.<br><br>A Maps search stops at about 120 places. This keeps going until it has the number you asked for, up to 10,000.<br><br>### How it works<br>1. The form collects the trade, the city, an optional condition and your maximum price.<br>2. SpecStream reads the request and quotes the most it can cost. Nothing is spent yet.<br>3. If the quote is at or below your maximum, the workflow approves it. If not, it stops and tells you the price.<br>4. It checks every 30 seconds until the run ends, then writes one row per business to your sheet.<br><br>### Set up (5 minutes)<br>1. Get a free API key at [specstream.ai](https://specstream.ai) under **Agents & API**. New accounts get $10 of work free, no card.<br>2. In n8n, create a **Bearer Auth** credential with the key and pick it in the five SpecStream HTTP nodes.<br>3. Connect Google Sheets and choose an empty sheet in **Add rows to your sheet**. The first run writes the headers.<br>4. Open the form URL from **Describe the list** and submit it.<br><br>### Price<br>10¢ a business for a plain list with phone and website, 30¢ when it has to meet a condition such as a rating. The workflow approves a quote only when it is at or below your maximum. You pay for the matches, and a run that finds nothing is free. |
| **Step 1** | `n8n-nodes-base.stickyNote` | Documentation | None | None | ### 1. Describe the list<br>The form builds one plain sentence, for example: *HVAC companies in Columbus, Ohio, with their phone number and website*. Add a condition such as *rated 4.5 or better with fewer than 20 Google reviews* to narrow it. |
| **Step 2** | `n8n-nodes-base.stickyNote` | Documentation | None | None | ### 2. Read the request and price it<br>Reading and pricing are free. **Get the price** returns the most this list can cost; **Within your maximum?** stops the run if the price is above what you typed. |
| **Step 3** | `n8n-nodes-base.stickyNote` | Documentation | None | None | ### 3. Approve, then wait for the list<br>**Approve and start** is the only step that spends. It reuses one key per execution, so a retry never buys the list twice. The loop checks progress every 30 seconds; most lists finish in 2 to 10 minutes. |
| **Step 4** | `n8n-nodes-base.stickyNote` | Documentation | None | None | ### 4. Into your sheet<br>One row per business. **Source** is the page the row came from and **List** opens the full list in SpecStream, where you can keep it current or send it to a CRM. |
| **Describe the list** | `n8n-nodes-base.formTrigger` | Form Trigger | None | Write the request | ### 1. Describe the list<br>The form builds one plain sentence, for example: *HVAC companies in Columbus, Ohio, with their phone number and website*. Add a condition such as *rated 4.5 or better with fewer than 20 Google reviews* to narrow it. |
| **Write the request** | `n8n-nodes-base.set` | Data Transformation | Describe the list | Read the request | ### 1. Describe the list<br>The form builds one plain sentence, for example: *HVAC companies in Columbus, Ohio, with their phone number and website*. Add a condition such as *rated 4.5 or better with fewer than 20 Google reviews* to narrow it. |
| **Read the request** | `n8n-nodes-base.httpRequest` | API Integration | Write the request | Clear enough to price? | ### 2. Read the request and price it<br>Reading and pricing are free. **Get the price** returns the most this list can cost; **Within your maximum?** stops the run if the price is above what you typed. |
| **Clear enough to price?** | `n8n-nodes-base.if` | Flow Control | Read the request | Stop: reword the request,<br>Get the price | ### 2. Read the request and price it<br>Reading and pricing are free. **Get the price** returns the most this list can cost; **Within your maximum?** stops the run if the price is above what you typed. |
| **Stop: reword the request** | `n8n-nodes-base.stopAndError` | Error Handling | Clear enough to price? | None | ### 2. Read the request and price it<br>Reading and pricing are free. **Get the price** returns the most this list can cost; **Within your maximum?** stops the run if the price is above what you typed. |
| **Get the price** | `n8n-nodes-base.httpRequest` | API Integration | Clear enough to price? | Within your maximum? | ### 2. Read the request and price it<br>Reading and pricing are free. **Get the price** returns the most this list can cost; **Within your maximum?** stops the run if the price is above what you typed. |
| **Within your maximum?** | `n8n-nodes-base.if` | Flow Control | Get the price | Stop: price above your maximum,<br>Approve and start | ### 2. Read the request and price it<br>Reading and pricing are free. **Get the price** returns the most this list can cost; **Within your maximum?** stops the run if the price is above what you typed. |
| **Stop: price above your maximum** | `n8n-nodes-base.stopAndError` | Error Handling | Within your maximum? | None | ### 2. Read the request and price it<br>Reading and pricing are free. **Get the price** returns the most this list can cost; **Within your maximum?** stops the run if the price is above what you typed. |
| **Approve and start** | `n8n-nodes-base.httpRequest` | API Integration | Within your maximum? | Wait 30 seconds | ### 3. Approve, then wait for the list<br>**Approve and start** is the only step that spends. It reuses one key per execution, so a retry never buys the list twice. The loop checks progress every 30 seconds; most lists finish in 2 to 10 minutes. |
| **Wait 30 seconds** | `n8n-nodes-base.wait` | Flow Control | Approve and start,<br>Finished? | Check progress | ### 3. Approve, then wait for the list<br>**Approve and start** is the only step that spends. It reuses one key per execution, so a retry never buys the list twice. The loop checks progress every 30 seconds; most lists finish in 2 to 10 minutes. |
| **Check progress** | `n8n-nodes-base.httpRequest` | API Integration | Wait 30 seconds | Finished? | ### 3. Approve, then wait for the list<br>**Approve and start** is the only step that spends. It reuses one key per execution, so a retry never buys the list twice. The loop checks progress every 30 seconds; most lists finish in 2 to 10 minutes. |
| **Finished?** | `n8n-nodes-base.if` | Flow Control | Check progress | Get every row,<br>Wait 30 seconds | ### 3. Approve, then wait for the list<br>**Approve and start** is the only step that spends. It reuses one key per execution, so a retry never buys the list twice. The loop checks progress every 30 seconds; most lists finish in 2 to 10 minutes. |
| **Get every row** | `n8n-nodes-base.httpRequest` | API Integration | Finished? | One row per business | ### 4. Into your sheet<br>One row per business. **Source** is the page the row came from and **List** opens the full list in SpecStream, where you can keep it current or send it to a CRM. |
| **One row per business** | `n8n-nodes-base.code` | Data Transformation | Get every row | Add rows to your sheet | ### 4. Into your sheet<br>One row per business. **Source** is the page the row came from and **List** opens the full list in SpecStream, where you can keep it current or send it to a CRM. |
| **Add rows to your sheet** | `n8n-nodes-base.googleSheets` | Data Integration | One row per business | None | ### 4. Into your sheet<br>One row per business. **Source** is the page the row came from and **List** opens the full list in SpecStream, where you can keep it current or send it to a CRM. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**
   - Add a **Form Trigger** node named `Describe the list`.
   - Configure form fields: *Business type* (Text, required), *City* (Text, required), *Anything else every business must meet (optional)* (Text), *How many businesses* (Number, required), *Maximum price in USD* (Number, required).
   - Uncheck attribution branding and set the submission text option.

2. **Create Data Formatting Node:**
   - Add a **Set** node named `Write the request`.
   - Connect `Describe the list` to `Write the request`.
   - Create assignments: `request` (String), `count` (Number), `max_cents` (Number) using the expressions detailed in Section 2.1.

3. **Create Query Parsing Node:**
   - Add an **HTTP Request** node named `Read the request`.
   - Configure POST to `https://api.specstream.ai/v1/parse`, select Generic Credential Type -> HTTP Bearer Auth.
   - Set Body (JSON) to `={{ JSON.stringify({ q: $json.request, locale: 'en-US' }) }}`. Enable retry on fail.
   - Connect `Write the request` to `Read the request`.

4. **Create Parser Validation Condition:**
   - Add an **If** node named `Clear enough to price?`.
   - Set condition to check if `{{ $json.data.status }}` equals `runnable`.
   - Connect `Read the request` to `Clear enough to price?`.

5. **Create Error Node for Unparseable Queries:**
   - Add a **Stop and Error** node named `Stop: reword the request`.
   - Configure the error message expression with the canonical sentence.
   - Connect the `false` output of `Clear enough to price?` to this node.

6. **Create Pricing Request Node:**
   - Add an **HTTP Request** node named `Get the price`.
   - Configure POST to `https://api.specstream.ai/v1/capacity/quotes`, using HTTP Bearer Auth.
   - Set Body (JSON) to `={{ JSON.stringify({ ast_hash: $json.data.ast_hash, operation: 'find', requested_units: $('Write the request').first().json.count }) }}`.
   - Connect the `true` output of `Clear enough to price?` to `Get the price`.

7. **Create Budget Verification Condition:**
   - Add an **If** node named `Within your maximum?`.
   - Set condition to check if `{{ $json.data.maximum_cents }}` is less than or equal to `={{ $('Write the request').first().json.max_cents }}`.
   - Connect `Get the price` to `Within your maximum?`.

8. **Create Error Node for Budget Exceedance:**
   - Add a **Stop and Error** node named `Stop: price above your maximum`.
   - Configure the error message formatting costs in dollars.
   - Connect the `false` output of `Within your maximum?` to this node.

9. **Create Execution Approval Node:**
   - Add an **HTTP Request** node named `Approve and start`.
   - Configure POST to `https://api.specstream.ai/v1/capacity/executions`, using HTTP Bearer Auth.
   - Set Body (JSON) to `={{ JSON.stringify({ quote_id: $json.data.quote_id, quote_hash: $json.data.quote_hash, confirm_max_cents: $json.data.maximum_cents, idempotency_key: 'local-list-' + $execution.id }) }}`.
   - Connect the `true` output of `Within your maximum?` to `Approve and start`.

10. **Create Wait Node:**
    - Add a **Wait** node named `Wait 30 seconds`.
    - Set unit to seconds, amount to `30`.
    - Connect `Approve and start` to `Wait 30 seconds`.

11. **Create Progress Check Node:**
    - Add an **HTTP Request** node named `Check progress`.
    - Configure GET to `=https://api.specstream.ai/v1/finds/{{ $('Approve and start').first().json.data.find_id }}/rows`, using HTTP Bearer Auth with query parameter `limit` = `1`.
    - Connect `Wait 30 seconds` to `Check progress`.

12. **Create Loop Condition Node:**
    - Add an **If** node named `Finished?`.
    - Set conditions to verify status is not equal to `queued`, `planning`, or `running`.
    - Connect `Check progress` to `Finished?`.
    - Connect the `false` output of `Finished?` back to `Wait 30 seconds`.

13. **Create Paginated Data Retrieval Node:**
    - Add an **HTTP Request** node named `Get every row`.
    - Configure GET to `=https://api.specstream.ai/v1/finds/{{ $('Approve and start').first().json.data.find_id }}/rows`, using HTTP Bearer Auth.
    - Enable pagination options with `cursor` query parameter mapped to `={{ $response.body.data.next_cursor }}` and complete expression `={{ !$response.body.data.next_cursor }}`.
    - Connect the `true` output of `Finished?` to `Get every row`.

14. **Create Data Normalization Code Node:**
    - Add a **Code** node named `One row per business`.
    - Paste the JavaScript snippet for extracting cell labels, values, and provenance metadata.
    - Connect `Get every row` to `One row per business`.

15. **Create Google Sheets Integration Node:**
    - Add a **Google Sheets** node named `Add rows to your sheet`.
    - Configure operation to `append`, handling extra data set to `insertInNewColumn`. Link your Google Sheets credential and select the target Document and Sheet.
    - Connect `One row per business` to `Add rows to your sheet`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| SpecStream API & Platform | Get a free API key at [specstream.ai](https://specstream.ai) under **Agents & API** |
| Authentication Requirement | Requires an n8n **Bearer Auth** credential populated with the SpecStream API key across all HTTP request nodes. |
| Google Sheets Setup | Connect Google Sheets and select an empty worksheet; the initial execution automatically inserts headers and dynamic columns. |
Track Greenhouse job board changes with Apify and Data Tables

https://n8nworkflows.xyz/workflows/track-greenhouse-job-board-changes-with-apify-and-data-tables-20802


# Track Greenhouse job board changes with Apify and Data Tables

### 1. Workflow Overview

This workflow automates tracking changes on public Greenhouse job boards using Apify and stores the latest state of each job in an n8n Data Table. It ensures data integrity by verifying run statuses, key-value store summaries, and dataset integrity before making any database modifications.

The logical execution flow is grouped into the following functional blocks:
- **1.1 Initialization and Actor Trigger:** Initiates the workflow manually and triggers the Apify Greenhouse Job Edit Feed Actor with predefined configuration limits.
- **1.2 Summary Validation & Storage Provisioning:** Fetches and validates the execution summary from Apify, ensures strict success criteria, and provisions the target n8n Data Table if it does not already exist.
- **1.3 Dataset Processing and Persistence:** Downloads the dataset from Apify, validates row counts and event signatures, maps the events by a stable composite key (`board` + `post_id`), and upserts the records into the n8n Data Table.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization and Actor Trigger
- **Overview:** Starts the execution sequence on-demand and kicks off the Apify web scraping actor with specific parameters and billing limits.
- **Nodes Involved:** 
  - `Run job board check` (Manual Trigger)
  - `Run Greenhouse Edit Feed Actor` (HTTP Request)
- **Node Details:**
  - **Run job board check**
    - *Type:* `n8n-nodes-base.manualTrigger`
    - *Technical Role:* Entry point for manual workflow executions.
    - *Configuration:* Default.
    - *Input/Output:* No inputs; outputs to the Apify HTTP Request node.
    - *Edge Cases:* None.
  - **Run Greenhouse Edit Feed Actor**
    - *Type:* `n8n-nodes-base.httpRequest`
    - *Technical Role:* Triggers the Apify Actor via REST API using a POST request.
    - *Configuration:* POST request to `https://api.apify.com/v2/acts/resolute_client~greenhouse-job-edit-feed/runs?waitForFinish=120&maxTotalChargeUsd=0.05&memory=256&timeout=120`, authenticated via HTTP Bearer token. Sends a JSON body defining the target board (`webflow`) and namespace.
    - *Key Expressions:* `={ "boards": ["webflow"], "namespace": "career-watchlist-demo" }`
    - *Input/Output:* Input from `Run job board check`; outputs to `Read Actor completion SUMMARY`.
    - *Edge Cases:* Authentication failure if the Bearer token is missing/invalid; Apify timeout or cost limit reached.

#### 1.2 Summary Validation & Storage Provisioning
- **Overview:** Retrieves the run manifest summary from Apify, validates that all board operations completed successfully without errors, and ensures the target data table exists.
- **Nodes Involved:**
  - `Read Actor completion SUMMARY` (HTTP Request)
  - `Validate completed Actor SUMMARY` (Code)
  - `Ensure persistent job table exists` (Data Table)
- **Node Details:**
  - **Read Actor completion SUMMARY**
    - *Type:* `n8n-nodes-base.httpRequest`
    - *Technical Role:* Fetches execution metadata and manifest logs from the Apify key-value store.
    - *Configuration:* GET request authenticated via HTTP Bearer token. URL dynamically references the dynamic key-value store ID generated in the previous step.
    - *Key Expressions:* `=https://api.apify.com/v2/key-value-stores/{{ $('Run Greenhouse Edit Feed Actor').first().json.defaultKeyValueStoreId }}/records/SUMMARY`
    - *Input/Output:* Input from `Run Greenhouse Edit Feed Actor`; outputs to `Validate completed Actor SUMMARY`.
    - *Edge Cases:* Invalid store ID or missing record.
  - **Validate completed Actor SUMMARY**
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Implements strict validation logic, throwing errors to abort execution if run statuses, summary metadata, or board manifests do not align.
    - *Configuration:* JavaScript execution mode (`runOnceForAllItems`).
    - *Input/Output:* Input from `Read Actor completion SUMMARY`; outputs to `Ensure persistent job table exists`.
    - *Edge Cases:* Throws explicit runtime errors if `run.status !== 'SUCCEEDED'`, if manifest boards do not match, or if verification flags are false.
  - **Ensure persistent job table exists**
    - *Type:* `n8n-nodes-base.dataTable`
    - *Technical Role:* Creates the destination table if it is absent.
    - *Configuration:* Resource: `table`, Operation: `create`, Table Name: `Greenhouse Job Watchlist`, Option: `createIfNotExists` enabled. Defines schema columns (`display_id`, `board`, `post_id`, `event`, `visibility`, `title`, `location`, `url`, `description_text`, `description_hash`, `updated_at`, `observed_at`, `event_id`, `source_run_id`, `changed_fields`).
    - *Input/Output:* Input from `Validate completed Actor SUMMARY`; outputs to `Read completed Actor dataset`.
    - *Edge Cases:* Database permission issues or naming conflicts.

#### 1.3 Dataset Processing and Persistence
- **Overview:** Pulls the scraped job records from Apify, maps them to stable unique job identifiers, computes visibility statuses, and upserts them into the n8n Data Table.
- **Nodes Involved:**
  - `Read completed Actor dataset` (HTTP Request)
  - `Validate dataset and map producer events by stable job ID` (Code)
  - `Upsert one persistent job row` (Data Table)
- **Node Details:**
  - **Read completed Actor dataset**
    - *Type:* `n8n-nodes-base.httpRequest`
    - *Technical Role:* Downloads the processed items dataset from Apify.
    - *Configuration:* GET request authenticated via HTTP Bearer token using the dataset ID from the initial actor run.
    - *Key Expressions:* `=https://api.apify.com/v2/datasets/{{ $('Run Greenhouse Edit Feed Actor').first().json.defaultDatasetId }}/items?clean=true&format=json&limit=10000`
    - *Input/Output:* Input from `Ensure persistent job table exists`; outputs to `Validate dataset and map producer events by stable job ID`.
    - *Edge Cases:* API rate limits, large datasets exceeding limits.
  - **Validate dataset and map producer events by stable job ID**
    - *Type:* `n8n-nodes-base.code`
    - *Technical Role:* Verifies row count consistency against the summary, screens for duplicate event IDs, calculates visibility levels (`visible`, `hidden`, `uncertain`), and groups items by composite key.
    - *Configuration:* JavaScript execution mode (`runOnceForAllItems`).
    - *Input/Output:* Input from `Read completed Actor dataset`; outputs to `Upsert one persistent job row`.
    - *Edge Cases:* Row count mismatches, malformed payloads, or duplicate event IDs trigger script failure.
  - **Upsert one persistent job row**
    - *Type:* `n8n-nodes-base.dataTable`
    - *Technical Role:* Performs row-level updates or inserts based on the job's stable identifier (`display_id`).
    - *Configuration:* Resource: `row`, Operation: `upsert`, Table Name: `Greenhouse Job Watchlist`, Match Type: Match by `display_id`. Maps fields from the previous node expressions.
    - *Key Expressions:* Dynamic bindings for fields like `={{ $json.url }}`, `={{ $json.board }}`, `={{ $json.display_id }}`, etc.
    - *Input/Output:* Input from `Validate dataset and map producer events by stable job ID`; terminal node.
    - *Edge Cases:* Table locked or database upsert constraints violated.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run job board check | `manualTrigger` | Entry point for manual executions | None | Run Greenhouse Edit Feed Actor | # Track Greenhouse job changes in an n8n Data Table<br><br>**Who this is for:** teams that monitor public Greenhouse job boards and want a persistent, reviewable change feed.<br><br>**What it does:** runs the public Greenhouse Job Edit Feed Actor for one configured board, checks that the run and every requested board completed, verifies the SUMMARY manifest against dataset readback, and upserts the latest event by the stable `(board, post_id)` key. Description-only edits update the text and hash; closed jobs become hidden; reopened and reappeared jobs become visible again. Incomplete or mismatched runs stop before job rows are written.<br><br>**Set up:** choose your board slug and namespace in “Run Greenhouse Edit Feed Actor”; use an Apify Bearer credential in the three HTTP Request nodes; keep the HTTP credential in n8n Credentials. Set the same credential on all three nodes. The Data Table is created if missing. Importantly, the first run may create the table before dataset validation; row upserts only occur after validation.<br><br>**Usage and costs:** the template is free. The Actor itself is a separate Apify product and is billed to your Apify account. The workflow caps each Actor run at $0.05 and requests 256 MB. On 12 Oct 2026, the public listing showed $0.01 per board snapshot plus $0.001 per Actor start; confirm the listing before use. Your n8n hosting plan may also have its own cost.<br><br>**Limits:** manual, on-demand checks; no email or Slack notifications. Review uncertain/missing candidates before acting. Public job data only. Do not put API keys in node fields or workflow exports. |
| Run Greenhouse Edit Feed Actor | `httpRequest` | Triggers the Apify Actor for Greenhouse job scraping | Run job board check | Read Actor completion SUMMARY | Configure the board slug and namespace. Store the Apify token in n8n Credentials and select that credential for every HTTP Request node. |
| Read Actor completion SUMMARY | `httpRequest` | Fetches run execution summary and manifest records | Run Greenhouse Edit Feed Actor | Validate completed Actor SUMMARY | The exact run ID, SUCCEEDED status, SUMMARY counts, board manifest, dataset source run IDs, and dataset row count must agree. A mismatch stops row writes. |
| Validate completed Actor SUMMARY | `code` | Validates summary records and execution constraints | Read Actor completion SUMMARY | Ensure persistent job table exists | The exact run ID, SUCCEEDED status, SUMMARY counts, board manifest, dataset source run IDs, and dataset row count must agree. A mismatch stops row writes. |
| Ensure persistent job table exists | `dataTable` | Provisions the persistent data table if missing | Validate completed Actor SUMMARY | Read completed Actor dataset | The exact run ID, SUCCEEDED status, SUMMARY counts, board manifest, dataset source run IDs, and dataset row count must agree. A mismatch stops row writes. |
| Read completed Actor dataset | `httpRequest` | Downloads scraped records from Apify dataset | Ensure persistent job table exists | Validate dataset and map producer events by stable job ID | Events are grouped by stable Greenhouse board slug and post ID. Replays update the same row; closed jobs are hidden and returned jobs are visible again. |
| Validate dataset and map producer events by stable job ID | `code` | Maps items by unique key and calculates visibility | Read completed Actor dataset | Upsert one persistent job row | Events are grouped by stable Greenhouse board slug and post ID. Replays update the same row; closed jobs are hidden and returned jobs are visible again. |
| Upsert one persistent job row | `dataTable` | Upserts job records into the data table | Validate dataset and map producer events by stable job ID | None | Events are grouped by stable Greenhouse board slug and post ID. Replays update the same row; closed jobs are hidden and returned jobs are visible again. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run job board check`.
2. **Configure the Apify Actor Trigger:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Run Greenhouse Edit Feed Actor`.
   - Set Method to `POST`.
   - Set URL to `https://api.apify.com/v2/acts/resolute_client~greenhouse-job-edit-feed/runs?waitForFinish=120&maxTotalChargeUsd=0.05&memory=256&timeout=120`.
   - Set Authentication to `Generic Credential Type` -> `HTTP Bearer Auth` (configure your Apify API Token).
   - Specify Body as `JSON` with value: `={"boards":["webflow"],"namespace":"career-watchlist-demo"}`.
   - Set options timeout to `180000` ms.
3. **Fetch the Execution Summary:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Read Actor completion SUMMARY`.
   - Set Method to `GET`.
   - Set URL to `=https://api.apify.com/v2/key-value-stores/{{ $('Run Greenhouse Edit Feed Actor').first().json.defaultKeyValueStoreId }}/records/SUMMARY`.
   - Use the same HTTP Bearer credential.
4. **Add Summary Validation Logic:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Validate completed Actor SUMMARY`.
   - Set Mode to `Run Once for All Items`.
   - Paste validation checks verifying `run.id`, `run.status === 'SUCCEEDED'`, `summary.source_run_id`, manifest validity, and board count parity.
5. **Provision the Data Table:**
   - Add a **Data Table** node (`n8n-nodes-base.dataTable`). Name it `Ensure persistent job table exists`.
   - Set Resource to `Table` and Operation to `Create`.
   - Set Table Name to `Greenhouse Job Watchlist`.
   - Enable option `Create If Not Exists` and configure the 15 required string/number columns (`display_id`, `board`, `post_id`, `event`, `visibility`, `title`, `location`, `url`, `description_text`, `description_hash`, `updated_at`, `observed_at`, `event_id`, `source_run_id`, `changed_fields`).
6. **Download the Dataset:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Read completed Actor dataset`.
   - Set Method to `GET`.
   - Set URL to `=https://api.apify.com/v2/datasets/{{ $('Run Greenhouse Edit Feed Actor').first().json.defaultDatasetId }}/items?clean=true&format=json&limit=10000`.
   - Use the same HTTP Bearer credential.
7. **Map and Validate Dataset Events:**
   - Add a **Code** node (`n8n-nodes-base.code`). Name it `Validate dataset and map producer events by stable job ID`.
   - Set Mode to `Run Once for All Items`.
   - Paste mapping logic that asserts row counts against the summary, removes duplicates via `event_id`, and assigns visibility states based on event categories (`missing_candidate` -> `uncertain`, `closed` -> `hidden`).
8. **Persist Records to Data Table:**
   - Add a **Data Table** node (`n8n-nodes-base.dataTable`). Name it `Upsert one persistent job row`.
   - Set Resource to `Row` and Operation to `Upsert`.
   - Set Table Name to `Greenhouse Job Watchlist`.
   - Configure Match Type to match records where `display_id` equals `={{ $json.display_id }}`.
   - Map all output fields using expressions matching respective property keys from the input data (e.g., `={{ $json.title }}`).
9. **Establish Connections:**
   - Connect `Run job board check` $\rightarrow$ `Run Greenhouse Edit Feed Actor`.
   - Connect `Run Greenhouse Edit Feed Actor` $\rightarrow$ `Read Actor completion SUMMARY`.
   - Connect `Read Actor completion SUMMARY` $\rightarrow$ `Validate completed Actor SUMMARY`.
   - Connect `Validate completed Actor SUMMARY` $\rightarrow$ `Ensure persistent job table exists`.
   - Connect `Ensure persistent job table exists` $\rightarrow$ `Read completed Actor dataset`.
   - Connect `Read completed Actor dataset` $\rightarrow$ `Validate dataset and map producer events by stable job ID`.
   - Connect `Validate dataset and map producer events by stable job ID` $\rightarrow$ `Upsert one persistent job row`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify Actor Product & Pricing | The Actor is a separate third-party product billed directly to your Apify account. Check current listings for baseline charges per board snapshot and start fees. |
| Data Table Behavior | The target n8n Data Table is created automatically if absent, but record upserts are strictly protected and only happen after validation checkpoints successfully pass. |
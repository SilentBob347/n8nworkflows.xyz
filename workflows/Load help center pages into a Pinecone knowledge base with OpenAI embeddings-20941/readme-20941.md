Load help center pages into a Pinecone knowledge base with OpenAI embeddings

https://n8nworkflows.xyz/workflows/load-help-center-pages-into-a-pinecone-knowledge-base-with-openai-embeddings-20941


# Load help center pages into a Pinecone knowledge base with OpenAI embeddings

### 1. Workflow Overview

This workflow is designed to automatically ingest public documentation or help center pages, chunk them into manageable text segments, generate vector embeddings using OpenAI, and upsert those vectors into a Pinecone knowledge base. This processed data supports downstream retrieval-augmented generation (RAG) applications, such as AI support assistants or customer service chat systems.

The execution logic is structured into four primary functional blocks:

- **1.1 Initialization and Configuration:** Receives manual trigger inputs, defines system-wide settings, variables, safety switches, and establishes fallback knowledge base parameters.
- **1.2 Document Retrieval and Processing:** Determines whether custom public URLs are configured. If active, it fetches, cleans, validates, and parses the HTML into plain text. If not, it utilizes a built-in demo dataset, subsequently breaking the content into stable, overlapping chunks.
- **1.3 Embedding Generation & Gate Control:** Evaluates whether upsert operations are authorized based on configuration settings. If disabled, it runs in preview mode; if enabled, it batches chunks and interfaces with the OpenAI Embeddings API.
- **1.4 Vector Upsert and Delivery:** Prepares and transmits embedding vectors to Pinecone, handling batch success verification, error management, and final completion status reporting.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Configuration
This block kicks off the workflow execution manually, loads system parameters, and verifies whether custom documentation URLs have been provided for ingestion.

- **Nodes Involved:**
  - `Run it now`
  - `config`
  - `document list configured?`

- **Node Details:**
  - **Run it now**
    - *Type and technical role:* `n8n-nodes-base.manualTrigger` — Acts as the manual entry point for workflow execution.
    - *Configuration choices:* Default settings (no parameters required).
    - *Input/Output:* Output connected to the `config` node.
    - *Edge cases:* None.
  - **config**
    - *Type and technical role:* `n8n-nodes-base.code` — Centralizes global settings including model names, vector dimensions, Pinecone routing properties, safety thresholds, and demo datasets.
    - *Configuration choices:* JavaScript implementation mapping settings object and passing incoming payload structures.
    - *Input/Output:* Input from `Run it now`; output connected to `document list configured?`.
    - *Edge cases:* Invalid parameter syntax or missing data types will default to fallback numerical/boolean operators.
  - **document list configured?**
    - *Type and technical role:* `n8n-nodes-base.if` — Conditional gate evaluating if `docs_enabled` is true.
    - *Configuration choices:* Compares `{{ $('config').first().json.docs_enabled }}` to boolean `true`.
    - *Input/Output:* Input from `config`; true branch routes to `list the documents`, false branch routes to `use the demo knowledge base`.
    - *Edge cases:* Boolean type mismatches handled via loose type validation.

---

#### 2.2 Document Retrieval and Processing
This block handles fetching target web pages, stripping HTML tags, validating content length, handling request errors, and slicing valid texts into standardized chunks.

- **Nodes Involved:**
  - `list the documents`
  - `Fetch the document`
  - `read the document text`
  - `use the demo knowledge base`
  - `chunk the documents`

- **Node Details:**
  - **list the documents**
    - *Type and technical role:* `n8n-nodes-base.code` — Iterates through `DOC_URLS` from the configuration and normalizes URLs into structured document IDs.
    - *Configuration choices:* JavaScript mapping function generating unique slug identifiers (up to 60 characters).
    - *Input/Output:* Input from `document list configured?` (true branch); output connected to `Fetch the document`.
    - *Edge cases:* Duplicate URLs or invalid string formats mapped to indexed fallbacks (`doc-n`).
  - **Fetch the document**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` — HTTP GET request fetching public webpage text with automated retries.
    - *Configuration choices:* URL set via expression `{{ $json.url }}`, 20-second timeout, plain text response format, 3 max tries with a 3000ms delay. `onError` set to `continueRegularOutput`.
    - *Input/Output:* Input from `list the documents`; output connected to `read the document text`.
    - *Edge cases:* Network timeouts, HTTP 4xx/5xx status codes, or invalid DNS resolutions captured via `continueRegularOutput` for status logging.
  - **read the document text**
    - *Type and technical role:* `n8n-nodes-base.code` — Parses HTML content, extracts page titles, strips styling/scripts, and filters out pages lacking sufficient text length.
    - *Configuration choices:* Regular expression cleaning functions checking for a minimum 20-word threshold.
    - *Input/Output:* Input from `Fetch the document`; output connected to `chunk the documents`.
    - *Edge cases:* Pages returning empty strings, error objects, or minimal content are flagged with an `ok: false` status and skip reason.
  - **use the demo knowledge base**
    - *Type and technical role:* `n8n-nodes-base.code` — Fallback node that loads built-in demo records when `DOC_URLS` is empty.
    - *Configuration choices:* Maps `DEMO_KNOWLEDGE` objects into standard document structures.
    - *Input/Output:* Input from `document list configured?` (false branch); output connected to `chunk the documents`.
    - *Edge cases:* None; relies on internal mock arrays.
  - **chunk the documents**
    - *Type and technical role:* `n8n-nodes-base.code` — Splits documents into sliding windows governed by `CHUNK_WORDS` and `CHUNK_OVERLAP` parameters, assigning stable IDs (`<doc_id>#<n>`).
    - *Configuration choices:* JavaScript array chunking logic.
    - *Input/Output:* Input from `read the document text` or `use the demo knowledge base`; output connected to `loading on?`.
    - *Edge cases:* Extremely short documents output a single chunk matching total word length.

---

#### 2.3 Embedding Generation & Gate Control
This block controls whether vector generation and cloud database synchronization proceed based on system flags, organizing chunks into uniform batches before requesting embeddings from OpenAI.

- **Nodes Involved:**
  - `loading on?`
  - `STOP: preview - vectors not upserted`
  - `build the embedding batches`
  - `[cred] OpenAI - embed the chunks`

- **Node Details:**
  - **loading on?**
    - *Type and technical role:* `n8n-nodes-base.if` — Evaluates whether vector upserting is enabled (`upsert_enabled`).
    - *Configuration choices:* Compares `{{ $('config').first().json.upsert_enabled }}` to boolean `true`.
    - *Input/Output:* Input from `chunk the documents`; true branch routes to `build the embedding batches`, false branch routes to `STOP: preview - vectors not upserted`.
    - *Edge cases:* Configuration flags defaulting to false halt active cloud sync.
  - **STOP: preview - vectors not upserted**
    - *Type and technical role:* `n8n-nodes-base.code` — Terminal preview handler outputting mock payload details without hitting external APIs.
    - *Configuration choices:* Summarizes chunk configurations and outputs a mock preview structure.
    - *Input/Output:* Input from `loading on?` (false branch); terminal execution endpoint.
    - *Edge cases:* None.
  - **build the embedding batches**
    - *Type and technical role:* `n8n-nodes-base.code` — Groups document chunks into batch arrays of 96 items for optimal processing limits.
    - *Configuration choices:* JavaScript batch slicing logic mapping title and text content to request body attributes.
    - *Input/Output:* Input from `loading on?` (true branch); output connected to `[cred] OpenAI - embed the chunks`.
    - *Edge cases:* Payload sizing exceeding token limits mitigated by fixed batch window constraints.
  - **[cred] OpenAI - embed the chunks**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` — HTTP POST request communicating with OpenAI's Embeddings API.
    - *Configuration choices:* Endpoint set to `https://api.openai.com/v1/embeddings`, uses pre-defined OpenAI API credentials (`openAiApi`), 60-second timeout, 3 max retries. `onError` set to `continueRegularOutput`.
    - *Input/Output:* Input from `build the embedding batches`; output connected to `build the Pinecone upsert`.
    - *Edge cases:* Rate limiting (HTTP 429), authentication failures, or malformed JSON bodies handled via automated retries and error forwarding.

---

#### 2.4 Vector Upsert and Delivery
This block pairs embedded vector datasets with chunk metadata, verifies processing success, manages Pinecone synchronization, and returns final operational metrics.

- **Nodes Involved:**
  - `build the Pinecone upsert`
  - `batch embedded?`
  - `STOP: embedding failed`
  - `[cred] Pinecone - upsert the vectors`
  - `STOP: knowledge base loaded`
  - `STOP: Pinecone refused the upsert`

- **Node Details:**
  - **build the Pinecone upsert**
    - *Type and technical role:* `n8n-nodes-base.code` — Pairs API embedding results with original chunk properties, formatting vector payload definitions.
    - *Configuration choices:* JavaScript mapping building vector objects with vector values, IDs, and metadata properties.
    - *Input/Output:* Input from `[cred] OpenAI - embed the chunks`; output connected to `batch embedded?`.
    - *Edge cases:* Dimension mismatches or truncated embedding arrays flagged as failed batches.
  - **batch embedded?**
    - *Type and technical role:* `n8n-nodes-base.if` — Validates whether embedding creation completed successfully for the current batch.
    - *Configuration choices:* Compares `{{ $json.embedded }}` to boolean `true`.
    - *Input/Output:* Input from `build the Pinecone upsert`; true branch routes to `[cred] Pinecone - upsert the vectors`, false branch routes to `STOP: embedding failed`.
    - *Edge cases:* Null response fields caught and routed to error termination.
  - **STOP: embedding failed**
    - *Type and technical role:* `n8n-nodes-base.code` — Error handler logging un-embedded batch failures while allowing independent workflow branches to proceed.
    - *Configuration choices:* Generates failure outcome strings.
    - *Input/Output:* Input from `batch embedded?` (false branch); terminal execution endpoint.
    - *Edge cases:* None.
  - **[cred] Pinecone - upsert the vectors**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` — HTTP POST request upserting vectors into the designated Pinecone index namespace.
    - *Configuration choices:* Dynamic URL using `{{ $('config').first().json.pinecone_index_host + '/vectors/upsert' }}`, Header Auth credential (`httpHeaderAuth`), includes custom header `X-Pinecone-API-Version: 2025-01`, 30-second timeout, 3 retries. `onError` set to `continueErrorOutput`.
    - *Input/Output:* Input from `batch embedded?` (true branch); outputs routed to `STOP: knowledge base loaded` (success) or `STOP: Pinecone refused the upsert` (error).
    - *Edge cases:* Index host misconfigurations, incorrect API keys, or Pinecone namespace connection drops caught via error output routing.
  - **STOP: knowledge base loaded**
    - *Type and technical role:* `n8n-nodes-base.code` — Success logging terminal node reporting vector sent counts and Pinecone acceptance metrics.
    - *Configuration choices:* JavaScript mapping extracting upserted count properties.
    - *Input/Output:* Input from `[cred] Pinecone - upsert the vectors` (success path); terminal execution endpoint.
    - *Edge cases:* Missing upsert counts display fallback messaging (`(not reported)`).
  - **STOP: Pinecone refused the upsert**
    - *Type and technical role:* `n8n-nodes-base.code` — Error terminal node capturing Pinecone rejection messages.
    - *Configuration choices:* JavaScript fallback mapping extracting error response messages.
    - *Input/Output:* Input from `[cred] Pinecone - upsert the vectors` (error path); terminal execution endpoint.
    - *Edge cases:* Unknown error structures default to generic string extraction.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Run it now` | `n8n-nodes-base.manualTrigger` | Manual workflow trigger | None | `config` | Load help pages into a Pinecone knowledge base with OpenAI embeddings... |
| `config` | `n8n-nodes-base.code` | Parameter & setting definitions | `Run it now` | `document list configured?` | Load help pages into a Pinecone knowledge base with OpenAI embeddings... |
| `document list configured?` | `n8n-nodes-base.if` | Check if doc URLs are specified | `config` | `list the documents`, `use the demo knowledge base` | Load help pages into a Pinecone knowledge base with OpenAI embeddings... |
| `list the documents` | `n8n-nodes-base.code` | Normalize documentation URLs | `document list configured?` | `Fetch the document` | 1. Read the pages and chunk them |
| `Fetch the document` | `n8n-nodes-base.httpRequest` | Fetch public page web content | `list the documents` | `read the document text` | 1. Read the pages and chunk them |
| `read the document text` | `n8n-nodes-base.code` | Clean HTML & validate text length | `Fetch the document` | `chunk the documents` | 1. Read the pages and chunk them |
| `use the demo knowledge base` | `n8n-nodes-base.code` | Provide fallback demo documents | `document list configured?` | `chunk the documents` | Load help pages into a Pinecone knowledge base with OpenAI embeddings... |
| `chunk the documents` | `n8n-nodes-base.code` | Split text into overlapping chunks | `read the document text`, `use the demo knowledge base` | `loading on?` | 1. Read the pages and chunk them |
| `loading on?` | `n8n-nodes-base.if` | Check if upserting is enabled | `chunk the documents` | `STOP: preview - vectors not upserted`, `build the embedding batches` | 2. Embed and upsert |
| `STOP: preview - vectors not upserted` | `n8n-nodes-base.code` | Output preview data without API calls | `loading on?` | None | 2. Embed and upsert |
| `build the embedding batches` | `n8n-nodes-base.code` | Group chunks into embedding batches | `loading on?` | `[cred] OpenAI - embed the chunks` | 2. Embed and upsert |
| `[cred] OpenAI - embed the chunks` | `n8n-nodes-base.httpRequest` | Generate embeddings via OpenAI API | `build the embedding batches` | `build the Pinecone upsert` | 2. Embed and upsert |
| `build the Pinecone upsert` | `n8n-nodes-base.code` | Format vectors and metadata payloads | `[cred] OpenAI - embed the chunks` | `batch embedded?` | 2. Embed and upsert |
| `batch embedded?` | `n8n-nodes-base.if` | Verify embedding batch success | `build the Pinecone upsert` | `STOP: embedding failed`, `[cred] Pinecone - upsert the vectors` | 2. Embed and upsert |
| `STOP: embedding failed` | `n8n-nodes-base.code` | Handle failed embedding batches | `batch embedded?` | None | 2. Embed and upsert |
| `[cred] Pinecone - upsert the vectors` | `n8n-nodes-base.httpRequest` | Upsert vector batches to Pinecone | `batch embedded?` | `STOP: knowledge base loaded`, `STOP: Pinecone refused the upsert` | 2. Embed and upsert |
| `STOP: knowledge base loaded` | `n8n-nodes-base.code` | Report successful batch upsert count | `[cred] Pinecone - upsert the vectors` | None | 2. Embed and upsert |
| `STOP: Pinecone refused the upsert` | `n8n-nodes-base.code` | Handle Pinecone rejection errors | `[cred] Pinecone - upsert the vectors` | None | 2. Embed and upsert |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in n8n, follow these sequential steps:

1. **Create the Entry Point:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`) and name it `Run it now`.

2. **Add the Configuration Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) named `config`.
   - Paste the global configuration and settings payload into the JavaScript code parameter (ensuring proper variable mapping for API models, thresholds, document URLs, and safety settings).
   - Connect `Run it now` to `config`.

3. **Set Up the Document Configuration Gate:**
   - Add an **If** node (`n8n-nodes-base.if`) named `document list configured?`.
   - Set the condition expression to evaluate `{{ $('config').first().json.docs_enabled }}` equals `true` (loose validation enabled).
   - Connect `config` to `document list configured?`.

4. **Build the Live Document Processing Branch:**
   - Add a **Code** node named `list the documents`. Configure it to extract and parse elements from `DOC_URLS`. Connect the true output of `document list configured?` to this node.
   - Add an **HTTP Request** node named `Fetch the document`. Set method to `GET`, URL to `{{ $json.url }}`, response format to `Text`, timeout to `20000ms`, and enable retries (3 attempts, 3000ms wait). Set `onError` to `Continue (Using Regular Output)`. Connect `list the documents` to it.
   - Add a **Code** node named `read the document text` to strip HTML tags, validate lengths (>20 words), and catch fetch errors. Connect `Fetch the document` to it.

5. **Build the Demo Knowledge Base Branch:**
   - Add a **Code** node named `use the demo knowledge base` to map fallback demo objects when `DOC_URLS` is empty. Connect the false output of `document list configured?` to this node.

6. **Implement Document Chunking:**
   - Add a **Code** node named `chunk the documents`. Merge paths from `read the document text` and `use the demo knowledge base`. Configure text sliding windows based on `CHUNK_WORDS` and `CHUNK_OVERLAP` settings with stable IDs (`<doc_id>#<n>`).

7. **Add the Upsert Control Gate:**
   - Add an **If** node named `loading on?`. Set the condition to evaluate `{{ $('config').first().json.upsert_enabled }}` equals `true`. Connect `chunk the documents` to it.
   - Add a **Code** node named `STOP: preview - vectors not upserted` for read-only preview execution. Connect the false output of `loading on?` to it.

8. **Implement Batching and OpenAI Embedding Integration:**
   - Add a **Code** node named `build the embedding batches` to group chunks into sets of 96 items. Connect the true output of `loading on?` to it.
   - Add an **HTTP Request** node named `[cred] OpenAI - embed the chunks`. Set method to `POST`, URL to `https://api.openai.com/v1/embeddings`, body content type to `JSON`, body expression to `={{ JSON.stringify($json._body) }}`, timeout to `60000ms`, and enable retries (3 attempts). Set Authentication to **Predefined Credential Type** using your OpenAI API credential (`openAiApi`). Set `onError` to `Continue (Using Regular Output)`. Connect `build the embedding batches` to it.

9. **Build the Pinecone Upsert Handler:**
   - Add a **Code** node named `build the Pinecone upsert` to format vector definitions and metadata. Connect `[cred] OpenAI - embed the chunks` to it.
   - Add an **If** node named `batch embedded?` checking `{{ $json.embedded }}` equals `true`. Connect `build the Pinecone upsert` to it.
   - Add a **Code** node named `STOP: embedding failed` on the false branch of `batch embedded?` to catch embedding errors.
   - Add an **HTTP Request** node named `[cred] Pinecone - upsert the vectors` on the true branch of `batch embedded?`. Set method to `POST`, URL to `{{ $('config').first().json.pinecone_index_host + '/vectors/upsert' }}`, body content type to `JSON`, body expression to `={{ JSON.stringify($json._body) }}`, timeout to `3000ms`, retries to 3. Set Authentication to **Generic Credential Type** using **Header Auth** (`httpHeaderAuth`) with header parameter `X-Pinecone-API-Version` set to `2025-01`. Set `onError` to `Continue (Using Error Output)`.
   - Add a **Code** node named `STOP: knowledge base loaded` connected to the success output of `[cred] Pinecone - upsert the vectors`.
   - Add a **Code** node named `STOP: Pinecone refused the upsert` connected to the error output of `[cred] Pinecone - upsert the vectors`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Load help pages into a Pinecone knowledge base with OpenAI embeddings workflow overview and configuration details. | Built-in workflow documentation notes and setup guidelines. |
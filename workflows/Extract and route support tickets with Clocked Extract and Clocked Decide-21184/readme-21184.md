Extract and route support tickets with Clocked Extract and Clocked Decide

https://n8nworkflows.xyz/workflows/extract-and-route-support-tickets-with-clocked-extract-and-clocked-decide-21184


# Extract and route support tickets with Clocked Extract and Clocked Decide

### 1. Workflow Overview

This workflow automates the ingestion, extraction, validation, classification, and routing of unstructured support ticket data using the Clocked.ai API. Its core purpose is to parse messy incoming support requests into clean structured fields, determine ticket categories and urgency levels, apply business logic and confidence thresholds, and safely route tickets to either an automated resolution path (ITSM update) or a human review queue while maintaining comprehensive audit logs.

The workflow logic is categorized into the following functional blocks:
- **1.1 Input Reception & Normalization:** Manually triggers the workflow, loads sample support ticket text along with an extraction schema, and normalizes the input while preserving the verbatim original source.
- **1.2 Extraction & Correlation:** Generates a correlation ID, sends the normalized text and schema to Clocked Extract, and merges the resulting API payload back with the original ticket context.
- **1.3 Parsing & Validation:** Analyzes extraction outputs against mandatory field requirements, verifies data types (such as ensuring array formats for lists and strict string validation), checks email validity, and flags missing data or extraction errors.
- **1.4 Classification & Decision Gate:** Prepares payload contexts, generates decision correlation IDs, queries Clocked Decide to classify categories and urgency, and evaluates confidence scores against a threshold ($0.85$) alongside business rules.
- **1.5 Routing & Auditing Execution:** Splits traffic based on approval status. Approved items pass to an ITSM update action (disabled by default) and generate an approved audit trail written to local disk. Unapproved or flagged items route to a human review queue, generating a review audit trail also archived to local disk.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
- **Overview:** Initializes the workflow execution, defines a sample support ticket payload alongside a JSON extraction schema, and normalizes line breaks and whitespace while preserving the original input text.
- **Nodes Involved:** 
  - `When clicking ‘Execute workflow’`
  - `Support Ticket Input with Extractor Rules`
  - `Normalize Input + Preserve Original`
- **Node Details:**
  - **When clicking ‘Execute workflow’**
    - *Type and technical role:* `n8n-nodes-base.manualTrigger` (Manual Trigger). Acts as the primary workflow entry point.
    - *Configuration choices:* Default settings.
    - *Key expressions or variables:* None.
    - *Connections:* Output connects to `Support Ticket Input with Extractor Rules`.
    - *Version requirements:* Version 1.
    - *Edge cases:* None.
  - **Support Ticket Input with Extractor Rules**
    - *Type and technical role:* `n8n-nodes-base.set` (Edit Fields). Defines static test properties including raw support text (`Ticket`) and a structural mapping template (`JSON Extract`).
    - *Configuration choices:* Sets two string variables.
    - *Key expressions or variables:* None.
    - *Connections:* Input from `When clicking ‘Execute workflow’`; output to `Normalize Input + Preserve Original`.
    - *Version requirements:* Version 3.4.
    - *Edge cases:* Malformed JSON string values in the schema definition will cause downstream parsing errors.
  - **Normalize Input + Preserve Original**
    - *Type and technical role:* `n8n-nodes-base.code` (Code Node). JavaScript processor that normalizes spacing, checks for empty strings, safely parses the schema JSON, and assigns tracking metadata.
    - *Configuration choices:* Executes custom JavaScript over input elements.
    - *Key expressions or variables:* Uses `$input.all()`.
    - *Connections:* Input from `Support Ticket Input with Extractor Rules`; output to `Generate Correlation ID For Extract`.
    - *Version requirements:* Version 2.
    - *Edge cases:* Throws explicit runtime errors if the input ticket source is empty or if the extraction schema is not a valid JSON object.

#### 2.2 Extraction & Correlation
- **Overview:** Generates a unique correlation UUID for the extraction phase, posts normalized ticket data to the Clocked Extract endpoint using bearer token authentication, and merges the asynchronous response back with the original ticket context.
- **Nodes Involved:**
  - `Generate Correlation ID For Extract`
  - `Clocked Extract`
  - `Merge Original + Extract Response`
- **Node Details:**
  - **Generate Correlation ID For Extract**
    - *Type and technical role:* `n8n-nodes-base.code` (Code Node). Generates an idempotency UUID string for tracking if not already present.
    - *Configuration choices:* JavaScript random UUID generator function.
    - *Key expressions or variables:* Uses `item.json.correlation_id`.
    - *Connections:* Input from `Normalize Input + Preserve Original`; outputs fan out to both `Clocked Extract` and `Merge Original + Extract Response`.
    - *Version requirements:* Version 2.
    - *Edge cases:* None.
  - **Clocked Extract**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (HTTP Request). Communicates with the Clocked.ai extraction service via POST.
    - *Configuration choices:* Method set to `POST`, timeout set to 30,000ms, redirect following disabled, body formatted as JSON, generic bearer token authentication selected, and `Idempotency-Key` header mapped. Configured to retry up to 3 times on failure with a 1000ms wait interval.
    - *Key expressions or variables:* Request body uses `={{ { text: $json.normalized_text, fields: $json.extraction_schema } }}`; header uses `={{ $json.correlation_id }}`.
    - *Connections:* Input from `Generate Correlation ID For Extract`; output to `Merge Original + Extract Response`.
    - *Version requirements:* Version 4.3.
    - *Credentials:* `ClockedAI` (HTTP Bearer Auth).
    - *Edge cases:* API timeouts, authentication failures, or invalid JSON response structures from the upstream service.
  - **Merge Original + Extract Response**
    - *Type and technical role:* `n8n-nodes-base.merge` (Merge Node). Reconciles data streams using positional combination.
    - *Configuration choices:* Mode set to `combine`, combination criteria set to `combineByPosition`.
    - *Key expressions or variables:* None.
    - *Connections:* Inputs from `Generate Correlation ID For Extract` and `Clocked Extract`; output to `Parse + Validate Extraction`.
    - *Version requirements:* Version 3.
    - *Edge cases:* Position mismatch if asynchronous request sequencing alters item order.

#### 2.3 Parsing & Validation
- **Overview:** Unwraps and validates extraction responses against mandatory business rules, verifying data types, list element constraints, email syntax compliance, and structural integrity.
- **Nodes Involved:**
  - `Parse + Validate Extraction`
- **Node Details:**
  - **Parse + Validate Extraction**
    - *Type and technical role:* `n8n-nodes-base.code` (Code Node). Comprehensive script evaluating required fields (`requester`, `ticket_id`, `email`, `device`, `assigned_engineer`, `error_message`, `affected_applications`, `attempted_fixes`, `callback_time`), checking array list types, validating email regex, and populating validation status flags.
    - *Configuration choices:* Iterates through input records running validation assertions.
    - *Key expressions or variables:* Uses `$input.all()`.
    - *Connections:* Input from `Merge Original + Extract Response`; outputs fan out to `Merge Decision + Ticket Context` and `Generate Correlation ID For Decide`.
    - *Version requirements:* Version 2.
    - *Edge cases:* Missing nesting properties inside raw provider payloads are safely caught and routed to review flags.

#### 2.4 Classification & Decision Gate
- **Overview:** Assigns decision-specific correlation identifiers, queries Clocked Decide to classify operational categories and urgencies, and evaluates model outputs against confidence thresholds and business rules.
- **Nodes Involved:**
  - `Generate Correlation ID For Decide`
  - `Clocked Decide`
  - `Merge Decision + Ticket Context`
  - `Evaluate Rules + Confidence Gate`
- **Node Details:**
  - **Generate Correlation ID For Decide**
    - *Type and technical role:* `n8n-nodes-base.code` (Code Node). Generates a fresh correlation UUID for the decision request stage.
    - *Configuration choices:* JavaScript random UUID snippet.
    - *Key expressions or variables:* Modifies `item.json.correlation_id`.
    - *Connections:* Input from `Parse + Validate Extraction`; output to `Clocked Decide`.
    - *Version requirements:* Version 2.
    - *Edge cases:* None.
  - **Clocked Decide**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (HTTP Request). Sends normalized states and extracted fields to the Clocked Decide engine.
    - *Configuration choices:* Method set to `POST`, 30,000ms timeout, redirection disabled, JSON body payload specified with explicit categorical criteria definitions, bearer token authentication enabled, and `Idempotency-Key` header included. Configured to retry up to 3 times.
    - *Key expressions or variables:* Request body uses structural state and question maps referencing `{{ $json.normalized_text }}` and `{{ $json.extracted_fields }}`; header uses `={{ $json.correlation_id }}`.
    - *Connections:* Input from `Generate Correlation ID For Decide`; output to `Merge Decision + Ticket Context`.
    - *Version requirements:* Version 4.3.
    - *Credentials:* `ClockedAI` (HTTP Bearer Auth).
    - *Edge cases:* API failure responses, malformed criteria schemas, or unexpected decision answer payloads.
  - **Merge Decision + Ticket Context**
    - *Type and technical role:* `n8n-nodes-base.merge` (Merge Node). Reconciles decision responses with existing ticket contexts.
    - *Configuration choices:* Mode set to `combine`, combination criteria set to `combineByPosition`.
    - *Key expressions or variables:* None.
    - *Connections:* Inputs from `Parse + Validate Extraction` (Index 0) and `Clocked Decide` (Index 1); output to `Evaluate Rules + Confidence Gate`.
    - *Version requirements:* Version 3.
    - *Edge cases:* Index misalignment if payloads go out of sync.
  - **Evaluate Rules + Confidence Gate**
    - *Type and technical role:* `n8n-nodes-base.code` (Code Node). Implements the confidence threshold gate ($0.85$), inspects review flags, compiles failure reasons, sets `update_approved` booleans, and formats standard audit records.
    - *Configuration choices:* Custom evaluation logic script.
    - *Key expressions or variables:* Uses `MIN_CONFIDENCE = 0.85` and `$input.all()`.
    - *Connections:* Input from `Merge Decision + Ticket Context`; output to `Update Approved?`.
    - *Version requirements:* Version 2.
    - *Edge cases:* Null or non-numeric confidence responses default to review queues.

#### 2.5 Routing & Auditing Execution
- **Overview:** Splits workflow execution paths based on the evaluation approval flag. Approved tickets proceed to ITSM update actions and storage logging, while flagged or unapproved tickets route to human review queues and separate archive logging.
- **Nodes Involved:**
  - `Update Approved?`
  - `ITSM Update - CONFIGURE BEFORE USE`
  - `Audit Record - Approved Path`
  - `Convert To Binary - Approved`
  - `Write Binary File - Approved`
  - `Human Review Queue - CONFIGURE`
  - `Audit Record - Review Path`
  - `Convert To Binary - Human Review`
  - `Write Binary File - Human Review`
- **Node Details:**
  - **Update Approved?**
    - *Type and technical role:* `n8n-nodes-base.if` (If Node). Evaluates approval status to split workflow execution.
    - *Configuration choices:* Strict type validation rule checking whether `update_approved` equals `true`.
    - *Key expressions or variables:* `={{ $json.update_approved }}`.
    - *Connections:* Input from `Evaluate Rules + Confidence Gate`; true branch outputs to `ITSM Update - CONFIGURE BEFORE USE`, false branch outputs to `Human Review Queue - CONFIGURE`.
    - *Version requirements:* Version 2.
    - *Edge cases:* Boolean evaluation mismatches will route items down incorrect paths.
  - **ITSM Update - CONFIGURE BEFORE USE**
    - *Type and technical role:* `n8n-nodes-base.httpRequest` (HTTP Request). Placeholder node for updating ticket management systems via PATCH requests. (Disabled by default).
    - *Configuration choices:* HTTP method set to `PATCH`, 20,000ms timeout, JSON body configured with categorized payloads, and custom `Content-Type` header set.
    - *Key expressions or variables:* URL targets `https://YOUR-ITSM-INSTANCE.example/api/tickets/{{ $json.extracted_fields.ticket_id }}`; body maps category, urgency, and extracted values.
    - *Connections:* Input from `Update Approved?` (True branch); output to `Audit Record - Approved Path`.
    - *Version requirements:* Version 4.3.
    - *Edge cases:* Disabled by default; must be configured with correct tenant credentials and endpoints before activation.
  - **Audit Record - Approved Path**
    - *Type and technical role:* `n8n-nodes-base.set` (Edit Fields). Assigns audit status flags and serializes records into string payloads.
    - *Configuration choices:* Sets `audit_status` to `approved_for_update` and serializes audit json.
    - *Key expressions or variables:* `={{ $json.toJsonString() }}`.
    - *Connections:* Input from `ITSM Update - CONFIGURE BEFORE USE`; output to `Convert To Binary - Approved`.
    - *Version requirements:* Version 3.4.
    - *Edge cases:* Circular structures in JSON payloads can cause stringification errors.
  - **Convert To Binary - Approved**
    - *Type and technical role:* `n8n-nodes-base.function` (Function Node). Converts JSON object strings into Buffer binary structures.
    - *Configuration choices:* Custom buffer conversion code.
    - *Key expressions or variables:* Uses `Buffer(JSON.stringify(...)).toString('base64')`.
    - *Connections:* Input from `Audit Record - Approved Path`; output to `Write Binary File - Approved`.
    - *Version requirements:* Version 1.
    - *Edge cases:* Memory limits when handling large payloads.
  - **Write Binary File - Approved**
    - *Type and technical role:* `n8n-nodes-base.writeBinaryFile` (Write Binary File). Writes approved binary payload streams to local storage destinations.
    - *Configuration choices:* File path defined as `C:\n8n\data\ticket_structured_approved_data.json`.
    - *Key expressions or variables:* None.
    - *Connections:* Input from `Convert To Binary - Approved`; output ends branch.
    - *Version requirements:* Version 1.
    - *Edge cases:* File system permission denials or missing directory paths on the target host environment.
  - **Human Review Queue - CONFIGURE**
    - *Type and technical role:* `n8n-nodes-base.set` (Edit Fields). Prepares payloads for manual review workflows or external review queues.
    - *Configuration choices:* Assigns review status and packages complete context payloads.
    - *Key expressions or variables:* `={{ { correlation_id: $json.correlation_id, source: $json.original_source, extraction: $json.extracted_fields, extraction_validation: $json.extraction_validation, decision: $json.decision_summary, audit: $json.audit_record } }}`.
    - *Connections:* Input from `Update Approved?` (False branch); output to `Audit Record - Review Path`.
    - *Version requirements:* Version 3.4.
    - *Edge cases:* Exposing sensitive raw ticket texts to unapproved channels if misconfigured.
  - **Audit Record - Review Path**
    - *Type and technical role:* `n8n-nodes-base.set` (Edit Fields). Assigns review audit statuses and serializes payloads.
    - *Configuration choices:* Sets `audit_status` to `sent_to_human_review`.
    - *Key expressions or variables:* `={{ $json.toJsonString() }}`.
    - *Connections:* Input from `Human Review Queue - CONFIGURE`; output to `Convert To Binary - Human Review`.
    - *Version requirements:* Version 3.4.
    - *Edge cases:* None.
  - **Convert To Binary - Human Review**
    - *Type and technical role:* `n8n-nodes-base.function` (Function Node). Converts review audit records into binary format.
    - *Configuration choices:* Buffer encoding snippet.
    - *Key expressions or variables:* Uses Node.js `Buffer` constructor.
    - *Connections:* Input from `Audit Record - Review Path`; output to `Write Binary File - Human Review`.
    - *Version requirements:* Version 1.
    - *Edge cases:* None.
  - **Write Binary File - Human Review**
    - *Type and technical role:* `n8n-nodes-base.writeBinaryFile` (Write Binary File). Writes pending human review payloads to local storage.
    - *Configuration choices:* File path defined as `C:\n8n\data\ticket_structured_pending_human_review_data.json`.
    - *Key expressions or variables:* None.
    - *Connections:* Input from `Convert To Binary - Human Review`; output ends branch.
    - *Version requirements:* Version 1.
    - *Edge cases:* Directory write restrictions on host servers.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When clicking ‘Execute workflow’ | `manualTrigger` | Entry point | None | Support Ticket Input with Extractor Rules | Support Ticket Data Extraction – Analysis & Routing Automation with Clocked.ai<br><br>### How it works<br><br>This workflow extracts structured data from support tickets using Clocked.ai, validates the extracted fields, and evaluates ticket decisions using Clocked Decide. It applies confidence and business rules to determine whether a ticket can be approved for automated updates or requires human review. Both paths generate audit records and save their outputs to files.<br><br>### Setup steps<br><br>- Configure the support ticket input and extraction rules.<br>- Set up Clocked.ai API credentials, extraction endpoint, and unique correlation IDs.<br>- Configure extraction parsing, validation, and Clocked Decide inputs.<br>- Review the business rules and confidence thresholds.<br>- Configure the approved and human-review output paths, audit records, and file destinations.<br>- Test both paths with valid, incomplete, and ambiguous tickets.<br><br>### Customization<br><br>Adjust extraction fields, decision criteria, confidence thresholds, and review rules. Connect the approved path to your ticketing system and configure a human-review destination for cases requiring manual intervention. |
| Support Ticket Input with Extractor Rules | `set` | Defines ticket text and schema | When clicking ‘Execute workflow’ | Normalize Input + Preserve Original | Support Ticket Data Extraction – Analysis & Routing Automation with Clocked.ai<br><br>### How it works<br><br>This workflow extracts structured data from support tickets using Clocked.ai, validates the extracted fields, and evaluates ticket decisions using Clocked Decide. It applies confidence and business rules to determine whether a ticket can be approved for automated updates or requires human review. Both paths generate audit records and save their outputs to files.<br><br>### Setup steps<br><br>- Configure the support ticket input and extraction rules.<br>- Set up Clocked.ai API credentials, extraction endpoint, and unique correlation IDs.<br>- Configure extraction parsing, validation, and Clocked Decide inputs.<br>- Review the business rules and confidence thresholds.<br>- Configure the approved and human-review output paths, audit records, and file destinations.<br>- Test both paths with valid, incomplete, and ambiguous tickets.<br><br>### Customization<br><br>Adjust extraction fields, decision criteria, confidence thresholds, and review rules. Connect the approved path to your ticketing system and configure a human-review destination for cases requiring manual intervention. |
| Normalize Input + Preserve Original | `code` | Normalizes text and preserves source | Support Ticket Input with Extractor Rules | Generate Correlation ID For Extract | Manual Input + Normalization<br><br>Provide the support ticket text and extraction rules. Normalize the input while preserving the original ticket content for traceability. |
| Generate Correlation ID For Extract | `code` | Creates extraction correlation ID | Normalize Input + Preserve Original | Clocked Extract, Merge Original + Extract Response | Clocked Extract and Merge<br><br>Extract structured ticket details using Clocked Extract (https://clocked.rezolve.ai/extract) and merge the API response with the original input and correlation ID. |
| Clocked Extract | `httpRequest` | Calls Clocked Extract API | Generate Correlation ID For Extract | Merge Original + Extract Response | Clocked Extract and Merge<br><br>Extract structured ticket details using Clocked Extract (https://clocked.rezolve.ai/extract) and merge the API response with the original input and correlation ID. |
| Merge Original + Extract Response | `merge` | Merges original input with extraction | Generate Correlation ID For Extract, Clocked Extract | Parse + Validate Extraction | Clocked Extract and Merge<br><br>Extract structured ticket details using Clocked Extract (https://clocked.rezolve.ai/extract) and merge the API response with the original input and correlation ID. |
| Parse + Validate Extraction | `code` | Validates fields and checks types | Merge Original + Extract Response | Merge Decision + Ticket Context, Generate Correlation ID For Decide | Parse & Validate<br><br>Parse the extraction results, validate required fields and data types, and flag missing or invalid information for review. |
| Generate Correlation ID For Decide | `code` | Creates decision correlation ID | Parse + Validate Extraction | Clocked Decide | Clocked Decide and Merge<br><br>Use Clocked Decide (https://clocked.rezolve.ai/decide) to evaluate the ticket based on the extracted data and context, then merge the decision with the ticket information. |
| Clocked Decide | `httpRequest` | Calls Clocked Decide API | Generate Correlation ID For Decide | Merge Decision + Ticket Context | Clocked Decide and Merge<br><br>Use Clocked Decide (https://clocked.rezolve.ai/decide) to evaluate the ticket based on the extracted data and context, then merge the decision with the ticket information. |
| Merge Decision + Ticket Context | `merge` | Merges decision with ticket context | Parse + Validate Extraction, Clocked Decide | Evaluate Rules + Confidence Gate | Clocked Decide and Merge<br><br>Use Clocked Decide (https://clocked.rezolve.ai/decide) to evaluate the ticket based on the extracted data and context, then merge the decision with the ticket information. |
| Evaluate Rules + Confidence Gate | `code` | Evaluates confidence and business rules | Merge Decision + Ticket Context | Update Approved? | Evaluate<br><br>Apply business rules and confidence thresholds to determine whether the ticket qualifies for automated processing or requires human review. |
| Update Approved? | `if` | Splits path based on approval status | Evaluate Rules + Confidence Gate | ITSM Update - CONFIGURE BEFORE USE, Human Review Queue - CONFIGURE | Evaluate<br><br>Apply business rules and confidence thresholds to determine whether the ticket qualifies for automated processing or requires human review. |
| ITSM Update - CONFIGURE BEFORE USE | `httpRequest` | Updates ITSM ticket (Disabled) | Update Approved? | Audit Record - Approved Path | Approved Flow<br><br>Approved tickets, prepare the ITSM update, generate an audit record, and save the approved output. Configure the ITSM integration before enabling this path. |
| Audit Record - Approved Path | `set` | Formats approved audit record | ITSM Update - CONFIGURE BEFORE USE | Convert To Binary - Approved | Approved Flow<br><br>Approved tickets, prepare the ITSM update, generate an audit record, and save the approved output. Configure the ITSM integration before enabling this path. |
| Convert To Binary - Approved | `function` | Converts approved JSON to binary | Audit Record - Approved Path | Write Binary File - Approved | Approved Flow<br><br>Approved tickets, prepare the ITSM update, generate an audit record, and save the approved output. Configure the ITSM integration before enabling this path. |
| Write Binary File - Approved | `writeBinaryFile` | Writes approved payload to disk | Convert To Binary - Approved | None | Approved Flow<br><br>Approved tickets, prepare the ITSM update, generate an audit record, and save the approved output. Configure the ITSM integration before enabling this path. |
| Human Review Queue - CONFIGURE | `set` | Packages review payload | Update Approved? | Audit Record - Review Path | Human Review Flow<br><br>Route tickets requiring manual intervention to a review queue, record the review audit data, and save the output for follow-up. |
| Audit Record - Review Path | `set` | Formats review audit record | Human Review Queue - CONFIGURE | Convert To Binary - Human Review | Human Review Flow<br><br>Route tickets requiring manual intervention to a review queue, record the review audit data, and save the output for follow-up. |
| Convert To Binary - Human Review | `function` | Converts review JSON to binary | Audit Record - Review Path | Write Binary File - Human Review | Human Review Flow<br><br>Route tickets requiring manual intervention to a review queue, record the review audit data, and save the output for follow-up. |
| Write Binary File - Human Review | `writeBinaryFile` | Writes review payload to disk | Convert To Binary - Human Review | None | Human Review Flow<br><br>Route tickets requiring manual intervention to a review queue, record the review audit data, and save the output for follow-up. |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow from scratch in n8n, follow these sequential setup instructions:

1. **Create the Entry Point:**
   - Add a **Manual Trigger** node (`When clicking ‘Execute workflow’`).
2. **Add Input Data and Extraction Rules:**
   - Add an **Edit Fields (Set)** node named `Support Ticket Input with Extractor Rules`.
   - Create two string assignments: `Ticket` (containing sample support text) and `JSON Extract` (containing the structural JSON schema definition).
   - Connect the Manual Trigger to this Set node.
3. **Normalize Input Data:**
   - Add a **Code** node named `Normalize Input + Preserve Original`.
   - Insert JavaScript to strip trailing whitespaces, normalize line endings, parse the JSON schema, and establish timestamps (`received_at`, `original_source`, `normalized_text`).
   - Connect `Support Ticket Input with Extractor Rules` to this node.
4. **Setup Extraction Tracking & API Call:**
   - Add a **Code** node named `Generate Correlation ID For Extract` to create idempotency UUIDs. Connect `Normalize Input + Preserve Original` to it.
   - Add an **HTTP Request** node named `Clocked Extract`.
     - Configure method as `POST`, URL as `https://clocked.rezolve.ai/clocked/extract`, and set a timeout of `30000ms`.
     - Configure authentication using a **Generic Credential Type** -> **HTTP Bearer Auth** (Credential Name: `ClockedAI`).
     - Set request headers: `Idempotency-Key` = `={{ $json.correlation_id }}`.
     - Set request body (JSON) to `={{ { text: $json.normalized_text, fields: $json.extraction_schema } }}`.
     - Enable retry on failure (3 attempts, 1000ms wait).
   - Add a **Merge** node named `Merge Original + Extract Response` set to positional combination. Connect both `Generate Correlation ID For Extract` and `Clocked Extract` to its inputs.
5. **Parse and Validate Extractions:**
   - Add a **Code** node named `Parse + Validate Extraction`.
   - Implement validation scripts to verify mandatory fields (`requester`, `ticket_id`, `email`, `device`, `assigned_engineer`, `error_message`, `affected_applications`, `attempted_fixes`, `callback_time`), enforce array types for list fields, and run email regex validation.
   - Connect `Merge Original + Extract Response` to this node.
6. **Setup Decision Tracking & API Call:**
   - Add a **Code** node named `Generate Correlation ID For Decide`. Connect `Parse + Validate Extraction` to it.
   - Add an **HTTP Request** node named `Clocked Decide`.
     - Configure method as `POST`, URL as `https://clocked.rezolve.ai/clocked/decide`, timeout `30000ms`.
     - Select HTTP Bearer Auth credential (`ClockedAI`).
     - Configure header `Idempotency-Key` = `={{ $json.correlation_id }}`.
     - Set request body (JSON) containing state and question blocks (`ticket_category`, `urgency`, `requires_human_review`).
     - Enable retry on failure.
   - Add a **Merge** node named `Merge Decision + Ticket Context` set to positional combination. Connect `Parse + Validate Extraction` (Index 0) and `Clocked Decide` (Index 1) into it.
7. **Evaluate Rules and Set Confidence Thresholds:**
   - Add a **Code** node named `Evaluate Rules + Confidence Gate`.
   - Implement logic checking whether confidence is `>= 0.85`, verifying missing category flags, reviewing model review requests, and outputting an `update_approved` boolean alongside an `audit_record`.
   - Connect `Merge Decision + Ticket Context` to this node.
8. **Configure Branching Logic (If Node):**
   - Add an **If** node named `Update Approved?`.
   - Set condition to check whether `{{ $json.update_approved }}` equals `true`.
   - Connect `Evaluate Rules + Confidence Gate` to this node.
9. **Configure the Approved Path:**
   - Connect the **True** output of `Update Approved?` to an **HTTP Request** node named `ITSM Update - CONFIGURE BEFORE USE` (Leave disabled). Set URL to your ITSM endpoint (`https://YOUR-ITSM-INSTANCE.example/api/tickets/{{ $json.extracted_fields.ticket_id }}`), method `PATCH`, and supply JSON body payloads.
   - Add an **Edit Fields (Set)** node named `Audit Record - Approved Path` to format audit tracking strings (`approved_for_update`).
   - Add a **Function** node named `Convert To Binary - Approved` to convert payload strings into Base64 Buffer binary items.
   - Add a **Write Binary File** node named `Write Binary File - Approved` targeting file path `C:\n8n\data\ticket_structured_approved_data.json`.
10. **Configure the Review Path:**
    - Connect the **False** output of `Update Approved?` to an **Edit Fields (Set)** node named `Human Review Queue - CONFIGURE` to package review data payloads.
    - Add an **Edit Fields (Set)** node named `Audit Record - Review Path` to format review statuses (`sent_to_human_review`).
    - Add a **Function** node named `Convert To Binary - Human Review` to encode review objects into Base64 binary items.
    - Add a **Write Binary File** node named `Write Binary File - Human Review` targeting file path `C:\n8n\data\ticket_structured_pending_human_review_data.json`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| ![Resolve.ai](https://www.rezolve.ai/brand/rezolve-wordmark-light.svg) | Branding asset utilized in workflow sticky notes representing Rezolve.ai / Clocked.ai |
| Clocked Extract API Endpoint | [https://clocked.rezolve.ai/clocked/extract](https://clocked.rezolve.ai/clocked/extract) |
| Clocked Decide API Endpoint | [https://clocked.rezolve.ai/clocked/decide](https://clocked.rezolve.ai/decide) |
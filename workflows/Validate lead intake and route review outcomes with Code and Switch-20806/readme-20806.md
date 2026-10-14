Validate lead intake and route review outcomes with Code and Switch

https://n8nworkflows.xyz/workflows/validate-lead-intake-and-route-review-outcomes-with-code-and-switch-20806


# Validate lead intake and route review outcomes with Code and Switch

### 1. Workflow Overview

This workflow is an educational, credential-free integration blueprint designed for sales operations builders and n8n learners. Its primary purpose is to ingest, validate, and diagnose malformed, incomplete, or duplicate lead payloads before any CRM handoff occurs. 

The processing logic is structured into the following functional blocks:
- **1.1 Input Reception & Synthetic Generation:** Manually starts the process and generates an offline batch of fictional lead payloads containing various edge cases (e.g., missing contact details, invalid email formats, duplicate IDs, malformed fields, and bad UTM parameters).
- **1.2 Preflight Evaluation & Normalization:** Processes the raw leads via custom JavaScript logic to sanitize string fields, validate event IDs, verify email/phone syntaxes, and track intra-run duplicates.
- **1.3 Conditional Routing (Switch):** Directs assessed items into specific review, hold, or fallback paths based on their assigned preflight status code.
- **1.4 Outcome Processing & Finalization:** Enriches items in each branch with a distinct route decision tag to allow granular inspection per pathway.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Synthetic Generation
- **Overview:** Initializes the execution manually and supplies a predefined array of synthetic lead records to simulate incoming webhook or form submissions.
- **Nodes Involved:** 
  - `Run Synthetic Preflight`
  - `Generate Synthetic Examples`
- **Node Details:**
  - **Run Synthetic Preflight**
    - *Type & Technical Role:* `n8n-nodes-base.manualTrigger` (Trigger node)
    - *Configuration:* Default settings; requires user interaction to start.
    - *Input/Output:* No inputs; outputs a single execution trigger object.
    - *Failure Types:* None (manual execution only).
  - **Generate Synthetic Examples**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution node, running once for all items).
    - *Configuration:* Hardcodes an array of 9 mock lead objects (`fixtures`) covering valid formats and specific edge cases (e.g., `.invalid` TLDs, non-string payloads, missing values). Maps them into standard n8n items (`{ json: fixture }`).
    - *Key Expressions/Variables:* Uses standard ES6 JavaScript array mapping.
    - *Input/Output:* Input from `Run Synthetic Preflight`; outputs 9 distinct item objects.
    - *Failure Types:* JavaScript syntax errors (if modified incorrectly).

#### 2.2 Preflight Evaluation & Normalization
- **Overview:** Evaluates the generated batch against strict validation logic, normalizes string fields, cleans whitespace, checks formatting constraints, and detects duplicates *within the current run*.
- **Nodes Involved:** 
  - `Evaluate Lead Preflight`
- **Node Details:**
  - **Evaluate Lead Preflight**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution node, running once for all items).
    - *Configuration:* Utilizes internal `Set` objects (`seenEventIds`, `seenEmails`) to track duplicates per batch. Implements helper functions (`readText`, `statusResult`) to sanitize input lengths and types, generating diagnostic reason codes and standardized statuses.
    - *Key Expressions/Variables:* Iterates over `$input.all()`, evaluating fields like `event_id`, `email`, `phone`, and UTM variables.
    - *Input/Output:* Input from `Generate Synthetic Examples`; outputs 9 evaluated items containing `case_id`, `status`, `reason_codes`, `normalized_fields`, and `diagnostic_scope`.
    - *Failure Types:* Type coercion errors if input schemas change unexpectedly.

#### 2.3 Conditional Routing (Switch)
- **Overview:** Inspects the evaluated status of each lead and branches execution into one of eight distinct review/hold paths or a fallback catcher.
- **Nodes Involved:** 
  - `Route by Preflight Status`
- **Node Details:**
  - **Route by Preflight Status**
    - *Type & Technical Role:* `n8n-nodes-base.switch` (Routing node, Version 3.4).
    - *Configuration:* Configured in `rules` mode with strict case-insensitive string matching on `={{ $json.status }}`. Defines 8 named output rules and 1 fallback output (`Fallback`).
    - *Input/Output:* Input from `Evaluate Lead Preflight`; outputs items to matching destination nodes based on status values (`READY_FOR_REVIEW`, `REVIEW_PHONE_ONLY`, `HOLD_MISSING_CONTACT`, `HOLD_BAD_EMAIL`, `REVIEW_DUPLICATE_IN_BATCH`, `REVIEW_SAME_EMAIL_IN_BATCH`, `HOLD_INVALID_EVENT`, `HOLD_MALFORMED_INPUT`, and Fallback).
    - *Failure Types:* Unhandled status strings routing to the fallback block.

#### 2.4 Outcome Processing & Finalization
- **Overview:** Assigns an explicit `route_decision` property matching the branch name to each item, ensuring traceability before inspection or logging.
- **Nodes Involved:** 
  - `Decision: READY_FOR_REVIEW`
  - `Decision: REVIEW_PHONE_ONLY`
  - `Decision: HOLD_MISSING_CONTACT`
  - `Decision: HOLD_BAD_EMAIL`
  - `Decision: REVIEW_DUPLICATE_IN_BATCH`
  - `Decision: REVIEW_SAME_EMAIL_IN_BATCH`
  - `Decision: HOLD_INVALID_EVENT`
  - `Decision: HOLD_MALFORMED_INPUT`
  - `Unexpected Status: Hold For Review`
- **Node Details:**
  - **Decision Nodes (Set)** (`n8n-nodes-base.set`)
    - *Type & Technical Role:* Edit Fields / Set node (Version 3.5).
    - *Configuration:* Manual mode with `includeOtherFields: true`. Adds a static string assignment for `route_decision` corresponding to the specific branch name.
    - *Input/Output:* Inputs from respective outputs of `Route by Preflight Status`; outputs enriched JSON items.
  - **Unexpected Status: Hold For Review**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution node).
    - *Configuration:* Catches unhandled statuses from the Switch fallback, forcing status to `HOLD_MALFORMED_INPUT`, `route_decision` to `FALLBACK_MANUAL_REVIEW`, and appending `UNEXPECTED_ROUTING_STATUS` to `reason_codes`.
    - *Input/Output:* Input from Switch fallback output; outputs modified safety-hold items.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Synthetic Preflight | manualTrigger | Manual trigger for offline batch run | None | Generate Synthetic Examples | Start synthetic preflight |
| Generate Synthetic Examples | code | Generates 9 fictional lead edge-case fixtures | Run Synthetic Preflight | Evaluate Lead Preflight | Generate and evaluate leads |
| Evaluate Lead Preflight | code | Sanitizes, validates, and flags lead records | Generate Synthetic Examples | Route by Preflight Status | Generate and evaluate leads |
| Route by Preflight Status | switch | Routes items based on preflight status | Evaluate Lead Preflight | All 9 Decision/Fallback nodes | Route validation status |
| Decision: READY_FOR_REVIEW | set | Assigns ready review route decision | Route by Preflight Status | None | Primary review outcomes |
| Decision: REVIEW_PHONE_ONLY | set | Assigns phone-only review decision | Route by Preflight Status | None | Primary review outcomes |
| Decision: HOLD_MISSING_CONTACT | set | Assigns missing contact hold decision | Route by Preflight Status | None | Primary review outcomes |
| Decision: HOLD_BAD_EMAIL | set | Assigns bad email hold decision | Route by Preflight Status | None | Primary review outcomes |
| Decision: REVIEW_DUPLICATE_IN_BATCH | set | Assigns duplicate batch review decision | Route by Preflight Status | None | Batch and event outcomes |
| Decision: REVIEW_SAME_EMAIL_IN_BATCH | set | Assigns same-email batch review decision | Route by Preflight Status | None | Batch and event outcomes |
| Decision: HOLD_INVALID_EVENT | set | Assigns invalid event hold decision | Route by Preflight Status | None | Batch and event outcomes |
| Decision: HOLD_MALFORMED_INPUT | set | Assigns malformed input hold decision | Route by Preflight Status | None | Malformed and fallback holds |
| Unexpected Status: Hold For Review | code | Fallback handler for unmapped statuses | Route by Preflight Status | None | Malformed and fallback holds |
| Overview: Lead intake preflight | stickyNote | High-level architectural documentation | None | None | Validate lead intake and route review outcomes with Code and Switch |
| Section: Start synthetic preflight | stickyNote | Visual boundary comment | None | None | Start synthetic preflight |
| Section: Generate and evaluate leads | stickyNote | Visual boundary comment | None | None | Generate and evaluate leads |
| Section: Route validation status | stickyNote | Visual boundary comment | None | None | Route validation status |
| Section: Primary review outcomes | stickyNote | Visual boundary comment | None | None | Primary review outcomes |
| Section: Batch and event outcomes | stickyNote | Visual boundary comment | None | None | Batch and event outcomes |
| Section: Malformed and fallback holds | stickyNote | Visual boundary comment | None | None | Malformed and fallback holds |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this workflow manually in an n8n instance, follow these sequential steps:

1. **Create the Trigger Node:**
   - Add a **Manual Trigger** node. Name it `Run Synthetic Preflight`. Leave parameters at default.

2. **Add the Synthetic Generator:**
   - Add a **Code** node named `Generate Synthetic Examples`. Connect `Run Synthetic Preflight` to its main input.
   - Set mode to `Run Once for All Items`.
   - Set Language to `JavaScript`.
   - Paste the array of 9 mock fixtures (`01_valid` through `09_bad_utm_safe`) returning `fixtures.map(fixture => ({ json: fixture }))`.

3. **Add the Preflight Evaluation Logic:**
   - Add a **Code** node named `Evaluate Lead Preflight`. Connect `Generate Synthetic Examples` to its main input.
   - Set mode to `Run Once for All Items`.
   - Implement JavaScript validation logic to initialize duplicate tracking `Set` objects, sanitize field types using length/regex checks, assign standardized status codes (`READY_FOR_REVIEW`, `HOLD_MALFORMED_INPUT`, etc.), and output structured diagnostic objects containing `case_id`, `status`, `reason_codes`, and `normalized_fields`.

4. **Configure the Switch Routing Node:**
   - Add a **Switch** node named `Route by Preflight Status`. Connect `Evaluate Lead Preflight` to its input.
   - Set mode to `Rules`. Configure string comparison rules checking `={{ $json.status }}` against the 8 expected status strings:
     1. `READY_FOR_REVIEW`
     2. `REVIEW_PHONE_ONLY`
     3. `HOLD_MISSING_CONTACT`
     4. `HOLD_BAD_EMAIL`
     5. `REVIEW_DUPLICATE_IN_BATCH`
     6. `REVIEW_SAME_EMAIL_IN_BATCH`
     7. `HOLD_INVALID_EVENT`
     8. `HOLD_MALFORMED_INPUT`
   - Enable fallback output and name it `Fallback`.

5. **Create the Decision (Set) Nodes:**
   - Add 8 **Edit Fields (Set)** nodes corresponding to each switch rule output:
     - `Decision: READY_FOR_REVIEW` (Assign string `route_decision` = `READY_FOR_REVIEW`)
     - `Decision: REVIEW_PHONE_ONLY` (Assign string `route_decision` = `REVIEW_PHONE_ONLY`)
     - `Decision: HOLD_MISSING_CONTACT` (Assign string `route_decision` = `HOLD_MISSING_CONTACT`)
     - `Decision: HOLD_BAD_EMAIL` (Assign string `route_decision` = `HOLD_BAD_EMAIL`)
     - `Decision: REVIEW_DUPLICATE_IN_BATCH` (Assign string `route_decision` = `REVIEW_DUPLICATE_IN_BATCH`)
     - `Decision: REVIEW_SAME_EMAIL_IN_BATCH` (Assign string `route_decision` = `REVIEW_SAME_EMAIL_IN_BATCH`)
     - `Decision: HOLD_INVALID_EVENT` (Assign string `route_decision` = `HOLD_INVALID_EVENT`)
     - `Decision: HOLD_MALFORMED_INPUT` (Assign string `route_decision` = `HOLD_MALFORMED_INPUT`)
   - Ensure `Include Other Fields` is enabled on all Set nodes.
   - Connect each corresponding output port of `Route by Preflight Status` to its matching Decision node.

6. **Configure the Fallback Node:**
   - Add a **Code** node named `Unexpected Status: Hold For Review`.
   - Connect the Switch node's Fallback output port to this node.
   - Add JavaScript logic to catch unexpected payloads, assign `HOLD_MALFORMED_INPUT`, `FALLBACK_MANUAL_REVIEW`, and append the `UNEXPECTED_ROUTING_STATUS` reason code.

7. **Finalize:**
   - Optionally, add **Sticky Note** nodes to visually section and document the workflow canvas according to the node layout guidelines.
   - Keep the workflow inactive and click **Execute Workflow** to verify the 9 expected output paths.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Validate lead intake and route review outcomes with Code and Switch | Educational preflight workflow demonstrating offline payload validation without external credentials or CRM writes. |
| Verification Environment | Built and verified on n8n version 2.42.7 using CLI directory import. |
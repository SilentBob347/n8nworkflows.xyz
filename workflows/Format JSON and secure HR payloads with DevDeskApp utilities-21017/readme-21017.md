Format JSON and secure HR payloads with DevDeskApp utilities

https://n8nworkflows.xyz/workflows/format-json-and-secure-hr-payloads-with-devdeskapp-utilities-21017


# Format JSON and secure HR payloads with DevDeskApp utilities

### 1. Workflow Overview

This workflow is designed to execute local data processing, security hashing, text transformation, and compensation calculations using the **DevDeskApp** community utility suite. It serves as an automated pipeline for preparing secure, standardized HR payloads and developer metrics without relying on external server calls. 

The logical flow is organized into the following functional blocks:
- **1.1 Input Initiation & Mock Data Generation:** Triggers the workflow manually and seeds the execution context with a minified JSON API payload, a candidate's full name, annual CTC, and a basic salary percentage.
- **1.2 Data Cleaning & Security Processing:** Formats the minified JSON into a readable tree structure, generates a tamper-evident SHA-256 cryptographic hash, and assigns an RFC 4122-compliant UUID v4 trace ID for tracking.
- **1.3 Business Logic & HR Calculations:** Slugifies the candidate's full name into a URL-safe identifier and processes a detailed CTC-to-in-hand salary breakdown (including monthly gross, take-home, provident fund, and estimated taxes).
- **1.4 Output Consolidation:** Merges all processed metadata, structured JSON elements, and financial computations into a final, unified JSON object.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initiation & Mock Data Generation
- **Overview:** Initiates the workflow execution manually and defines initial variables containing raw minified JSON data alongside candidate and compensation parameters.
- **Nodes Involved:** 
  - `Manual Run (Instant Test)`
  - `Sample Developer & HR Payload`
- **Node Details:**
  - **Manual Run (Instant Test)**
    - *Type & Role:* `n8n-nodes-base.manualTrigger` (Trigger node).
    - *Configuration:* Default execution trigger requiring a manual click in the UI.
    - *Input/Output:* No inputs; outputs a single execution trigger item to downstream nodes.
    - *Edge Cases:* Manual execution only; cannot be scheduled or web-hooked natively without replacing the trigger.
  - **Sample Developer & HR Payload**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript execution node).
    - *Configuration:* Returns a hardcoded array containing a minified JSON string (`rawMinifiedApiData`), a candidate full name (`candidateFullName`), annual CTC (`annualCtc`), and a basic salary percentage (`basicPercentage`).
    - *Key Expressions:* Uses `JSON.stringify()` to structure the initial mock payload.
    - *Input/Output:* Input from `Manual Run (Instant Test)`; outputs structured data items to `DevDeskApp: JSON Prettifier`.
    - *Edge Cases:* Syntax errors in custom JavaScript if modified; ensures numeric types are properly passed for downstream calculations.

#### 2.2 Data Cleaning & Security Processing
- **Overview:** Unpacks minified JSON into a clean, human-readable structure, creates a cryptographic hash of the formatted output, and generates a unique trace ID.
- **Nodes Involved:**
  - `DevDeskApp: JSON Prettifier`
  - `DevDeskApp: SHA-256 Digest`
  - `DevDeskApp: UUID Generator`
- **Node Details:**
  - **DevDeskApp: JSON Prettifier**
    - *Type & Role:* `n8n-nodes-devdeskapp.devDeskApp` (Utility node).
    - *Configuration:* Resource set to `json`, operation set to `prettify`, indentation set to `2`.
    - *Key Expressions:* `={{ $json.rawMinifiedApiData }}`
    - *Input/Output:* Input from `Sample Developer & HR Payload`; outputs formatted JSON text to `DevDeskApp: SHA-256 Digest`.
    - *Version Requirements:* Requires the `n8n-nodes-devdeskapp` community package installed.
    - *Edge Cases:* Fails if the input string is not valid JSON.
  - **DevDeskApp: SHA-256 Digest**
    - *Type & Role:* `n8n-nodes-devdeskapp.devDeskApp` (Crypto utility node).
    - *Configuration:* Resource set to `crypto`, operation set to `hash`, hash algorithm set to `sha256`.
    - *Key Expressions:* `={{ $json.formatted }}`
    - *Input/Output:* Input from `DevDeskApp: JSON Prettifier`; outputs the resulting hash string to `DevDeskApp: UUID Generator`.
    - *Edge Cases:* Expects a string payload; errors out if input is null or undefined.
  - **DevDeskApp: UUID Generator**
    - *Type & Role:* `n8n-nodes-devdeskapp.devDeskApp` (Identifier utility node).
    - *Configuration:* Resource set to `crypto`, operation set to `uuid`.
    - *Input/Output:* Input from `DevDeskApp: SHA-256 Digest`; outputs a generated UUID v4 object to `DevDeskApp: Slugifier`.
    - *Edge Cases:* Relies on secure random generation context within the local environment.

#### 2.3 Business Logic & HR Calculations
- **Overview:** Converts the candidate's full name into a URL-safe slug and computes a comprehensive compensation breakdown based on annual CTC and basic salary percentages.
- **Nodes Involved:**
  - `DevDeskApp: Slugifier`
  - `DevDeskApp: CTC Calculator`
- **Node Details:**
  - **DevDeskApp: Slugifier**
    - *Type & Role:* `n8n-nodes-devdeskapp.devDeskApp` (Text utility node).
    - *Configuration:* Resource set to `text`, operation set to `slugify`.
    - *Key Expressions:* `={{ $('Sample Developer & HR Payload').item.json.candidateFullName }}`
    - *Input/Output:* Input from `DevDeskApp: UUID Generator`; outputs the slugified text to `DevDeskApp: CTC Calculator`.
    - *Edge Cases:* Special characters or empty strings in candidate names may result in generic or empty slugs.
  - **DevDeskApp: CTC Calculator**
    - *Type & Role:* `n8n-nodes-devdeskapp.devDeskApp` (Financial calculator node).
    - *Configuration:* Resource set to `calculator`, operation set to `ctcToInHand`.
    - *Key Expressions:* 
      - Annual CTC: `={{ $('Sample Developer & HR Payload').item.json.annualCtc }}`
      - Basic Percentage: `={{ $('Sample Developer & HR Payload').item.json.basicPercentage }}`
    - *Input/Output:* Input from `DevDeskApp: Slugifier`; outputs salary breakdown metrics (monthly gross, take-home, PF, TDS) to `Consolidated Final Output`.
    - *Edge Cases:* Zero or negative values for annual CTC will result in division or calculation errors.

#### 2.4 Output Consolidation
- **Overview:** Aggregates data from preceding execution steps, parses the formatted JSON, formats financial values into localized strings, and builds the final consolidated payload.
- **Nodes Involved:**
  - `Consolidated Final Output`
- **Node Details:**
  - **Consolidated Final Output**
    - *Type & Role:* `n8n-nodes-base.code` (JavaScript execution node).
    - *Configuration:* Custom script utilizing node-linking expressions (`$('Node Name').first().json`) to pull data across the entire execution history and format it into a localized Indian Rupee (`en-IN`) reporting schema.
    - *Key Expressions:* Uses multiple data-referencing methods (e.g., `$('DevDeskApp: JSON Prettifier').first().json`, `.toLocaleString('en-IN')`).
    - *Input/Output:* Input from `DevDeskApp: CTC Calculator`; outputs the final aggregated JSON object.
    - *Edge Cases:* Missing node references will throw runtime errors if upstream nodes fail or are skipped.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Manual Run (Instant Test)` | `n8n-nodes-base.manualTrigger` | Workflow Entry Point | None | `Sample Developer & HR Payload` | ## 🚀 DevDeskApp All-in-One Suite<br>**How it works:**<br>This workflow demonstrates real-world developer utilities running 100% locally with zero external server dependencies.<br><br>### ⚡ Test with 1 Click:<br>Just click **'Test workflow'** at the bottom - no webhook or API setup required! All sample data is pre-configured. |
| `Sample Developer & HR Payload` | `n8n-nodes-base.code` | Mock Data Seeder | `Manual Run (Instant Test)` | `DevDeskApp: JSON Prettifier` | ## 🚀 DevDeskApp All-in-One Suite<br>**How it works:**<br>This workflow demonstrates real-world developer utilities running 100% locally with zero external server dependencies.<br><br>### ⚡ Test with 1 Click:<br>Just click **'Test workflow'** at the bottom - no webhook or API setup required! All sample data is pre-configured. |
| `DevDeskApp: JSON Prettifier` | `n8n-nodes-devdeskapp.devDeskApp` | JSON Formatter | `Sample Developer & HR Payload` | `DevDeskApp: SHA-256 Digest` | ## 🔐 Data Cleaning & Security<br>1. **Prettifier:** Unpacks minified JSON into clean human-readable trees.<br>2. **SHA-256 Hasher:** Generates cryptographic tamper-proof digest.<br>3. **UUID v4:** Stamps every payload with a unique RFC 4122 trace ID. |
| `DevDeskApp: SHA-256 Digest` | `n8n-nodes-devdeskapp.devDeskApp` | Cryptographic Hasher | `DevDeskApp: JSON Prettifier` | `DevDeskApp: UUID Generator` | ## 🔐 Data Cleaning & Security<br>1. **Prettifier:** Unpacks minified JSON into clean human-readable trees.<br>2. **SHA-256 Hasher:** Generates cryptographic tamper-proof digest.<br>3. **UUID v4:** Stamps every payload with a unique RFC 4122 trace ID. |
| `DevDeskApp: UUID Generator` | `n8n-nodes-devdeskapp.devDeskApp` | Trace ID Generator | `DevDeskApp: SHA-256 Digest` | `DevDeskApp: Slugifier` | ## 🔐 Data Cleaning & Security<br>1. **Prettifier:** Unpacks minified JSON into clean human-readable trees.<br>2. **SHA-256 Hasher:** Generates cryptographic tamper-proof digest.<br>3. **UUID v4:** Stamps every payload with a unique RFC 4122 trace ID. |
| `DevDeskApp: Slugifier` | `n8n-nodes-devdeskapp.devDeskApp` | Text Slugifier | `DevDeskApp: UUID Generator` | `DevDeskApp: CTC Calculator` | ## 📊 HR & Business Math<br>4. **Slugifier:** Cleans candidate name into URL-safe ID.<br>5. **CTC Calculator:** Computes monthly take-home, employee/employer PF, professional tax, and income tax breakdown. |
| `DevDeskApp: CTC Calculator` | `n8n-nodes-devdeskapp.devDeskApp` | Financial Calculator | `DevDeskApp: Slugifier` | `Consolidated Final Output` | ## 📊 HR & Business Math<br>4. **Slugifier:** Cleans candidate name into URL-safe ID.<br>5. **CTC Calculator:** Computes monthly take-home, employee/employer PF, professional tax, and income tax breakdown. |
| `Consolidated Final Output` | `n8n-nodes-base.code` | Payload Aggregator | `DevDeskApp: CTC Calculator` | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Install Prerequisites:** Ensure the `n8n-nodes-devdeskapp` community package is installed in your target n8n instance.
2. **Create Trigger Node:** 
   - Add a **Manual Run (Instant Test)** node (`n8n-nodes-base.manualTrigger`).
3. **Create Mock Data Node:** 
   - Add a **Code** node (`n8n-nodes-base.code`) named `Sample Developer & HR Payload`.
   - Set the execution mode to JavaScript and configure the return block with a minified API JSON string, candidate name (`Rahul Verma - Principal Cloud Architect`), annual CTC (`2400000`), and basic percentage (`50`).
   - Connect `Manual Run (Instant Test)` to `Sample Developer & HR Payload`.
4. **Create Prettifier Node:**
   - Add a **DevDeskApp** node (`n8n-nodes-devdeskapp.devDeskApp`) named `DevDeskApp: JSON Prettifier`.
   - Set Resource to `json`, Operation to `prettify`, and Indentation to `2`.
   - Set `jsonInput` parameter expression to `={{ $json.rawMinifiedApiData }}`.
   - Connect `Sample Developer & HR Payload` to this node.
5. **Create SHA-256 Hash Node:**
   - Add a **DevDeskApp** node named `DevDeskApp: SHA-256 Digest`.
   - Set Resource to `crypto`, Operation to `hash`, and Hash Algorithm to `sha256`.
   - Set `textPayload` parameter expression to `={{ $json.formatted }}`.
   - Connect `DevDeskApp: JSON Prettifier` to this node.
6. **Create UUID Generator Node:**
   - Add a **DevDeskApp** node named `DevDeskApp: UUID Generator`.
   - Set Resource to `crypto` and Operation to `uuid`.
   - Connect `DevDeskApp: SHA-256 Digest` to this node.
7. **Create Slugifier Node:**
   - Add a **DevDeskApp** node named `DevDeskApp: Slugifier`.
   - Set Resource to `text` and Operation to `slugify`.
   - Set `textInput` parameter expression to `={{ $('Sample Developer & HR Payload').item.json.candidateFullName }}`.
   - Connect `DevDeskApp: UUID Generator` to this node.
8. **Create CTC Calculator Node:**
   - Add a **DevDeskApp** node named `DevDeskApp: CTC Calculator`.
   - Set Resource to `calculator` and Operation to `ctcToInHand`.
   - Set `annualCtc` parameter expression to `={{ $('Sample Developer & HR Payload').item.json.annualCtc }}`.
   - Set `basicPercentage` parameter expression to `={{ $('Sample Developer & HR Payload').item.json.basicPercentage }}`.
   - Connect `DevDeskApp: Slugifier` to this node.
9. **Create Final Output Aggregator:**
   - Add a **Code** node named `Consolidated Final Output`.
   - Populate the code body to pull data from preceding nodes (`JSON Prettifier`, `SHA-256 Digest`, `UUID Generator`, `Slugifier`, `CTC Calculator`), parse structured items, map financial breakdowns, and apply localized formatting (`en-IN`).
   - Connect `DevDeskApp: CTC Calculator` to `Consolidated Final Output`.
10. **Add Sticky Notes (Optional):**
    - Place corresponding visual comment blocks around the workflow nodes to document sections for "How It Works", "Data Cleaning & Security", and "HR & Business Math".

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| DevDeskApp Official Website | [https://www.devdeskapp.com](https://www.devdeskapp.com) |
| DevDeskApp CTC Calculator Reference | [https://www.devdeskapp.com/ctc-to-in-hand-calculator](https://www.devdeskapp.com/ctc-to-in-hand-calculator) |
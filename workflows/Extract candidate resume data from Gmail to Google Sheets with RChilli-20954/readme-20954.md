Extract candidate resume data from Gmail to Google Sheets with RChilli

https://n8nworkflows.xyz/workflows/extract-candidate-resume-data-from-gmail-to-google-sheets-with-rchilli-20954


# Extract candidate resume data from Gmail to Google Sheets with RChilli

### 1. Workflow Overview

This workflow automates the candidate screening process by monitoring incoming emails in Gmail for resumes, extracting structured candidate details via the RChilli Model Context Protocol (MCP), and organizing the extracted data into a Google Sheets tracker. It handles various document formats, validates file constraints, processes entries in batches to manage rate limits, and maintains an exception log for invalid or corrupt submissions.

The workflow logic is categorized into four sequential blocks:
- **1.1 Input Reception & Schema Preparation:** Monitors incoming emails, extracts attachments, and reads the target Google Sheet's column headers to establish a dynamic mapping schema.
- **1.2 File Validation & Text Extraction:** Opens, verifies, unzips, and converts diverse document formats (PDF, Word, RTF, ODT, HTML, MHT, TXT, and image OCR) into clean text streams.
- **1.3 Batch Processing & Data Extraction:** Processes candidate files in batches of five, communicating with the RChilli MCP service to parse structured resume fields.
- **1.4 Data Mapping & Destination Routing:** Maps extracted attributes or error codes against the dynamic sheet schema, routing successful records to the candidate database and logging failures to an exception tracking tab.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Schema Preparation
- **Overview:** Triggers on incoming emails matching specified criteria, extracts attached files, and concurrently reads the destination Google Sheet headers to configure runtime data mapping.
- **Nodes Involved:** 
  - `1. Receive Resume Email`
  - `2. Read Sheet Column Headers`
- **Node Details:**
  - **1. Receive Resume Email**
    - *Type & Technical Role:* `n8n-nodes-base.gmailTrigger` (Trigger Node)
    - *Configuration Choices:* Polls every minute for emails containing specific file extensions in attachments, filtering for a specific label ID (`Label_888463215918477448`). Automatically downloads attachments with the prefix `attachment_`.
    - *Key Expressions or Variables:* Uses Gmail search query filtering: `has:attachment {filename:pdf filename:docx ...}`.
    - *Input / Output Connections:* Input: None (Trigger). Output: Connected to both `2. Read Sheet Column Headers` and `3. Check Resume Files & Unzip`.
    - *Edge Cases & Potential Failures:* OAuth token expiration or missing label permissions.
  - **2. Read Sheet Column Headers**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Action Node)
    - *Configuration Choices:* Configured to read row 1 of the target Google Sheet tab once per execution. Set to continue regular output on error (`onError: continueRegularOutput`) and retry up to 4 times with a 3-second delay.
    - *Input / Output Connections:* Input: Triggered by `1. Receive Resume Email`. Output: Passes sheet configuration data downstream (though functionally acts as a reference input for mapping nodes).
    - *Edge Cases & Potential Failures:* Google Sheets API rate limits, permission revocations, or missing headers like "Date Received".

#### 1.2 File Validation & Text Extraction
- **Overview:** Validates attachment sizes, handles ZIP archives securely, applies text extraction logic across multiple document types, and uses optical character recognition (OCR) for image-based resumes.
- **Nodes Involved:** 
  - `3. Check Resume Files & Unzip`
- **Node Details:**
  - **3. Check Resume Files & Unzip**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code Node)
    - *Configuration Choices:* Executes a custom Node.js script implementing RFC 1951 DEFLATE decoding, ZIP directory verification, robust text extraction parsers for PDF, DOC, DOCX, RTF, ODT, HTML, MHT, TXT, and optional RChilli OCR configuration for image files.
    - *Key Expressions or Variables:* Defines size constraints (`MAX_FILE_BYTES: 5 MiB`), ZIP unpack limits, and signature image filtering rules.
    - *Input / Output Connections:* Input: Receives items from `1. Receive Resume Email`. Output: Passes normalized resume text items to `4. Send 5 Resumes at a Time`.
    - *Edge Cases & Potential Failures:* ZIP bombs, password-protected files, empty documents, unsupported binaries, and extraction failures resulting in exception flags.

#### 1.3 Batch Processing & Data Extraction
- **Overview:** Groups parsed candidate items into manageable batches and invokes the RChilli MCP service to extract standardized candidate data fields.
- **Nodes Involved:** 
  - `4. Send 5 Resumes at a Time`
  - `5. Resume Data Extraction (RChilli MCP)`
- **Node Details:**
  - **4. Send 5 Resumes at a Time**
    - *Type & Technical Role:* `n8n-nodes-base.splitInBatches` (Flow Control Node)
    - *Configuration Choices:* Sets a batch size of `5`. Loops execution back from mapping steps until all batches are completed.
    - *Input / Output Connections:* Input: Receives items from `3. Check Resume Files & Unzip` or `6. Map Candidate Data to Sheet Columns`. Output: Branches output to `7. Candidate Data Extracted?` (when complete) or `5. Resume Data Extraction (RChilli MCP)` (per batch).
    - *Edge Cases & Potential Failures:* Infinite loops if batch indexing fails to increment correctly.
  - **5. Resume Data Extraction (RChilli MCP)**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.mcpClient` (AI / MCP Tool Integration Node)
    - *Configuration Choices:* Calls the `extract_resume_data` tool on the RChilli MCP endpoint with a 90-second timeout. Continues regular output on error.
    - *Key Expressions or Variables:* `={{ JSON.stringify({ resume_text: $json.status === 'ok' ? String($json.text || '') : '', filename: $json.filename }) }}`
    - *Input / Output Connections:* Input: Receives batches from `4. Send 5 Resumes at a Time`. Output: Connects to `6. Map Candidate Data to Sheet Columns`.
    - *Edge Cases & Potential Failures:* API timeouts, network disconnections, invalid RChilli credentials, or credit exhaustion.

#### 1.4 Data Mapping & Destination Routing
- **Overview:** Parses the RChilli response or error codes, maps data dynamically to the spreadsheet's column schema, and routes entries to either the primary database or exception logs.
- **Nodes Involved:** 
  - `6. Map Candidate Data to Sheet Columns`
  - `7. Candidate Data Extracted?`
  - `8a. Save Candidate Record to Google Sheets`
  - `8b. Log Failed Resume to Exception Log`
- **Node Details:**
  - **6. Map Candidate Data to Sheet Columns**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript Code Node)
    - *Configuration Choices:* Runs once for each incoming item. Parses RChilli error codes (1001–1061), normalizes column headers read from sheet row 1, extracts label-value text pairs, and builds a clean row payload.
    - *Input / Output Connections:* Input: Receives data from `5. Resume Data Extraction (RChilli MCP)`. Output: Loops back to `4. Send 5 Resumes at a Time` or flows forward to `7. Candidate Data Extracted?`.
    - *Edge Cases & Potential Failures:* Schema mismatches, missing Google Sheet header references, or unparseable JSON payloads.
  - **7. Candidate Data Extracted?**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Branching Node)
    - *Configuration Choices:* Evaluates whether the processed item contains a failure reason (`$json.Reason`).
    - *Key Expressions or Variables:* `={{ !$json.Reason }}`
    - *Input / Output Connections:* Input: Receives mapped items from `6. Map Candidate Data to Sheet Columns` (via the batch completion loop). Output: True branch connects to `8a. Save Candidate Record to Google Sheets`; False branch connects to `8b. Log Failed Resume to Exception Log`.
    - *Edge Cases & Potential Failures:* Incorrect evaluation criteria leading to misrouted records.
  - **8a. Save Candidate Record to Google Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Action Node)
    - *Configuration Choices:* Appends successful candidate rows to the sheet using dynamic column mapping. Configured with retry logic (up to 3 tries, 2-second interval).
    - *Input / Output Connections:* Input: True branch of `7. Candidate Data Extracted?`. Output: None (Terminal node for successful branch).
    - *Edge Cases & Potential Failures:* Sheet permission issues, API rate limiting, or structural constraint violations.
  - **8b. Log Failed Resume to Exception Log**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Action Node)
    - *Configuration Choices:* Appends rejected files and error reasons to the exception tracking tab. Retries up to 3 times on failure.
    - *Input / Output Connections:* Input: False branch of `7. Candidate Data Extracted?`. Output: None (Terminal node for failure branch).
    - *Edge Cases & Potential Failures:* Missing exception log tabs or incorrect column mappings.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1. Receive Resume Email | n8n-nodes-base.gmailTrigger | Checks the Resumes label every minute for emails with resume files or ZIPs and downloads the attachments. Passes the email to 3. Check Resume Files & Unzip (and to 2. Read Sheet Column Headers). | None (Trigger) | 2. Read Sheet Column Headers, 3. Check Resume Files & Unzip | 📬 Step 1: Get, check and read the resume files |
| 2. Read Sheet Column Headers | n8n-nodes-base.googleSheets | Reads row 1 (the column names) of the Candidate tab ONCE per run, so Google is not called once per resume. It sits above the main line, so it runs first. 5. Map Fields to Sheet Row fills exactly these columns: add a header to the sheet any time and the next run uses it. The header "Date Received" must exist (the read looks for it). | 1. Receive Resume Email | None | 📬 Step 1: Get, check and read the resume files |
| 3. Check Resume Files & Unzip | n8n-nodes-base.code | Opens every attachment and ZIP, checks each file (not empty, smaller than 5 MiB, a real resume type read from the file content) and READS ITS TEXT, so no other node is needed for PDF, Word, RTF, ODT, HTML, MHT or TXT. One item per resume with its text. Problem files are marked and logged with their reason, never parsed. Pictures (PNG, JPEG, JPG) are read with RChilli OCR here (set your userkey and subuserid in the PICTURE OCR settings at the top of the code). | 1. Receive Resume Email | 4. Send 5 Resumes at a Time | 📬 Step 1: Get, check and read the resume files |
| 4. Send 5 Resumes at a Time | n8n-nodes-base.splitInBatches | Batch processing: sends the resumes to RChilli 5 at a time. The next batch starts when the previous 5 are mapped (5. Parse > 6. Map loops back here). When all batches are done, everything moves on to 7. Candidate Data Extracted?. | 3. Check Resume Files & Unzip, 6. Map Candidate Data to Sheet Columns | 5. Resume Data Extraction (RChilli MCP), 7. Candidate Data Extracted? | 🤖 Step 2: Resume Data Extraction with RChilli MCP, 5 at a time |
| 5. Resume Data Extraction (RChilli MCP) | @n8n/n8n-nodes-langchain.mcpClient | RChilli MCP (extract_resume_data): sends the resume TEXT (never shortened, RChilli applies its own 60,000 character limit) and the file name, and gets the structured resume back, one resume at a time inside each batch of 5. Problem files send empty text and are logged by node 6 with their own reason. Errors are caught per resume and logged with the RChilli code. | 4. Send 5 Resumes at a Time | 6. Map Candidate Data to Sheet Columns | 🤖 Step 2: Resume Data Extraction with RChilli MCP, 5 at a time |
| 6. Map Candidate Data to Sheet Columns | n8n-nodes-base.code | Builds one sheet row, no AI. Fills ONLY the columns in row 1: standard columns first, then any header you add is matched to the stored RChilli field list (211 fields), then to a "Label: value" line in the resume text. Nothing found = empty cell. A rejected file becomes an exception row. Goes back to node 4 for the next batch. | 5. Resume Data Extraction (RChilli MCP) | 4. Send 5 Resumes at a Time | 🧩 Step 3: Map candidate data to your columns |
| 7. Candidate Data Extracted? | n8n-nodes-base.if | A good row goes up to 7a. Save to Candidate Data, a rejected file goes down to 7b. Log Rejected Resume. | 4. Send 5 Resumes at a Time | 8a. Save Candidate Record to Google Sheets, 8b. Log Failed Resume to Exception Log | 📊 Step 4: Save the candidate record (after all batches) |
| 8a. Save Candidate Record to Google Sheets | n8n-nodes-base.googleSheets | Adds the good resumes as new rows. The columns come from row 1 of the sheet itself (no stored column list), so a header you add is written automatically. The extra "_mapping_info" value is ignored here; it only shows in the execution view how each column was filled. | 7. Candidate Data Extracted? | None | 📊 Step 4: Save the candidate record (after all batches) |
| 8b. Log Failed Resume to Exception Log | n8n-nodes-base.googleSheets | Appends the rejected file to the Exception Logs tab (LLM Testing Logs) with sender, subject, file name and the plain-English Reason. | 7. Candidate Data Extracted? | None | 📊 Step 4: Save the candidate record (after all batches) |
| Sticky Note | n8n-nodes-base.stickyNote | ## 📋 What this workflow does<br>You email resumes to Gmail. This workflow reads them with the **RChilli MCP** and fills a Google Sheet for you. No AI.<br><br>1️⃣ **Get** the files from the email, unzip ZIPs, check every file and **read its text** (PDF, Word, RTF, ODT, TXT, HTML, MHT)<br>2️⃣ **Send 5 resumes at a time** to RChilli (it takes resume text)<br>3️⃣ **Map** the result to the columns in row 1 of your sheet. Add a header any time and it is filled when RChilli found that information<br>4️⃣ **Save** ✅ good resumes → **Candidate** tab · ⚠️ problem files → **Exception Logs** tab, with the reason<br><br>👉 First time? Follow the **SETUP** note on the right. | None | None | |
| Sticky Note2 | n8n-nodes-base.stickyNote | ## 🛠️ SETUP (do this once, about 5 minutes)<br><br>**A. Your Google Sheet** (one file, two tabs)<br>1. **Candidate Data** tab: row 1 = your column names. Required: **Date Received**. Suggested: Date Received, Full Name, Email, Phone, Location, Current Role, Employer, Years Of Experience, Skills, Institute Name, Qualification, Profile Link, Source, Resume Link, Status, Sender Email, Subject, File Name, Reason, Date Of Birth, Nationality.<br>2. **Exception Logs** tab: row 1 must be **Date Received, Sender Email, Subject, File Name, Reason**.<br>✅ **To add a new column later, just add a header in row 1.** It is filled by itself when RChilli found that information (about 210 RChilli fields are known, e.g. Gender, Passport Number, Total Experience In Months, Executive Summary, Languages Known, Category).<br><br>B. Gmail label – create a label **Resumes** and move resume emails there.<br><br>C. Credentials – 1. Receive Resume Email → your Gmail · 2. Read Sheet Column Headers, 8a, 8b → your Google account · **5. Resume Data Extraction (RChilli MCP) → your RChilli MCP credential** (RChilli MCP Authentication).<br><br>D. Publish – send 3 test files first (a PDF, a Word file and a ZIP), then turn the workflow on. Run one execution at a time. | None | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Gmail Trigger** node (`1. Receive Resume Email`).
   - Set filters to check the `Resumes` label (`Label_888463215918477448`) every minute, enabling attachment downloads with the prefix `attachment_`.
   - Configure a Gmail OAuth2 credential.

2. **Create the Schema Reader Node:**
   - Add a **Google Sheets** node (`2. Read Sheet Column Headers`).
   - Configure it to read row 1 of the Candidate tab once per run, setting error handling to continue regular output and enabling retries.
   - Configure a Google Sheets OAuth2 API credential.

3. **Create the File Parsing & Validation Code Node:**
   - Add a **Code** node (`3. Check Resume Files & Unzip`).
   - Paste the provided JavaScript parsing logic for handling ZIP archives, document text extraction (PDF, DOC, DOCX, RTF, ODT, HTML, MHT, TXT), and optional RChilli OCR settings (`USERKEY` and `SUBUSERID`).
   - Connect `1. Receive Resume Email` to this node.

4. **Create the Batch Processing Flow Control Node:**
   - Add a **Split In Batches** node (`4. Send 5 Resumes at a Time`) with a batch size of `5`.
   - Connect the output of `3. Check Resume Files & Unzip` to this node.

5. **Create the AI/MCP Extraction Node:**
   - Add an **MCP Client** node (`5. Resume Data Extraction (RChilli MCP)`).
   - Set the endpoint URL to `https://mcp.rchilli.ai/mcp`, choose the `extract_resume_data` tool, and set the input JSON expression to serialize resume text and filenames.
   - Configure required RChilli MCP credentials.
   - Connect the batch flow output to this node.

6. **Create the Mapping & Normalization Code Node:**
   - Add a **Code** node (`6. Map Candidate Data to Sheet Columns`).
   - Paste the provided mapping script that correlates RChilli error codes, standard schema fields, and resume text label-value pairs.
   - Connect `5. Resume Data Extraction (RChilli MCP)` to this node, and route its output loop back to `4. Send 5 Resumes at a Time`.

7. **Create the Conditional Routing Node:**
   - Add an **If** node (`7. Candidate Data Extracted?`).
   - Set condition expression to check that `$json.Reason` is empty (`={{ !$json.Reason }}`).
   - Connect the batch completion output from `4. Send 5 Resumes at a Time` to this node.

8. **Create the Destination Sheets Nodes:**
   - Add two **Google Sheets** action nodes (`8a. Save Candidate Record to Google Sheets` and `8b. Log Failed Resume to Exception Log`).
   - Configure both to append rows using dynamic sheet headers, applying retry configurations on failure.
   - Connect the True branch of `7. Candidate Data Extracted?` to `8a`, and the False branch to `8b`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| RChilli MCP Documentation & Setup Guides | [RChilli Documentation](https://docs.rchilli.com) |
| RChilli MCP Endpoint | `https://mcp.rchilli.ai/mcp` |
Send 80G receipts and CSR proposals with Razorpay, GPT-4o-mini, and Google

https://n8nworkflows.xyz/workflows/send-80g-receipts-and-csr-proposals-with-razorpay--gpt-4o-mini--and-google-20879


# Send 80G receipts and CSR proposals with Razorpay, GPT-4o-mini, and Google

### 1. Workflow Overview

This workflow is an end-to-end automation system designed for Indian non-profit organizations (NGOs) to process digital donations, manage compliance requirements (such as 80G tax exemption certificates and Schedule VII CSR frameworks), generate AI-powered communications, and manage quarterly stakeholder reporting and corporate partnership leads. 

The system logic is divided into three primary functional flows:
1. **Donation Ingestion & 80G Receipt Processing:** Receives and validates Razorpay webhooks, verifies transactions against external APIs, prevents duplicate ledger entries, calculates Indian financial year (FY) sequential receipt numbers and donor lifetime tiers, generates a formatted HTML 80G tax receipt converted to PDF via Gotenberg, distributes personalized thank-you messages via Gmail and WhatsApp, updates central ledgers in Google Sheets, and triggers Slack alerts for significant gifts or tier upgrades.
2. **Quarterly Impact Reporting:** Executes on a cron-based schedule every quarter to aggregate financial contributions and program impact metrics from Google Sheets, uses OpenAI (`gpt-4o-mini`) to compile narrative impact stories, dynamically segments donors by tier, and generates reviewable Gmail draft campaigns alongside Slack notifications.
3. **Corporate Social Responsibility (CSR) Proposal Generation:** Processes form submissions from prospective corporate partners, matches brief requirements against an internal program catalogue using a multi-variable scoring model, drafts a tailored compliance-ready CSR proposal using OpenAI, converts the layout into a PDF, archives files to Google Drive, sets up Gmail draft attachments, updates the internal CSR pipeline ledger, and announces new leads via Slack.

---

### 2. Block-by-Block Analysis

#### Block 1: Donation Ingestion & Verification
- **Overview:** Receives raw payment webhook notifications, isolates captured transactions, performs backend API verification against Razorpay, and cross-references historical ledger records to calculate sequential financial year receipt numbering and donor lifetime classifications.
- **Nodes Involved:** `Razorpay Webhook`, `Parse Payment Event`, `Is Captured Payment?`, `Verify with Razorpay API`, `Read Donations Ledger`, `Validate, Dedupe & Tier`, `Route by Donation Status`

- **Node Details:**
  - **Razorpay Webhook**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Trigger)
    - *Configuration Choices:* Listens for incoming HTTP POST requests on path `razorpay-donation`.
    - *Input/Output Connections:* Output connects to `Parse Payment Event`.
    - *Edge Cases/Failures:* Signature verification failure or network interruption from Razorpay dispatchers.
  - **Parse Payment Event**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Transformation)
    - *Configuration Choices:* Extracts payment entity attributes, normalizes contact strings, email formats, PAN identifiers, and metadata notes from raw JSON payloads.
    - *Input/Output Connections:* Input from `Razorpay Webhook`; Output to `Is Captured Payment?`.
  - **Is Captured Payment?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router)
    - *Configuration Choices:* Evaluates whether `event` equals `payment.captured` and checks that `payment_id` is not empty.
    - *Input/Output Connections:* Input from `Parse Payment Event`; Output to `Verify with Razorpay API`.
  - **Verify with Razorpay API**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (API Integration)
    - *Configuration Choices:* Executes a GET request to `https://api.razorpay.com/v1/payments/{{ $json.payment_id }}` using HTTP Basic Authentication (`httpBasicAuth`). Configured with `neverError: true` to catch non-200 responses gracefully.
    - *Input/Output Connections:* Input from `Is Captured Payment?`; Output to `Read Donations Ledger`.
    - *Edge Cases/Failures:* API timeouts, rate limiting, or invalid credentials resulting in HTTP error codes.
  - **Read Donations Ledger**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Retrieval)
    - *Configuration Choices:* Reads rows from the `Donations` sheet using document ID `REPLACE_WITH_GOOGLE_SHEET_ID`. Executes once (`executeOnce: true`).
    - *Input/Output Connections:* Input from `Verify with Razorpay API`; Output to `Validate, Dedupe & Tier`.
  - **Validate, Dedupe & Tier**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Business Logic & Calculations)
    - *Configuration Choices:* Defines NGO profile constants (80G, 12A, CSR-1, PAN, signatories), cross-checks the incoming payment ID against historical entries to flag duplicates, validates financial-year calculation rules (running April to March), assigns sequential numbering (`80G/YYYY-YY/000000`), aggregates lifetime totals, assigns donor tiers (*Patron*, *Champion*, *Supporter*, *Friend*), and converts numerical amounts to Indian currency words.
    - *Input/Output Connections:* Input from `Read Donations Ledger` and previous nodes; Output to `Route by Donation Status`.
  - **Route by Donation Status**
    - *Type and Technical Role:* `n8n-nodes-base.switch` (Conditional Router)
    - *Configuration Choices:* Splits flow based on the `status` string property (`new` vs `mismatch`).
    - *Input/Output Connections:* Input from `Validate, Dedupe & Tier`; Output 0 (`New`) connects to `AI Write Thank-you Letter`, Output 1 (`Mismatch`) connects to `Alert Slack: Payment Mismatch`.

---

#### Block 2: 80G Receipt PDF Generation & Exception Handling
- **Overview:** Handles valid donations by prompting an AI model to draft personalized acknowledgements and compiles a structured HTML document that is subsequently converted into a downloadable PDF via Gotenberg. Mismatched transactions are routed directly to Slack.
- **Nodes Involved:** `AI Write Thank-you Letter`, `Alert Slack: Payment Mismatch`, `Build 80G Receipt HTML`, `Receipt HTML to File`, `Render Receipt PDF`

- **Node Details:**
  - **AI Write Thank-you Letter**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (AI Text Generation)
    - *Configuration Choices:* Uses the `gpt-4o-mini` model with a temperature setting of `0.6`. Configured with a system prompt enforcing JSON-only output containing email subjects, HTML paragraphs, WhatsApp text, and PAN requests.
    - *Input/Output Connections:* Input from `Route by Donation Status`; Output to `Build 80G Receipt HTML`.
    - *Edge Cases/Failures:* LLM quota exhaustion, malformed JSON responses, or rate limits.
  - **Alert Slack: Payment Mismatch**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Messaging Notification)
    - *Configuration Choices:* Posts warning alerts containing mismatch reasons, payment IDs, and amounts to the designated channel ID (`REPLACE_WITH_SLACK_CHANNEL_ID`).
    - *Input/Output Connections:* Input from `Route by Donation Status`.
  - **Build 80G Receipt HTML**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Template Rendering)
    - *Configuration Choices:* Generates a compliant A4-formatted HTML string containing organization registration details, sequential receipt numbers, donor particulars, currency breakdowns, and statutory disclaimers.
    - *Input/Output Connections:* Input from `AI Write Thank-you Letter`; Output to `Receipt HTML to File`.
  - **Receipt HTML to File**
    - *Type and Technical Role:* `n8n-nodes-base.convertToFile` (Binary Transformation)
    - *Configuration Choices:* Converts the HTML string payload into a text file asset named `index.html`.
    - *Input/Output Connections:* Input from `Build 80G Receipt HTML`; Output to `Render Receipt PDF`.
  - **Render Receipt PDF**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (PDF Conversion Service)
    - *Configuration Choices:* Sends a multipart form-data POST request to a local Gotenberg instance (`http://gotenberg:3000/forms/chromium/convert/html`) with configured paper dimensions (8.27 × 11.7 inches) and background printing enabled. Configured with a 60-second timeout and up to 2 retries on failure.
    - *Input/Output Connections:* Input from `Receipt HTML to File`; Outputs connect to `Archive Receipt to Drive` and `Email Thank-you & 80G Receipt`.
    - *Edge Cases/Failures:* Gotenberg service unavailability or rendering timeouts.

---

#### Block 3: Delivery, Ledger Logging, and Tier Escalations
- **Overview:** Archives the generated PDF to Google Drive, distributes communications via Gmail and WhatsApp, updates accounting and CRM ledgers in Google Sheets, and sends internal team alerts on Slack when milestone tier changes or major gifts occur.
- **Nodes Involved:** `Archive Receipt to Drive`, `Email Thank-you & 80G Receipt`, `Send WhatsApp Thank-you`, `Append Donations Ledger`, `Upsert Donor Profile`, `Tier Upgrade or Major Gift?`, `Alert Fundraising Team on Slack`

- **Node Details:**
  - **Archive Receipt to Drive**
    - *Type and Technical Role:* `n8n-nodes-base.googleDrive` (Cloud Storage)
    - *Configuration Choices:* Uploads the binary file to the specified folder ID (`REPLACE_WITH_RECEIPTS_FOLDER_ID`) under My Drive.
    - *Input/Output Connections:* Input from `Render Receipt PDF`.
  - **Email Thank-you & 80G Receipt**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Email Dispatch)
    - *Configuration Choices:* Composes an HTML message incorporating AI-generated copy, PAN requests if applicable, and attaches the generated 80G receipt binary file.
    - *Input/Output Connections:* Input from `Render Receipt PDF`.
  - **Send WhatsApp Thank-you**
    - *Type and Technical Role:* `n8n-nodes-base.whatsApp` (Messaging Integration)
    - *Configuration Choices:* Sends outbound text messages using a configured phone number ID (`REPLACE_WITH_WHATSAPP_PHONE_NUMBER_ID`). Configured with `onError: continueRegularOutput` to prevent message failures from halting ledger updates.
    - *Input/Output Connections:* Input from `Email Thank-you & 80G Receipt`; Output to `Append Donations Ledger`.
  - **Append Donations Ledger**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Ledger Logging)
    - *Configuration Choices:* Appends a structured row containing transaction metadata, financial year, receipt links, payment IDs, and PAN verification statuses into the `Donations` sheet.
    - *Input/Output Connections:* Input from `Send WhatsApp Thank-you`; Output to `Upsert Donor Profile`.
  - **Upsert Donor Profile**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (CRM Upsert)
    - *Configuration Choices:* Matches rows on `donor_key` (email/phone) within the `Donors` sheet, appending new profiles or updating lifetime totals, donation counts, and tier classifications.
    - *Input/Output Connections:* Input from `Append Donations Ledger`; Output to `Tier Upgrade or Major Gift?`.
  - **Tier Upgrade or Major Gift?**
    - *Type and Technical Role:* `n8n-nodes-base.if` (Conditional Router)
    - *Configuration Choices:* Evaluates whether `tier_upgraded` is true or `major_gift` is true.
    - *Input/Output Connections:* Input from `Upsert Donor Profile`; Output connects to `Alert Fundraising Team on Slack`.
  - **Alert Fundraising Team on Slack**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Internal Notification)
    - *Configuration Choices:* Dispatches formatted messages highlighting upgraded donor tiers or major gifts, including lifetime giving totals and receipt references.
    - *Input/Output Connections:* Input from `Tier Upgrade or Major Gift?`.

---

#### Block 4: Quarterly Impact Updates
- **Overview:** Executes on a scheduled cron trigger every quarter to aggregate financial contributions and program indicators, generates a contextual impact story via OpenAI, compiles segmented email drafts by donor tier, and notifies the communications team on Slack.
- **Nodes Involved:** `Every Quarter (1 Jan, Apr, Jul, Oct)`, `Set Quarter Window`, `Read Donations (Quarter)`, `Read Impact Metrics`, `Aggregate Quarter`, `AI Write Impact Story`, `Build Tier Drafts`, `Create Gmail Draft per Tier`, `Notify Slack: Drafts Ready`

- **Node Details:**
  - **Every Quarter (1 Jan, Apr, Jul, Oct)**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Cron Trigger)
    - *Configuration Choices:* Uses cron expression `0 10 1 1,4,7,10 *` (runs at 10:00 AM on the first day of January, April, July, and October).
    - *Input/Output Connections:* Output connects to `Set Quarter Window`.
  - **Set Quarter Window**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Date Utility)
    - *Configuration Choices:* Calculates date boundaries for the previous completed quarter and assigns financial-year quarter labels (e.g., `Q1 FY2026-27`).
    - *Input/Output Connections:* Input from `Every Quarter (1 Jan, Apr, Jul, Oct)`; Output to `Read Donations (Quarter)`.
  - **Read Donations (Quarter)**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Retrieval)
    - *Configuration Choices:* Reads all rows from the `Donations` sheet (`executeOnce: true`).
    - *Input/Output Connections:* Input from `Set Quarter Window`; Output to `Read Impact Metrics`.
  - **Read Impact Metrics**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Retrieval)
    - *Configuration Choices:* Reads all rows from the `Impact Metrics` sheet (`executeOnce: true`).
    - *Input/Output Connections:* Input from `Read Donations (Quarter)`; Output to `Aggregate Quarter`.
  - **Aggregate Quarter**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Data Aggregation)
    - *Configuration Choices:* Filters donations matching the quarter window, computes total raised, gift counts, recurring totals, campaign distributions, and groups email audiences by donor tier.
    - *Input/Output Connections:* Input from `Read Impact Metrics`; Output to `AI Write Impact Story`.
  - **AI Write Impact Story**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (AI Text Generation)
    - *Configuration Choices:* Uses `gpt-4o-mini` with a temperature of `0.5` to generate structured JSON containing email subject lines, headlines, body HTML sections, and tier-specific notes.
    - *Input/Output Connections:* Input from `Aggregate Quarter`; Output to `Build Tier Drafts`.
  - **Build Tier Drafts**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Payload Construction)
    - *Configuration Choices:* Maps aggregated tier audiences into distinct batch segments (BCC capped at 450 recipients per draft) and assembles customized HTML payloads.
    - *Input/Output Connections:* Input from `AI Write Impact Story`; Output to `Create Gmail Draft per Tier`.
  - **Create Gmail Draft per Tier**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Draft Creation)
    - *Configuration Choices:* Sets resource type to `draft`, using HTML email formatting with dynamic BCC recipient arrays.
    - *Input/Output Connections:* Input from `Build Tier Drafts`; Output to `Notify Slack: Drafts Ready`.
  - **Notify Slack: Drafts Ready**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Internal Notification)
    - *Configuration Choices:* Posts summary metrics and tier draft counts to the Slack channel (`executeOnce: true`).
    - *Input/Output Connections:* Input from `Create Gmail Draft per Tier`.

---

#### Block 5: CSR Partnership Lead Processing & Proposal Generation
- **Overview:** Receives form submissions from corporate prospects, calculates program matches based on geographical, budgetary, and thematic alignment, generates a compliance-ready CSR proposal via OpenAI, renders it to PDF, archives files, creates Gmail drafts, logs the lead in a CRM pipeline, and alerts the team on Slack.
- **Nodes Involved:** `CSR Enquiry Form`, `Read Program Catalogue`, `Match Programs to CSR Brief`, `AI Write CSR Proposal`, `Build CSR Proposal HTML`, `CSR HTML to File`, `Render CSR Proposal PDF`, `Archive CSR Proposal to Drive`, `Create Proposal Gmail Draft`, `Append to CSR Pipeline`, `Alert Slack: New CSR Lead`

- **Node Details:**
  - **CSR Enquiry Form**
    - *Type and Technical Role:* `n8n-nodes-base.formTrigger` (Form Trigger)
    - *Configuration Choices:* Configures a web form capturing company names, contact designations, email addresses, budget bands, Schedule VII focus areas, and geographic preferences.
    - *Input/Output Connections:* Output connects to `Read Program Catalogue`.
  - **Read Program Catalogue**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Data Retrieval)
    - *Configuration Choices:* Reads program records from the `Programs` sheet (`executeOnce: true`).
    - *Input/Output Connections:* Input from `CSR Enquiry Form`; Output to `Match Programs to CSR Brief`.
  - **Match Programs to CSR Brief**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Scoring Algorithm)
    - *Configuration Choices:* Scores catalogue entries against prospect criteria (focus areas, state coverage, budget constraints), picks top matching programs, and generates a unique lead identifier string (`CSR-YYMMDD-XXXX`).
    - *Input/Output Connections:* Input from `Read Program Catalogue`; Output to `AI Write CSR Proposal`.
  - **AI Write CSR Proposal**
    - *Type and Technical Role:* `@n8n/n8n-nodes-langchain.openAi` (AI Text Generation)
    - *Configuration Choices:* Uses `gpt-4o-mini` with a temperature of `0.4` to generate structured JSON detailing executive summaries, Schedule VII alignments, budget allocations, KPIs, and M&E frameworks.
    - *Input/Output Connections:* Input from `Match Programs to CSR Brief`; Output to `Build CSR Proposal HTML`.
  - **Build CSR Proposal HTML**
    - *Type and Technical Role:* `n8n-nodes-base.code` (Template Rendering)
    - *Configuration Choices:* Renders a formal A4-compliant HTML proposal document featuring custom styling, cover blocks, budget tables, and compliance credentials.
    - *Input/Output Connections:* Input from `AI Write CSR Proposal`; Output to `CSR HTML to File`.
  - **CSR HTML to File**
    - *Type and Technical Role:* `n8n-nodes-base.convertToFile` (Binary Transformation)
    - *Configuration Choices:* Converts the proposal HTML string into an `index.html` binary file.
    - *Input/Output Connections:* Input from `Build CSR Proposal HTML`; Output to `Render CSR Proposal PDF`.
  - **Render CSR Proposal PDF**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (PDF Conversion Service)
    - *Configuration Choices:* Sends a POST request to Gotenberg (`http://gotenberg:3000/forms/chromium/convert/html`) with a 60-second timeout and 2 retries on failure.
    - *Input/Output Connections:* Input from `CSR HTML to File`; Outputs connect to `Archive CSR Proposal to Drive` and `Create Proposal Gmail Draft`.
  - **Archive CSR Proposal to Drive**
    - *Type and Technical Role:* `n8n-nodes-base.googleDrive` (Cloud Storage)
    - *Configuration Choices:* Uploads the rendered PDF file to the specified CSR folder ID (`REPLACE_WITH_CSR_FOLDER_ID`).
    - *Input/Output Connections:* Input from `Render CSR Proposal PDF`.
  - **Create Proposal Gmail Draft**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Draft Creation)
    - *Configuration Choices:* Creates a Gmail draft addressed to the prospect containing cover email copy and attaching the binary PDF proposal.
    - *Input/Output Connections:* Input from `Render CSR Proposal PDF`; Output to `Append to CSR Pipeline`.
  - **Append to CSR Pipeline**
    - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (CRM Logging)
    - *Configuration Choices:* Appends the lead record, budget bands, proposal links, and status stage (`Proposal Drafted`) to the `CSR Pipeline` sheet.
    - *Input/Output Connections:* Input from `Create Proposal Gmail Draft`; Output to `Alert Slack: New CSR Lead`.
  - **Alert Slack: New CSR Lead**
    - *Type and Technical Role:* `n8n-nodes-base.slack` (Internal Notification)
    - *Configuration Choices:* Posts lead summaries, fit ratings, matched programs, and proposal links to the designated Slack channel ID (`REPLACE_WITH_SLACK_CHANNEL_ID`).
    - *Input/Output Connections:* Input from `Append to CSR Pipeline`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note – Overview | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 💛 NGO-OS – Donations, 80G Receipts, Impact Updates & CSR<br><br>For Indian NGOs that still issue 80G receipts by hand from a Razorpay export. Receipts go out days late, numbering slips, PAN goes missing at Form 10BD time, and nobody notices when a regular donor becomes a major one.<br><br>### How it works<br>Each captured Razorpay payment is verified against the Razorpay API and deduplicated, then given a sequential financial-year receipt number (`80G/2026-27/000123`). The donor gets an AI-personalised thank-you with an 80G PDF receipt by email and a WhatsApp note. Their lifetime giving sets a tier (Friend, Supporter, Champion, Patron), and the team hears about upgrades and major gifts.<br><br>Every quarter, giving and program metrics become an AI impact story, with one Gmail draft per donor tier for review. CSR enquiries from a web form are matched to your programs and turned into a tailored proposal PDF.<br><br>### Setup steps<br>1. In Razorpay, send `payment.captured` webhooks to **Razorpay Webhook**, and add your key ID and secret as Basic Auth. Collect `name`, `pan`, `address` and `campaign` as payment notes.<br>2. Run Gotenberg (`docker run -p 3000:3000 gotenberg/gotenberg:8`) where n8n can reach it.<br>3. Connect Google Sheets, Drive, Gmail, Slack, WhatsApp and OpenAI.<br>4. Fill in the NGO profile (80G, 12A, CSR-1, PAN) in **Validate, Dedupe & Tier** and **Match Programs to CSR Brief**.<br>5. Create the **Impact Metrics** and **Programs** sheets. Donations, Donors and CSR Pipeline fill themselves. |
| Sticky Note – Verify & Dedupe Donation | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## ✅ Verify & Dedupe Donation<br>Only `payment.captured` events go through. Each payment is re-checked with the Razorpay API, then matched against the ledger by payment ID, so webhook retries never create a second receipt. Lifetime giving, tier and the next FY receipt number are worked out here. |
| Sticky Note – 80G Receipt PDF | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 🧾 80G Receipt PDF<br>GPT-4o-mini writes a short thank-you that mentions what the gift pays for. The receipt includes 80G and 12A details, the amount in words, and a request for PAN if it's missing. Gotenberg renders the PDF. Mismatched payments go to Slack instead. |
| Sticky Note – Send, Log & Tier Alert | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 📬 Send, Log & Tier Alert<br>The donor gets the PDF by email and a WhatsApp thank-you. The ledger and donor profile are updated, and the team is pinged on Slack when someone moves up a tier or makes a major gift. |
| Sticky Note – Quarter Data | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 📅 Quarter Data<br>On 1 January, April, July and October, last quarter's donations and program metrics are loaded and totalled by tier and program. |
| Sticky Note – Impact Story & Tier Drafts | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## ✍️ Impact Story & Tier Drafts<br>The AI turns the numbers into an impact story. One Gmail draft is created per donor tier, BCC'd to that tier's donors, with a more personal letter for Patrons. Nothing is sent until someone reviews it. |
| Sticky Note – CSR Matching & Proposal | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 🏢 CSR Matching & Proposal<br>A company's CSR brief is matched against your program catalogue by Schedule VII area, states and budget. The AI writes a full proposal covering alignment, budget, KPIs, monitoring and compliance, and Gotenberg renders it to PDF. |
| Sticky Note – Deliver CSR Proposal | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 📤 Deliver CSR Proposal<br>The proposal is archived to Drive and saved as a Gmail draft for your partnerships lead to review. The lead is logged to the CSR pipeline and announced on Slack. |
| Sticky Note – Credentials & Security | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## 🔐 Credentials & Security<br>Store the Razorpay key ID and secret as Basic Auth, never in node fields. Donor PAN and addresses live in your sheets and receipts, so keep the sheets and Drive folder private. Every payment is re-verified with Razorpay before a receipt is issued. |
| Razorpay Webhook | `n8n-nodes-base.webhook` | Webhook Trigger | None | Parse Payment Event | |
| Parse Payment Event | `n8n-nodes-base.code` | Data Transformation | Razorpay Webhook | Is Captured Payment? | |
| Is Captured Payment? | `n8n-nodes-base.if` | Conditional Router | Parse Payment Event | Verify with Razorpay API | |
| Verify with Razorpay API | `n8n-nodes-base.httpRequest` | API Integration | Is Captured Payment? | Read Donations Ledger | |
| Read Donations Ledger | `n8n-nodes-base.googleSheets` | Data Retrieval | Verify with Razorpay API | Validate, Dedupe & Tier | |
| Validate, Dedupe & Tier | `n8n-nodes-base.code` | Business Logic | Read Donations Ledger | Route by Donation Status | |
| Route by Donation Status | `n8n-nodes-base.switch` | Conditional Router | Validate, Dedupe & Tier | AI Write Thank-you Letter, Alert Slack: Payment Mismatch | |
| AI Write Thank-you Letter | `@n8n/n8n-nodes-langchain.openAi` | AI Generation | Route by Donation Status | Build 80G Receipt HTML | |
| Alert Slack: Payment Mismatch | `n8n-nodes-base.slack` | Notification | Route by Donation Status | None | |
| Build 80G Receipt HTML | `n8n-nodes-base.code` | Template Rendering | AI Write Thank-you Letter | Receipt HTML to File | |
| Receipt HTML to File | `n8n-nodes-base.convertToFile` | Binary Transformation | Build 80G Receipt HTML | Render Receipt PDF | |
| Render Receipt PDF | `n8n-nodes-base.httpRequest` | PDF Conversion | Receipt HTML to File | Archive Receipt to Drive, Email Thank-you & 80G Receipt | |
| Archive Receipt to Drive | `n8n-nodes-base.googleDrive` | Cloud Storage | Render Receipt PDF | None | |
| Email Thank-you & 80G Receipt | `n8n-nodes-base.gmail` | Email Dispatch | Render Receipt PDF | Send WhatsApp Thank-you | |
| Send WhatsApp Thank-you | `n8n-nodes-base.whatsApp` | Messaging Integration | Email Thank-you & 80G Receipt | Append Donations Ledger | |
| Append Donations Ledger | `n8n-nodes-base.googleSheets` | Ledger Logging | Send WhatsApp Thank-you | Upsert Donor Profile | |
| Upsert Donor Profile | `n8n-nodes-base.googleSheets` | CRM Upsert | Append Donations Ledger | Tier Upgrade or Major Gift? | |
| Tier Upgrade or Major Gift? | `n8n-nodes-base.if` | Conditional Router | Upsert Donor Profile | Alert Fundraising Team on Slack | |
| Alert Fundraising Team on Slack | `n8n-nodes-base.slack` | Internal Notification | Tier Upgrade or Major Gift? | None | |
| Every Quarter (1 Jan, Apr, Jul, Oct) | `n8n-nodes-base.scheduleTrigger` | Cron Trigger | None | Set Quarter Window | |
| Set Quarter Window | `n8n-nodes-base.code` | Date Utility | Every Quarter (1 Jan, Apr, Jul, Oct) | Read Donations (Quarter) | |
| Read Donations (Quarter) | `n8n-nodes-base.googleSheets` | Data Retrieval | Set Quarter Window | Read Impact Metrics | |
| Read Impact Metrics | `n8n-nodes-base.googleSheets` | Data Retrieval | Read Donations (Quarter) | Aggregate Quarter | |
| Aggregate Quarter | `n8n-nodes-base.code` | Data Aggregation | Read Impact Metrics | AI Write Impact Story | |
| AI Write Impact Story | `@n8n/n8n-nodes-langchain.openAi` | AI Generation | Aggregate Quarter | Build Tier Drafts | |
| Build Tier Drafts | `n8n-nodes-base.code` | Payload Construction | AI Write Impact Story | Create Gmail Draft per Tier | |
| Create Gmail Draft per Tier | `n8n-nodes-base.gmail` | Draft Creation | Build Tier Drafts | Notify Slack: Drafts Ready | |
| Notify Slack: Drafts Ready | `n8n-nodes-base.slack` | Internal Notification | Create Gmail Draft per Tier | None | |
| CSR Enquiry Form | `n8n-nodes-base.formTrigger` | Web Form Trigger | None | Read Program Catalogue | |
| Read Program Catalogue | `n8n-nodes-base.googleSheets` | Data Retrieval | CSR Enquiry Form | Match Programs to CSR Brief | |
| Match Programs to CSR Brief | `n8n-nodes-base.code` | Scoring Algorithm | Read Program Catalogue | AI Write CSR Proposal | |
| AI Write CSR Proposal | `@n8n/n8n-nodes-langchain.openAi` | AI Generation | Match Programs to CSR Brief | Build CSR Proposal HTML | |
| Build CSR Proposal HTML | `n8n-nodes-base.code` | Template Rendering | AI Write CSR Proposal | CSR HTML to File | |
| CSR HTML to File | `n8n-nodes-base.convertToFile` | Binary Transformation | Build CSR Proposal HTML | Render CSR Proposal PDF | |
| Render CSR Proposal PDF | `n8n-nodes-base.httpRequest` | PDF Conversion | CSR HTML to File | Archive CSR Proposal to Drive, Create Proposal Gmail Draft | |
| Archive CSR Proposal to Drive | `n8n-nodes-base.googleDrive` | Cloud Storage | Render Receipt PDF / Render CSR Proposal PDF | None | |
| Create Proposal Gmail Draft | `n8n-nodes-base.gmail` | Draft Creation | Render CSR Proposal PDF | Append to CSR Pipeline | |
| Append to CSR Pipeline | `n8n-nodes-base.googleSheets` | CRM Logging | Create Proposal Gmail Draft | Alert Slack: New CSR Lead | |
| Alert Slack: New CSR Lead | `n8n-nodes-base.slack` | Internal Notification | Append to CSR Pipeline | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow this step-by-step procedure to rebuild the entire automation system manually in n8n.

#### Step 1: Initialize Google Sheets Document & Structure
1. Create a new Google Spreadsheet and share it with your Google service account/credentials.
2. Create five separate sheets within the document with the following exact names and column headers:
   - **Donations**: `payment_id`, `order_id`, `receipt_no`, `fy`, `name`, `email`, `phone`, `pan`, `address`, `amount`, `currency`, `method`, `campaign`, `donor_type`, `tier`, `recurring`, `donated_at`, `receipt_link`, `pan_status`, `form_10bd`, `receipted_at`
   - **Donors**: `donor_key`, `name`, `email`, `phone`, `pan`, `tier`, `lifetime_total`, `donation_count`, `recurring`, `last_gift`, `last_campaign`, `last_donated_at`
   - **Impact Metrics**: `program`, `quarter`, `metric`, `value`
   - **Programs**: `program`, `schedule_vii`, `states`, `min_budget`, `cost_per_beneficiary`, `focus_areas`, `outcomes`
   - **CSR Pipeline**: `lead_id`, `company`, `contact`, `designation`, `email`, `phone`, `budget_band`, `focus_areas`, `geography`, `timeline`, `fit`, `programs`, `proposed_budget`, `proposal_link`, `stage`, `created_at`

#### Step 2: Set Up Gotenberg PDF Engine
1. Deploy a local or remote Gotenberg instance accessible to n8n (e.g., via Docker: `docker run -p 3000:3000 gotenberg/gotenberg:8`).

#### Step 3: Configure Credentials in n8n
Set up and verify the following credentials in your n8n instance:
- **Razorpay**: HTTP Basic Auth (Key ID as Username, Secret as Password).
- **Google Sheets / Drive**: OAuth2 or Service Account credentials.
- **Gmail**: OAuth2 credentials.
- **Slack**: Slack API OAuth2 or Bot Token credentials.
- **WhatsApp**: WhatsApp Business Cloud API credentials.
- **OpenAI**: OpenAI API credential (`gpt-4o-mini`).

#### Step 4: Build Donation Ingestion Flow
1. **Webhook Node**: Create node named `Razorpay Webhook`, set type to `Webhook`, method to `POST`, and path to `razorpay-donation`.
2. **Code Node**: Create `Parse Payment Event` to extract payload entities and normalize contact data, PAN, email, and amounts (dividing by 100).
3. **If Node**: Create `Is Captured Payment?` checking that `event` equals `payment.captured` and `payment_id` is not empty.
4. **HTTP Request Node**: Create `Verify with Razorpay API`, set URL to `https://api.razorpay.com/v1/payments/{{ $json.payment_id }}`, set Authentication to Generic Credential Type (`httpBasicAuth`), and enable `neverError: true`.
5. **Google Sheets Node**: Create `Read Donations Ledger`, select operation `Get Many`, sheet name `Donations`, document ID `REPLACE_WITH_GOOGLE_SHEET_ID`, and enable `executeOnce: true`.
6. **Code Node**: Create `Validate, Dedupe & Tier`. Populate NGO profile metadata, implement financial year and receipt number calculation logic (`80G/YYYY-YY/000000`), aggregate lifetime giving totals, assign donor tiers (*Patron*, *Champion*, *Supporter*, *Friend*), and implement the Indian currency word conversion algorithm.
7. **Switch Node**: Create `Route by Donation Status` with rules matching `$json.status` against `new` and `mismatch`.

#### Step 5: Build Receipt Generation & Delivery Flow
1. **OpenAI Node**: Connect Output 0 (`New`) of the Switch node to `AI Write Thank-you Letter`. Configure model `gpt-4o-mini`, temperature `0.6`, system prompt for personalized donor letters, and JSON output formatting.
2. **Slack Node**: Connect Output 1 (`Mismatch`) of the Switch node to `Alert Slack: Payment Mismatch`. Configure message text and channel ID `REPLACE_WITH_SLACK_CHANNEL_ID`.
3. **Code Node**: Create `Build 80G Receipt HTML` to construct the professional HTML receipt markup.
4. **Convert to File Node**: Create `Receipt HTML to File` converting the HTML string property to a text binary file named `index.html`.
5. **HTTP Request Node**: Create `Render Receipt PDF`. Set method to `POST`, URL to `http://gotenberg:3000/forms/chromium/convert/html`, content type to `multipart/form-data`, and add binary file parameter `files` from input data field `data`. Set timeout to `60000`.
6. **Google Drive Node**: Create `Archive Receipt to Drive`. Set operation `Upload`, file name expression `={{ $('Build 80G Receipt HTML').first().json.file_name }}`, and folder ID `REPLACE_WITH_RECEIPTS_FOLDER_ID`.
7. **Gmail Node**: Create `Email Thank-you & 80G Receipt`. Set resource `Message`, operation `Send`, recipient to donor email, subject to AI-generated subject, and attach the binary PDF file.
8. **WhatsApp Node**: Create `Send WhatsApp Thank-you`. Set operation `Send`, phone number ID `REPLACE_WITH_WHATSAPP_PHONE_NUMBER_ID`, recipient phone number, and configure `onError: continueRegularOutput`.
9. **Google Sheets Node**: Create `Append Donations Ledger`. Set operation `Append`, sheet name `Donations`, and map all transaction columns.
10. **Google Sheets Node**: Create `Upsert Donor Profile`. Set operation `Append or Update`, sheet name `Donors`, matching column `donor_key`, and map all donor profile fields.
11. **If Node**: Create `Tier Upgrade or Major Gift?` checking if `tier_upgraded` or `major_gift` is true.
12. **Slack Node**: Create `Alert Fundraising Team on Slack` to notify the team of major gifts or tier escalations.

#### Step 6: Build Quarterly Impact Reporting Flow
1. **Schedule Trigger Node**: Create `Every Quarter (1 Jan, Apr, Jul, Oct)` with cron expression `0 10 1 1,4,7,10 *`.
2. **Code Node**: Create `Set Quarter Window` to compute start and end ISO timestamps for the previous quarter.
3. **Google Sheets Nodes**: Create `Read Donations (Quarter)` and `Read Impact Metrics` to fetch historical records (`executeOnce: true`).
4. **Code Node**: Create `Aggregate Quarter` to calculate totals, campaign distributions, and audience tier groupings.
5. **OpenAI Node**: Create `AI Write Impact Story` using `gpt-4o-mini` (temperature `0.5`) to generate impact narratives and segmented tier notes.
6. **Code Node**: Create `Build Tier Drafts` to segment recipient lists into BCC batches under 450 addresses.
7. **Gmail Node**: Create `Create Gmail Draft per Tier` setting resource to `draft`, email type to `html`, and mapping BCC lists.
8. **Slack Node**: Create `Notify Slack: Drafts Ready` to announce draft availability.

#### Step 7: Build CSR Proposal Flow
1. **Form Trigger Node**: Create `CSR Enquiry Form` with form title `CSR Partnership Enquiry` and fields for Company Name, Contact Person, Designation, Email, Phone, Budget Band, Schedule VII Focus Areas, Geography, Timeline, and Notes.
2. **Google Sheets Node**: Create `Read Program Catalogue` to pull rows from the `Programs` sheet (`executeOnce: true`).
3. **Code Node**: Create `Match Programs to CSR Brief` to evaluate multi-variable scoring (focus areas, geography, budget) and generate unique lead IDs (`CSR-YYMMDD-XXXX`).
4. **OpenAI Node**: Create `AI Write CSR Proposal` using `gpt-4o-mini` (temperature `0.4`) to generate Schedule VII compliant proposal sections.
5. **Code Node**: Create `Build CSR Proposal HTML` to render the professional proposal document layout.
6. **Convert to File Node**: Create `CSR HTML to File` converting the HTML string to an `index.html` binary file.
7. **HTTP Request Node**: Create `Render CSR Proposal PDF` targeting Gotenberg (`http://gotenberg:3000/forms/chromium/convert/html`).
8. **Google Drive Node**: Create `Archive CSR Proposal to Drive` uploading the PDF to `REPLACE_WITH_CSR_FOLDER_ID`.
9. **Gmail Node**: Create `Create Proposal Gmail Draft` attaching the PDF proposal and cover email text.
10. **Google Sheets Node**: Create `Append to CSR Pipeline` logging lead records into the `CSR Pipeline` sheet.
11. **Slack Node**: Create `Alert Slack: New CSR Lead` to announce new corporate partnership briefs to the team.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Gotenberg Docker Deployment Instruction | Run locally using Docker: `docker run -p 3000:3000 gotenberg/gotenberg:8` |
| Statutory Compliance Notice | Ensure PAN and address details are collected accurately on payment notes or forms to comply with Indian Income Tax Act requirements for Form 10BD and Form 10BE generation. |
Send solar ROI proposals from electricity bills with WhatsApp, OpenAI and Google Drive

https://n8nworkflows.xyz/workflows/send-solar-roi-proposals-from-electricity-bills-with-whatsapp--openai-and-google-drive-20875


# Send solar ROI proposals from electricity bills with WhatsApp, OpenAI and Google Drive

### 1. Workflow Overview

This workflow automates the rooftop solar lead generation, engineering sizing, subsidy calculation, proposal document creation, and appointment booking process. Designed for solar EPC companies in India, it removes manual overhead by instantaneously extracting electricity bill data via AI, calculating technical system specifications and financial returns (ROI, PM Surya Ghar subsidy, EMI), generating a branded PDF proposal, communicating via WhatsApp, updating the CRM, and coordinating site surveys with field engineers.

The system logic operates across several interdependent functional blocks:
- **1.1 Input Reception:** Captures incoming prospective customer data via WhatsApp messages (with or without bill photos) or an n8n web form with uploaded file attachments.
- **1.2 Bill OCR & Validation:** Downloads media files, queries OpenAI’s GPT-4o for high-accuracy bill data extraction (consumption, tariff, sanctioned load), and validates legibility.
- **1.3 Sizing, Subsidy & Financial Analysis:** Calculates optimal solar system capacity based on state-specific solar yields, calculates the PM Surya Ghar subsidy, projects 25-year financial savings, creates a QuickChart graphical representation, and evaluates lead priority scores.
- **1.4 Proposal Generation:** Uses OpenAI (gpt-4o-mini) to generate customized proposal text, compiles the elements into structured HTML, converts the text to a file, and utilizes a self-hosted Gotenberg container to output a finalized PDF.
- **1.5 Delivery & CRM Hand-Off:** Archives the proposal PDF to Google Drive, transmits the PDF and a Cal.com booking link to the lead via WhatsApp, upserts lead data into Google Sheets, and alerts internal sales personnel.
- **1.6 Survey Booking & Engineering Dispatch:** Listens for Cal.com webhook payloads, updates lead statuses based on booking actions (booked, rescheduled, or cancelled), dispatches automated confirmation/rebooking notices to leads, and briefs site engineers with operational details.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Receives raw webhook payloads from WhatsApp Business Cloud or the n8n web form, cleanses metadata, drops empty delivery status updates, normalizes phone numbers, and manages file binaries.
- **Nodes Involved:**
  - `On WhatsApp Message`
  - `Parse WhatsApp Message`
  - `Has Bill Photo?`
  - `Get WhatsApp Media URL`
  - `Ask Lead for Bill Photo`
  - `Download Bill Image`
  - `Solar Proposal Web Form`
  - `Normalize Form Lead`
- **Node Details:**
  - **On WhatsApp Message**
    - *Type & Technical Role:* `n8n-nodes-base.whatsAppTrigger` – Listens for incoming WhatsApp message events.
    - *Configuration:* Configured to trigger on the `messages` update array.
    - *Expressions:* Uses default webhook registration (`aa354940-57e7-4802-b8a6-70bddbd72709`).
    - *Connections:* Input: None (Trigger); Output: `Parse WhatsApp Message`.
  - **Parse WhatsApp Message**
    - *Type & Technical Role:* `n8n-nodes-base.code` – Extracts core message properties and drops delivery receipts.
    - *Configuration:* JavaScript execution processing raw input JSON.
    - *Expressions:* Iterates through `v.messages`, evaluates `contact` profiles, and extracts media/text bodies.
    - *Connections:* Input: `On WhatsApp Message`; Output: `Has Bill Photo?`.
  - **Has Bill Photo?**
    - *Type & Technical Role:* `n8n-nodes-base.if` – Conditional branch evaluating whether an uploaded bill image is present.
    - *Configuration:* Loose type validation comparing boolean states.
    - *Expressions:* `={{ $json.has_bill_photo }}`
    - *Connections:* Input: `Parse WhatsApp Message`; Output (True): `Get WhatsApp Media URL`; Output (False): `Ask Lead for Bill Photo`.
  - **Get WhatsApp Media URL**
    - *Type & Technical Role:* `n8n-nodes-base.whatsApp` – Fetches the download link for media uploaded to WhatsApp.
    - *Configuration:* Resource: `media`, Operation: `mediaUrlGet`.
    - *Expressions:* `={{ $json.media_id }}`
    - *Connections:* Input: `Has Bill Photo?` (True); Output: `Download Bill Image`.
  - **Ask Lead for Bill Photo**
    - *Type & Technical Role:* `n8n-nodes-base.whatsApp` – Sends an automated text prompting users to upload a clear electricity bill image.
    - *Configuration:* Operation: `send`, specifies phone number ID.
    - *Expressions:* Dynamic text body greeting the lead by name; Recipient: `={{ $json.phone }}`.
    - *Connections:* Input: `Has Bill Photo?` (False); Output: None (Terminal branch).
  - **Download Bill Image**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` – Downloads binary file data from the secured WhatsApp media URL.
    - *Configuration:* Response format set to `file`, uses `whatsAppApi` credentials.
    - *Expressions:* `={{ $json.url }}`
    - *Connections:* Input: `Get WhatsApp Media URL`; Output: `Prepare Lead & OCR Request`.
  - **Solar Proposal Web Form**
    - *Type & Technical Role:* `n8n-nodes-base.formTrigger` – Serves an interactive web interface for capturing lead metadata and file uploads.
    - *Configuration:* Form title and custom fields configured (Full Name, WhatsApp Number, City, Property Type, Roof Area, Electricity Bill Photo).
    - *Expressions:* Custom form submission success text.
    - *Connections:* Input: None (Trigger); Output: `Normalize Form Lead`.
  - **Normalize Form Lead**
    - *Type & Technical Role:* `n8n-nodes-base.code` – Cleans form fields, formats phone numbers to international standards, and handles binary data mapping.
    - *Configuration:* JavaScript execution per item.
    - *Expressions:* Normalizes form inputs and assigns binary property `bill`.
    - *Connections:* Input: `Solar Proposal Web Form`; Output: `Prepare Lead & OCR Request`.

---

#### 2.2 Bill OCR & Validation
- **Overview:** Prepares the extracted bill image payload, calls OpenAI’s GPT-4o model to parse consumption details in JSON format, and validates legibility and data quality.
- **Nodes Involved:**
  - `Prepare Lead & OCR Request`
  - `AI Read Electricity Bill (OCR)`
  - `Parse & Validate OCR`
  - `Bill Readable?`
  - `Ask for Clearer Bill Photo`
- **Node Details:**
  - **Prepare Lead & OCR Request**
    - *Type & Technical Role:* `n8n-nodes-base.code` – Converts binary files to base64, generates unique lead IDs (`SOL-YYMMDD-XXXX-RAND`), and formats the OpenAI Vision API request body.
    - *Configuration:* JavaScript execution handling asynchronous buffer reading.
    - *Expressions:* Dynamically builds JSON payload with strict system instructions for Indian DISCOM OCR.
    - *Connections:* Input: `Download Bill Image` or `Normalize Form Lead`; Output: `AI Read Electricity Bill (OCR)`.
  - **AI Read Electricity Bill (OCR)**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` – Makes a REST API call to OpenAI chat completions for vision processing.
    - *Configuration:* Method: `POST`, Timeout: 120,000ms, Retries on fail enabled, uses `openAiApi` credentials.
    - *Expressions:* `={{ JSON.stringify($json.vision_body) }}`
    - *Connections:* Input: `Prepare Lead & OCR Request`; Output: `Parse & Validate OCR`.
  - **Parse & Validate OCR**
    - *Type & Technical Role:* `n8n-nodes-base.code` – Parses the OpenAI OCR output, normalizes bi-monthly consumption metrics to monthly figures, calculates peak usage, and determines validity.
    - *Configuration:* JavaScript execution per item.
    - *Expressions:* Evaluates consumption thresholds and confidence scores.
    - *Connections:* Input: `AI Read Electricity Bill (OCR)`; Output: `Bill Readable?`.
  - **Bill Readable?**
    - *Type & Technical Role:* `n8n-nodes-base.if` – Evaluates whether the OCR extraction passed validation metrics.
    - *Configuration:* Loose boolean type validation.
    - *Expressions:* `={{ $json.readable }}`
    - *Connections:* Input: `Parse & Validate OCR`; Output (True): `Size System, Subsidy & ROI`; Output (False): `Ask for Clearer Bill Photo`.
  - **Ask for Clearer Bill Photo**
    - *Type & Technical Role:* `n8n-nodes-base.whatsApp` – Sends an automated text message requesting a higher quality bill upload due to parsing errors.
    - *Configuration:* Operation: `send`.
    - *Expressions:* Dynamic template referencing OCR issue metrics and recipient phone numbers.
    - *Connections:* Input: `Bill Readable?` (False); Output: None (Terminal branch).

---

#### 2.3 Sizing, Subsidy & Financial Analysis
- **Overview:** Executes core engineering and financial algorithms to size the solar installation, compute state-level PM Surya Ghar subsidies, forecast 25-year returns, generate charting URLs, and assign lead priority scores.
- **Nodes Involved:**
  - `Size System, Subsidy & ROI`
- **Node Details:**
  - **Size System, Subsidy & ROI**
    - *Type & Technical Role:* `n8n-nodes-base.code` – Comprehensive calculation script processing regional solar yields, sanctioned load caps, tier-based pricing structures, loan EMIs, and QuickChart JSON encoding.
    - *Configuration:* JavaScript execution containing extensive lookup maps (e.g., state yields, price brackets, special category states).
    - *Expressions:* Complex calculations outputting a consolidated `calc` object containing system sizing, financial returns, and QuickChart URLs.
    - *Connections:* Input: `Bill Readable?` (True); Output: `AI Write Proposal Copy`.

---

#### 2.4 Proposal Generation
- **Overview:** Uses a lightweight LLM to generate customer-facing proposal copy and FAQs, combines calculated metrics into a responsive HTML document, converts the HTML to text, and renders a PDF via Gotenberg.
- **Nodes Involved:**
  - `AI Write Proposal Copy`
  - `Build Proposal HTML`
  - `Convert HTML to File`
  - `Render Proposal PDF (Gotenberg)`
- **Node Details:**
  - **AI Write Proposal Copy**
    - *Type & Technical Role:* `@n8n/n8n-nodes-langchain.openAi` – Advanced AI node generating localized proposal copy.
    - *Configuration:* Model: `gpt-4o-mini`, Temperature: 0.4, JSON output mode enforced.
    - *Expressions:* Passes stringified lead consumption and financial metrics to the system prompt.
    - *Connections:* Input: `Size System, Subsidy & ROI`; Output: `Build Proposal HTML`.
  - **Build Proposal HTML**
    - *Type & Technical Role:* `n8n-nodes-base.code` – Constructs a fully styled A4-compatible HTML proposal document embedding data tables and QuickChart images.
    - *Configuration:* JavaScript execution per item.
    - *Expressions:* Combines calculation parameters, AI headlines, introductions, FAQs, and CSS styling.
    - *Connections:* Input: `AI Write Proposal Copy`; Output: `Convert HTML to File`.
  - **Convert HTML to File**
    - *Type & Technical Role:* `n8n-nodes-base.convertToFile` – Converts HTML string data into a binary file format.
    - *Configuration:* Operation: `toText`, source property `html`.
    - *Connections:* Input: `Build Proposal HTML`; Output: `Render Proposal PDF (Gotenberg)`.
  - **Render Proposal PDF (Gotenberg)**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` – Communicates with a self-hosted Gotenberg instance to convert HTML files into formatted PDF documents.
    - *Configuration:* Method: `POST`, Content-Type: `multipart/form-data`, Timeout: 60,000ms. Body parameters include file payloads, paper width (8.27 in), paper height (11.7 in), print background, and a 2s wait delay.
    - *Connections:* Input: `Convert HTML to File`; Output: `Archive Proposal to Drive`, `Send Proposal PDF on WhatsApp`.

---

#### 2.5 Deliver & Hand Off
- **Overview:** Archives the generated PDF to Google Drive, transmits the proposal document and interactive Cal.com survey booking link via WhatsApp, upserts lead data to Google Sheets, and notifies internal sales teams.
- **Nodes Involved:**
  - `Archive Proposal to Drive`
  - `Send Proposal PDF on WhatsApp`
  - `Send Survey Booking Link`
  - `Upsert Lead in CRM Sheet`
  - `Alert Sales Engineer on WhatsApp`
- **Node Details:**
  - **Archive Proposal to Drive**
    - *Type & Technical Role:* `n8n-nodes-base.googleDrive` – Uploads binary files to a designated Google Drive folder.
    - *Configuration:* Resource: File, Operation: Upload, uses configured Google Drive credentials and folder ID placeholders.
    - *Expressions:* File name mapped to `={{ $('Build Proposal HTML').item.json.file_name }}`.
    - *Connections:* Input: `Render Proposal PDF (Gotenberg)`; Output: None (Parallel branch).
  - **Send Proposal PDF on WhatsApp**
    - *Type & Technical Role:* `n8n-nodes-base.whatsApp` – Sends document-type messages via WhatsApp Business API.
    - *Configuration:* Operation: `send`, message type `document`.
    - *Expressions:* Dynamic media caption and custom filenames referencing calculated system metrics.
    - *Connections:* Input: `Render Proposal PDF (Gotenberg)`; Output: `Send Survey Booking Link`.
  - **Send Survey Booking Link**
    - *Type & Technical Role:* `n8n-nodes-base.whatsApp` – Sends a message containing English and Hindi Cal.com site survey booking links with metadata parameters.
    - *Configuration:* Operation: `send`.
    - *Expressions:* Includes dynamic survey URLs with lead ID metadata parameters and AI-generated message bodies.
    - *Connections:* Input: `Send Proposal PDF on WhatsApp`; Output: `Upsert Lead in CRM Sheet`.
  - **Upsert Lead in CRM Sheet**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` – Creates or updates lead status records in Google Sheets.
    - *Configuration:* Operation: `appendOrUpdate`, matching columns: `lead_id`, sheet name: `Solar Leads`.
    - *Expressions:* Maps structured properties (system kW, subsidy, net cost, payback, ROI, proposal links, timestamps) to spreadsheet columns.
    - *Connections:* Input: `Send Survey Booking Link`; Output: `Alert Sales Engineer on WhatsApp`.
  - **Alert Sales Engineer on WhatsApp**
    - *Type & Technical Role:* `n8n-nodes-base.whatsApp` – Sends lead summary alerts to internal sales personnel.
    - *Configuration:* Operation: `send`.
    - *Expressions:* Conditional evaluation formatting lead scores (HOT/WARM/NURTURE), energy metrics, and Drive proposal links.
    - *Connections:* Input: `Upsert Lead in CRM Sheet`; Output: None (Terminal node).

---

#### 2.6 Survey Booking & Engineering Dispatch
- **Overview:** Receives Cal.com webhooks for appointment actions, parses booking details, updates CRM statuses, dispatches WhatsApp appointment confirmations or rebooking nudges to leads, and briefs site engineers.
- **Nodes Involved:**
  - `On Cal.com Survey Booking`
  - `Parse Booking Details`
  - `Look Up Lead in CRM`
  - `Route by Booking Status`
  - `Mark Survey Scheduled in CRM`
  - `Mark Survey Cancelled in CRM`
  - `Confirm Survey to Lead`
  - `Send Rebooking Nudge`
  - `Brief Site Engineer on WhatsApp`
- **Node Details:**
  - **On Cal.com Survey Booking**
    - *Type & Technical Role:* `n8n-nodes-base.webhook` – HTTP webhook receiver listening for scheduling events.
    - *Configuration:* Path: `solar-survey-booked`, Method: `POST`.
    - *Connections:* Input: None (Trigger); Output: `Parse Booking Details`.
  - **Parse Booking Details**
    - *Type & Technical Role:* `n8n-nodes-base.code` – Parses Cal.com payload structures, extracts attendee details, and formats timestamps.
    - *Configuration:* JavaScript execution.
    - *Expressions:* Pulls metadata parameters like `lead_id` and attendee properties.
    - *Connections:* Input: `On Cal.com Survey Booking`; Output: `Look Up Lead in CRM`.
  - **Look Up Lead in CRM**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` – Queries the Google Sheets CRM to retrieve existing proposal and lead data.
    - *Configuration:* Operation: `get` (lookup value filter on `lead_id`), execute once enabled.
    - *Expressions:* Uses `={{ $json.lead_id }}` for column matching.
    - *Connections:* Input: `Parse Booking Details`; Output: `Route by Booking Status`.
  - **Route by Booking Status**
    - *Type & Technical Role:* `n8n-nodes-base.switch` – Directs execution flow based on whether the event contains cancellation indicators.
    - *Configuration:* Rule-based switching evaluating event strings.
    - *Expressions:* Evaluates if trigger event strings contain `CANCEL`.
    - *Connections:* Input: `Look Up Lead in CRM`; Output 1 (Booked/Rescheduled): `Mark Survey Scheduled in CRM`; Output 2 (Cancelled): `Mark Survey Cancelled in CRM`.
  - **Mark Survey Scheduled in CRM**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` – Updates CRM row status to "Survey Scheduled".
    - *Configuration:* Operation: `update`, matching column: `lead_id`.
    - *Expressions:* Updates `status`, `survey_at`, `updated_at`, and `survey_booking_uid`.
    - *Connections:* Input: `Route by Booking Status` (Branch 1); Output: `Confirm Survey to Lead`.
  - **Mark Survey Cancelled in CRM**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` – Updates CRM row status to "Survey Cancelled".
    - *Configuration:* Operation: `update`, matching column: `lead_id`.
    - *Expressions:* Updates `status` and `updated_at`.
    - *Connections:* Input: `Route by Booking Status` (Branch 2); Output: `Send Rebooking Nudge`.
  - **Confirm Survey to Lead**
    - *Type & Technical Role:* `n8n-nodes-base.whatsApp` – Sends appointment confirmation messages to customers.
    - *Configuration:* Operation: `send`.
    - *Expressions:* Displays scheduled display times, system capacity metrics retrieved from CRM, and preparation reminders.
    - *Connections:* Input: `Mark Survey Scheduled in CRM`; Output: `Brief Site Engineer on WhatsApp`.
  - **Send Rebooking Nudge**
    - *Type & Technical Role:* `n8n-nodes-base.whatsApp` – Sends polite rescheduling links following a booking cancellation.
    - *Configuration:* Operation: `send`.
    - *Expressions:* References original sizing proposals and provides booking URLs.
    - *Connections:* Input: `Mark Survey Cancelled in CRM`; Output: None (Terminal node).
  - **Brief Site Engineer on WhatsApp**
    - *Type & Technical Role:* `n8n-nodes-base.whatsApp` – Dispatches technical summaries and location coordinates to field engineers.
    - *Configuration:* Operation: `send`.
    - *Expressions:* Formats lead scores, contact info, utility details, proposed system capacities, and Google Maps routing URLs.
    - *Connections:* Input: `Confirm Survey to Lead`; Output: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note – Overview | n8n-nodes-base.stickyNote | Workflow documentation and configuration guide | None | None | ## ☀️ SolarOS – Rooftop Solar Lead to Proposal... |
| Sticky Note – WhatsApp Intake | n8n-nodes-base.stickyNote | Intake logic documentation | None | None | ## 💬 WhatsApp Intake... |
| Sticky Note – Web Form Intake | n8n-nodes-base.stickyNote | Web form intake documentation | None | None | ## 📝 Web Form Intake... |
| Sticky Note – Bill OCR | n8n-nodes-base.stickyNote | OCR processing documentation | None | None | ## 🔎 Bill OCR... |
| Sticky Note – Sizing, Subsidy & Proposal | n8n-nodes-base.stickyNote | Sizing and proposal generation documentation | None | None | ## ☀️ Sizing, Subsidy & Proposal... |
| Sticky Note – Deliver & Hand Off | n8n-nodes-base.stickyNote | Delivery and CRM hand-off documentation | None | None | ## 📲 Deliver & Hand Off... |
| Sticky Note – Survey Booking | n8n-nodes-base.stickyNote | Booking webhook documentation | None | None | ## 📅 Survey Booking... |
| Sticky Note – Scheduled or Cancelled | n8n-nodes-base.stickyNote | Booking routing documentation | None | None | ## 🛠️ Scheduled or Cancelled... |
| Sticky Note – Credentials & Security | n8n-nodes-base.stickyNote | Security notes | None | None | ## 🔐 Credentials & Security... |
| On WhatsApp Message | n8n-nodes-base.whatsAppTrigger | Listens for incoming WhatsApp messages | None | Parse WhatsApp Message | |
| Solar Proposal Web Form | n8n-nodes-base.formTrigger | Serves proposal intake web form | None | Normalize Form Lead | |
| Parse WhatsApp Message | n8n-nodes-base.code | Cleans message payloads and filters read receipts | On WhatsApp Message | Has Bill Photo? | |
| Normalize Form Lead | n8n-nodes-base.code | Formats lead metadata and normalizes phone numbers | Solar Proposal Web Form | Prepare Lead & OCR Request | |
| Has Bill Photo? | n8n-nodes-base.if | Evaluates if media attachment exists in message | Parse WhatsApp Message | Get WhatsApp Media URL, Ask Lead for Bill Photo | |
| Get WhatsApp Media URL | n8n-nodes-base.whatsApp | Retrieves media file download URL | Has Bill Photo? | Download Bill Image | |
| Ask Lead for Bill Photo | n8n-nodes-base.whatsApp | Prompts lead to upload a bill photo | Has Bill Photo? | None | |
| Download Bill Image | n8n-nodes-base.httpRequest | Downloads bill image binary | Get WhatsApp Media URL | Prepare Lead & OCR Request | |
| Prepare Lead & OCR Request | n8n-nodes-base.code | Generates IDs and formats OpenAI Vision payload | Download Bill Image, Normalize Form Lead | AI Read Electricity Bill (OCR) | |
| AI Read Electricity Bill (OCR) | n8n-nodes-base.httpRequest | Queries GPT-4o for structured OCR data | Prepare Lead & OCR Request | Parse & Validate OCR | |
| Parse & Validate OCR | n8n-nodes-base.code | Normalizes consumption metrics and validates legibility | AI Read Electricity Bill (OCR) | Bill Readable? | |
| Bill Readable? | n8n-nodes-base.if | Checks if OCR confidence meets criteria | Parse & Validate OCR | Size System, Subsidy & ROI, Ask for Clearer Bill Photo | |
| Ask for Clearer Bill Photo | n8n-nodes-base.whatsApp | Requests re-upload for unreadable bills | Bill Readable? | None | |
| Size System, Subsidy & ROI | n8n-nodes-base.code | Computes capacity, subsidies, payback, and charts | Bill Readable? | AI Write Proposal Copy | |
| AI Write Proposal Copy | @n8n/n8n-nodes-langchain.openAi | Generates customized proposal copy and FAQs | Size System, Subsidy & ROI | Build Proposal HTML | |
| Build Proposal HTML | n8n-nodes-base.code | Generates formatted HTML proposal document | AI Write Proposal Copy | Convert HTML to File | |
| Convert HTML to File | n8n-nodes-base.convertToFile | Converts HTML string data to binary text file | Build Proposal HTML | Render Proposal PDF (Gotenberg) | |
| Render Proposal PDF (Gotenberg) | n8n-nodes-base.httpRequest | Renders HTML to PDF via Gotenberg | Convert HTML to File | Archive Proposal to Drive, Send Proposal PDF on WhatsApp | |
| Archive Proposal to Drive | n8n-nodes-base.googleDrive | Uploads proposal PDF to Google Drive | Render Proposal PDF (Gotenberg) | None | |
| Send Proposal PDF on WhatsApp | n8n-nodes-base.whatsApp | Sends proposal document via WhatsApp | Render Proposal PDF (Gotenberg) | Send Survey Booking Link | |
| Send Survey Booking Link | n8n-nodes-base.whatsApp | Sends scheduling booking links | Send Proposal PDF on WhatsApp | Upsert Lead in CRM Sheet | |
| Upsert Lead in CRM Sheet | n8n-nodes-base.googleSheets | Appends or updates lead records in CRM sheet | Send Survey Booking Link | Alert Sales Engineer on WhatsApp | |
| Alert Sales Engineer on WhatsApp | n8n-nodes-base.whatsApp | Alerts sales personnel with lead score summaries | Upsert Lead in CRM Sheet | None | |
| On Cal.com Survey Booking | n8n-nodes-base.webhook | Receives booking lifecycle webhooks | None | Parse Booking Details | |
| Parse Booking Details | n8n-nodes-base.code | Parses webhook event payloads | On Cal.com Survey Booking | Look Up Lead in CRM | |
| Look Up Lead in CRM | n8n-nodes-base.googleSheets | Retrieves corresponding CRM record | Parse Booking Details | Route by Booking Status | |
| Route by Booking Status | n8n-nodes-base.switch | Routes execution based on booking action | Look Up Lead in CRM | Mark Survey Scheduled in CRM, Mark Survey Cancelled in CRM | |
| Mark Survey Scheduled in CRM | n8n-nodes-base.googleSheets | Updates status to Survey Scheduled | Route by Booking Status | Confirm Survey to Lead | |
| Mark Survey Cancelled in CRM | n8n-nodes-base.googleSheets | Updates status to Survey Cancelled | Route by Booking Status | Send Rebooking Nudge | |
| Confirm Survey to Lead | n8n-nodes-base.whatsApp | Sends booking confirmation to lead | Mark Survey Scheduled in CRM | Brief Site Engineer on WhatsApp | |
| Send Rebooking Nudge | n8n-nodes-base.whatsApp | Sends rescheduling prompt on cancellation | Mark Survey Cancelled in CRM | None | |
| Brief Site Engineer on WhatsApp | n8n-nodes-base.whatsApp | Briefs site engineer with details and maps link | Confirm Survey to Lead | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to manually recreate the workflow in an n8n instance:

1. **Create Intake Triggers:**
   - Add an **On WhatsApp Message** (`n8n-nodes-base.whatsAppTrigger`) node. Set updates to listen to `messages`.
   - Add a **Solar Proposal Web Form** (`n8n-nodes-base.formTrigger`) node. Configure form fields: *Full Name* (required), *WhatsApp Number* (required), *City* (required), *Property Type* (Dropdown: Residential, Housing Society, Commercial), *Shadow-free Roof Area (sq ft)* (number), and *Electricity Bill Photo* (file upload, `.jpg,.jpeg,.png,.webp`). Set submission success text.

2. **Setup Intake Processing & Validation:**
   - Connect **On WhatsApp Message** to a **Code** node named `Parse WhatsApp Message` configured to extract valid customer text/media messages and filter out delivery status events.
   - Connect **Solar Proposal Web Form** to a **Code** node named `Normalize Form Lead` to sanitize phone numbers and map binary file objects.
   - Attach an **If** node named `Has Bill Photo?` following `Parse WhatsApp Message`. Set condition: `={{ $json.has_bill_photo }}` equals `true`.
   - On the `true` output, add a **WhatsApp** node named `Get WhatsApp Media URL` (Resource: `media`, Operation: `mediaUrlGet`) using expression `={{ $json.media_id }}`. On the `false` output, add a **WhatsApp** node named `Ask Lead for Bill Photo` to prompt users.
   - Connect **Get WhatsApp Media URL** to an **HTTP Request** node named `Download Bill Image` (Response format: File, using WhatsApp API credentials).

3. **Configure AI Bill OCR:**
   - Connect **Download Bill Image** and **Normalize Form Lead** to a **Code** node named `Prepare Lead & OCR Request` to generate unique lead IDs (`SOL-...`) and structure the OpenAI Vision request.
   - Connect to an **HTTP Request** node named `AI Read Electricity Bill (OCR)` targeting `https://api.openai.com/v1/chat/completions` (Method: `POST`, OpenAI API credentials, JSON body containing GPT-4o vision prompt). Enable retries on fail.
   - Connect to a **Code** node named `Parse & Validate OCR` to parse JSON output, normalize bi-monthly consumption, and evaluate legibility.
   - Add an **If** node named `Bill Readable?` evaluating `={{ $json.readable }}`. Connect the `false` branch to a **WhatsApp** node named `Ask for Clearer Bill Photo`.

4. **Implement Sizing, Subsidy, and LLM Proposal Generation:**
   - Connect the `true` branch of `Bill Readable?` to a **Code** node named `Size System, Subsidy & ROI`. Input the extensive JavaScript logic containing state yield maps, PM Surya Ghar subsidy formulas, turnkey price bands, ROI calculations, and QuickChart URL string builders.
   - Connect to an **OpenAI Chat Model** node named `AI Write Proposal Copy` (`@n8n/n8n-nodes-langchain.openAi`) utilizing model `gpt-4o-mini`, temperature `0.4`, and JSON output enabled. Pass calculated metrics in the user message.
   - Connect to a **Code** node named `Build Proposal HTML` to assemble the responsive A4 HTML template embedding tables and chart images.
   - Connect to a **Convert to File** node named `Convert HTML to File` (Operation: `toText`, Source property: `html`).
   - Connect to an **HTTP Request** node named `Render Proposal PDF (Gotenberg)` targeting your self-hosted Gotenberg endpoint (`http://gotenberg:3000/forms/chromium/convert/html`, Method: `POST`, Content-Type: `multipart/form-data`) with parameters: `files` (binary data), `paperWidth` (`8.27`), `paperHeight` (`11.7`), `printBackground` (`true`), and `waitDelay` (`2s`). Enable retries.

5. **Configure Delivery and CRM Hand-Off:**
   - From `Render Proposal PDF (Gotenberg)`, create two parallel branches:
     - **Branch A:** Add a **Google Drive** node named `Archive Proposal to Drive` (Resource: File, Operation: Upload) using your target folder ID.
     - **Branch B:** Add a **WhatsApp** node named `Send Proposal PDF on WhatsApp` (Operation: `send`, Message type: `document`) with dynamic captions and filenames.
   - Connect `Send Proposal PDF on WhatsApp` to a **WhatsApp** node named `Send Survey Booking Link` to send survey URLs containing metadata parameters.
   - Connect to a **Google Sheets** node named `Upsert Lead in CRM Sheet` (Operation: `appendOrUpdate`, matching column: `lead_id`, Sheet name: `Solar Leads`) mapping all calculated metrics, proposal links, and timestamps.
   - Connect to a **WhatsApp** node named `Alert Sales Engineer on WhatsApp` targeting the sales team's phone number with lead score summaries.

6. **Configure Survey Booking Webhook:**
   - Add a **Webhook** trigger node named `On Cal.com Survey Booking` (Path: `solar-survey-booked`, Method: `POST`).
   - Connect to a **Code** node named `Parse Booking Details` to extract booking UIDs, timestamps, attendee data, and lead IDs.
   - Connect to a **Google Sheets** node named `Look Up Lead in CRM` (Operation: `get`, filter by `lead_id`).
   - Connect to a **Switch** node named `Route by Booking Status` to route based on whether the trigger event contains cancellation keywords.
   - **Scheduled/Rescheduled Branch:**
     - Connect to a **Google Sheets** node named `Mark Survey Scheduled in CRM` (Operation: `update`, status: `Survey Scheduled`).
     - Connect to a **WhatsApp** node named `Confirm Survey to Lead` to confirm appointment times.
     - Connect to a **WhatsApp** node named `Brief Site Engineer on WhatsApp` to forward location coordinates and system specs to the site engineer.
   - **Cancelled Branch:**
     - Connect to a **Google Sheets** node named `Mark Survey Cancelled in CRM` (Operation: `update`, status: `Survey Cancelled`).
     - Connect to a **WhatsApp** node named `Send Rebooking Nudge` to deliver rescheduling links.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Privacy & Security** | Electricity bills contain sensitive consumer PII (names, addresses, consumer numbers). Ensure Google Drive folders and Google Sheets CRM documents remain private and restricted to authorized personnel. |
| **Cal.com Integration** | Secure production webhooks with a Cal.com webhook secret. Ensure the booking link metadata structure (`?metadata[lead_id]=...`) matches the extraction pattern in `Parse Booking Details`. |
| **Gotenberg Deployment** | Run Gotenberg via Docker (`docker run -d --name gotenberg -p 3000:3000 gotenberg/gotenberg:8`) within a private internal network accessible to your n8n execution environment rather than exposing endpoints publicly. |
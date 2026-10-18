Log maintenance tickets from forms into Google Sheets and notify via Gmail

https://n8nworkflows.xyz/workflows/log-maintenance-tickets-from-forms-into-google-sheets-and-notify-via-gmail-20890


# Log maintenance tickets from forms into Google Sheets and notify via Gmail

### 1. Workflow Overview

The "Automated Maintenance Ticketing System" workflow automates the intake, recording, and notification of facility or property maintenance requests. Its primary purpose is to eliminate manual data entry, provide immediate visibility of damage reports, and keep a centralized, searchable ledger of ongoing maintenance tasks.

The workflow’s logic is organized into the following functional blocks:
- **1.1 Input Reception:** Captures user-submitted maintenance reports via an interactive n8n web form, including textual details and photographic evidence.
- **1.2 Media Processing:** Transforms binary file uploads into publicly accessible URLs to enable remote viewing within spreadsheets and emails.
- **1.3 Database Logging:** Persists structured ticket records, including status metadata and dynamic formula previews, into a designated Google Sheets document.
- **1.4 Stakeholder Notification:** Dispatches a formatted HTML alert via Gmail to the maintenance manager, containing embedded visual proof and direct links to the master tracker.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block acts as the entry point of the workflow, serving an interactive form to users and capturing required maintenance parameters alongside binary image data.
- **Nodes Involved:** 
  - `Form: Submit Maintenance Request`

- **Node Details:**
  - **Type & Technical Role:** `n8n-nodes-base.formTrigger` (v2.6) – Webhook-driven form trigger that generates a standalone UI endpoint for user input.
  - **Configuration Choices:** 
    - Title: "Repair Request"
    - Fields: 
      - "Reporter Name" (Required, Text)
      - "Apartment / Room Number" (Required, Text)
      - "Describe the Issue" (Required, Textarea)
      - "Upload Photo of the Damage" (Required, File, accepted types: `.jpg`, `.png`)
  - **Key Expressions / Variables:** N/A (Trigger node)
  - **Input / Output Connections:** 
    - Input: None (Trigger)
    - Output: Connects to `Create Public Photo Link`
  - **Version Requirements:** v2.6 or higher.
  - **Edge Cases & Failure Types:** 
    - Users uploading unsupported file formats (handled natively by form validation if restricted via accept properties, otherwise fails downstream).
    - Incomplete mandatory fields preventing form submission.

---

#### 2.2 Media Processing

- **Overview:** This block takes the binary image file captured by the form trigger and uploads it to an external hosting utility, returning a stable public URL.
- **Nodes Involved:**
  - `Create Public Photo Link`

- **Node Details:**
  - **Type & Technical Role:** `n8n-nodes-uploadtourl.uploadToUrl` (v1) – Third-party integration node used to host binary files temporarily or permanently on a public URL.
  - **Configuration Choices:** 
    - Expiry Type: `never`
  - **Key Expressions / Variables:** Automatically maps binary data streams from the preceding form trigger.
  - **Input / Output Connections:** 
    - Input: Connected from `Form: Submit Maintenance Request`
    - Output: Connects to `Log Ticket in Database`
  - **Version Requirements:** v1.
  - **Edge Cases & Failure Types:** 
    - Authentication failures with the UploadToURL service.
    - Rate limits or payload size restrictions imposed by the file hosting provider.

---

#### 2.3 Database Logging

- **Overview:** This block maps the collected form inputs and public image URL into structured columns, appending a new row to a centralized Google Sheets tracker.
- **Nodes Involved:**
  - `Log Ticket in Database`

- **Node Details:**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (v4.7) – Interacts with Google Sheets API to perform row-level operations (append).
  - **Configuration Choices:** 
    - Operation: `append`
    - Document: Custom URL pointing to the target spreadsheet.
    - Sheet Name: `Sheet1` (gid=0)
    - Mapping Mode: `defineBelow` (Manual column-to-value mapping)
  - **Key Expressions / Variables:**
    - `Location`: `={{ $('Form: Submit Maintenance Request').item.json['Apartment / Room Number'] }}`
    - `Reporter`: `={{ $('Form: Submit Maintenance Request').item.json['Reporter Name'] }}`
    - `Date Reported`: `={{ $('Form: Submit Maintenance Request').item.json.submittedAt }}`
    - `Photo Preview`: `==IMAGE("{{ $json.url }}", 2)`
    - `Ticket Status`: `Pending`
    - `Issue Description`: `={{ $('Form: Submit Maintenance Request').item.json['Describe the Issue'] }}`
  - **Input / Output Connections:** 
    - Input: Connected from `Create Public Photo Link`
    - Output: Connects to `Email Alert to Manager`
  - **Version Requirements:** v4.7.
  - **Edge Cases & Failure Types:** 
    - Google Sheets API rate limits or quota exhaustion.
    - Schema mismatches if spreadsheet column headers are altered or deleted.
    - Authentication expiry on OAuth2 credentials.

---

#### 2.4 Stakeholder Notification

- **Overview:** This block formats and sends an HTML-rich email alert to the maintenance manager containing ticket details, an inline preview of the damage photo, and a tracker link.
- **Nodes Involved:**
  - `Email Alert to Manager`

- **Node Details:**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` (v2.2) – Sends emails through the authenticated user's Gmail account using the Gmail API.
  - **Configuration Choices:** 
    - Send To: `user@example.com` (placeholder)
    - Subject: `=🚨 New Repair Request: Apt {{ $('Form: Submit Maintenance Request').item.json["Apartment / Room Number"] }}`
    - Message: HTML-formatted string containing inline CSS, node expression evaluations for text fields, an `<img>` tag sourcing `{{ $('Create Public Photo Link').item.json.url }}`, and a styled anchor tag linking to the Google Sheets tracker.
  - **Key Expressions / Variables:** 
    - Pulls reporter name, location, and description directly from `Form: Submit Maintenance Request`.
    - Pulls the image URL directly from `Create Public Photo Link`.
  - **Input / Output Connections:** 
    - Input: Connected from `Log Ticket in Database`
    - Output: None (Terminal node)
  - **Version Requirements:** v2.2.
  - **Edge Cases & Failure Types:** 
    - Gmail API sending limits or daily quotas.
    - Invalid recipient email configurations.
    - Rendering issues in strict email clients that block external images or inline CSS.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Form: Submit Maintenance Request` | `n8n-nodes-base.formTrigger` | Captures resident repair requests and file uploads via web form. | None (Trigger) | `Create Public Photo Link` | ## 🛠️ Automated Maintenance Ticketing System<br><br>### Workflow Overview<br>1. A resident submits a maintenance request through an online form.<br>2. The form collects the reporter's name, apartment or room number, issue description, and damage photo.<br>3. UploadToURL uploads the photo and generates a public URL.<br>4. Google Sheets records the request details, photo preview, submission timestamp, and initial ticket status.<br>5. Gmail sends an HTML email alert to the maintenance manager with the request details and damage photo.<br>6. The manager can use the ticket tracker link to review the logged request.<br><br>## 📝 Form: Submit Maintenance Request<br><br>### Purpose<br>Provides a form for residents or users to report maintenance issues and upload a photo of the damage.<br><br>### How It Works<br>- Collects the reporter's name.<br>- Collects the apartment or room number.<br>- Collects a detailed description of the issue.<br>- Requires a JPG or PNG photo of the damage. |
| `Create Public Photo Link` | `n8n-nodes-uploadtourl.uploadToUrl` | Generates a public URL for the uploaded damage photo. | `Form: Submit Maintenance Request` | `Log Ticket in Database` | ## 🛠️ Automated Maintenance Ticketing System<br><br>### Workflow Overview<br>1. A resident submits a maintenance request through an online form.<br>2. The form collects the reporter's name, apartment or room number, issue description, and damage photo.<br>3. UploadToURL uploads the photo and generates a public URL.<br>4. Google Sheets records the request details, photo preview, submission timestamp, and initial ticket status.<br>5. Gmail sends an HTML email alert to the maintenance manager with the request details and damage photo.<br>6. The manager can use the ticket tracker link to review the logged request.<br><br>## 🔗 Create Public Photo Link<br><br>### Purpose<br>Uploads the damage photo and generates a URL that can be used in the maintenance tracker and notification email.<br><br>### How It Works<br>- Receives the submitted photo from the form node.<br>- Uploads the file through the UploadToURL integration.<br><br>### Why It's Needed<br>Allows the maintenance team to view the damage photo directly from the spreadsheet or email instead of searching for the original submission. |
| `Log Ticket in Database` | `n8n-nodes-base.googleSheets` | Appends maintenance ticket data and spreadsheet formulas to Google Sheets. | `Create Public Photo Link` | `Email Alert to Manager` | ## 🛠️ Automated Maintenance Ticketing System<br><br>### Workflow Overview<br>1. A resident submits a maintenance request through an online form.<br>2. The form collects the reporter's name, apartment or room number, issue description, and damage photo.<br>3. UploadToURL uploads the photo and generates a public URL.<br>4. Google Sheets records the request details, photo preview, submission timestamp, and initial ticket status.<br>5. Gmail sends an HTML email alert to the maintenance manager with the request details and damage photo.<br>6. The manager can use the ticket tracker link to review the logged request.<br><br>## 📊 Log Ticket in Database<br><br>### Purpose<br>Stores each submitted maintenance request in Google Sheets to create a centralized maintenance ticket tracker.<br><br>### Spreadsheet Columns<br>- Date Reported<br>- Reporter<br>- Location<br>- Issue Description<br>- Photo Preview<br>- Ticket Status<br><br>### Why It's Needed<br>Maintains a searchable record of maintenance requests, helping property managers organize reported issues and monitor pending work. |
| `Email Alert to Manager` | `n8n-nodes-base.gmail` | Sends an HTML email notification with ticket details and image preview to the manager. | `Log Ticket in Database` | None (Terminal) | ## 🛠️ Automated Maintenance Ticketing System<br><br>### Workflow Overview<br>1. A resident submits a maintenance request through an online form.<br>2. The form collects the reporter's name, apartment or room number, issue description, and damage photo.<br>3. UploadToURL uploads the photo and generates a public URL.<br>4. Google Sheets records the request details, photo preview, submission timestamp, and initial ticket status.<br>5. Gmail sends an HTML email alert to the maintenance manager with the request details and damage photo.<br>6. The manager can use the ticket tracker link to review the logged request.<br><br>## 📧 Email Alert to Manager<br><br>### Purpose<br>Sends an email notification to the maintenance manager when a new repair request has been logged.<br><br>### Why It's Needed<br>Notifies the maintenance manager promptly so that reported issues can be reviewed and assigned for resolution. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in your n8n instance and name it `Automated Maintenance Ticketing System`.
2. **Add Node 1:** Create a **Form Trigger** node (`n8n-nodes-base.formTrigger`).
   - Set form title to `Repair Request`.
   - Add field 1: Type `Text`, Label `Reporter Name`, Required `true`.
   - Add field 2: Type `Text`, Label `Apartment / Room Number`, Required `true`.
   - Add field 3: Type `Textarea`, Label `Describe the Issue`, Required `true`.
   - Add field 4: Type `File`, Label `Upload Photo of the Damage`, Required `true`, Accept File Types `.jpg, .png`.
3. **Add Node 2:** Create an **Upload to URL** node (`n8n-nodes-uploadtourl.uploadToUrl`).
   - Set Expiry Type to `never`.
   - Configure credentials for your Upload to URL account.
   - Connect the output of `Form: Submit Maintenance Request` to this node.
4. **Add Node 3:** Create a **Google Sheets** node (`n8n-nodes-base.googleSheets`).
   - Set Operation to `append`.
   - Set Document to your target Google Sheets URL.
   - Set Sheet Name to your target sheet (e.g., `Sheet1`).
   - Set Mapping Mode to `defineBelow`.
   - Map columns and values using expressions:
     - `Date Reported`: `={{ $('Form: Submit Maintenance Request').item.json.submittedAt }}`
     - `Reporter`: `={{ $('Form: Submit Maintenance Request').item.json['Reporter Name'] }}`
     - `Location`: `={{ $('Form: Submit Maintenance Request').item.json['Apartment / Room Number'] }}`
     - `Issue Description`: `={{ $('Form: Submit Maintenance Request').item.json['Describe the Issue'] }}`
     - `Photo Preview`: `==IMAGE("{{ $json.url }}", 2)`
     - `Ticket Status`: `Pending`
   - Configure Google Sheets OAuth2 credentials.
   - Connect the output of `Create Public Photo Link` to this node.
5. **Add Node 4:** Create a **Gmail** node (`n8n-nodes-base.gmail`).
   - Set Send To address to the target manager email (e.g., `user@example.com`).
   - Set Subject to `=🚨 New Repair Request: Apt {{ $('Form: Submit Maintenance Request').item.json["Apartment / Room Number"] }}`.
   - Populate the Message field with an HTML structure utilizing expressions for reporter name, location, description, image source (`{{ $('Create Public Photo Link').item.json.url }}`), and a tracker link button.
   - Configure Gmail OAuth2 credentials.
   - Connect the output of `Log Ticket in Database` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheets Template Reference | [Google Sheets Document Link](https://docs.google.com/spreadsheets/d/1eZp-t6yJ9H8ut5BG5-W_7PDxrylLTQ9CWgnRFoHglO0/edit?usp=sharing) |
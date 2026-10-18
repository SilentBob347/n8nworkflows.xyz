Send QR codes for form-uploaded files with UploadToURL and Gmail

https://n8nworkflows.xyz/workflows/send-qr-codes-for-form-uploaded-files-with-uploadtourl-and-gmail-20888


# Send QR codes for form-uploaded files with UploadToURL and Gmail

### 1. Workflow Overview

This workflow is designed to automate the generation and distribution of QR codes for user-submitted files. It captures a file upload alongside metadata (project name and email address) via an n8n web form, hosts the file publicly using an external upload service, converts the public URL into a scannable QR code image, and delivers the finalized asset directly to the user via email. 

The logical execution is grouped into the following functional blocks:
- **1.1 Form Input Reception:** Collects metadata and binary file payloads from end-users through an interactive web form.
- **1.2 File Hosting & Public Link Generation:** Sends the binary file payload to UploadToURL to create a public HTTP resource link.
- **1.3 QR Code Rendering:** Transforms the public resource URL into a PNG QR code image using a specialized toolkit node.
- **1.4 Email Distribution:** Composes and transmits an HTML-formatted email containing a personalized greeting, a direct link, and an attached copy of the generated QR code via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Form Input Reception
- **Overview:** Acts as the primary workflow entry point by presenting a web form to users, capturing text metadata, and accepting file uploads.
- **Nodes Involved:** `Form: Upload Any File`
- **Node Details:**
  - **Type & Technical Role:** `n8n-nodes-base.formTrigger` (Trigger Node). Pauses execution until a user submits the published web form.
  - **Configuration Choices:** Configured with three mandatory fields: "Project Name" (Text), "Email Address" (Email), and "Upload File (PDF, Image, or Document)" (File). Form options use default platform settings.
  - **Key Expressions or Variables:** N/A (Triggers output data directly).
  - **Input & Output Connections:** 
    - Inputs: None (Entry point).
    - Outputs: Connects to `Create Public File Link`.
  - **Version-Specific Requirements:** Version 2.6.
  - **Edge Cases & Potential Failures:** File size limits imposed by n8n instance settings or the browser; missing mandatory field submissions handled natively by the form interface.
  - **Sub-Workflow Reference:** None.

---

#### 1.2 File Hosting & Public Link Generation
- **Overview:** Takes the binary file payload received from the trigger block and uploads it to a cloud repository to make it publicly accessible via a direct URL.
- **Nodes Involved:** `Create Public File Link`
- **Node Details:**
  - **Type & Technical Role:** `n8n-nodes-uploadtourl.uploadToUrl` (Action Node). Interacts with the UploadToURL API to store binary objects.
  - **Configuration Choices:** `expiryType` parameter set to `never`.
  - **Key Expressions or Variables:** Automatically maps incoming binary file data from the preceding form node.
  - **Input & Output Connections:** 
    - Inputs: Receives execution from `Form: Upload Any File`.
    - Outputs: Connects to `Generate QR Code`.
  - **Version-Specific Requirements:** Version 1.
  - **Edge Cases & Potential Failures:** Authentication failures due to expired credentials; API rejection due to unsupported file types or excessive file sizes.
  - **Sub-Workflow Reference:** None.

---

#### 1.3 QR Code Rendering
- **Overview:** Consumes the public HTTP URL generated in the previous step and encodes it into a visual graphical representation (QR code).
- **Nodes Involved:** `Generate QR Code`
- **Node Details:**
  - **Type & Technical Role:** `@cyberace/n8n-nodes-qr.qrToolkit` (Action Node). Generates matrix barcode representations from string input.
  - **Configuration Choices:** `content` parameter targets the URL property returned by the file hosting node.
  - **Key Expressions or Variables:** `={{ $json.url }}`
  - **Input & Output Connections:** 
    - Inputs: Receives execution from `Create Public File Link`.
    - Outputs: Connects to `Email QR Code to User`.
  - **Version-Specific Requirements:** Version 1.
  - **Edge Cases & Potential Failures:** Expression resolution failures if the upstream node fails to supply a valid `url` property.
  - **Sub-Workflow Reference:** None.

---

#### 1.4 Email Distribution
- **Overview:** Assembles a structured HTML email incorporating dynamic data fields and attaches the binary QR code image for user delivery.
- **Nodes Involved:** `Email QR Code to User`
- **Node Details:**
  - **Type & Technical Role:** `n8n-nodes-base.gmail` (Action Node). Sends messages via the Gmail API utilizing OAuth2 authorization.
  - **Configuration Choices:** Sets custom sender name (`Jitesh`), dynamic recipient routing, and HTML message body structures with binary attachment options enabled (`attachmentsUi.attachmentsBinary`).
  - **Key Expressions or Variables:** 
    - Recipient: `={{ $('Form: Upload Any File').item.json['Email Address'] }}`
    - Project Name in HTML: `={{ $('Form: Upload Any File').item.json["Project Name"] }}`
    - Direct URL in HTML: `={{ $('Create Public File Link').item.json.url }}`
  - **Input & Output Connections:** 
    - Inputs: Receives execution from `Generate QR Code`.
    - Outputs: None (Terminal node).
  - **Version-Specific Requirements:** Version 2.2.
  - **Edge Cases & Potential Failures:** OAuth token expiration; invalid recipient email string format; API quota limits on the connected Gmail account.
  - **Sub-Workflow Reference:** None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Form: Upload Any File` | `n8n-nodes-base.formTrigger` | Collects user metadata and files via a published web form. | None (Trigger) | `Create Public File Link` | ## 📝 Form: Upload Any File<br><br>### Purpose<br>Collects the user's project name, email address, and document or image through an n8n form.<br><br>### How It Works<br>- Collects the user's project name.<br>- Collects and validates the user's email address.<br>- Allows the user to upload a document or image.<br>- Requires the configured fields before submission.<br><br>### Why It's Needed<br>Provides a simple interface for users to submit files without accessing the n8n editor. |
| `Create Public File Link` | `n8n-nodes-uploadtourl.uploadToUrl` | Uploads the submitted file to generate a public URL. | `Form: Upload Any File` | `Generate QR Code` | ## 🔗 Create Public File Link<br><br>### Purpose<br>Uploads the submitted document or image and generates a URL that can be used to create a QR code.<br><br>### How It Works<br>- Receives the uploaded file from the form node.<br>- Uploads the file using the UploadToURL integration.<br><br>### Why It's Needed<br>The QR code requires a destination URL that users can open after scanning it.<br><br>### Setup Notes<br>- Configure the required UploadToURL integration.<br>- Ensure the destination URL remains accessible for the intended period. |
| `Generate QR Code` | `@cyberace/n8n-nodes-qr.qrToolkit` | Converts the public file URL into a scannable PNG QR code image. | `Create Public File Link` | `Email QR Code to User` | ## 🔲 Generate QR Code<br><br>### Purpose<br>Converts the uploaded file's public URL into a QR code image.<br><br>### How It Works<br>- Receives the file URL from the previous node.<br>- Generates a QR code representing the destination URL.<br>- Produces the QR code as a PNG image.<br><br>### Why It's Needed<br>Allows users to access documents and images by scanning a QR code instead of manually entering a URL. |
| `Email QR Code to User` | `n8n-nodes-base.gmail` | Sends an HTML email with the direct link and attached QR code image. | `Generate QR Code` | None (Terminal) | ## 📧 Email QR Code to User<br><br>### Purpose<br>Sends the generated QR code to the email address provided in the original form submission.<br><br>### How It Works<br>- Creates an HTML email explaining that the QR code is ready.<br>- Attaches the generated QR code image.<br>- Includes a direct link to the uploaded file.<br><br>### Output<br>- Email sent to the user<br>- QR code image attachment<br>- Direct link to the uploaded file |

*(Note: The global overview sticky note spanning coordinates `[-96, -688]` provides overarching context for the entire workflow graph and has been omitted from individual node rows per structure rules.)*

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Form Trigger** node (`n8n-nodes-base.formTrigger`, v2.6).
   - Set the form title to `Instant QR`.
   - Add three form fields:
     1. Field 1: Label `Project Name`, type `Text`, required.
     2. Field 2: Label `Email Address`, type `Email`, required.
     3. Field 3: Label `Upload File (PDF, Image, or Document)`, type `File`, required.

2. **Create the File Hosting Node:**
   - Add an **Upload to URL** node (`n8n-nodes-uploadtourl.uploadToUrl`, v1).
   - Configure parameters: Set `Expiry Type` to `never`.
   - Connect credentials: Link or create an **Upload to URL account** credential.
   - Connect the output of `Form: Upload Any File` to this node's input.

3. **Create the QR Generation Node:**
   - Add a **QR Code Toolkit** node (`@cyberace/n8n-nodes-qr.qrToolkit`, v1).
   - Configure the content parameter using the expression: `={{ $json.url }}`.
   - Connect the output of `Create Public File Link` to this node's input.

4. **Create the Email Notification Node:**
   - Add a **Gmail** node (`n8n-nodes-base.gmail`, v2.2).
   - Connect credentials: Link or create a **Gmail OAuth2** credential.
   - Configure parameters:
     - **Send To:** `={{ $('Form: Upload Any File').item.json['Email Address'] }}`
     - **Subject:** `Here is your ready-to-print QR Code!`
     - **Options -> Sender Name:** `Jitesh`
     - **Options -> Attachments:** Enable attachment configuration and select binary attachment handling.
     - **Message (HTML):** Paste the complete styled HTML template containing expressions mapping to `{{ $('Form: Upload Any File').item.json["Project Name"] }}` and `{{ $('Create Public File Link').item.json.url }}`.
   - Connect the output of `Generate QR Code` to this node's input.

5. **Finalize:**
   - Save the workflow, test execution using form preview mode, and verify outgoing emails and attachments.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Instant Document & Image to QR Code Maker Workflow | Core template workflow design integrating form collection, web hosting, matrix barcode generation, and authenticated email delivery. |
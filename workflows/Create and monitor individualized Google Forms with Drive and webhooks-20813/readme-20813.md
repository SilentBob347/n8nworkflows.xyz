Create and monitor individualized Google Forms with Drive and webhooks

https://n8nworkflows.xyz/workflows/create-and-monitor-individualized-google-forms-with-drive-and-webhooks-20813


# Create and monitor individualized Google Forms with Drive and webhooks

### 1. Workflow Overview

This workflow automates the creation, distribution, monitoring, and response extraction of individualized Google Forms for specific customers. It combines Google Drive, Google Forms, Google Cloud Pub/Sub, and n8n data persistence to handle asynchronous user interactions. 

The execution is split into two primary functional paths:
1. **1.1 Form Initialization & Setup:** Triggered manually, this path defines customer parameters, duplicates a master Google Form template via Google Drive API, registers the form ID and execution resume URL in an n8n Data Table, updates form titles/questions, publishes the form, and sets up a response watch via Google Cloud Pub/Sub. The workflow then pauses using a Wait node.
2. **1.2 Response Handling & Processing:** Triggered asynchronously via an n8n webhook receiving Google Cloud Pub/Sub push notifications, this path queries the n8n Data Table for the corresponding execution resume URL, resumes the paused workflow execution, fetches form responses from the Google Forms API, extracts user choices, and passes the result to a placeholder node for custom downstream processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger Start
- **Overview:** Initiates the workflow execution manually on demand.
- **Nodes Involved:** `When Workflow Executed`
- **Node Details:**
  - **When Workflow Executed**
    - Type: `n8n-nodes-base.manualTrigger`
    - Technical Role: Entry point for manual executions.
    - Configuration Choices: Standard default trigger settings.
    - Input/Output: Output connects to `Set Form Parameters`.
    - Edge Cases: None.

#### 2.2 Copy and Store Form
- **Overview:** Sets static parameters for the target customer and template, duplicates the master form inside a specified Google Drive folder, and persists the new form ID and resume URL mapping in a Data Table.
- **Nodes Involved:** `Set Form Parameters`, `Post Copy Form to Drive API`, `Store Form Webhook Info`
- **Node Details:**
  - **Set Form Parameters**
    - Type: `n8n-nodes-base.set`
    - Technical Role: Defines configuration variables for the execution.
    - Configuration Choices: Assigns `template_id`, `custname`, and `parent_folder_id`.
    - Key Expressions: Static placeholder strings requiring user update.
    - Input/Output: Input from `When Workflow Executed`; output connects to `Post Copy Form to Drive API`.
  - **Post Copy Form to Drive API**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Calls the Google Drive REST API to copy a file.
    - Configuration Choices: POST request to `https://www.googleapis.com/drive/v3/files/{{ $json.template_id }}/copy`. Sends JSON body containing the new form name and parent folder ID. Uses generic OAuth2 authentication.
    - Key Expressions: `{{ $json.template_id }}` and `{{ $json.parent_folder_id }}`.
    - Input/Output: Input from `Set Form Parameters`; output connects to `Store Form Webhook Info`.
    - Potential Failure Types: Google API authorization errors, invalid folder ID, or rate limits.
  - **Store Form Webhook Info**
    - Type: `n8n-nodes-base.dataTable`
    - Technical Role: Persists data mapping between the Google Form ID and the n8n execution resume URL.
    - Configuration Choices: Operation set to create/insert row in Data Table `forms_to_hooks`.
    - Key Expressions: `form_id` mapped to `{{ $json.id }}`; `hook_url` mapped to `{{ $execution.resumeUrl }}`.
    - Input/Output: Input from `Post Copy Form to Drive API`; output connects to `Fetch Form Details`.
    - Potential Failure Types: Missing table schema or database constraint violations.

#### 2.3 Form Modification and Publication
- **Overview:** Retrieves metadata for the newly copied form, dynamically updates its title, description, and question options, and publishes it to accept responses.
- **Nodes Involved:** `Fetch Form Details`, `Post Modify Form Fields`, `Post Publish Form Changes`
- **Node Details:**
  - **Fetch Form Details**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Fetches form structure and items via Google Forms API.
    - Configuration Choices: GET request to `https://forms.googleapis.com/v1/forms/{{ $('Post Copy Form to Drive API').item.json.id }}`. Uses generic OAuth2 authentication.
    - Key Expressions: `{{ $('Post Copy Form to Drive API').item.json.id }}`.
    - Input/Output: Input from `Store Form Webhook Info`; output connects to `Post Modify Form Fields`.
    - Potential Failure Types: Form ID not found or API permission issues (`forms.body` scope required).
  - **Post Modify Form Fields**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Updates form questions and info via batch update.
    - Configuration Choices: POST request to `https://forms.googleapis.com/v1/forms/{{ $json.formId }}:batchUpdate`. Modifies title, description, and radio options. Uses generic OAuth2 authentication.
    - Key Expressions: References `{{ $('Set Form Parameters').item.json.custname }}` and `{{ $json.items[0].itemId }}`.
    - Input/Output: Input from `Fetch Form Details`; output connects to `Post Publish Form Changes`.
    - Potential Failure Types: Invalid item ID referencing or malformed JSON body structure.
  - **Post Publish Form Changes**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Publishes the form and enables response acceptance.
    - Configuration Choices: POST request to `https://forms.googleapis.com/v1/forms/{{ $('Fetch Form Details').item.json.formId }}:setPublishSettings`. Sets `isPublished` and `isAcceptingResponses` to true. Uses generic OAuth2 authentication.
    - Key Expressions: `{{ $('Fetch Form Details').item.json.formId }}`.
    - Input/Output: Input from `Post Modify Form Fields`; output connects to `Post Create Form Watch`.
    - Potential Failure Types: API error if form is locked or insufficient permissions.

#### 2.4 Create and Manage Watch
- **Overview:** Registers a webhook subscription (watch) on the Google Form to forward response events to Google Cloud Pub/Sub, and pauses the workflow awaiting form submission.
- **Nodes Involved:** `Post Create Form Watch`, `Send Notification Email`, `Wait for Watch Event`
- **Node Details:**
  - **Post Create Form Watch**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Subscribes to form response notifications via Google Forms API.
    - Configuration Choices: POST request to `https://forms.googleapis.com/v1/forms/{{ $('Fetch Form Details').item.json.formId }}/watches`. Target Pub/Sub topic configuration required. Uses generic OAuth2 authentication.
    - Key Expressions: `{{ $('Fetch Form Details').item.json.formId }}`.
    - Input/Output: Input from `Post Publish Form Changes`; output connects to `Send Notification Email`.
    - Potential Failure Types: Incorrect Google Cloud Pub/Sub topic ARN or missing Pub/Sub publisher permissions for the Google Forms service account.
  - **Send Notification Email**
    - Type: `n8n-nodes-base.noOp`
    - Technical Role: Placeholder node intended for notification dispatch (e.g., email, Slack).
    - Configuration Choices: No-Operation (No configuration).
    - Input/Output: Input from `Post Create Form Watch`; output connects to `Wait for Watch Event`.
  - **Wait for Watch Event**
    - Type: `n8n-nodes-base.wait`
    - Technical Role: Pauses execution until an external webhook resumes it.
    - Configuration Choices: Resume type set to `webhook`.
    - Input/Output: Input from `Send Notification Email`; output connects to `Get Form Response Data` (upon resume).
    - Potential Failure Types: Timeout if no webhook calls resume URL within limits.

#### 2.5 Webhook Trigger Response
- **Overview:** Listens for incoming push notifications from Google Cloud Pub/Sub, matches the incoming form ID to an active workflow execution, and invokes the resume URL.
- **Nodes Involved:** `Google Form Webhook`, `Find Form Webhook Instance`, `Fetch Webhook Callback`
- **Node Details:**
  - **Google Form Webhook**
    - Type: `n8n-nodes-base.webhook`
    - Technical Role: Entry point for external Google Cloud Pub/Sub push notifications.
    - Configuration Choices: HTTP POST method on path `c15852f3-7247-41cb-9c09-61dbf2c69138`.
    - Input/Output: Output connects to `Find Form Webhook Instance`.
    - Potential Failure Types: Invalid webhook payload structure from Pub/Sub.
  - **Find Form Webhook Instance**
    - Type: `n8n-nodes-base.dataTable`
    - Technical Role: Queries the Data Table to find the resume URL associated with the submitted form ID.
    - Configuration Choices: Operation set to `get` with filter where `form_id` matches `{{ $json.body.message.attributes.formId }}`.
    - Key Expressions: `{{ $json.body.message.attributes.formId }}`.
    - Input/Output: Input from `Google Form Webhook`; output connects to `Fetch Webhook Callback`.
    - Potential Failure Types: Record not found if form ID was not properly stored.
  - **Fetch Webhook Callback**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Calls the n8n execution resume URL to wake up the paused workflow branch.
    - Configuration Choices: GET request to `{{ $json.hook_url }}`.
    - Key Expressions: `{{ $json.hook_url }}`.
    - Input/Output: Input from `Find Form Webhook Instance`; output has no direct downstream nodes.
    - Potential Failure Types: Expired or invalid resume URL resulting in 404/410 errors.

#### 2.6 Response Processing
- **Overview:** Resumes execution upon webhook callback, fetches form submission details, extracts the user's selected answer, and routes it to a processing placeholder.
- **Nodes Involved:** `Get Form Response Data`, `Set Choice Data`, `Process Choice Data`
- **Node Details:**
  - **Get Form Response Data**
    - Type: `n8n-nodes-base.httpRequest`
    - Technical Role: Retrieves submitted response data via Google Forms API.
    - Configuration Choices: GET request to `https://forms.googleapis.com/v1/forms/{{ $('Fetch Form Details').item.json.formId }}/responses`. Uses generic OAuth2 authentication.
    - Key Expressions: `{{ $('Fetch Form Details').item.json.formId }}`.
    - Input/Output: Input from `Wait for Watch Event`; output connects to `Set Choice Data`.
    - Potential Failure Types: API errors or scope missing (`forms.responses.readonly`).
  - **Set Choice Data**
    - Type: `n8n-nodes-base.set`
    - Technical Role: Extracts the specific text answer from the raw response payload.
    - Configuration Choices: Assigns extracted value to field `choice`.
    - Key Expressions: `={{ $json.responses[0].answers[$('Fetch Form Details').item.json.items[0].questionItem.question.questionId].textAnswers.answers[0].value }}`.
    - Input/Output: Input from `Get Form Response Data`; output connects to `Process Choice Data`.
    - Potential Failure Types: Index out of bounds if no responses exist or question ID reference mismatches.
  - **Process Choice Data**
    - Type: `n8n-nodes-base.noOp`
    - Technical Role: Placeholder node for custom downstream business logic (e.g., CRM update, email confirmation).
    - Configuration Choices: No-Operation.
    - Input/Output: Input from `Set Choice Data`; terminal node of the primary workflow branch.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Documentation and setup instructions | None | None | ## Individual Google forms... |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Section label for manual trigger start | None | None | ## Manual trigger start |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Section label for copy and store form | None | None | ## Copy and store form |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Section label for form modification | None | None | ## Form modification and publication |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Section label for watch creation | None | None | ## Create and manage watch |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Section label for webhook response handling | None | None | ## Webhook trigger response |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Section label for response processing | None | None | ## Response processing |
| When Workflow Executed | `n8n-nodes-base.manualTrigger` | Entry point for manual executions | None | Set Form Parameters | |
| Set Form Parameters | `n8n-nodes-base.set` | Sets customer and template parameters | When Workflow Executed | Post Copy Form to Drive API | |
| Post Copy Form to Drive API | `n8n-nodes-base.httpRequest` | Duplicates Google Form template via Drive API | Set Form Parameters | Store Form Webhook Info | |
| Store Form Webhook Info | `n8n-nodes-base.dataTable` | Stores form ID and resume URL mapping | Post Copy Form to Drive API | Fetch Form Details | |
| Fetch Form Details | `n8n-nodes-base.httpRequest` | Retrieves form structure and IDs | Store Form Webhook Info | Post Modify Form Fields | |
| Post Modify Form Fields | `n8n-nodes-base.httpRequest` | Updates form title, description, and choices | Fetch Form Details | Post Publish Form Changes | |
| Post Publish Form Changes | `n8n-nodes-base.httpRequest` | Publishes form to accept responses | Post Modify Form Fields | Post Create Form Watch | |
| Post Create Form Watch | `n8n-nodes-base.httpRequest` | Sets up Pub/Sub response watch | Post Publish Form Changes | Send Notification Email | |
| Send Notification Email | `n8n-nodes-base.noOp` | Placeholder for email notifications | Post Create Form Watch | Wait for Watch Event | |
| Wait for Watch Event | `n8n-nodes-base.wait` | Pauses execution until form response webhook | Send Notification Email | Get Form Response Data | |
| Google Form Webhook | `n8n-nodes-base.webhook` | Receives Pub/Sub push notification | None | Find Form Webhook Instance | |
| Find Form Webhook Instance | `n8n-nodes-base.dataTable` | Finds resume URL by form ID | Google Form Webhook | Fetch Webhook Callback | |
| Fetch Webhook Callback | `n8n-nodes-base.httpRequest` | Resumes paused workflow execution | Find Form Webhook Instance | None | |
| Get Form Response Data | `n8n-nodes-base.httpRequest` | Fetches form submissions | Wait for Watch Event | Set Choice Data | |
| Set Choice Data | `n8n-nodes-base.set` | Extracts selected choice from responses | Get Form Response Data | Process Choice Data | |
| Process Choice Data | `n8n-nodes-base.noOp` | Placeholder for custom choice processing | Set Choice Data | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an n8n Data Table:** Create a new Data Table named `forms_to_hooks` with two text columns: `form_id` and `hook_url`.
2. **Configure Credentials:** Set up a Generic OAuth2 API credential in n8n named `Google forms auth` using the following parameters:
   - Grant Type: `Authorization Code`
   - Authorization URL: `https://accounts.google.com/o/oauth2/v2/auth`
   - Access Token URL: `https://oauth2.googleapis.com/token`
   - Scopes: `https://www.googleapis.com/auth/drive https://www.googleapis.com/auth/forms.body https://www.googleapis.com/auth/forms.responses.readonly`
3. **Build Execution Path A (Initialization):**
   - **Step 1:** Add a `When Workflow Executed` (Manual Trigger) node.
   - **Step 2:** Add a `Set` node (`Set Form Parameters`). Add string assignments for `template_id` (your template form ID), `custname` (customer name), and `parent_folder_id` (destination Google Drive folder ID). Connect `When Workflow Executed` to this node.
   - **Step 3:** Add an `HTTP Request` node (`Post Copy Form to Drive API`). Set method to `POST`, URL to `https://www.googleapis.com/drive/v3/files/{{ $json.template_id }}/copy`, and body type to JSON with payload `{"name": "Customer-specific form for {{ $json.custname }}", "parents": ["{{ $json.parent_folder_id }}"]}`. Assign the Google OAuth2 credential. Connect `Set Form Parameters` here.
   - **Step 4:** Add a `Data Table` node (`Store Form Webhook Info`). Select the `forms_to_hooks` table, operation `Append`, mapping `form_id` to `{{ $json.id }}` and `hook_url` to `{{ $execution.resumeUrl }}`. Connect `Post Copy Form to Drive API` here.
   - **Step 5:** Add an `HTTP Request` node (`Fetch Form Details`). Set method to `GET`, URL to `https://forms.googleapis.com/v1/forms/{{ $('Post Copy Form to Drive API').item.json.id }}`. Assign credentials. Connect `Store Form Webhook Info` here.
   - **Step 6:** Add an `HTTP Request` node (`Post Modify Form Fields`). Set method to `POST`, URL to `https://forms.googleapis.com/v1/forms/{{ $json.formId }}:batchUpdate`, and body type to JSON containing requests to update `updateFormInfo` and `updateItem` (RADIO choices). Assign credentials. Connect `Fetch Form Details` here.
   - **Step 7:** Add an `HTTP Request` node (`Post Publish Form Changes`). Set method to `POST`, URL to `https://forms.googleapis.com/v1/forms/{{ $('Fetch Form Details').item.json.formId }}:setPublishSettings`, and body payload to publish and accept responses. Assign credentials. Connect `Post Modify Form Fields` here.
   - **Step 8:** Add an `HTTP Request` node (`Post Create Form Watch`). Set method to `POST`, URL to `https://forms.googleapis.com/v1/forms/{{ $('Fetch Form Details').item.json.formId }}/watches`, and body payload targeting your Google Cloud Pub/Sub topic. Assign credentials. Connect `Post Publish Form Changes` here.
   - **Step 9:** Add a `No-Op` node (`Send Notification Email`). Connect `Post Create Form Watch` here.
   - **Step 10:** Add a `Wait` node (`Wait for Watch Event`). Set resume mechanism to `Webhook`. Connect `Send Notification Email` here.
4. **Build Execution Path B (Webhook Listener):**
   - **Step 11:** Add a `Webhook` node (`Google Form Webhook`). Set HTTP method to `POST`.
   - **Step 12:** Add a `Data Table` node (`Find Form Webhook Instance`). Select table `forms_to_hooks`, operation `Get`, with filter condition `form_id` equal to `{{ $json.body.message.attributes.formId }}`. Connect `Google Form Webhook` here.
   - **Step 13:** Add an `HTTP Request` node (`Fetch Webhook Callback`). Set method to `GET` and URL to `{{ $json.hook_url }}`. Connect `Find Form Webhook Instance` here.
5. **Build Execution Path C (Response Processing - Resumed):**
   - **Step 14:** Connect the output of `Wait for Watch Event` to an `HTTP Request` node (`Get Form Response Data`). Set method to `GET`, URL to `https://forms.googleapis.com/v1/forms/{{ $('Fetch Form Details').item.json.formId }}/responses`. Assign credentials.
   - **Step 15:** Add a `Set` node (`Set Choice Data`). Assign `choice` using expression: `={{ $json.responses[0].answers[$('Fetch Form Details').item.json.items[0].questionItem.question.questionId].textAnswers.answers[0].value }}`. Connect `Get Form Response Data` here.
   - **Step 16:** Add a `No-Op` node (`Process Choice Data`). Connect `Set Choice Data` here as the terminal step.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Cloud Pub/Sub Setup | Ensure a Pub/Sub topic and push subscription pointing to the n8n webhook URL are configured in the Google Cloud Console. |
| OAuth2 Scopes Requirement | Credentials must include `drive`, `forms.body`, and `forms.responses.readonly` scopes. |
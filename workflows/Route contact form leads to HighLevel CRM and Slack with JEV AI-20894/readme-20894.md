Route contact form leads to HighLevel CRM and Slack with JEV AI

https://n8nworkflows.xyz/workflows/route-contact-form-leads-to-highlevel-crm-and-slack-with-jev-ai-20894


# Route contact form leads to HighLevel CRM and Slack with JEV AI

### 1. Workflow Overview

The purpose of this workflow is to capture incoming contact form submissions, synchronize contact data with GoHighLevel (GHL) CRM, classify the lead using an external AI routing engine (BeatAPI / JEV model), create appropriate CRM opportunities based on classification, and notify the corresponding teams via Slack.

The workflow is divided into the following logical blocks:
- **1.1 Input Reception & Validation:** Captures form entries and validates that an email address is present.
- **1.2 CRM Contact Resolution:** Queries GHL for existing contacts and upserts new records if necessary.
- **1.3 AI Lead Classification:** Submits lead details to BeatAPI to route inquiries to the sales team, partnership team, or ignore them.
- **1.4 Sales Route Processing:** Fetches existing sales opportunities, checks their status, creates a new opportunity if missing, and notifies the sales channel on Slack.
- **1.5 Partnership Route Processing:** Fetches existing partnership opportunities, checks their status, creates a new opportunity if missing, and notifies the operations/partnership channel on Slack.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
- **Overview:** Receives contact data from a published n8n web form and ensures a valid email address is provided before proceeding.
- **Nodes Involved:** 
  - `When Form Submitted`
  - `If Email Is Provided`
- **Node Details:**
  - **When Form Submitted**
    - *Type & Technical Role:* `n8n-nodes-base.formTrigger` (Webhook/Trigger node)
    - *Configuration:* Captures form fields: `Your Name`, `email`, `reason`, and `Phone number`.
    - *Key Expressions:* None (triggers on form submit).
    - *Input/Output:* No inputs; outputs form submission payload to `If Email Is Provided`.
    - *Edge Cases/Failures:* Submissions without an email fail validation downstream.
  - **If Email Is Provided**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Checks if `{{ $json.email }}` is not empty.
    - *Key Expressions:* `{{ $json.email }}`
    - *Input/Output:* Input from `When Form Submitted`; outputs to `Retrieve Contact from GHL`.
    - *Edge Cases/Failures:* Halts execution if the email field is blank.

#### 2.2 CRM Contact Resolution
- **Overview:** Searches GoHighLevel for an existing contact matching the email, evaluates if the contact exists, and upserts the contact if no match is found.
- **Nodes Involved:**
  - `Retrieve Contact from GHL`
  - `If Contact Exists`
  - `Upsert Contact in GHL`
- **Node Details:**
  - **Retrieve Contact from GHL**
    - *Type & Technical Role:* `n8n-nodes-base.highLevel` (CRM Integration)
    - *Configuration:* Retrieves contacts (`getAll`) filtering by email query.
    - *KeyExpressions:* `={{ $json.email }}`
    - *Input/Output:* Input from `If Email Is Provided`; outputs to `If Contact Exists`.
    - *Credentials:* `HighLevel account` (OAuth2)
  - **If Contact Exists**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Validates whether the returned GHL contact object contains an ID (`{{ $json.id }}`).
    - *Key Expressions:* `={{ $json.id }}`
    - *Input/Output:* Input from `Retrieve Contact from GHL`; outputs True branch to `Post to Jev Routing API` and False branch to `Upsert Contact in GHL`.
  - **Upsert Contact in GHL**
    - *Type & Technical Role:* `n8n-nodes-base.highLevel` (CRM Integration)
    - *Configuration:* Creates or updates contact records in GHL.
    - *Key Expressions:* `={{ $('When Voice Recording Received').item.json.body.email }}` (Note: References form submission context).
    - *Input/Output:* Input from `If Contact Exists` (False branch); outputs to `Post to Jev Routing API`.
    - *Credentials:* `HighLevel account` (OAuth2)

#### 2.3 AI Lead Classification
- **Overview:** Sends lead data to the BeatAPI (JEV model) to classify the inquiry intent into a specific routing category.
- **Nodes Involved:**
  - `Post to Jev Routing API`
  - `Route by Jev Rules`
- **Node Details:**
  - **Post to Jev Routing API**
    - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (API Request)
    - *Configuration:* POST request to `https://api.beatapi.io/v1/systemone` using HTTP Header authentication. Sends JSON body containing ICP descriptions, lead name, reason, and classification rules.
    - *Key Expressions:* Lead name and reason expressions tied to the form submission.
    - *Input/Output:* Input from `If Contact Exists` or `Upsert Contact in GHL`; outputs to `Route by Jev Rules`.
    - *Credentials:* `beatapi` (HTTP Header Auth)
    - *Edge Cases/Failures:* API timeouts or authentication failures will halt classification.
  - **Route by Jev Rules**
    - *Type & Technical Role:* `n8n-nodes-base.switch` (Flow Control)
    - *Configuration:* Routes execution based on `{{ $json.answers.route_to.choice }}`. Matches "sales team", "partnership team", or "ignore".
    - *Key Expressions:* `={{ $json.answers.route_to.choice }}`
    - *Input/Output:* Input from `Post to Jev Routing API`; outputs Route 0 (`sales team`) to `Fetch GHL Opportunities`, Route 1 (`partnership team`) to `Retrieve Additional GHL Opportunities`.

#### 2.4 Sales Route Processing
- **Overview:** Fetches GHL opportunities for the sales branch, evaluates if a new opportunity is required, creates it in the sales pipeline, and posts a notification to Slack.
- **Nodes Involved:**
  - `Fetch GHL Opportunities`
  - `Classify GHL Opportunities`
  - `If Opportunity Is Active`
  - `Add to Sales Pipeline`
  - `Prepare Sales Slack Message`
  - `Post to Sales Slack`
- **Node Details:**
  - **Fetch GHL Opportunities**
    - *Type & Technical Role:* `n8n-nodes-base.highLevel` (CRM Integration)
    - *Configuration:* Retrieves all opportunities matching the lead's email.
    - *Key Expressions:* `={{ $('If Email Is Provided').item.json.email }}`
  - **Classify GHL Opportunities**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Normalizes phone numbers, deduplicates records, and matches opportunities against contacts by ID, phone, or email to set `lead_status` to `new_opportunity` or `old_opportunity`.
  - **If Opportunity Is Active**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Checks if `lead_status` equals `new_opportunity`.
  - **Add to Sales Pipeline**
    - *Type & Technical Role:* `n8n-nodes-base.highLevel` (CRM Integration)
    - *Configuration:* Creates an opportunity using `pipelineId` (`smEMRhNOilT5Pm0QZCWO`) and `stageId` (`f0fed07b-9189-4970-88e6-5dc55357347b`).
  - **Prepare Sales Slack Message**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Assigns a formatted Markdown string containing lead details, AI classification confidence, and CRM IDs to `slack_message`.
  - **Post to Sales Slack**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Messaging Integration)
    - *Configuration:* Sends message to the specified Slack channel ID (`C0B8VH1M5PX`).
    - *Credentials:* `Slack account 2` (Slack OAuth2/API)

#### 2.5 Partnership Route Processing
- **Overview:** Fetches GHL opportunities for the partnership branch, evaluates status, creates an opportunity in the partnership pipeline, and notifies the partnership Slack channel.
- **Nodes Involved:**
  - `Retrieve Additional GHL Opportunities`
  - `Organize Additional Opportunities`
  - `Check Additional Opportunity Status`
  - `Add to Opportunity Pipeline`
  - `Prepare Partnership Slack Message`
  - `Post to Partnership Slack`
- **Node Details:**
  - **Retrieve Additional GHL Opportunities**
    - *Type & Technical Role:* `n8n-nodes-base.highLevel` (CRM Integration)
    - *Configuration:* Retrieves opportunities for the partnership path.
  - **Organize Additional Opportunities**
    - *Type & Technical Role:* `n8n-nodes-base.code` (JavaScript execution)
    - *Configuration:* Identical logic to `Classify GHL Opportunities`, resolving contact mapping and setting lead status.
  - **Check Additional Opportunity Status**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Validates if `lead_status` equals `new_opportunity`.
  - **Add to Opportunity Pipeline**
    - *Type & Technical Role:* `n8n-nodes-base.highLevel` (CRM Integration)
    - *Configuration:* Creates an opportunity in the partnership pipeline (`pipelineId`: `P8DK5W78aOu5jMBHuyPZ`, `stageId`: `9d6d7c43-a631-450a-838f-0bf62011bd79`).
  - **Prepare Partnership Slack Message**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation)
    - *Configuration:* Formats a partnership-specific Slack notification payload.
  - **Post to Partnership Slack**
    - *Type & Technical Role:* `n8n-nodes-base.slack` (Messaging Integration)
    - *Configuration:* Posts message to Slack channel ID `C0B8VH1M5PX`.
    - *Credentials:* `Slack account 2` (Slack OAuth2/API)

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Workflow overview, setup steps, customization, and link | None | None | ## Contact Form to GHL CRM sync and Lead Routing by JEV, and creating oppurtunities in CRM and notify sales and opporational teams... @[youtube](dmzbxqZRGxs) |
| Sticky Note1 | n8n-nodes-base.stickyNote | Overview of form reception and validation | None | None | ## Receive and validate form<br><br>Starts the workflow from a contact form submission and checks that an email address was provided before attempting CRM lookup. |
| Sticky Note2 | n8n-nodes-base.stickyNote | Overview of CRM contact resolution | None | None | ## Resolve CRM contact<br><br>Looks up the submitted contact in GoHighLevel, determines whether a matching contact exists, and upserts contact details when needed. |
| Sticky Note3 | n8n-nodes-base.stickyNote | Overview of lead routing via JEV API | None | None | ## Route lead by JEV<br><br>Sends the lead details to the JEV routing API and uses the returned result to split the workflow into the correct downstream path. |
| Sticky Note4 | n8n-nodes-base.stickyNote | Overview of sales opportunities evaluation | None | None | ## Evaluate sales opportunities<br><br>Top branch that fetches GoHighLevel opportunities, categorizes them, and checks whether the lead should be pushed into the sales pipeline. |
| Sticky Note5 | n8n-nodes-base.stickyNote | Overview of partnership opportunities evaluation | None | None | ## Evaluate partnership opportunities<br><br>Bottom branch that fetches GoHighLevel opportunities, categorizes them, and checks whether the lead should be pushed into the partnership or opportunity pipeline. |
| Sticky Note6 | n8n-nodes-base.stickyNote | Overview of sales follow-up creation and Slack notification | None | None | ## Create sales follow-up<br><br>Creates or updates the lead in the sales pipeline, generates a Slack notification message, and sends it to the sales team channel. |
| Sticky Note7 | n8n-nodes-base.stickyNote | Overview of partnership follow-up creation and Slack notification | None | None | ## Create partnership follow-up<br><br>Creates or updates the lead in the opportunity/partnership pipeline, prepares the Slack message, and notifies the partnership channel. |
| Retrieve Contact from GHL | n8n-nodes-base.highLevel | Query GHL contacts by email | If Email Is Provided | If Contact Exists | |
| If Email Is Provided | n8n-nodes-base.if | Verify email is not empty | When Form Submitted | Retrieve Contact from GHL | |
| If Contact Exists | n8n-nodes-base.if | Check if GHL contact ID exists | Retrieve Contact from GHL | Post to Jev Routing API, Upsert Contact in GHL | |
| Upsert Contact in GHL | n8n-nodes-base.highLevel | Create or update contact in GHL | If Contact Exists | Post to Jev Routing API | |
| When Form Submitted | n8n-nodes-base.formTrigger | Capture form trigger inputs | None | If Email Is Provided | |
| Route by Jev Rules | n8n-nodes-base.switch | Route execution based on AI category | Post to Jev Routing API | Fetch GHL Opportunities, Retrieve Additional GHL Opportunities | |
| Fetch GHL Opportunities | n8n-nodes-base.highLevel | Retrieve GHL opportunities for sales | Route by Jev Rules | Classify GHL Opportunities | |
| Classify GHL Opportunities | n8n-nodes-base.code | Normalize, deduplicate, and match opportunities | Fetch GHL Opportunities | If Opportunity Is Active | |
| If Opportunity Is Active | n8n-nodes-base.if | Check if opportunity is new | Classify GHL Opportunities | Add to Sales Pipeline | |
| Retrieve Additional GHL Opportunities | n8n-nodes-base.highLevel | Retrieve GHL opportunities for partnership | Route by Jev Rules | Organize Additional Opportunities | |
| Organize Additional Opportunities | n8n-nodes-base.code | Normalize and deduplicate partnership opportunities | Retrieve Additional GHL Opportunities | Check Additional Opportunity Status | |
| Check Additional Opportunity Status | n8n-nodes-base.if | Check if partnership opportunity is new | Organize Additional Opportunities | Add to Opportunity Pipeline | |
| Prepare Sales Slack Message | n8n-nodes-base.set | Format sales notification message | Add to Sales Pipeline | Post to Sales Slack | |
| Prepare Partnership Slack Message | n8n-nodes-base.set | Format partnership notification message | Add to Opportunity Pipeline | Post to Partnership Slack | |
| Post to Partnership Slack | n8n-nodes-base.slack | Send Slack notification to partnership team | Prepare Partnership Slack Message | None | |
| Post to Sales Slack | n8n-nodes-base.slack | Send Slack notification to sales team | Prepare Sales Slack Message | None | |
| Add to Sales Pipeline | n8n-nodes-base.highLevel | Create opportunity in sales pipeline | If Opportunity Is Active | Prepare Sales Slack Message | |
| Add to Opportunity Pipeline | n8n-nodes-base.highLevel | Create opportunity in partnership pipeline | Check Additional Opportunity Status | Prepare Partnership Slack Message | |
| Post to Jev Routing API | n8n-nodes-base.httpRequest | Call BeatAPI for intent classification | If Contact Exists, Upsert Contact in GHL | Route by Jev Rules | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:** Add a `When Form Submitted` node (`n8n-nodes-base.formTrigger`). Configure form fields: `Your Name`, `email`, `reason`, and `Phone number`.
2. **Add Email Validation:** Create an `If` node named `If Email Is Provided`. Set condition to check that `{{ $json.email }}` is not empty. Connect `When Form Submitted` output to this node.
3. **Retrieve Contact from CRM:** Add a GoHighLevel node (`n8n-nodes-base.highLevel`) named `Retrieve Contact from GHL`. Set operation to `getAll` with query `={{ $json.email }}` using the `HighLevel account` OAuth2 credential. Connect the true output of `If Email Is Provided`.
4. **Evaluate Contact Existence:** Add an `If` node named `If Contact Exists`. Check if `{{ $json.id }}` is not empty. Connect `Retrieve Contact from GHL` output here.
5. **Upsert Contact (False Branch):** Add a GoHighLevel node named `Upsert Contact in GHL`. Configure it to save contact data and connect the false output of `If Contact Exists`.
6. **AI Routing API Request:** Add an HTTP Request node named `Post to Jev Routing API`. Configure method as `POST`, URL as `https://api.beatapi.io/v1/systemone`, and use generic HTTP Header Auth (`beatapi`). Pass the JSON body containing the model payload (`jev-1.13-free`) and rules. Connect both the true output of `If Contact Exists` and the output of `Upsert Contact in GHL` into this node.
7. **Switch Routing Node:** Add a Switch node named `Route by Jev Rules`. Configure rules to evaluate `{{ $json.answers.route_to.choice }}` for values `sales team` and `partnership team`. Connect `Post to Jev Routing API` output here.
8. **Sales Branch Setup:**
   - Add a GHL node `Fetch GHL Opportunities` (operation: `getAll`, filter query matching lead email). Connect to Rule 0 output.
   - Add a Code node `Classify GHL Opportunities` running the deduplication and matching JavaScript provided in the workflow definition.
   - Add an `If` node `If Opportunity Is Active` checking `{{ $json.lead_status }} === 'new_opportunity'`.
   - Add a GHL node `Add to Sales Pipeline` (resource: `opportunity`, pipeline ID `smEMRhNOilT5Pm0QZCWO`, stage ID `f0fed07b-9189-4970-88e6-5dc55357347b`).
   - Add a Set node `Prepare Sales Slack Message` formatting the markdown message string.
   - Add a Slack node `Post to Sales Slack` using credentials `Slack account 2` targeting channel ID `C0B8VH1M5PX`.
9. **Partnership Branch Setup:**
   - Add a GHL node `Retrieve Additional GHL Opportunities` connected to Rule 1 output.
   - Add a Code node `Organize Additional Opportunities` with the matching script.
   - Add an `If` node `Check Additional Opportunity Status` checking `{{ $json.lead_status }} === 'new_opportunity'`.
   - Add a GHL node `Add to Opportunity Pipeline` (resource: `opportunity`, pipeline ID `P8DK5W78aOu5jMBHuyPZ`, stage ID `9d6d7c43-a631-450a-838f-0bf62011bd79`).
   - Add a Set node `Prepare Partnership Slack Message` formatting the partnership markdown message string.
   - Add a Slack node `Post to Partnership Slack` using credentials `Slack account 2` targeting channel ID `C0B8VH1M5PX`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Youtube Video Walkthrough | [https://youtu.be/dmzbxqZRGxs](https://youtu.be/dmzbxqZRGxs) |
| BeatAPI Endpoint | `https://api.beatapi.io/v1/systemone` |
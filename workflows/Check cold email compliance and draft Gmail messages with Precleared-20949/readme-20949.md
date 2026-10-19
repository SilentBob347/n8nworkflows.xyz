Check cold email compliance and draft Gmail messages with Precleared

https://n8nworkflows.xyz/workflows/check-cold-email-compliance-and-draft-gmail-messages-with-precleared-20949


# Check cold email compliance and draft Gmail messages with Precleared

### 1. Workflow Overview

This workflow automates the process of checking cold-email compliance for a batch of leads using the [Precleared](https://precleared.dev) API. It ensures that emails are only drafted if permitted by regional regulations (such as UK PECR, GDPR, US CAN-SPAM, Canada's CASL, and German/French laws). Compliant leads generate customized Gmail drafts including mandatory footers and receipt IDs, while non-compliant leads are held back with clear reasoning and suggested remediation steps.

The workflow logic is grouped into three main functional blocks:
- **1.1 Input Reception & Configuration:** Initializes global sender parameters and generates or loads the target lead batch.
- **1.2 Compliance Verification:** Processes each lead sequentially through the Precleared API to evaluate regulatory admissibility.
- **1.3 Conditional Drafting & Routing:** Splits the flow based on the compliance verdict, either creating a Gmail draft with required disclosures or isolating the lead for review.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Configuration
- **Overview:** Establishes sender metadata (corporate address, pitching details) and provides a sample lead dataset to feed the processing pipeline.
- **Nodes Involved:** `Start`, `Your settings`, `Sample leads`, `Map lead fields`
- **Node Details:**
  - **Start**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` — Manual execution entry point.
    - *Configuration Choices:* Standard manual trigger configuration.
    - *Key Expressions:* None.
    - *Input/Output:* Output connected to `Your settings`.
    - *Edge Cases:* None.
  - **Your settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields) — Defines global sender attributes.
    - *Configuration Choices:* Sets static strings for `sender_country`, `your_name`, `your_company`, `postal_address`, `opt_out_line`, and `pitch`.
    - *Key Expressions:* None.
    - *Input/Output:* Input from `Start`; output connected to `Sample leads`.
    - *Edge Cases:* Missing required fields may cause compliance check failures downstream.
  - **Sample leads**
    - *Type and Technical Role:* `n8n-nodes-base.code` — JavaScript execution node returning 3 static test records.
    - *Configuration Choices:* Outputs objects containing `email`, `first_name`, `company`, and `address_source`.
    - *Key Expressions:* None.
    - *Input/Output:* Input from `Your settings`; output connected to `Map lead fields`.
    - *Edge Cases:* Custom database or CRM integrations should replace this node.
  - **Map lead fields**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields) — Normalizes lead properties.
    - *Configuration Choices:* Maps incoming fields to standard data names.
    - *Key Expressions:* `={{ $json.email }}`, `={{ $json.first_name }}`, `={{ $json.company }}`, `={{ $json.address_source || 'unknown' }}`.
    - *Input/Output:* Input from `Sample leads`; output connected to `Check with Precleared`.
    - *Edge Cases:* Unmapped or null email fields will cause HTTP request errors.

#### Block 1.2: Compliance Verification
- **Overview:** Sends lead payloads and sender context to the external Precleared compliance engine and assesses the returned verdict.
- **Nodes Involved:** `Check with Precleared`, `Allowed to email?`
- **Node Details:**
  - **Check with Precleared**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` — External API call to the Precleared compliance service.
    - *Configuration Choices:* POST method to `https://precleared.dev/api/v1/check`. Configured with batching (batch size 5, interval 1000ms) and response error suppression (`neverError: true`).
    - *Key Expressions:* `recipient_email` (`={{ $json.email }}`), `address_source` (`={{ $json.address_source }}`), `sender_country` (`={{ $('Your settings').first().json.sender_country }}`), `pitch` (`={{ $('Your settings').first().json.pitch }}`).
    - *Input/Output:* Input from `Map lead fields`; output connected to `Allowed to email?`.
    - *Edge Cases:* Rate limiting (free tier restrictions) or network timeouts. Optional bearer token authentication (`Authorization: Bearer ...`) required for higher volumes.
  - **Allowed to email?**
    - *Type and Technical Role:* `n8n-nodes-base.if` — Conditional router based on API output.
    - *Configuration Choices:* Evaluates if `verdict` equals `ok_to_email`.
    - *Key Expressions:* `={{ $json.verdict }}`.
    - *Input/Output:* Input from `Check with Precleared`; outputs `true` branch to `Write the email`, `false` branch to `Hold back`.
    - *Edge Cases:* Malformed API responses resulting in missing verdict properties route to the false branch.

#### Block 1.3: Conditional Drafting & Routing
- **Overview:** Generates email content incorporating legal footer requirements for compliant leads, or captures failure reasons for non-compliant leads.
- **Nodes Involved:** `Write the email`, `Create Gmail draft`, `Hold back`
- **Node Details:**
  - **Write the email**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields) — Assembles email subject, body, and compliance metadata.
    - *Configuration Choices:* Constructs formatted text templates.
    - *Key Expressions:* Uses item-linking expressions referencing `Map lead fields`, `Your settings`, and `Check with Precleared` (e.g., `={{ $('Map lead fields').item.json.email }}`, `={{ $json.footer_text }}`, `={{ $json.receipt.id }}`).
    - *Input/Output:* Input from `Allowed to email?` (true branch); output connected to `Create Gmail draft`.
    - *Edge Cases:* Missing fallback values for optional lead properties.
  - **Create Gmail draft**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` — Creates an email draft inside the connected mailbox.
    - *Configuration Choices:* Resource: `draft`, Email Type: `text`.
    - *Key Expressions:* `message` (`={{ $json.body }}`), `sendTo` (`={{ $json.to }}`), `subject` (`={{ $json.subject }}`).
    - *Input/Output:* Input from `Write the email`; terminal node.
    - *Credentials:* Requires valid Google OAuth2 / Gmail API authentication.
    - *Edge Cases:* Authentication expiration, invalid recipient address formats.
  - **Hold back**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Edit Fields) — Aggregates rejection details, rule violations, and remediation URLs.
    - *Configuration Choices:* Maps error text, verdicts, and receipts.
    - *Key Expressions:* `verdict` (`={{ $json.verdict || 'not checked' }}`), `reason` (`={{ ($json.reasons && $json.reasons[0] && ($json.reasons[0].text || $json.reasons[0].note)) || $json.message || '' }}`), `next_step` (`={{ $json.next_step || ($json.get_key_url ? 'Get a free key: ' + $json.get_key_url : '') || ($json.buy_url ? 'Buy a pack: ' + $json.buy_url : '') }}`).
    - *Input/Output:* Input from `Allowed to email?` (false branch); terminal node.
    - *Edge Cases:* Unhandled nested object structures in the error response.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Sticky Note` | `n8n-nodes-base.stickyNote` | Documentation | None | None | ## Check cold email compliance before drafting in Gmail<br><br>Checks every lead with [Precleared](https://precleared.dev) before an email is written, so you only draft emails the law allows: UK PECR and GDPR, US CAN-SPAM, Canada's CASL, Germany's UWG and French rules.<br><br>### How it works<br>1. **Sample leads** gives three test leads. Swap in Google Sheets, Airtable or your CRM.<br>2. **Check with Precleared** asks whether a cold email to each address is allowed. It returns a verdict, the rule with its source, what the footer must include and a signed receipt id.<br>3. If the verdict is `ok_to_email`, **Write the email** adds your footer (company, postal address, opt-out line) and keeps Precleared's requirement as `footer_rule`. **Create Gmail draft** saves it for you to review.<br>4. Anything else goes to **Hold back**, with the reason and the next step.<br><br>### Setup<br>1. Fill in **Your settings**: your country (GB or US), name, company, postal address, opt-out line and what you offer.<br>2. Connect Gmail in **Create Gmail draft**.<br>3. Run it. The sample domains have no mail server, so they land in **Hold back**. Then point **Map lead fields** at your own leads.<br><br>No account needed: 10 free checks a day. A free key from [precleared.dev/get-key](https://precleared.dev/get-key) gives 25 a day, and packs add more. To use a key, set **Check with Precleared** to Header Auth with the name `Authorization` and the value `Bearer pc_live_...`.<br><br>### Customization tips<br>- Change the wording in **Write the email**, and check your footer meets `footer_rule`.<br>- Switch the draft to a send once you trust it.<br>- Save the receipt id with each lead as proof of the check.<br><br>Precleared is decision support with sources, not legal advice. |
| `Section: your leads` | `n8n-nodes-base.stickyNote` | Section Grouping | None | None | ## 1. Your leads<br>Add your details, then map your lead source to email, first name, company and where each address came from. |
| `Section: check` | `n8n-nodes-base.stickyNote` | Section Grouping | None | None | ## 2. Check each address<br>One Precleared check per lead. Only `ok_to_email` goes on to an email. |
| `Section: draft or hold` | `n8n-nodes-base.stickyNote` | Section Grouping | None | None | ## 3. Draft or hold<br>Allowed leads become Gmail drafts with your footer. The rest are held with the reason and the next step. |
| `Start` | `n8n-nodes-base.manualTrigger` | Trigger execution | None | `Your settings` | |
| `Your settings` | `n8n-nodes-base.set` | Define sender settings | `Start` | `Sample leads` | |
| `Sample leads` | `n8n-nodes-base.code` | Generate sample data | `Your settings` | `Map lead fields` | |
| `Map lead fields` | `n8n-nodes-base.set` | Normalize lead fields | `Sample leads` | `Check with Precleared` | |
| `Check with Precleared` | `n8n-nodes-base.httpRequest` | Verify compliance API | `Map lead fields` | `Allowed to email?` | |
| `Allowed to email?` | `n8n-nodes-base.if` | Route by verdict | `Check with Precleared` | `Write the email`, `Hold back` | |
| `Write the email` | `n8n-nodes-base.set` | Format email payload | `Allowed to email?` | `Create Gmail draft` | |
| `Create Gmail draft` | `n8n-nodes-base.gmail` | Save draft to Gmail | `Write the email` | None | |
| `Hold back` | `n8n-nodes-base.set` | Capture rejection details | `Allowed to email?` | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger:** Add a **Manual Trigger** (`n8n-nodes-base.manualTrigger`) node named `Start`.
2. **Configure Sender Settings:** Add an **Edit Fields (Set)** node named `Your settings`. Connect `Start` to it. Configure assignments:
   - `sender_country` (String) = `GB`
   - `your_name` (String) = `Your name`
   - `your_company` (String) = `Your Company Ltd`
   - `postal_address` (String) = `Your full postal address`
   - `opt_out_line` (String) = `If you'd rather not hear from me, reply STOP and I won't email you again.`
   - `pitch` (String) = `bookkeeping software for small agencies`
3. **Add Sample Data:** Add a **Code** (`n8n-nodes-base.code`) node named `Sample leads`. Connect `Your settings` to it. Insert the JavaScript snippet returning an array of sample lead objects containing `email`, `first_name`, `company`, and `address_source`.
4. **Map Lead Fields:** Add an **Edit Fields (Set)** node named `Map lead fields`. Connect `Sample leads` to it. Map incoming parameters:
   - `email` = `={{ $json.email }}`
   - `first_name` = `={{ $json.first_name }}`
   - `company` = `={{ $json.company }}`
   - `address_source` = `={{ $json.address_source || 'unknown' }}`
5. **Set up API Verification:** Add an **HTTP Request** (`n8n-nodes-base.httpRequest`) node named `Check with Precleared`. Connect `Map lead fields` to it.
   - Method: `POST`
   - URL: `https://precleared.dev/api/v1/check`
   - Body Parameters: Add `recipient_email` (`={{ $json.email }}`), `address_source` (`={{ $json.address_source }}`), `sender_country` (`={{ $('Your settings').first().json.sender_country }}`), and `pitch` (`={{ $('Your settings').first().json.pitch }}`).
   - Options: Enable response option `Never Error`. Set batching options (Batch Size: `5`, Batch Interval: `1000`). Configure optional Header Auth (`Authorization: Bearer <API_KEY>`) if using a paid/free key.
6. **Add Conditional Router:** Add an **If** (`n8n-nodes-base.if`) node named `Allowed to email?`. Connect `Check with Precleared` to it. Set condition: `{{ $json.verdict }}` equals `ok_to_email`.
7. **Configure Compliant Branch:**
   - Add an **Edit Fields (Set)** node named `Write the email`. Connect the `true` output of `Allowed to email?` to it. Map fields:
     - `to` = `={{ $('Map lead fields').item.json.email }}`
     - `subject` = `=A question for {{ $('Map lead fields').item.json.company }}`
     - `body` = Compose template incorporating sender name, company, pitch, postal address, and opt-out line.
     - `footer_rule` = `={{ $json.footer_text }}`
     - `receipt_id` = `={{ $json.receipt.id }}`
   - Add a **Gmail** (`n8n-nodes-base.gmail`) node named `Create Gmail draft`. Connect `Write the email` to it.
     - Resource: `Draft`
     - Operation: `Create`
     - Subject: `={{ $json.subject }}`
     - Message: `={{ $json.body }}`
     - Send To: `={{ $json.to }}`
     - *Credentials:* Configure Google OAuth2.
8. **Configure Hold-Back Branch:**
   - Add an **Edit Fields (Set)** node named `Hold back`. Connect the `false` output of `Allowed to email?` to it. Map fields:
     - `email` = `={{ $('Map lead fields').item.json.email }}`
     - `verdict` = `={{ $json.verdict || 'not checked' }}`
     - `reason` = `={{ ($json.reasons && $json.reasons[0] && ($json.reasons[0].text || $json.reasons[0].note)) || $json.message || '' }}`
     - `next_step` = `={{ $json.next_step || ($json.get_key_url ? 'Get a free key: ' + $json.get_key_url : '') || ($json.buy_url ? 'Buy a pack: ' + $json.buy_url : '') }}`
     - `receipt_id` = `={{ $json.receipt ? $json.receipt.id : '' }}`

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Precleared compliance service homepage | [Precleared Website](https://precleared.dev) |
| Free API key registration | [Get Free Key](https://precleared.dev/get-key) |
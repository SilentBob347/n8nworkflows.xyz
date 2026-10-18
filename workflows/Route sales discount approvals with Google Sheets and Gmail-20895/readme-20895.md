Route sales discount approvals with Google Sheets and Gmail

https://n8nworkflows.xyz/workflows/route-sales-discount-approvals-with-google-sheets-and-gmail-20895


# Route sales discount approvals with Google Sheets and Gmail

### 1. Workflow Overview

The **Sales Deal Desk & Discount Approval Control System** automates the end-to-end lifecycle of sales deal submissions and discount requests. It eliminates manual routing delays by capturing deal proposals via an n8n Form, comparing them against pricing thresholds stored in Google Sheets, assigning the correct approval tier (Auto, Manager, or Senior), logging audits, sending out requests, and processing subsequent approver decisions via a secondary form interface.

The workflow is categorized into the following logical blocks:
- **1.1 Deal Intake & Policy Lookup:** Receives incoming sales proposals, normalizes parameters, and queries Google Sheets for matching product/service pricing policies.
- **1.2 Policy Validation & Routing Logic:** Evaluates discount percentages and gross margins against policy limits to branch the workflow into Auto-Approval, Manager-Approval, or Senior-Approval paths, handling missing policies as exceptions.
- **1.3 Audit Logging & Notification Dispatch:** Appends deal audit details to a centralized tracking sheet and triggers initial emails either confirming auto-approval or requesting review from human approvers.
- **1.4 Approval Decision Processing:** Captures manager or senior leadership decisions via a dedicated feedback form, updates the centralized deal log accordingly, and communicates the final outcome to the sales representative.

---

### 2. Block-by-Block Analysis

#### 2.1 Deal Intake & Policy Lookup
- **Overview:** Captures form submissions from sales reps, formats the raw inputs with calculated fields and timestamps, and fetches corresponding policy definitions from Google Sheets.
- **Nodes Involved:** 
  - `When Deal Form Submitted`
  - `Set Deal Details`
  - `Read Pricing Policy in Sheets`

- **Node Details:**
  - **When Deal Form Submitted**
    - *Type & Technical Role:* `n8n-nodes-base.formTrigger` (Trigger Node). Creates an accessible web form to collect deal variables (e.g., product/service name, proposed discount, expected gross margin).
    - *Configuration Choices:* Standard webhook-backed form trigger configuration.
    - *Key Expressions/Variables:* Collects user-submitted form payload fields.
    - *Input/Output Connections:* Output connects to `Set Deal Details`.
    - *Failure Types:* Webhook connectivity drops or invalid payload types submitted by users.
  - **Set Deal Details**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation). Normalizes input data, adds system timestamps, generates unique deal identifiers, and prepares payload properties for validation.
    - *Configuration Choices:* Assigns explicit properties to map incoming form fields to standardized internal variables.
    - *Key Expressions/Variables:* Uses expressions to generate unique IDs and parse numeric values for margins and discounts.
    - *Input/Output Connections:* Input from `When Deal Form Submitted`; output to `Read Pricing Policy in Sheets`.
    - *Failure Types:* Expression evaluation errors if form inputs contain unexpected string formats instead of numbers.
  - **Read Pricing Policy in Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Integration). Queries the `Pricing_Policy` spreadsheet using the submitted product or service identifier.
    - *Configuration Choices:* Configured to read rows (`alwaysOutputData: true`) to ensure downstream nodes still fire if no matching policy exists.
    - *Key Expressions/Variables:* Filters or searches rows based on the product/service name provided in the previous step.
    - *Input/Output Connections:* Input from `Set Deal Details`; output to `Check Policy Existence`.
    - *Credentials:* Google Sheets OAuth2 / Service Account.
    - *Failure Types:* API rate limits, invalid Spreadsheet ID or GID configuration, missing permissions.

---

#### 2.2 Policy Validation & Routing Logic
- **Overview:** Evaluates whether a valid pricing policy exists for the requested item and applies conditional threshold checks to assign the appropriate routing path.
- **Nodes Involved:**
  - `Check Policy Existence`
  - `If Auto Approval Possible`
  - `If Manager Approval Needed`
  - `Prepare Auto Approval Route`
  - `Prepare Manager Approval Route`
  - `Prepare Senior Approval Route`
  - `Log Missing Policy in Sheets`

- **Node Details:**
  - **Check Policy Existence**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Logic). Checks if the Google Sheets lookup returned a valid active policy record.
    - *Configuration Choices:* Boolean evaluation based on the presence of policy data.
    - *Input/Output Connections:* Input from `Read Pricing Policy in Sheets`. True branch leads to `If Auto Approval Possible`; False branch leads to `Log Missing Policy in Sheets`.
  - **If Auto Approval Possible**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Logic). Determines if the requested discount and margin fall within the `Max_Auto_Discount_Pct` thresholds.
    - *Configuration Choices:* Comparison expressions evaluating requested discount against policy limits.
    - *Input/Output Connections:* True branch leads to `Prepare Auto Approval Route`; False branch leads to `If Manager Approval Needed`.
  - **If Manager Approval Needed**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Logic). Evaluates if the discount exceeds auto-approval limits but stays within `Manager_Approval_Max_Discount_Pct`.
    - *Configuration Choices:* Comparison against manager threshold limits.
    - *Input/Output Connections:* True branch leads to `Prepare Manager Approval Route`; False branch leads to `Prepare Senior Approval Route`.
  - **Prepare Auto Approval Route**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation). Sets the approval status flag to "Auto-Approved" and populates relevant metadata.
    - *Input/Output Connections:* Input from `If Auto Approval Possible`; output to `Append to Deal Log in Sheets`.
  - **Prepare Manager Approval Route**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation). Assigns the designated manager's email address and sets the status to "Pending Manager Approval".
    - *Input/Output Connections:* Input from `If Manager Approval Needed`; output to `Append to Deal Log in Sheets`.
  - **Prepare Senior Approval Route**
    - *Type & Technical Role:* `n8n-nodes-base.set` (Data Transformation). Assigns the senior approver's email address and sets the status to "Pending Senior Approval".
    - *Input/Output Connections:* Input from `If Manager Approval Needed` (False branch); output to `Append to Deal Log in Sheets`.
  - **Log Missing Policy in Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Integration). Appends an exception record to the sheet when a deal references a product without an active policy.
    - *Input/Output Connections:* Input from `Check Policy Existence` (False branch).
    - *Credentials:* Google Sheets.

---

#### 2.3 Audit Logging & Notification Dispatch
- **Overview:** Centralizes all processed deals into a Google Sheets audit log and dispatches the corresponding email notifications to sales reps or designated approvers.
- **Nodes Involved:**
  - `Append to Deal Log in Sheets`
  - `If Human Approval Needed`
  - `Send Approval Request via Gmail`
  - `Send Auto Approval via Gmail`

- **Node Details:**
  - **Append to Deal Log in Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Integration). Appends a new row containing deal attributes, calculated margins, and assigned approval tiers to the `Deal_Log` sheet.
    - *Input/Output Connections:* Inputs from route preparation nodes (`Prepare Auto/Manager/Senior Approval Route`); output to `If Human Approval Needed`.
  - **If Human Approval Needed**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Conditional Logic). Splits flow depending on whether human intervention (Manager/Senior) is required or if the deal was auto-approved.
    - *Input/Output Connections:* Input from `Append to Deal Log in Sheets`. True branch routes to `Send Approval Request via Gmail`; False branch routes to `Send Auto Approval via Gmail`.
  - **Send Approval Request via Gmail**
    - *Type & Technical Role:* `n8n-nodes-base.gmail` (Communication). Sends an approval notification email containing decision links/form details to the assigned manager or senior approver.
    - *Input/Output Connections:* Input from `If Human Approval Needed`.
    - *Credentials:* Gmail OAuth2.
    - *Failure Types:* Invalid recipient email addresses, OAuth token expiration, daily Gmail sending limits.
  - **Send Auto Approval via Gmail**
    - *Type & Technical Role:* `n8n-nodes-base.gmail` (Communication). Sends an instant confirmation email to the sales representative confirming the deal has been auto-approved.
    - *Input/Output Connections:* Input from `If Human Approval Needed`.
    - *Credentials:* Gmail OAuth2.

---

#### 2.4 Approval Decision Processing
- **Overview:** Receives review decisions from managers or senior leaders via a secondary form, updates the audit sheet with the outcome, and emails the final status to the salesperson.
- **Nodes Involved:**
  - `When Approval Form Submitted`
  - `Read Deal for Approval in Sheets`
  - `Route by Approval Decision`
  - `Update Approved Deal in Sheets`
  - `Update Rejected Deal in Sheets`
  - `Send Decision Email via Gmail`

- **Node Details:**
  - **When Approval Form Submitted**
    - *Type & Technical Role:* `n8n-nodes-base.formTrigger` (Trigger Node). Webhook-powered form capturing the approver's decision (Approve/Reject) and remarks alongside the Deal ID.
    - *Input/Output Connections:* Output to `Read Deal for Approval in Sheets`.
  - **Read Deal for Approval in Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Integration). Queries the `Deal_Log` sheet to retrieve original deal metadata using the Deal ID submitted in the approval form.
    - *Input/Output Connections:* Input from `When Approval Form Submitted`; output to `Route by Approval Decision`.
  - **Route by Approval Decision**
    - *Type & Technical Role:* `n8n-nodes-base.switch` (Routing Logic). Branches execution based on whether the approver selected "Approved" or "Rejected".
    - *Configuration Choices:* Rules evaluating the decision form output field.
    - *Input/Output Connections:* Input from `Read Deal for Approval in Sheets`. Branch 0 leads to `Update Approved Deal in Sheets`; Branch 1 leads to `Update Rejected Deal in Sheets`.
  - **Update Approved Deal in Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Integration). Updates the corresponding row in the `Deal_Log` sheet, marking status as approved and adding remarks.
    - *Input/Output Connections:* Input from `Route by Approval Decision`; output to `Send Decision Email via Gmail`.
  - **Update Rejected Deal in Sheets**
    - *Type & Technical Role:* `n8n-nodes-base.googleSheets` (Data Integration). Updates the matching row in the `Deal_Log` sheet, marking status as rejected with approver remarks.
    - *Input/Output Connections:* Input from `Route by Approval Decision`; output to `Send Decision Email via Gmail`.
  - **Send Decision Email via Gmail**
    - *Type & Technical Role:* `n8n-nodes-base.gmail` (Communication). Emails the sales representative with the final approval or rejection decision and any remarks provided by the approver.
    - *Input/Output Connections:* Inputs from both `Update Approved Deal in Sheets` and `Update Rejected Deal in Sheets`.
    - *Credentials:* Gmail OAuth2.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | `n8n-nodes-base.stickyNote` | Visual grouping container | None | None | |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Visual grouping container | None | None | |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Visual grouping container | None | None | |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Visual grouping container | None | None | |
| Sticky Note4 | `n8n-nodes-base.stickyNote` | Visual grouping container | None | None | |
| Sticky Note5 | `n8n-nodes-base.stickyNote` | Visual grouping container | None | None | |
| Sticky Note6 | `n8n-nodes-base.stickyNote` | Visual grouping container | None | None | |
| Sticky Note7 | `n8n-nodes-base.stickyNote` | Visual grouping container | None | None | |
| Sticky Note8 | `n8n-nodes-base.stickyNote` | Visual grouping container | None | None | |
| Sticky Note9 | `n8n-nodes-base.stickyNote` | Visual grouping container | None | None | |
| When Deal Form Submitted | `n8n-nodes-base.formTrigger` | Collects deal submissions | None | Set Deal Details | |
| Set Deal Details | `n8n-nodes-base.set` | Normalizes and adds metadata to deal input | When Deal Form Submitted | Read Pricing Policy in Sheets | |
| Read Pricing Policy in Sheets | `n8n-nodes-base.googleSheets` | Queries pricing thresholds | Set Deal Details | Check Policy Existence | |
| Check Policy Existence | `n8n-nodes-base.if` | Validates if policy record was found | Read Pricing Policy in Sheets | If Auto Approval Possible, Log Missing Policy in Sheets | |
| If Auto Approval Possible | `n8n-nodes-base.if` | Checks auto-approval eligibility | Check Policy Existence | Prepare Auto Approval Route, If Manager Approval Needed | |
| If Manager Approval Needed | `n8n-nodes-base.if` | Evaluates manager vs senior approval tiers | If Auto Approval Possible | Prepare Manager Approval Route, Prepare Senior Approval Route | |
| Prepare Manager Approval Route | `n8n-nodes-base.set` | Assigns manager route metadata | If Manager Approval Needed | Append to Deal Log in Sheets | |
| Prepare Senior Approval Route | `n8n-nodes-base.set` | Assigns senior route metadata | If Manager Approval Needed | Append to Deal Log in Sheets | |
| Prepare Auto Approval Route | `n8n-nodes-base.set` | Assigns auto-approval route metadata | If Auto Approval Possible | Append to Deal Log in Sheets | |
| Append to Deal Log in Sheets | `n8n-nodes-base.googleSheets` | Writes deal record to audit log | Prepare Auto/Manager/Senior Approval Route | If Human Approval Needed | |
| If Human Approval Needed | `n8n-nodes-base.if` | Separates automated vs manual routing | Append to Deal Log in Sheets | Send Approval Request via Gmail, Send Auto Approval via Gmail | |
| Send Approval Request via Gmail | `n8n-nodes-base.gmail` | Emails approver for manual review | If Human Approval Needed | None | |
| When Approval Form Submitted | `n8n-nodes-base.formTrigger` | Captures approver decision form | None | Read Deal for Approval in Sheets | |
| Read Deal for Approval in Sheets | `n8n-nodes-base.googleSheets` | Retrieves deal record for decision processing | When Approval Form Submitted | Route by Approval Decision | |
| Route by Approval Decision | `n8n-nodes-base.switch` | Branches based on approve/reject choice | Read Deal for Approval in Sheets | Update Approved Deal in Sheets, Update Rejected Deal in Sheets | |
| Update Approved Deal in Sheets | `n8n-nodes-base.googleSheets` | Updates deal log row to approved | Route by Approval Decision | Send Decision Email via Gmail | |
| Update Rejected Deal in Sheets | `n8n-nodes-base.googleSheets` | Updates deal log row to rejected | Route by Approval Decision | Send Decision Email via Gmail | |
| Send Decision Email via Gmail | `n8n-nodes-base.gmail` | Emails final outcome to salesperson | Update Approved Deal in Sheets, Update Rejected Deal in Sheets | None | |
| Send Auto Approval via Gmail | `n8n-nodes-base.gmail` | Emails auto-approval confirmation | If Human Approval Needed | None | |
| Log Missing Policy in Sheets | `n8n-nodes-base.googleSheets` | Logs pricing policy exceptions | Check Policy Existence | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Points (Forms):**
   - Add a `When Deal Form Submitted` (`n8n-nodes-base.formTrigger`, v2.6) node to capture sales submissions. Configure form fields for Product/Service Name, Requested Discount Percentage, and Expected Gross Margin.
   - Add a second `When Approval Form Submitted` (`n8n-nodes-base.formTrigger`, v2.6) node to handle manager/senior review responses. Configure fields for Deal ID, Decision (Approve/Reject), and Remarks.

2. **Set Up Deal Data Normalization:**
   - Create a `Set Deal Details` (`n8n-nodes-base.set`, v3.5) node. Connect it to `When Deal Form Submitted`. Configure expressions to assign timestamps and generate a unique Deal ID.

3. **Configure Pricing Policy Lookup & Validation:**
   - Add a `Read Pricing Policy in Sheets` (`n8n-nodes-base.googleSheets`, v4.7) node configured with Google Sheets credentials. Set it to read rows from your `Pricing_Policy` sheet based on the submitted product name. Enable `alwaysOutputData: true`.
   - Add a `Check Policy Existence` (`n8n-nodes-base.if`, v2.3) node. Connect True to auto-approval evaluation and False to an exception logging node.
   - Add `If Auto Approval Possible` and `If Manager Approval Needed` (`n8n-nodes-base.if`, v2.3) nodes to compare discount and margin values against policy limits.

4. **Configure Routing & Preparation Nodes:**
   - Add three `Set` nodes: `Prepare Auto Approval Route`, `Prepare Manager Approval Route`, and `Prepare Senior Approval Route` (`n8n-nodes-base.set`, v3.5). Map approval statuses and designate recipient emails accordingly.

5. **Set Up Audit Logging (Google Sheets):**
   - Add an `Append to Deal Log in Sheets` (`n8n-nodes-base.googleSheets`, v4.7) node connected to all three route preparation nodes. Map parameters to your `Deal_Log` sheet columns.
   - Add a `Log Missing Policy in Sheets` (`n8n-nodes-base.googleSheets`, v4.7) node connected to the negative branch of `Check Policy Existence`.

6. **Configure Email Dispatch (Gmail):**
   - Add an `If Human Approval Needed` (`n8n-nodes-base.if`, v2.3) node after the deal log append.
   - Add a `Send Approval Request via Gmail` (`n8n-nodes-base.gmail`, v2.2) node for manual approvals using Gmail OAuth2 credentials.
   - Add a `Send Auto Approval via Gmail` (`n8n-nodes-base.gmail`, v2.2) node for instant sales rep notifications.

7. **Configure Decision Processing Path:**
   - Connect `When Approval Form Submitted` to `Read Deal for Approval in Sheets` (`n8n-nodes-base.googleSheets`, v4.7) to look up deals by ID.
   - Add a `Route by Approval Decision` (`n8n-nodes-base.switch`, v3.4) node to split execution between approved and rejected paths.
   - Add `Update Approved Deal in Sheets` and `Update Rejected Deal in Sheets` (`n8n-nodes-base.googleSheets`, v4.7) nodes to update the respective row statuses.
   - Terminate both update paths into a shared `Send Decision Email via Gmail` (`n8n-nodes-base.gmail`, v2.2) node to notify the salesperson.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Ensure Google Sheets API is enabled and OAuth2 / Service Account credentials are active with editor access. | Google Sheets Integration Setup |
| Configure Gmail OAuth2 credentials with adequate scopes to send emails on behalf of the organization. | Gmail API Authentication |
| Replace spreadsheet IDs and sheet GIDs (`YOUR_GOOGLE_SHEET_ID`, `YOUR_PRICING_POLICY_SHEET_GID`, `YOUR_DEAL_LOG_SHEET_GID`) in all Google Sheets node configurations prior to activation. | Configuration Requirement |
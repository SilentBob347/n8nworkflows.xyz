Generate qualified eBay B2B seller leads from item IDs with Apify

https://n8nworkflows.xyz/workflows/generate-qualified-ebay-b2b-seller-leads-from-item-ids-with-apify-21001


# Generate qualified eBay B2B seller leads from item IDs with Apify

### 1. Workflow Overview

The "eBay B2B Seller Contact Lead Generator" workflow automates the extraction and qualification of B2B seller leads from eBay. Its primary purpose is to collect business disclosure information (such as corporate emails, phone numbers, and physical addresses) for specific eBay merchants, score their commercial value, and prepare the records for downstream CRM integration or cold outreach campaigns.

The logic is organized into three functional blocks:
- **1.1 Input Reception:** Manages manual execution and automated scheduling (every Monday at 8 AM) to kick off the lead generation process.
- **1.2 Scraping & Qualification:** Interacts with the Apify platform to execute an actor that extracts eBay seller contact data, then processes the raw payload through a custom script to score merchants, assign trust tiers, and generate tailored outreach assets.
- **1.3 CRM & Outreach Preparation:** Appends administrative metadata (lead status and enrichment timestamp) to finalize records for downstream tools.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Acts as the entry point for the workflow, allowing users to initiate lead generation manually or automatically on a weekly schedule.
- **Nodes Involved:** 
  - `Manual Trigger`
  - `Weekly Lead Scan (Mon 8 AM)`

- **Node Details:**
  - **Manual Trigger**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (v1) - Initiates workflow execution on demand.
    - *Configuration Choices:* Default configuration.
    - *Input/Output Connections:* Output connects to `Scrape eBay Merchant Contacts`.
    - *Edge Cases/Failures:* None.
  - **Weekly Lead Scan (Mon 8 AM)**
    - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (v1.4) - Automatically triggers the workflow on a recurring schedule.
    - *Configuration Choices:* Configured to trigger weekly on Mondays at 8 AM (`triggerAtDay: [1]`, `triggerAtHour: 8`).
    - *Input/Output Connections:* Output connects to `Scrape eBay Merchant Contacts`.
    - *Edge Cases/Failures:* Ensure n8n instance time zone settings match expectations to prevent unexpected execution times.

---

#### 2.2 Scraping & Qualification
- **Overview:** Extracts raw merchant contact details from eBay via an Apify actor, then runs custom JavaScript logic to filter, evaluate feedback metrics, assign merchant tiers, and formulate cold outreach elements.
- **Nodes Involved:**
  - `Scrape eBay Merchant Contacts`
  - `Qualify & Score B2B Leads`

- **Node Details:**
  - **Scrape eBay Merchant Contacts**
    - *Type and Technical Role:* `@apify/n8n-nodes-apify.apify` (v2) - Executes an external Apify actor to scrape structured data.
    - *Configuration Choices:* Uses actor URL mode (`https://console.apify.com/actors/OqXDpvrWE6qntxscU`) with the operation set to run the actor and retrieve the dataset. Custom JSON body specifies target eBay `itemIds` and a `poolCount` of 10.
    - *Credentials:* Requires an Apify API credential (`Apify account 1`).
    - *Input/Output Connections:* Inputs from `Manual Trigger` or `Weekly Lead Scan (Mon 8 AM)`; output connects to `Qualify & Score B2B Leads`.
    - *Edge Cases/Failures:* Apify rate limits, insufficient monthly credits, or invalid item IDs causing empty payloads.
  - **Qualify & Score B2B Leads**
    - *Type and Technical Role:* `n8n-nodes-base.code` (v2) - Executes custom JavaScript to parse, score, and restructure items.
    - *Configuration Choices:* Iterates through input items, evaluates feedback scores and positive ratings, assigns merchant tiers (`🌟 VIP Enterprise Merchant`, `⭐ High-Volume Verified Seller`, or `Standard Merchant`), and constructs icebreakers and subject lines.
    - *Key Expressions/Variables:* Uses `$input.all()` to access incoming datasets and inspect nested properties (`seller.email`, `seller.feedbackScore`, etc.).
    - *Input/Output Connections:* Input from `Scrape eBay Merchant Contacts`; output connects to `Prepare CRM & Outreach Record`.
    - *Edge Cases/Failures:* Runtime errors if the incoming payload lacks expected properties (mitigated by default fallback values like `item.seller || {}`).

---

#### 2.3 CRM & Outreach Preparation
- **Overview:** Finalizes the lead dataset by appending metadata required for CRM synchronization and tracking.
- **Nodes Involved:**
  - `Prepare CRM & Outreach Record`

- **Node Details:**
  - **Prepare CRM & Outreach Record**
    - *Type and Technical Role:* `n8n-nodes-base.set` (v3.4) - Assigns new values or updates existing properties on items.
    - *Configuration Choices:* Includes other incoming fields while adding `lead_status` (set to `"New Qualified Lead"`) and `enriched_at` (set via expression to current ISO timestamp).
    - *Key Expressions/Variables:* `={{ $now.toISO() }}`
    - *Input/Output Connections:* Input from `Qualify & Score B2B Leads`; output is ready for external CRM/database integrations.
    - *Edge Cases/Failures:* Expression evaluation failure if the execution context fails to resolve `$now`.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Manual Trigger` | `n8n-nodes-base.manualTrigger` | Manual entry point | None | `Scrape eBay Merchant Contacts` | Start lead scans |
| `Weekly Lead Scan (Mon 8 AM)` | `n8n-nodes-base.scheduleTrigger` | Scheduled entry point (Mon 8 AM) | None | `Scrape eBay Merchant Contacts` | Start lead scans |
| `Scrape eBay Merchant Contacts` | `@apify/n8n-nodes-apify.apify` | Scrapes eBay merchant contact details via Apify | `Manual Trigger`, `Weekly Lead Scan (Mon 8 AM)` | `Qualify & Score B2B Leads` | Scrape and score leads<br>Apify Actors Reference & Links |
| `Qualify & Score B2B Leads` | `n8n-nodes-base.code` | Scores leads, assigns tiers, and builds outreach copy | `Scrape eBay Merchant Contacts` | `Prepare CRM & Outreach Record` | Scrape and score leads |
| `Prepare CRM & Outreach Record` | `n8n-nodes-base.set` | Appends CRM status and enrichment timestamp | `Qualify & Score B2B Leads` | None | Prepare outreach record |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Entry Points:**
   - Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`).
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`), configure the rule interval to trigger weekly on day `1` at hour `8`.

2. **Configure the Apify Scraping Node:**
   - Add an **Apify** node (`@apify/n8n-nodes-apify.apify`).
   - Set the operation to **Run actor and get dataset**.
   - Set the actor ID mode to URL and provide: `https://console.apify.com/actors/OqXDpvrWE6qntxscU`.
   - Provide a custom JSON body containing target `itemIds` and `poolCount`:
     ```json
     {
       "itemIds": [
         "318445065346",
         "277961689107",
         "253995891196"
       ],
       "poolCount": 10
     }
     ```
   - Connect your **Apify API credentials**.
   - Connect both the Manual Trigger and Weekly Schedule Trigger to this node.

3. **Add the Qualification and Scoring Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`) and connect it to the Apify node output.
   - Set the Mode to run JavaScript for all items.
   - Paste the qualification script that checks `feedbackScore` and `positiveFeedbackPercent`, determines the merchant tier, and structures company details, emails, phones, and cold outreach text fields.

4. **Prepare the CRM Record:**
   - Add an **Edit Fields (Set)** node (`n8n-nodes-base.set`) and connect it to the Code node output.
   - Enable **Include Other Fields**.
   - Add an assignment where Name is `lead_status`, Type is `String`, and Value is `New Qualified Lead`.
   - Add a second assignment where Name is `enriched_at`, Type is `String`, and Value is expression `={{ $now.toISO() }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| eBay Seller Contact Scraper Actor | [Apify Actor Documentation](https://apify.com/magic-scraper/ebay-seller-contact-scraper) |
| Apify Usage Credits | Apify provides $5 in free monthly usage credits (~15,000+ items free/mo). |
| Workflow Target Audience | Designed for B2B Sales Teams, Lead Generation Agencies, Sourcing Scouts, Arbitrageurs & Dropshippers. |
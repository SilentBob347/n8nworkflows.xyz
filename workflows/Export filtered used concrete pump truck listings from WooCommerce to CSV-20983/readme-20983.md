Export filtered used concrete pump truck listings from WooCommerce to CSV

https://n8nworkflows.xyz/workflows/export-filtered-used-concrete-pump-truck-listings-from-woocommerce-to-csv-20983


# Export filtered used concrete pump truck listings from WooCommerce to CSV

### 1. Workflow Overview

This workflow is designed to automate the extraction, filtering, and export of used concrete pump truck listings from a public WooCommerce storefront into a downloadable CSV file. It targets equipment buyers, sourcing agents, and technical analysts who require a repeatable, code-driven method to compile specific machinery shortlists based on brand and operational reach. 

The execution model is sequential and synchronous, divided into three core logical blocks:
- **1.1 Initialization and Configuration:** Manually triggers the workflow and establishes global settings such as target API endpoints, preferred machinery brands, and boom reach boundaries.
- **1.2 Data Retrieval and Filtering:** Connects to the target WooCommerce Store API via an HTTP GET request, retrieves up to 100 recent product entries, and passes them to a JavaScript execution environment that performs regex-based text parsing, brand matching, and numerical range filtering.
- **1.3 Export Generation:** Formats the filtered dataset into structured tabular data and converts it into a downloadable CSV file containing catalog references, product URLs, and mandatory compliance notes.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Initialization and Configuration
- **Overview:** This block handles the manual entry point of the workflow and injects the operational parameters (API endpoint, brand preferences, and minimum/maximum reach limits) required downstream.
- **Nodes Involved:** 
  - `Run shortlist manually`
  - `Shortlist settings`

- **Node Details:**
  - **Run shortlist manually**
    - *Type and Technical Role:* `n8n-nodes-base.manualTrigger` (Trigger Node). Initiates execution on-demand.
    - *Configuration Choices:* Default parameters with no scheduled or webhook triggers.
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Output connects to `Shortlist settings`.
    - *Version-Specific Requirements:* Version 1.
    - *Edge Cases / Potential Failures:* None (manual execution).
    - *Sub-Workflow Reference:* None.

  - **Shortlist settings**
    - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation / Parameter Node). Establishes raw JSON configurations containing execution constants.
    - *Configuration Choices:* Mode set to `raw` with a JSON payload defining `apiUrl`, `preferredBrand`, `minReachM`, and `maxReachM`.
    - *Key Expressions/Variables:* 
      ```json
      {
        "apiUrl": "https://www.mechabro.com/wp-json/wc/store/v1/products?category=220&per_page=100&orderby=date&order=desc",
        "preferredBrand": "SANY",
        "minReachM": 40,
        "maxReachM": 70
      }
      ```
    - *Input/Output Connections:* Input from `Run shortlist manually`; output connects to `Fetch 100 recent public catalog entries`.
    - *Version-Specific Requirements:* Version 3.4.
    - *Edge Cases / Potential Failures:* Malformed JSON structures will throw syntax errors.
    - *Sub-Workflow Reference:* None.

---

#### Block 1.2: Data Retrieval and Filtering
- **Overview:** This block executes an HTTP request to the configured WooCommerce Store API to fetch the catalog and processes the response using custom JavaScript logic to filter products by category, brand name, and boom reach metrics.
- **Nodes Involved:**
  - `Fetch 100 recent public catalog entries`
  - `Filter brand and boom reach`

- **Node Details:**
  - **Fetch 100 recent public catalog entries**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (Network Request Node). Fetches public product data from the target WooCommerce store without authentication.
    - *Configuration Choices:* Method set to `GET`. Timeout configured to 15,000ms.
    - *Key Expressions/Variables:* `={{ $json.apiUrl }}`
    - *Input/Output Connections:* Input from `Shortlist settings`; output connects to `Filter brand and boom reach`.
    - *Version-Specific Requirements:* Version 4.3.
    - *Edge Cases / Potential Failures:* HTTP timeout errors, DNS resolution failures, or invalid endpoint URLs resulting in 404/500 responses.
    - *Sub-Workflow Reference:* None.

  - **Filter brand and boom reach**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Execution Node). Executes inline script to parse, sanitize, and filter product data.
    - *Configuration Choices:* Mode set to `runOnceForAllItems`. Validates numerical limits, strips injection-prone characters via a `safe` utility function, and performs regex matching for brand names (`SANY`, `Zoomlion`, `XCMG`) and metric boom reach (`XX` or `XX.X` followed by `m` or `meters`).
    - *Key Expressions/Variables:* Reads upstream parameters using `$('Shortlist settings').first().json`.
    - *Input/Output Connections:* Input from `Fetch 100 recent public catalog entries`; output connects to `Download shortlist CSV`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases / Potential Failures:* Throws an error if `minReachM` exceeds `maxReachM`. Returns a placeholder fallback row (`No matching catalog references`) if no items match the criteria.
    - *Sub-Workflow Reference:* None.

---

#### Block 1.3: Export Generation
- **Overview:** This block takes the filtered JSON array and transforms it into a structured CSV file ready for local download.
- **Nodes Involved:**
  - `Download shortlist CSV`

- **Node Details:**
  - **Download shortlist CSV**
    - *Type and Technical Role:* `n8n-nodes-base.convertToFile` (File Conversion Node). Converts structured JSON items into comma-separated values.
    - *Configuration Choices:* Operation set to `csv`. Options configured with `fileName` set to `used-concrete-pump-truck-shortlist.csv` and `headerRow` set to `true`.
    - *Key Expressions/Variables:* None.
    - *Input/Output Connections:* Input from `Filter brand and boom reach`; no downstream outputs.
    - *Version-Specific Requirements:* Version 1.1.
    - *Edge Cases / Potential Failures:* Empty binary output handling if upstream array is malformed.
    - *Sub-Workflow Reference:* None.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Run shortlist manually` | `n8n-nodes-base.manualTrigger` | Triggers execution manually | None | `Shortlist settings` | Export used concrete pump truck listings from WooCommerce to CSV<br><br>**Who’s it for:** Buyers and sourcing assistants who need a repeatable shortlist of published equipment references.<br><br>**How it works:** Run manually. The HTTP node reads up to 100 recently published truck-mounted concrete pump entries from a public WooCommerce Store API. The Code node filters by brand and boom reach. Convert to File creates a CSV with catalog ID, parsed reach, title, and original product URL.<br><br>**Set up:** In **Shortlist settings**, edit `preferredBrand`, `minReachM`, and `maxReachM`. Use an empty brand for all supported brands; set a reach limit to `0` to disable it. Click **Execute workflow**, then download the CSV from the final node.<br><br>**Requirements:** The example uses [Mechabro’s public used pump truck catalog](https://www.mechabro.com/en/product-category/concrete-machinery/truck-mounted-concrete-pump/) and needs no credential or paid API.<br><br>**Customize:** Replace `apiUrl` with another WooCommerce Store API product endpoint and category ID. This run checks only the first 100 returned entries. When reach filtering is enabled, entries without a reach in their title are omitted. These are catalog references, not verified current stock. Confirm the specific machine’s availability, condition, quotation, and export terms with the seller. |
| `Shortlist settings` | `n8n-nodes-base.set` | Defines configuration parameters | `Run shortlist manually` | `Fetch 100 recent public catalog entries` | Export used concrete pump truck listings from WooCommerce to CSV<br><br>**Who’s it for:** Buyers and sourcing assistants who need a repeatable shortlist of published equipment references.<br><br>**How it works:** Run manually. The HTTP node reads up to 100 recently published truck-mounted concrete pump entries from a public WooCommerce Store API. The Code node filters by brand and boom reach. Convert to File creates a CSV with catalog ID, parsed reach, title, and original product URL.<br><br>**Set up:** In **Shortlist settings**, edit `preferredBrand`, `minReachM`, and `maxReachM`. Use an empty brand for all supported brands; set a reach limit to `0` to disable it. Click **Execute workflow**, then download the CSV from the final node.<br><br>**Requirements:** The example uses [Mechabro’s public used pump truck catalog](https://www.mechabro.com/en/product-category/concrete-machinery/truck-mounted-concrete-pump/) and needs no credential or paid API.<br><br>**Customize:** Replace `apiUrl` with another WooCommerce Store API product endpoint and category ID. This run checks only the first 100 returned entries. When reach filtering is enabled, entries without a reach in their title are omitted. These are catalog references, not verified current stock. Confirm the specific machine’s availability, condition, quotation, and export terms with the seller.<br><br>### 1. Choose a public catalog and filters<br><br>`apiUrl` is a public WooCommerce Store API products URL. The example points to the [Mechabro truck-mounted concrete pump catalog](https://www.mechabro.com/en/product-category/concrete-machinery/truck-mounted-concrete-pump/) and requests its 100 most recent entries. No API key is needed.<br><br>Edit `preferredBrand`, `minReachM`, and `maxReachM` in **Shortlist settings** before running. Use an empty brand for all supported brands, or `0` for an open reach limit. The HTTP node only reads public product data; it does not change the store. |
| `Fetch 100 recent public catalog entries` | `n8n-nodes-base.httpRequest` | Calls WooCommerce Store API | `Shortlist settings` | `Filter brand and boom reach` | Export used concrete pump truck listings from WooCommerce to CSV<br><br>**Who’s it for:** Buyers and sourcing assistants who need a repeatable shortlist of published equipment references.<br><br>**How it works:** Run manually. The HTTP node reads up to 100 recently published truck-mounted concrete pump entries from a public WooCommerce Store API. The Code node filters by brand and boom reach. Convert to File creates a CSV with catalog ID, parsed reach, title, and original product URL.<br><br>**Set up:** In **Shortlist settings**, edit `preferredBrand`, `minReachM`, and `maxReachM`. Use an empty brand for all supported brands; set a reach limit to `0` to disable it. Click **Execute workflow**, then download the CSV from the final node.<br><br>**Requirements:** The example uses [Mechabro’s public used pump truck catalog](https://www.mechabro.com/en/product-category/concrete-machinery/truck-mounted-concrete-pump/) and needs no credential or paid API.<br><br>**Customize:** Replace `apiUrl` with another WooCommerce Store API product endpoint and category ID. This run checks only the first 100 returned entries. When reach filtering is enabled, entries without a reach in their title are omitted. These are catalog references, not verified current stock. Confirm the specific machine’s availability, condition, quotation, and export terms with the seller.<br><br>### 1. Choose a public catalog and filters<br><br>`apiUrl` is a public WooCommerce Store API products URL. The example points to the [Mechabro truck-mounted concrete pump catalog](https://www.mechabro.com/en/product-category/concrete-machinery/truck-mounted-concrete-pump/) and requests its 100 most recent entries. No API key is needed.<br><br>Edit `preferredBrand`, `minReachM`, and `maxReachM` in **Shortlist settings** before running. Use an empty brand for all supported brands, or `0` for an open reach limit. The HTTP node only reads public product data; it does not change the store. |
| `Filter brand and boom reach` | `n8n-nodes-base.code` | Parses and filters product listings | `Fetch 100 recent public catalog entries` | `Download shortlist CSV` | Export used concrete pump truck listings from WooCommerce to CSV<br><br>**Who’s it for:** Buyers and sourcing assistants who need a repeatable shortlist of published equipment references.<br><br>**How it works:** Run manually. The HTTP node reads up to 100 recently published truck-mounted concrete pump entries from a public WooCommerce Store API. The Code node filters by brand and boom reach. Convert to File creates a CSV with catalog ID, parsed reach, title, and original product URL.<br><br>**Set up:** In **Shortlist settings**, edit `preferredBrand`, `minReachM`, and `maxReachM`. Use an empty brand for all supported brands; set a reach limit to `0` to disable it. Click **Execute workflow**, then download the CSV from the final node.<br><br>**Requirements:** The example uses [Mechabro’s public used pump truck catalog](https://www.mechabro.com/en/product-category/concrete-machinery/truck-mounted-concrete-pump/) and needs no credential or paid API.<br><br>**Customize:** Replace `apiUrl` with another WooCommerce Store API product endpoint and category ID. This run checks only the first 100 returned entries. When reach filtering is enabled, entries without a reach in their title are omitted. These are catalog references, not verified current stock. Confirm the specific machine’s availability, condition, quotation, and export terms with the seller.<br><br>### 2. Review the CSV shortlist<br><br>The Code node keeps pump truck listings whose title matches the selected brand and whose title contains a boom reach within range. It exports the catalog ID, parsed brand/reach, listing title, original URL, and a review note. If nothing matches, the CSV contains one explanatory row.<br><br>The result covers only the first 100 API entries. Missing or ambiguous reach values are excluded when a reach filter is active. Open the original listing and confirm availability, machine condition, quotation, and export terms with the seller; this workflow does not verify those facts. |
| `Download shortlist CSV` | `n8n-nodes-base.convertToFile` | Generates downloadable CSV file | `Filter brand and boom reach` | None | Export used concrete pump truck listings from WooCommerce to CSV<br><br>**Who’s it for:** Buyers and sourcing assistants who need a repeatable shortlist of published equipment references.<br><br>**How it works:** Run manually. The HTTP node reads up to 100 recently published truck-mounted concrete pump entries from a public WooCommerce Store API. The Code node filters by brand and boom reach. Convert to File creates a CSV with catalog ID, parsed reach, title, and original product URL.<br><br>**Set up:** In **Shortlist settings**, edit `preferredBrand`, `minReachM`, and `maxReachM`. Use an empty brand for all supported brands; set a reach limit to `0` to disable it. Click **Execute workflow**, then download the CSV from the final node.<br><br>**Requirements:** The example uses [Mechabro’s public used pump truck catalog](https://www.mechabro.com/en/product-category/concrete-machinery/truck-mounted-concrete-pump/) and needs no credential or paid API.<br><br>**Customize:** Replace `apiUrl` with another WooCommerce Store API product endpoint and category ID. This run checks only the first 100 returned entries. When reach filtering is enabled, entries without a reach in their title are omitted. These are catalog references, not verified current stock. Confirm the specific machine’s availability, condition, quotation, and export terms with the seller.<br><br>### 2. Review the CSV shortlist<br><br>The Code node keeps pump truck listings whose title matches the selected brand and whose title contains a boom reach within range. It exports the catalog ID, parsed brand/reach, listing title, original URL, and a review note. If nothing matches, the CSV contains one explanatory row.<br><br>The result covers only the first 100 API entries. Missing or ambiguous reach values are excluded when a reach filter is active. Open the original listing and confirm availability, machine condition, quotation, and export terms with the seller; this workflow does not verify those facts. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Node 1:** Add a **Manual Trigger** node (`n8n-nodes-base.manualTrigger`). Name it `Run shortlist manually`. Keep default parameters.
2. **Create Node 2:** Add a **Set** node (`n8n-nodes-base.set`). Name it `Shortlist settings`. Set mode to `raw` and configure the JSON output parameter with:
   ```json
   {
     "apiUrl": "https://www.mechabro.com/wp-json/wc/store/v1/products?category=220&per_page=100&orderby=date&order=desc",
     "preferredBrand": "SANY",
     "minReachM": 40,
     "maxReachM": 70
   }
   ```
3. **Connect Nodes 1 & 2:** Link `Run shortlist manually` main output to `Shortlist settings` main input.
4. **Create Node 3:** Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`). Name it `Fetch 100 recent public catalog entries`. Set method to `GET`, URL to `={{ $json.apiUrl }}`, and timeout options to `15000`.
5. **Connect Nodes 2 & 3:** Link `Shortlist settings` main output to `Fetch 100 recent public catalog entries` main input.
6. **Create Node 4:** Add a **Code** node (`n8n-nodes-base.code`). Name it `Filter brand and boom reach`. Set mode to `runOnceForAllItems` and insert the JavaScript filtering logic that reads settings from `$('Shortlist settings').first().json`, parses item names/categories via regex, applies boundary checks for `minReachM` and `maxReachM`, sanitizes strings, and formats the output schema (`catalogId`, `brand`, `boomReachM`, `listingTitle`, `productUrl`, `reviewNote`).
7. **Connect Nodes 3 & 4:** Link `Fetch 100 recent public catalog entries` main output to `Filter brand and boom reach` main input.
8. **Create Node 5:** Add a **Convert to File** node (`n8n-nodes-base.convertToFile`). Name it `Download shortlist CSV`. Set operation to `csv`, enable `headerRow`, and set `fileName` to `used-concrete-pump-truck-shortlist.csv`.
9. **Connect Nodes 4 & 5:** Link `Filter brand and boom reach` main output to `Download shortlist CSV` main input.
10. **Credentials:** No external credentials are required as this workflow uses public WooCommerce Store API endpoints.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Example public data source catalog | [Mechabro used truck-mounted concrete pump catalog](https://www.mechabro.com/en/product-category/concrete-machinery/truck-mounted-concrete-pump/) |
| Operational disclaimer | The workflow outputs catalog references only. Confirm current availability, condition, quotation, and export terms directly with the seller. |
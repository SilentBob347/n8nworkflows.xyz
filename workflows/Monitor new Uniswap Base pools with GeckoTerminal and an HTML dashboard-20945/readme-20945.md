Monitor new Uniswap Base pools with GeckoTerminal and an HTML dashboard

https://n8nworkflows.xyz/workflows/monitor-new-uniswap-base-pools-with-geckoterminal-and-an-html-dashboard-20945


# Monitor new Uniswap Base pools with GeckoTerminal and an HTML dashboard

### 1. Workflow Overview

This workflow monitors newly created Uniswap V3 liquidity pools on the Base network using the public GeckoTerminal API, evaluates their risk levels based on liquidity, volume, and transaction counts, records them in an n8n Data Table, and renders a live, auto-refreshing HTML dashboard via a webhook.

The workflow logic is split into two distinct functional paths:
- **1.1 Pool Ingestion and Scoring Pipeline:** Triggered manually, this path fetches the latest Base network new-pools feed, extracts token pairs and fee tiers, filters out major stable/wrapped token pairs and low-liquidity pools, calculates a comprehensive risk metric, and saves valid records to the n8n Data Table.
- **1.2 Dashboard Delivery Pipeline:** Triggered via an incoming HTTP webhook request, this path retrieves all stored pool records from the Data Table, deduplicates and sorts them chronologically, compiles summary statistics, builds a responsive HTML page with a 30-second auto-refresh, and returns the response.

---

### 2. Block-by-Block Analysis

#### Block 1: Pool Ingestion and Scoring Pipeline
This block handles data retrieval from the GeckoTerminal API, performs complex string parsing and mathematical risk calculations, filters irrelevant or micro-liquidity pools, and persists qualified data.

- **Start Here**
  - **Type & Role:** `n8n-nodes-base.manualTrigger` — Manual execution entry point to kick off the ingestion pipeline.
  - **Configuration:** Default parameters (no input requirements).
  - **Connections:** Output connects to `Fetch New Pools`.
  - **Edge Cases:** None.

- **Fetch New Pools**
  - **Type & Role:** `n8n-nodes-base.httpRequest` — HTTP request node to query the GeckoTerminal API.
  - **Configuration:** Method `GET`, URL set to `https://api.geckoterminal.com/api/v2/networks/base/new_pools`.
  - **Connections:** Input from `Start Here`; Output connects to `Parse Pool Data`.
  - **Edge Cases:** Network timeouts or API rate-limiting by GeckoTerminal.

- **Parse Pool Data**
  - **Type & Role:** `n8n-nodes-base.code` — JavaScript execution node for transforming raw API payloads, parsing pool names, scoring risk, and filtering pools.
  - **Configuration:** Custom JS code parsing token symbols (e.g., `IOF / WETH 0.3%`), assigning risk scores based on liquidity (`reserve_in_usd`), 24h volume (`volume_usd.h24`), and transaction counts, and applying filters to exclude major token pairs (e.g., USDC, USDT, WETH) and pools with less than $500 liquidity.
  - **Connections:** Input from `Fetch New Pools`; Output connects to `Store New Pool`.
  - **Edge Cases:** Unexpected API schema changes leading to unparsable pool names or missing attributes.

- **Store New Pool**
  - **Type & Role:** `n8n-nodes-base.dataTable` — Database integration node to insert parsed pool records.
  - **Configuration:** Operation set to Append/Insert using mapping mode `defineBelow` targeting the data table named `new_pools_official`. Fields mapped include `pool_address`, `token0_symbol`, `token1_symbol`, `fee_tier`, `liquidity`, `volume_24h`, `tx_count`, `created_at`, `risk_score`, `risk_level`, `risk_label`, and `network`.
  - **Connections:** Input from `Parse Pool Data`.
  - **Edge Cases:** Data table ID mismatch or schema validation errors if columns do not match the expected data types.

---

#### Block 2: Dashboard Delivery Pipeline
This block listens for dashboard access requests, aggregates and prepares stored records, formats HTML output with styling and inline statistics, and responds to the client.

- **Dashboard Webhook**
  - **Type & Role:** `n8n-nodes-base.webhook` — Webhook trigger node acting as the HTTP server entry point for the dashboard.
  - **Configuration:** Path set to `new-pools-dashboard`, response mode set to `responseNode`.
  - **Connections:** Output connects to `Get All Pools`.
  - **Edge Cases:** Webhook URL conflicts or unauthorized access if network exposure is public.

- **Get All Pools**
  - **Type & Role:** `n8n-nodes-base.dataTable` — Database integration node to retrieve historical pool records.
  - **Configuration:** Operation set to `get` with `returnAll` enabled, targeting the data table `new_pools_official`.
  - **Connections:** Input from `Dashboard Webhook`; Output connects to `Generate Dashboard HTML`.
  - **Edge Cases:** Empty data table returning null items, handled gracefully by downstream script logic.

- **Generate Dashboard HTML**
  - **Type & Role:** `n8n-nodes-base.code` — JavaScript execution node for data aggregation, deduplication, statistical grouping, and HTML generation.
  - **Configuration:** Custom JS that deduplicates records by `pool_address`, sorts by creation time descending, computes risk distribution counts, formats timestamps (UTC and LA timezones), renders UI cards with dynamic risk border colors, embeds an auto-refresh meta tag (`content="30"`), and produces a self-contained HTML document string.
  - **Connections:** Input from `Get All Pools`; Output connects to `Respond Dashboard`.
  - **Edge Cases:** Large datasets causing high memory usage in JavaScript execution.

- **Respond Dashboard**
  - **Type & Role:** `n8n-nodes-base.respondToWebhook` — HTTP response node returning the generated HTML payload to the browser.
  - **Configuration:** Response format set to `text`, body mapped to `={{ $json.html }}`, and response header `Content-Type` set to `text/html; charset=utf-8`.
  - **Connections:** Input from `Generate Dashboard HTML`.
  - **Edge Cases:** Header configuration errors causing browsers to download the file instead of rendering the HTML.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Start Here** | `n8n-nodes-base.manualTrigger` | Manual workflow execution trigger | None | Fetch New Pools | 🚀 GETTING STARTED<br><br>This template monitors newly created Uniswap V3 pools on Base.<br><br>BEFORE RUNNING:<br>1. Create a Data Table named "new_pools_official" with these 14 columns:<br>   pool_address (String), token0_symbol (String), token0_address (String),<br>   token1_symbol (String), token1_address (String), fee_tier (String),<br>   liquidity (Number), volume_24h (Number), tx_count (Number),<br>   created_at (String), risk_score (Number), risk_level (String),<br>   risk_label (String), network (String)<br><br>2. Open the "Store New Pool" and "Get All Pools" nodes and select the new table.<br><br>3. Click "Execute Workflow" to test.<br><br>4. Publish the workflow, then access the dashboard at:<br>   /webhook/new-pools-dashboard<br><br>REQUIREMENTS:<br>- No API keys needed (uses free GeckoTerminal API)<br>- Base chain only |
| **Fetch New Pools** | `n8n-nodes-base.httpRequest` | Fetch new pool data from GeckoTerminal API | Start Here | Parse Pool Data | 🚀 GETTING STARTED<br><br>This template monitors newly created Uniswap V3 pools on Base.<br><br>BEFORE RUNNING:<br>1. Create a Data Table named "new_pools_official" with these 14 columns:<br>   pool_address (String), token0_symbol (String), token0_address (String),<br>   token1_symbol (String), token1_address (String), fee_tier (String),<br>   liquidity (Number), volume_24h (Number), tx_count (Number),<br>   created_at (String), risk_score (Number), risk_level (String),<br>   risk_label (String), network (String)<br><br>2. Open the "Store New Pool" and "Get All Pools" nodes and select the new table.<br><br>3. Click "Execute Workflow" to test.<br><br>4. Publish the workflow, then access the dashboard at:<br>   /webhook/new-pools-dashboard<br><br>REQUIREMENTS:<br>- No API keys needed (uses free GeckoTerminal API)<br>- Base chain only |
| **Parse Pool Data** | `n8n-nodes-base.code` | Parse pool names, calculate risk metrics, filter noise | Fetch New Pools | Store New Pool | 🧠 RISK SCORING LOGIC<br><br>Score is calculated based on:<br>- Liquidity: < $1K = +50, < $10K = +30, < $100K = +10<br>- 24h Volume: < $100 = +30, < $1K = +15<br>- Tx Count: < 10 = +20, < 100 = +10<br><br>Risk levels:<br>- 0-19: Low Risk<br>- 20-39: Medium Risk<br>- 40-59: High Risk<br>- 60+: Critical Risk |
| **Store New Pool** | `n8n-nodes-base.dataTable` | Save validated and scored pool records to Data Table | Parse Pool Data | None | 🚀 GETTING STARTED<br><br>This template monitors newly created Uniswap V3 pools on Base.<br><br>BEFORE RUNNING:<br>1. Create a Data Table named "new_pools_official" with these 14 columns:<br>   pool_address (String), token0_symbol (String), token0_address (String),<br>   token1_symbol (String), token1_address (String), fee_tier (String),<br>   liquidity (Number), volume_24h (Number), tx_count (Number),<br>   created_at (String), risk_score (Number), risk_level (String),<br>   risk_label (String), network (String)<br><br>2. Open the "Store New Pool" and "Get All Pools" nodes and select the new table.<br><br>3. Click "Execute Workflow" to test.<br><br>4. Publish the workflow, then access the dashboard at:<br>   /webhook/new-pools-dashboard<br><br>REQUIREMENTS:<br>- No API keys needed (uses free GeckoTerminal API)<br>- Base chain only |
| **Dashboard Webhook** | `n8n-nodes-base.webhook` | Receive HTTP requests for the dashboard | None | Get All Pools | 📊 DASHBOARD ACCESS<br><br>After publishing, access the live dashboard at:<br><br>http://YOUR_N8N_DOMAIN/webhook/new-pools-dashboard<br><br>The dashboard auto-refreshes every 30 seconds and shows:<br>- Total new pools (last 30)<br>- Risk distribution (Low/Medium/High/Critical)<br>- Pool details (liquidity, volume, tx count, risk score) |
| **Get All Pools** | `n8n-nodes-base.dataTable` | Retrieve all historical pool records from Data Table | Dashboard Webhook | Generate Dashboard HTML | |
| **Generate Dashboard HTML** | `n8n-nodes-base.code` | Deduplicate, sort, compute stats, and generate HTML dashboard | Get All Pools | Respond Dashboard | |
| **Respond Dashboard** | `n8n-nodes-base.respondToWebhook` | Return HTML response with proper headers to the client browser | Generate Dashboard HTML | None | |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n:

1. **Create the Data Table:**
   - In your n8n instance, create a new Data Table named `new_pools_official` with the following 14 columns:
     - `pool_address` (String)
     - `token0_symbol` (String)
     - `token0_address` (String)
     - `token1_symbol` (String)
     - `token1_address` (String)
     - `fee_tier` (String)
     - `liquidity` (Number)
     - `volume_24h` (Number)
     - `tx_count` (Number)
     - `created_at` (String)
     - `risk_score` (Number)
     - `risk_level` (String)
     - `risk_label` (String)
     - `network` (String)

2. **Build the Ingestion Pipeline:**
   - **Node 1:** Add a `Manual Trigger` (`n8n-nodes-base.manualTrigger`) named `Start Here`.
   - **Node 2:** Add an `HTTP Request` (`n8n-nodes-base.httpRequest`) named `Fetch New Pools`. Set Method to `GET` and URL to `https://api.geckoterminal.com/api/v2/networks/base/new_pools`. Connect `Start Here` to `Fetch New Pools`.
   - **Node 3:** Add a `Code` node (`n8n-nodes-base.code`) named `Parse Pool Data`. Paste the JavaScript transformation, token filtering, and risk-scoring logic provided in the reference JSON. Connect `Fetch New Pools` to `Parse Pool Data`.
   - **Node 4:** Add a `Data Table` node (`n8n-nodes-base.dataTable`) named `Store New Pool`. Select the `new_pools_official` table, set operation to append/insert, and map each item field (`={{ $json.fieldName }}`) to the corresponding table column. Connect `Parse Pool Data` to `Store New Pool`.

3. **Build the Dashboard Pipeline:**
   - **Node 5:** Add a `Webhook` (`n8n-nodes-base.webhook`) named `Dashboard Webhook`. Set HTTP Method to `GET`, path to `new-pools-dashboard`, and Response Mode to `Response Node`.
   - **Node 6:** Add a `Data Table` node (`n8n-nodes-base.dataTable`) named `Get All Pools`. Set operation to `Get`, enable `Return All`, and select the `new_pools_official` table. Connect `Dashboard Webhook` to `Get All Pools`.
   - **Node 7:** Add a `Code` node (`n8n-nodes-base.code`) named `Generate Dashboard HTML`. Paste the provided HTML generation script that handles deduplication, sorting, statistics, and templating. Connect `Get All Pools` to `Generate Dashboard HTML`.
   - **Node 8:** Add a `Respond to Webhook` node (`n8n-nodes-base.respondToWebhook`) named `Respond Dashboard`. Set Response Body to `={{ $json.html }}`, response type to `Text`, and add a response header with Name `Content-Type` and Value `text/html; charset=utf-8`. Connect `Generate Dashboard HTML` to `Respond Dashboard`.

4. **Activate and Test:**
   - Execute the ingestion pipeline manually once to populate initial data.
   - Publish the workflow and navigate to your n8n domain at `/webhook/new-pools-dashboard` to view the live dashboard.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Free public API usage (no authentication required) | GeckoTerminal API (`https://api.geckoterminal.com`) |
| Base network monitoring scope only | Base Chain (Layer 2) |
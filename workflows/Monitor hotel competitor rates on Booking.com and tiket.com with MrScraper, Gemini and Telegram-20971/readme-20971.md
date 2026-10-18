Monitor hotel competitor rates on Booking.com and tiket.com with MrScraper, Gemini and Telegram

https://n8nworkflows.xyz/workflows/monitor-hotel-competitor-rates-on-booking-com-and-tiket-com-with-mrscraper--gemini-and-telegram-20971


# Monitor hotel competitor rates on Booking.com and tiket.com with MrScraper, Gemini and Telegram

### 1. Workflow Overview

This workflow is a daily automated intelligence system designed to track, analyze, and report competitor hotel rates across multiple online travel agencies (Booking.com and tiket.com). Its primary target audience consists of hotel, guesthouse, and villa managers looking to monitor competitor pricing behavior without manual web browsing. 

The logic executes daily at 15:00 Asia/Jakarta time and is structurally split into two parallel operational tracks:
- **Watchlist Scan Track:** Reads a curated list of properties from Google Sheets, generates targeted stay dates for upcoming weekends, scrapes room rates using structured AI or markdown extraction paths depending on the platform, evaluates price changes against historical baselines, logs all observations, and generates a structured market summary.
- **Area Scan Track:** Searches Booking.com for broader city-level listings, extracts competing property details, saves the top results, and benchmarks the local market average.

Finally, the workflow compiles both data streams into an Indonesian-language summary using Google Gemini (with deterministic programmatic fallback mechanisms) and dispatches a unified notification report to a designated Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Schedule Trigger
- **Overview:** Initializes the daily execution cycle, loads the competitor property registry, and fetches historical tracking data from Google Sheets to establish analytical baselines.
- **Nodes Involved:** 
  - `Schedule: Daily 15:00 (Watchlist + Area Scan)`
  - `Get Listings`
  - `Get Rate History`
- **Node Details:**
  - `Schedule: Daily 15:00` (n8n-nodes-base.scheduleTrigger)
    - **Role:** Cron-based entry point firing daily at 15:00 (Asia/Jakarta timezone).
    - **Connections:** Triggers `Get Listings` and `Build Area Scan URL` in parallel.
  - `Get Listings` (n8n-nodes-base.googleSheets)
    - **Role:** Reads target competitor definitions from the `Listings` tab.
    - **Configuration:** Document ID set via `YOUR_GOOGLE_SHEET_ID`; reads row items.
    - **Connections:** Input from `Schedule`, output to `Get Rate History`.
    - **Edge Cases:** API timeout, authorization failure, or empty spreadsheet tab.
  - `Get Rate History` (n8n-nodes-base.googleSheets)
    - **Role:** Pulls historical rate logs from the `Rate_History` tab for delta calculations.
    - **Configuration:** Document ID set via `YOUR_GOOGLE_SHEET_ID`; executes once to feed downstream request generation.
    - **Connections:** Input from `Get Listings`, output to `Build Requests`.

#### 1.2 Request Builder & Platform Routing
- **Overview:** Calculates target stay dates for the next two weekends, normalizes and parameterizes URLs per platform requirements, and bifurcates processing logic based on platform-specific scraping paths.
- **Nodes Involved:**
  - `Build Requests`
  - `Is Booking (AI Path)?`
- **Node Details:**
  - `Build Requests` (n8n-nodes-base.code)
    - **Role:** JavaScript code node generating stay date offsets (weekends) and constructing localized platform URLs (Booking.com, tiket.com) with enforced currency parameters (`IDR`).
    - **Key Expressions:** Evaluates `$today` using Luxon to determine upcoming Friday/Saturday stay dates.
    - **Connections:** Input from `Get Rate History`, output to `Is Booking (AI Path)?`.
    - **Edge Cases:** Sandbox limitations regarding global objects; improperly formatted base URLs in the Google Sheet.
  - `Is Booking (AI Path)?` (n8n-nodes-base.if)
    - **Role:** Conditional router directing Booking.com requests to structured AI extraction and tiket.com requests to raw markdown retrieval.
    - **Configuration:** Checks if `{{ $json.extraction_path }}` equals `ai`.
    - **Connections:** Input from `Build Requests`; true branch outputs to `Loop Booking Pages`, false branch outputs to `Loop Markdown Pages`.

#### 1.3a Booking.com Scraping Track
- **Overview:** Iterates through Booking.com target URLs one by one, extracts structured hotel room rates via MrScraper, and normalizes the results against historical records.
- **Nodes Involved:**
  - `Loop Booking Pages`
  - `MrScraper Extract Booking`
  - `Normalize Booking Rates`
- **Node Details:**
  - `Loop Booking Pages` (n8n-nodes-base.splitInBatches)
    - **Role:** Manages sequential batch execution for Booking.com pages to respect rate limits.
    - **Connections:** Input from `Is Booking (AI Path)?` (true branch), outputs to `MrScraper Extract Booking` and `Merge Rates`.
  - `MrScraper Extract Booking` (n8n-nodes-mrscraper.mrscraper)
    - **Role:** Community node invoking MrScraper to extract structured hotel data (room types, prices, cancellation policies) using proxy country set to Indonesia (`ID`).
    - **Configuration:** Operation set to `scrapeStructuredData`; category set to `hotel`. Error handling set to `Continue (regular output)`.
    - **Connections:** Input from `Loop Booking Pages`, output to `Normalize Booking Rates`.
    - **Edge Cases:** Scraper timeouts, `empty_response` tokens, or upstream IP blocking.
  - `Normalize Booking Rates` (n8n-nodes-base.code)
    - **Role:** Parses raw scraper payloads, extracts numerical pricing, normalizes tax bases, compares values with historical baselines, and flags price anomalies (alerts).
    - **Connections:** Input from `MrScraper Extract Booking`, output loops back to `Loop Booking Pages`.
    - **Edge Cases:** Missing room type nodes, unexpected currency types, or unparsable price strings.

#### 1.3b Tiket.com Scraping Track
- **Overview:** Fetches rendered HTML markdowns for tiket.com in batches of five, cleans unnecessary assets, extracts pricing contexts via Gemini, and normalizes rates.
- **Nodes Involved:**
  - `Loop Markdown Pages`
  - `MrScraper Fetch Markdown`
  - `Prepare Gemini Batches`
  - `Gemini Extract Rooms`
  - `Normalize Markdown Rates`
- **Node Details:**
  - `Loop Markdown Pages` (n8n-nodes-base.splitInBatches)
    - **Role:** Batches markdown retrieval requests in groups of 5 to remain below platform parallel request limits.
    - **Connections:** Input from `Is Booking (AI Path)?` (false branch), outputs to `MrScraper Fetch Markdown` and `Prepare Gemini Batches`.
  - `MrScraper Fetch Markdown` (n8n-nodes-mrscraper.mrscraper)
    - **Role:** Fetches rendered page content in markdown format.
    - **Configuration:** Operation set to `scrape`, `markdown: true`, `homePage: true`. Error handling set to `Continue`.
    - **Connections:** Input from `Loop Markdown Pages`, output to `Loop Markdown Pages`.
  - `Prepare Gemini Batches` (n8n-nodes-base.code)
    - **Role:** Trims markdown content around price patterns to reduce token usage and groups pages into batches for Gemini extraction.
    - **Connections:** Input from `Loop Markdown Pages` (done), output to `Gemini Extract Rooms`.
  - `Gemini Extract Rooms` (@n8n/n8n-nodes-langchain.googleGemini)
    - **Role:** Uses Google Gemini (`models/gemini-3.6-flash`) to parse structured room names and pricing data from the cleaned markdown excerpts.
    - **Configuration:** Response forced to JSON without markdown fences. Error handling set to `Continue`.
    - **Connections:** Input from `Prepare Gemini Batches`, output to `Normalize Markdown Rates`.
    - **Edge Cases:** API quota exhaustion, HTTP 503 errors, or invalid JSON response structures (`llm_bad_json`).
  - `Normalize Markdown Rates` (n8n-nodes-base.code)
    - **Role:** Maps Gemini responses back to page IDs, processes unavailable rooms or fetch failures, and executes historical comparisons identically to the booking track.
    - **Connections:** Input from `Gemini Extract Rooms`, output to `Merge Rates`.

#### 1.4 Rate Persistence & Watchlist Reporting
- **Overview:** Consolidates all scraped rates, appends run logs to historical records, updates current status records, assesses data quality, and generates an AI-driven summary report.
- **Nodes Involved:**
  - `Merge Rates`
  - `Append Rate_History`
  - `Upsert Current_Status`
  - `Build Report`
  - `Has Data Issue?`
  - `Gemini Write Report`
  - `Merge Report`
- **Node Details:**
  - `Merge Rates` (n8n-nodes-base.merge)
    - **Role:** Combines normalized outputs from both Booking.com and tiket.com tracks.
    - **Connections:** Inputs from `Loop Booking Pages` and `Normalize Markdown Rates`, output to `Append Rate_History`.
  - `Append Rate_History` (n8n-nodes-base.googleSheets)
    - **Role:** Appends every extracted rate observation to the `Rate_History` tab along with execution metadata (`run_id`, `timestamp`).
    - **Connections:** Input from `Merge Rates`, output to `Upsert Current_Status`.
  - `Upsert Current_Status` (n8n-nodes-base.googleSheets)
    - **Role:** Updates or inserts the latest price observation per `rate_key` into the `Current_Status` tab.
    - **Configuration:** Matching column set to `rate_key`.
    - **Connections:** Input from `Append Rate_History`, output to `Build Report`.
  - `Build Report` (n8n-nodes-base.code)
    - **Role:** Aggregates platform health metrics, filters confirmed price alerts ($\ge 5\%$ delta over 2 consecutive scans), detects sold-out conditions, and compiles market trends.
    - **Connections:** Input from `Upsert Current_Status`, output to `Has Data Issue?`.
  - `Has Data Issue?` (n8n-nodes-base.if)
    - **Role:** Validates platform health; blocks AI report generation if no platforms returned valid pricing data.
    - **Configuration:** Evaluates `{{ $json.is_data_quality_issue }}`.
    - **Connections:** True branch outputs directly to `Merge Report` (bypassing AI), false branch outputs to `Gemini Write Report`.
  - `Gemini Write Report` (@n8n/n8n-nodes-langchain.googleGemini)
    - **Role:** Instructs Google Gemini to synthesize an Indonesian-language revenue analyst report based on JSON report payloads.
    - **Configuration:** Model: `models/gemini-3.6-flash`. Enforces strict character limits ($\le 1800$ chars) and specific content rules.
    - **Connections:** Input from `Has Data Issue?` (false branch), output to `Merge Report`.
  - `Merge Report` (n8n-nodes-base.merge)
    - **Role:** Reconciles the output paths from either the Gemini report generator or the data-quality error handler.
    - **Connections:** Inputs from `Has Data Issue?` (true branch) and `Gemini Write Report`, output to `Merge Watchlist and Area`.

#### 1.5 Area Scan & Telegram Distribution
- **Overview:** Scrapes competitive city-wide listings from Booking.com search result pages, summarizes local market pricing, merges watchlist data with area scan findings, and dispatches a unified Telegram notification.
- **Nodes Involved:**
  - `Build Area Scan URL`
  - `MrScraper Listing Agent`
  - `Split Listings`
  - `Has Listings?`
  - `Loop Area Listings`
  - `MrScraper Extract Listing`
  - `Normalize Area Listing`
  - `Append Area_Scan`
  - `Merge Area`
  - `Merge Watchlist and Area`
  - `Compose Telegram Message`
  - `Send Telegram Report`
- **Node Details:**
  - `Build Area Scan URL` (n8n-nodes-base.code)
    - **Role:** Generates search query URLs targeting a specified city (`Bandung`) with a defined check-in day offset (`7`).
    - **Connections:** Triggered in parallel by `Schedule`, output to `MrScraper Listing Agent`.
  - `MrScraper Listing Agent` (n8n-nodes-mrscraper.mrscraper)
    - **Role:** Executes an autonomous paginated listing agent to retrieve hotel search cards from Booking.com.
    - **Configuration:** Operation set to `listingAgentPaginated`; schema defines name, price, rating, and detail URL. Timeout: 600,000ms. Error handling set to `Continue`.
    - **Connections:** Input from `Build Area Scan URL`, output to `Split Listings`.
  - `Split Listings` (n8n-nodes-base.code)
    - **Role:** Cleans and normalizes search result objects, extracting up to the top 5 valid competitor property listings.
    - **Connections:** Input from `MrScraper Listing Agent`, output to `Has Listings?`.
  - `Has Listings?` (n8n-nodes-base.if)
    - **Role:** Checks whether valid listings were successfully extracted by the listing agent.
    - **Connections:** True branch routes to `Loop Area Listings`, false branch routes to `Merge Area` with an error marker.
  - `Loop Area Listings` (n8n-nodes-base.splitInBatches)
    - **Role:** Iterates sequentially through each discovered area listing.
    - **Connections:** Input from `Has Listings?` (true), outputs to `MrScraper Extract Listing` and `Append Area_Scan`.
  - `MrScraper Extract Listing` (n8n-nodes-mrscraper.mrscraper)
    - **Role:** Scrapes structured detail pages for individual area listings.
    - **Connections:** Input from `Loop Area Listings`, output to `Normalize Area Listing`.
  - `Normalize Area Listing` (n8n-nodes-base.code)
    - **Role:** Extracts minimum nightly room prices, falls back to search card pricing if detail extraction fails, and maps data to sheet schemas.
    - **Connections:** Input from `MrScraper Extract Listing`, output loops back to `Loop Area Listings`.
  - `Append Area_Scan` (n8n-nodes-base.googleSheets)
    - **Role:** Appends scraped area scan results to the `Area_Scan` spreadsheet tab.
    - **Connections:** Input from `Loop Area Listings`, output to `Merge Area`.
  - `Merge Area` (n8n-nodes-base.merge)
    - **Role:** Merges successfully appended area scans with empty-state fallback signals.
    - **Connections:** Inputs from `Append Area_Scan` and `Has Listings?` (false), output to `Merge Watchlist and Area`.
  - `Merge Watchlist and Area` (n8n-nodes-base.merge)
    - **Role:** Joins the completed watchlist intelligence payload with the area scan metrics payload.
    - **Connections:** Inputs from `Merge Report` and `Merge Area`, output to `Compose Telegram Message`.
  - `Compose Telegram Message` (n8n-nodes-base.code)
    - **Role:** Compiles both sections into a single HTML-formatted string ($\le 4096$ characters), applying programmatic fallbacks if Gemini text generation failed.
    - **Connections:** Input from `Merge Watchlist and Area`, output to `Send Telegram Report`.
  - `Send Telegram Report` (n8n-nodes-base.telegram)
    - **Role:** Sends the combined HTML notification message to the designated Telegram chat ID.
    - **Configuration:** Parse Mode set to `HTML`; chat ID configured via `YOUR_TELEGRAM_CHAT_ID`. Retry enabled.
    - **Connections:** Input from `Compose Telegram Message`.
    - **Edge Cases:** Telegram API character limit violations (mitigated by truncation code), invalid bot tokens, or chat permission blocks.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Schedule: Daily 15:00 (Watchlist + Area Scan)` | scheduleTrigger | Triggers the daily execution flow at 15:00. | None | `Get Listings`, `Build Area Scan URL` | Daily trigger<br>Runs every day at 15:00 in the workflow time zone. Starts the watchlist (top) and the area scan (bottom). |
| `Get Listings` | googleSheets | Loads monitored hotel competitor URLs and metadata. | `Schedule: Daily 15:00...` | `Get Rate History` | 1. Load the watchlist<br>**Get Listings** reads the Listings tab. **Get Rate History** reads earlier scans. New prices are compared with them. |
| `Get Rate History` | googleSheets | Reads historical rate logs from Google Sheets. | `Get Listings` | `Build Requests` | 1. Load the watchlist<br>**Get Listings** reads the Listings tab. **Get Rate History** reads earlier scans. New prices are compared with them. |
| `Build Requests` | code | Builds stay-date URLs for target platforms. | `Get Rate History` | `Is Booking (AI Path)?` | 2. Build URLs<br>**Build Requests** creates 1 URL per listing per stay date (next 2 weekends). Set `EXTRACTION_PATH` and `WEEKEND_COUNT` here. Booking.com goes to the AI path, tiket.com to the markdown path. |
| `Is Booking (AI Path)?` | if | Routes execution based on extraction path (AI vs. Markdown). | `Build Requests` | `Loop Booking Pages`, `Loop Markdown Pages` | |
| `Loop Booking Pages` | splitInBatches | Batches Booking.com requests. | `Is Booking (AI Path)?` | `MrScraper Extract Booking`, `Merge Rates` | 3a. Booking.com: MrScraper Extract Structured Data<br>1 page per batch, hotel category. Returns room types, prices, breakfast and cancellation policy. **Normalize Booking Rates** compares each rate with history. A failed page is saved with its error status (On Error: Continue). |
| `MrScraper Extract Booking` | mrscraper | Extracts structured hotel data from Booking.com. | `Loop Booking Pages` | `Normalize Booking Rates` | 3a. Booking.com: MrScraper Extract Structured Data<br>1 page per batch, hotel category. Returns room types, prices, breakfast and cancellation policy. **Normalize Booking Rates** compares each rate with history. A failed page is saved with its error status (On Error: Continue). |
| `Normalize Booking Rates` | code | Normalizes rates and computes price deltas for Booking. | `MrScraper Extract Booking` | `Loop Booking Pages` | 3a. Booking.com: MrScraper Extract Structured Data<br>1 page per batch, hotel category. Returns room types, prices, breakfast and cancellation policy. **Normalize Booking Rates** compares each rate with history. A failed page is saved with its error status (On Error: Continue). |
| `Loop Markdown Pages` | splitInBatches | Batches markdown requests for tiket.com. | `Is Booking (AI Path)?` | `MrScraper Fetch Markdown`, `Prepare Gemini Batches` | 3b. tiket.com: MrScraper Fetch Rendered HTML<br>5 pages per batch, below the Free plan limit of 10 parallel requests. Home Page = on (MrScraper team advice for tiket.com). **Prepare Gemini Batches** keeps the price lines only. |
| `MrScraper Fetch Markdown` | mrscraper | Fetches rendered page markdowns. | `Loop Markdown Pages` | `Loop Markdown Pages` | 3b. tiket.com: MrScraper Fetch Rendered HTML<br>5 pages per batch, below the Free plan limit of 10 parallel requests. Home Page = on (MrScraper team advice for tiket.com). **Prepare Gemini Batches** keeps the price lines only. |
| `Prepare Gemini Batches` | code | Prepares markdown excerpts for Gemini extraction. | `Loop Markdown Pages` | `Gemini Extract Rooms` | 3b. tiket.com: MrScraper Fetch Rendered HTML<br>5 pages per batch, below the Free plan limit of 10 parallel requests. Home Page = on (MrScraper team advice for tiket.com). **Prepare Gemini Batches** keeps the price lines only. |
| `Gemini Extract Rooms` | googleGemini | Extracts room types and prices using Google Gemini. | `Prepare Gemini Batches` | `Normalize Markdown Rates` | 4. Gemini room extraction<br>**Gemini Extract Rooms** reads room names and prices from the markdown. **Normalize Markdown Rates** maps them to each page and compares with history. **Merge Rates** joins both platforms. |
| `Normalize Markdown Rates` | code | Normalizes and compares tiket.com rates. | `Gemini Extract Rooms` | `Merge Rates` | 4. Gemini room extraction<br>**Gemini Extract Rooms** reads room names and prices from the markdown. **Normalize Markdown Rates** maps them to each page and compares with history. **Merge Rates** joins both platforms. |
| `Merge Rates` | merge | Merges normalized rates from all platforms. | `Loop Booking Pages`, `Normalize Markdown Rates` | `Append Rate_History` | |
| `Append Rate_History` | googleSheets | Appends rate observations to historical log sheet. | `Merge Rates` | `Upsert Current_Status` | 5. Save rates<br>**Append Rate_History** adds every rate of this run. **Upsert Current_Status** keeps the latest price per rate_key. Do not delete Rate_History rows. |
| `Upsert Current_Status` | googleSheets | Upserts latest rate status into Google Sheets. | `Append Rate_History` | `Build Report` | 5. Save rates<br>**Append Rate_History** adds every rate of this run. **Upsert Current_Status** keeps the latest price per rate_key. Do not delete Rate_History rows. |
| `Build Report` | code | Compiles market stats, alerts, and platform health. | `Upsert Current_Status` | `Has Data Issue?` | 6. Watchlist report<br>**Build Report** finds alerts (a change of 5% or more in 2 scans in a row), the cheapest listing per date, and platform health. If no platform has prices, a warning replaces the report. Otherwise **Gemini Write Report** writes it (Indonesian). |
| `Has Data Issue?` | if | Checks if valid platform coverage exists. | `Build Report` | `Merge Report`, `Gemini Write Report` | 6. Watchlist report<br>**Build Report** finds alerts (a change of 5% or more in 2 scans in a row), the cheapest listing per date, and platform health. If no platform has prices, a warning replaces the report. Otherwise **Gemini Write Report** writes it (Indonesian). |
| `Gemini Write Report` | googleGemini | Generates Indonesian analysis report using AI. | `Has Data Issue?` | `Merge Report` | 6. Watchlist report<br>**Build Report** finds alerts (a change of 5% or more in 2 scans in a row), the cheapest listing per date, and platform health. If no platform has prices, a warning replaces the report. Otherwise **Gemini Write Report** writes it (Indonesian). |
| `Merge Report` | merge | Reconciles report generation paths. | `Has Data Issue?`, `Gemini Write Report` | `Merge Watchlist and Area` | |
| `Build Area Scan URL` | code | Builds area search URLs for Booking.com. | `Schedule: Daily 15:00...` | `MrScraper Listing Agent` | 7. Area scan: MrScraper Listing Agent<br>**Build Area Scan URL** searches Booking.com for `AREA_CITY`, check-in in `DAY_OFFSET` days. Listing Agent timeout = 10 min, retry off: 1 search page can take more than 5 min, and each retry starts a new MrScraper job. **Split Listings** keeps the top 5. |
| `MrScraper Listing Agent` | mrscraper | Scrapes city-level competitor listings. | `Build Area Scan URL` | `Split Listings` | 7. Area scan: MrScraper Listing Agent<br>**Build Area Scan URL** searches Booking.com for `AREA_CITY`, check-in in `DAY_OFFSET` days. Listing Agent timeout = 10 min, retry off: 1 search page can take more than 5 min, and each retry starts a new MrScraper job. **Split Listings** keeps the top 5. |
| `Split Listings` | code | Filters top listings from area search results. | `MrScraper Listing Agent` | `Has Listings?` | 7. Area scan: MrScraper Listing Agent<br>**Build Area Scan URL** searches Booking.com for `AREA_CITY`, check-in in `DAY_OFFSET` days. Listing Agent timeout = 10 min, retry off: 1 search page can take more than 5 min, and each retry starts a new MrScraper job. **Split Listings** keeps the top 5. |
| `Has Listings?` | if | Validates presence of valid area listings. | `Split Listings` | `Loop Area Listings`, `Merge Area` | |
| `Loop Area Listings` | splitInBatches | Batches area listing detail scrapes. | `Has Listings?` | `MrScraper Extract Listing`, `Append Area_Scan` | 8. Listing details<br>**MrScraper Extract Listing** reads each listing page. **Normalize Area Listing** keeps the cheapest rate. If the detail page fails, it uses the price from the search page. |
| `MrScraper Extract Listing` | mrscraper | Extracts structured detail data for area listings. | `Loop Area Listings` | `Normalize Area Listing` | 8. Listing details<br>**MrScraper Extract Listing** reads each listing page. **Normalize Area Listing** keeps the cheapest rate. If the detail page fails, it uses the price from the search page. |
| `Normalize Area Listing` | code | Normalizes and selects cheapest rate per area listing. | `MrScraper Extract Listing` | `Loop Area Listings` | 8. Listing details<br>**MrScraper Extract Listing** reads each listing page. **Normalize Area Listing** keeps the cheapest rate. If the detail page fails, it uses the price from the search page. |
| `Append Area_Scan` | googleSheets | Appends area scan results to spreadsheet. | `Loop Area Listings` | `Merge Area` | 9. Save area scan<br>**Append Area_Scan** writes 1 row per listing. **Merge Area** joins this path with the no-listings path. |
| `Merge Area` | merge | Merges area scan save paths. | `Has Listings?`, `Append Area_Scan` | `Merge Watchlist and Area` | 9. Save area scan<br>**Append Area_Scan** writes 1 row per listing. **Merge Area** joins this path with the no-listings path. |
| `Merge Watchlist and Area` | merge | Combines watchlist report payload with area scan data. | `Merge Report`, `Merge Area` | `Compose Telegram Message` | 10. Message<br>Joins both reports into 1 HTML message (max 4096 characters). If Gemini fails, it uses a fixed format. |
| `Compose Telegram Message` | code | Composes final HTML Telegram message. | `Merge Watchlist and Area` | `Send Telegram Report` | 10. Message<br>Joins both reports into 1 HTML message (max 4096 characters). If Gemini fails, it uses a fixed format. |
| `Send Telegram Report` | telegram | Sends notification message to Telegram. | `Compose Telegram Message` | None | 11. Send<br>Set your chat ID here. See the **Telegram setup** note above. |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Set Up Credentials and Base Spreadsheet
1. Create a Google Sheet containing the following 4 tabs with exact header structures:
   - **Listings:** `listing_id`, `nama_properti`, `url`, `platform`, `lokasi`
   - **Rate_History:** `timestamp`, `run_id`, `listing_id`, `nama_properti`, `platform`, `target_date`, `room_type`, `price`, `previous_price`, `delta_pct`, `availability_status`, `breakfast_included`, `cancellation_policy`, `extraction_status`, `rate_key`, `is_new`, `status_changed`, `is_alert`
   - **Current_Status:** `rate_key`, `listing_id`, `nama_properti`, `platform`, `target_date`, `room_type`, `price`, `availability_status`, `last_updated`
   - **Area_Scan:** `scan_date`, `platform`, `nama_properti`, `price_per_night`, `rating`, `detail_url`
2. Configure **Google Sheets OAuth2** credentials in n8n.
3. Configure **MrScraper** API credentials and install the community node `n8n-nodes-mrscraper`.
4. Configure **Google Gemini (PaLM) API** credentials.
5. Configure a Telegram Bot via BotFather and create a **Telegram** bot credential in n8n.

#### Step 2: Build the Trigger and Input Loaders
1. Add a **Schedule Trigger** node (`Schedule: Daily 15:00`) set to fire daily at 15:00 (`Asia/Jakarta`).
2. Add a **Google Sheets** node (`Get Listings`):
   - Operation: `Get`
   - Sheet Name: `Listings`
   - Document ID: Your Spreadsheet ID.
   - Connect input: `Schedule: Daily 15:00`.
3. Add a **Google Sheets** node (`Get Rate History`):
   - Operation: `Get`
   - Sheet Name: `Rate_History`
   - Document ID: Your Spreadsheet ID.
   - Connect input: `Get Listings`.

#### Step 3: Configure URL Building and Platform Routing
1. Add a **Code** node (`Build Requests`):
   - Insert JavaScript logic to calculate upcoming weekend dates and build parameterized URLs for Booking.com (`en-gb.html`, IDR currency) and tiket.com.
   - Connect input: `Get Rate History`.
2. Add an **If** node (`Is Booking (AI Path)?`):
   - Condition: `{{ $json.extraction_path }}` equals `ai`.
   - Connect input: `Build Requests`.

#### Step 4: Implement Booking.com Path
1. Add a **Split In Batches** node (`Loop Booking Pages`), connected to the `true` branch of `Is Booking (AI Path)?`.
2. Add a **MrScraper** node (`MrScraper Extract Booking`):
   - Resource: `scraping`, Operation: `scrapeStructuredData`
   - URL: `{{ $json.url }}`
   - Category: `hotel`, Proxy Country: `ID`
   - On Error: `Continue (regular output)`.
   - Connect input: `Loop Booking Pages`.
3. Add a **Code** node (`Normalize Booking Rates`):
   - Insert normalization, price parsing, and historical comparison algorithms.
   - Connect input: `MrScraper Extract Booking`, output loops back to `Loop Booking Pages`.

#### Step 5: Implement Tiket.com Path
1. Add a **Split In Batches** node (`Loop Markdown Pages`), connected to the `false` branch of `Is Booking (AI Path)?` with batch size set to `5`.
2. Add a **MrScraper** node (`MrScraper Fetch Markdown`):
   - Resource: `scraping`, Operation: `scrape`
   - URL: `{{ $json.url }}`, Markdown: `true`, Home Page: `true`
   - Proxy Country: `id`, On Error: `Continue`.
   - Connect input: `Loop Markdown Pages`, output loops back to `Loop Markdown Pages`.
3. Add a **Code** node (`Prepare Gemini Batches`):
   - Filters and excerpts price lines from markdowns into batches.
   - Connect input: `Loop Markdown Pages` (done), output to `Gemini Extract Rooms`.
4. Add a **Google Gemini** node (`Gemini Extract Rooms`):
   - Model: `models/gemini-3.6-flash`
   - System/User prompt: Instruct extraction of room types and prices into strict JSON format.
   - On Error: `Continue`.
   - Connect input: `Prepare Gemini Batches`, output to `Normalize Markdown Rates`.
5. Add a **Code** node (`Normalize Markdown Rates`):
   - Parses LLM output, handles errors, and executes historical comparisons.
   - Connect input: `Gemini Extract Rooms`, output to `Merge Rates`.

#### Step 6: Consolidate Rates and Update Storage
1. Add a **Merge** node (`Merge Rates`):
   - Combines inputs from `Loop Booking Pages` and `Normalize Markdown Rates`.
2. Add a **Google Sheets** node (`Append Rate_History`):
   - Operation: `Append`, Sheet Name: `Rate_History`
   - Map all rate fields (`timestamp`, `run_id`, `listing_id`, `price`, `rate_key`, etc.).
   - Connect input: `Merge Rates`.
3. Add a **Google Sheets** node (`Upsert Current_Status`):
   - Operation: `Append or Update`, Sheet Name: `Current_Status`
   - Matching Column: `rate_key`.
   - Connect input: `Append Rate_History`.

#### Step 7: Build Watchlist Report & AI Summary
1. Add a **Code** node (`Build Report`):
   - Calculates platform health, detects market minimums, and compiles alert lists.
   - Connect input: `Upsert Current_Status`.
2. Add an **If** node (`Has Data Issue?`):
   - Condition: `{{ $json.is_data_quality_issue }}` equals `true`.
   - Connect input: `Build Report`.
3. Add a **Google Gemini** node (`Gemini Write Report`):
   - Model: `models/gemini-3.6-flash`
   - Prompt: Instructs synthesis of Indonesian-language revenue report based on report JSON.
   - Connect input: `Has Data Issue?` (false branch).
4. Add a **Merge** node (`Merge Report`):
   - Reconciles paths from `Has Data Issue?` (true branch) and `Gemini Write Report`.

#### Step 8: Build Area Scan Track
1. Add a **Code** node (`Build Area Scan URL`):
   - Generates Booking.com search URLs for target city (`Bandung`) with check-in offset (`7`).
   - Connect input: `Schedule: Daily 15:00...`.
2. Add a **MrScraper** node (`MrScraper Listing Agent`):
   - Resource: `scraping`, Operation: `listingAgentPaginated`
   - URL: `{{ $json.area_scan_url }}`, Prompt: Extract names, prices, ratings, detail URLs. Timeout: `600000ms`.
   - Connect input: `Build Area Scan URL`.
3. Add a **Code** node (`Split Listings`):
   - Filters and cleans top 5 listing items.
   - Connect input: `MrScraper Listing Agent`.
4. Add an **If** node (`Has Listings?`):
   - Condition: `{{ $json.has_listing }}` equals `true`.
   - Connect input: `Split Listings`.
5. Add a **Split In Batches** node (`Loop Area Listings`), connected to true branch of `Has Listings?`.
6. Add a **MrScraper** node (`MrScraper Extract Listing`):
   - Resource: `scraping`, Operation: `scrapeStructuredData`
   - URL: `{{ $json.detail_url }}`, Category: `hotel`.
   - Connect input: `Loop Area Listings`.
7. Add a **Code** node (`Normalize Area Listing`):
   - Extracts cheapest nightly rate.
   - Connect input: `MrScraper Extract Listing`, output loops back to `Loop Area Listings`.
8. Add a **Google Sheets** node (`Append Area_Scan`):
   - Operation: `Append`, Sheet Name: `Area_Scan`.
   - Connect input: `Loop Area Listings`.
9. Add a **Merge** node (`Merge Area`):
   - Reconciles `Append Area_Scan` output with `Has Listings?` (false branch).

#### Step 9: Compose and Send Telegram Notification
1. Add a **Merge** node (`Merge Watchlist and Area`):
   - Inputs: `Merge Report` and `Merge Area`.
2. Add a **Code** node (`Compose Telegram Message`):
   - Formats watchlist and area scan data into an HTML-compliant string ($\le 4096$ characters) with programmatic fallbacks.
   - Connect input: `Merge Watchlist and Area`.
3. Add a **Telegram** node (`Send Telegram Report`):
   - Resource: `message`, Operation: `send`
   - Text: `{{ $json.telegram_html }}`
   - Chat ID: `YOUR_TELEGRAM_CHAT_ID`
   - Additional Fields: Parse Mode = `HTML`, Append Attribution = `false`.
   - Connect input: `Compose Telegram Message`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Google Sheet Template | [Google Sheets Template Copy Link](https://docs.google.com/spreadsheets/d/1toPSjbY7JspOkHNheB7D3L-ezYAvjM7rBaFr8oHOdlI/copy) |
| MrScraper Service Sign-Up | [MrScraper Platform](https://mrscraper.com) |
| MrScraper n8n Documentation | [MrScraper n8n Integration Docs](https://docs.mrscraper.com/docs/integrations/n8n) |
| Telegram Bot Setup Instructions | Create a bot via [@BotFather](https://t.me/BotFather) using `/newbot`, retrieve updates via `https://api.telegram.org/bot<TOKEN>/getUpdates` to identify chat IDs. |
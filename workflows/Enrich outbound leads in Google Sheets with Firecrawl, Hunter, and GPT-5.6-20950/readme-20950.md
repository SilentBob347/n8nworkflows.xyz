Enrich outbound leads in Google Sheets with Firecrawl, Hunter, and GPT-5.6

https://n8nworkflows.xyz/workflows/enrich-outbound-leads-in-google-sheets-with-firecrawl--hunter--and-gpt-5-6-20950


# Enrich outbound leads in Google Sheets with Firecrawl, Hunter, and GPT-5.6

### 1. Workflow Overview

This workflow is an outbound lead enrichment and qualification pipeline designed to process pending leads stored in a Google Sheet. It automates the discovery of corporate domains, website scraping, AI-driven firmographic extraction, decision-maker discovery, multi-provider email waterfall verification, and personalized cold opener generation using strict anti-hallucination guardrails. Finally, it updates the spreadsheet with results and sends an aggregated HTML run report via email.

The logic is organized into the following sequential blocks:
- **1.1 Initialization and Intake:** Handles manual or scheduled triggers, configuration loading, database reading, batch sizing, and execution loops.
- **1.2 Company Discovery and Web Scraping:** Normalizes user-provided websites or discovers missing domains via search engines, then scrapes homepages and "About" pages to build structured research contexts.
- **1.3 AI Company Research & Grounding:** Uses an LLM (OpenRouter) to extract firmographics, signals, and ICP scores, verifying every extracted quote against source markdown to prevent hallucinations.
- **1.4 Prospect Discovery & Email Waterfall:** Identifies decision-makers by role/department and executes a sequential waterfall (website emails -> Hunter -> Dropcontact) validated by Hunter’s email verifier within strict budget constraints.
- **1.5 Grounded Cold Opener Generation:** Evaluates prospect qualification and uses an LLM to generate a single-sentence personalized cold email opener anchored in a verified fact, checking against strict style and grounding rules.
- **1.6 Write-Back and Reporting:** Assembles terminal row statuses, writes results back to Google Sheets, summarizes run economics, and dispatches an HTML email report.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Initialization and Intake
- **Overview:** Initializes the execution environment on a schedule or via manual invocation, loads global configuration parameters, fetches pending leads from Google Sheets, caps processing to a defined batch limit, and loops through rows individually.
- **Nodes Involved:** `Run Table Now`, `Every 30 Minutes`, `Table Settings`, `Read Pending Rows`, `Pick Batch`, `Loop Rows`, `Check Run Budget`, `Row Needs Work`, `Skip To Write-Back`
- **Node Details:**
  - `Run Table Now` (n8n-nodes-base.manualTrigger): Initiates manual executions. No parameters. Outputs to `Table Settings`.
  - `Every 30 Minutes` (n8n-nodes-base.scheduleTrigger): Triggers the pipeline automatically every 30 minutes via interval rules. Outputs to `Table Settings`.
  - `Table Settings` (n8n-nodes-base.set): Establishes global control variables including `batch_size` (25), `max_credits_per_run` (40), `min_verifier_score` (80), `min_icp_fit_for_opener` (50), credit valuations, target ICP descriptions, value propositions, target departments/titles, and report email.
  - `Read Pending Rows` (n8n-nodes-base.googleSheets): Queries the Google Sheets document using lookup filters (`status` equals `pending`). Requires Google Sheets OAuth2 credentials.
  - `Pick Batch` (n8n-nodes-base.code): JavaScript node that slices raw rows to match `batch_size`. Marks rows lacking a company name as `skipped` immediately.
  - `Loop Rows` (n8n-nodes-base.splitInBatches): Iterates over the batch items one by one. Connects back from write-back operations or loops to summaries.
  - `Check Run Budget` (n8n-nodes-base.code): Calculates cumulative credit consumption across prior loop iterations to ensure strict adherence to `max_credits_per_run`.
  - `Row Needs Work` (n8n-nodes-base.if): Evaluates whether a row's status is set to `skipped`. Bypasses processing for invalid rows.
  - `Skip To Write-Back` (n8n-nodes-base.noOp): Pass-through hub collecting unprocessed or domain-less rows, routing them directly to `Finalize Row`.

---

#### Block 1.2: Company Discovery and Web Scraping
- **Overview:** Extracts or discovers corporate domains from raw inputs or search engines, scrapes homepage content and relevant subpages (such as "About" or "Team"), and compiles a raw research corpus.
- **Nodes Involved:** `Has Website`, `Normalize Website`, `Search Company Domain`, `Pick Domain`, `Domain Found`, `Scrape Homepage`, `Find About Page`, `About Page Exists`, `Scrape About Page`, `Build Research Context`
- **Node Details:**
  - `Has Website` (n8n-nodes-base.if): Branches execution based on whether the `website` field is populated.
  - `Normalize Website` (n8n-nodes-base.code): Cleans user-submitted URLs by removing protocol handlers (`https://`), `www.` prefixes, and trailing paths.
  - `Search Company Domain` (@mendable/n8n-nodes-firecrawl.firecrawl): Performs a Firecrawl MapSearch query using the company name to discover official URLs. Requires Firecrawl API credentials.
  - `Pick Domain` (n8n-nodes-base.code): Filters search results against a list of banned host directories (e.g., LinkedIn, Wikipedia) and matches tokens against the company name.
  - `Domain Found` (n8n-nodes-base.if): Checks if a valid domain was successfully derived. Routes to scraping or bypasses to write-back if missing.
  - `Scrape Homepage` (@mendable/n8n-nodes-firecrawl.firecrawl): Scrapes the homepage markdown and extracts outgoing links. Configured with error handling (`continueRegularOutput`).
  - `Find About Page` (n8n-nodes-base.code): Parses scraped homepage links to identify "About", "Team", or "Company" subpages sharing the target domain.
  - `About Page Exists` (n8n-nodes-base.if): Determines whether a valid About page URL was identified.
  - `Scrape About Page` (@mendable/n8n-nodes-firecrawl.firecrawl): Scrapes markdown content from the identified About page.
  - `Build Research Context` (n8n-nodes-base.code): Merges homepage and About page markdown (capped at 12,000 characters) and extracts same-domain email addresses found within the page text.

---

#### Block 1.3: AI Company Research & Grounding
- **Overview:** Analyzes the scraped website text using an LLM via OpenRouter to extract firmographics, value propositions, recent signals, and ICP fit scores, followed by strict string-matching verification to eliminate ungrounded claims.
- **Nodes Involved:** `Research Company`, `Research Model`, `Research Schema`, `Check Research Grounding`
- **Node Details:**
  - `Research Company` (@n8n/n8n-nodes-langchain.chainLlm): LangChain structured LLM chain configured with prompt instructions enforcing strict B2B research guidelines and verbatim quotation rules.
  - `Research Model` (@n8n/n8n-nodes-langchain.lmChatOpenRouter): OpenRouter chat model integration configured for `openai/gpt-5.6-luna` with temperature set to `0`. Requires OpenRouter API credentials.
  - `Research Schema` (@n8n/n8n-nodes-langchain.outputParserStructured): Enforces a JSON schema requiring fields for products, customer types, signals, ICP scores, justifications, and grounded facts with exact quotes.
  - `Check Research Grounding` (n8n-nodes-base.code): Anti-hallucination code gate that performs case-insensitive substring checks of every extracted quote against the raw website text. Drops unverified facts and categorizes research status (`grounded`, `partial`, or `failed`).

---

#### Block 1.4: Prospect Discovery & Email Waterfall
- **Overview:** Identifies decision-makers by querying Hunter if contact names are missing, then runs a sequential waterfall (website emails -> Hunter Email Finder -> Dropcontact enrichment) with continuous verification via Hunter.
- **Nodes Involved:** `Has Person`, `Find Decision Maker`, `Pick Decision Maker`, `Person Found`, `Waterfall Stopped`, `Plan Waterfall`, `Website Email Found`, `Verify Website Email`, `Judge Website Email`, `Settled After Website`, `Hunter Budget Left`, `Hunter Email Finder`, `Hunter Found Email`, `Verify Hunter Email`, `Judge Hunter Email`, `Settled After Hunter`, `Dropcontact Budget Left`, `Dropcontact Enrich`, `Dropcontact Found Email`, `Verify Dropcontact Email`, `Judge Dropcontact Email`, `Assemble Email Result`
- **Node Details:**
  - `Has Person` (n8n-nodes-base.if): Checks if `first_name` and `last_name` are populated.
  - `Find Decision Maker` (n8n-nodes-base.hunter): Executes a Hunter domain search to retrieve senior executive contacts matching target departments. Requires Hunter API credentials.
  - `Pick Decision Maker` (n8n-nodes-base.code): Ranks returned contacts based on title and department relevance against `target_titles` and `target_departments`.
  - `Person Found` (n8n-nodes-base.if): Validates whether a decision-maker was successfully identified.
  - `Waterfall Stopped` (n8n-nodes-base.noOp): Convergence point collecting early exits from the waterfall.
  - `Plan Waterfall` (n8n-nodes-base.code): Matches scraped website emails against name-pattern permutations and initializes credit tracking variables.
  - `Website Email Found` / `Verify Website Email` / `Judge Website Email` / `Settled After Website`: Checks, verifies via Hunter, and evaluates whether a website-extracted email is deliverable or accept-all.
  - `Hunter Budget Left` / `Hunter Email Finder` / `Hunter Found Email` / `Verify Hunter Email` / `Judge Hunter Email` / `Settled After Hunter`: Evaluates remaining run budget, queries Hunter Email Finder, verifies results, and assesses deliverability status.
  - `Dropcontact Budget Left` / `Dropcontact Enrich` / `Dropcontact Found Email` / `Verify Dropcontact Email` / `Judge Dropcontact Email`: Fallback enrichment node querying Dropcontact for missing contact details, followed by Hunter verification.
  - `Assemble Email Result` (n8n-nodes-base.code): Finalizes email waterfall results, aggregates audit trails (`providers_tried`), and computes total credits used.

---

#### Block 1.5: Grounded Cold Opener Generation
- **Overview:** Determines whether a row is fully qualified for outreach, then uses an LLM to generate a concise, personalized cold email opener referencing a single verified fact without using canned phrases or unverified metrics.
- **Nodes Involved:** `Worth An Opener`, `Write Opener`, `Opener Model`, `Opener Schema`, `Check Opener`
- **Node Details:**
  - `Worth An Opener` (n8n-nodes-base.if): Evaluates whether the email status is verified/accept-all, research succeeded, and ICP fit meets or exceeds the minimum threshold.
  - `Write Opener` (@n8n/n8n-nodes-langchain.chainLlm): LangChain AI node instructed to draft a single sentence (12–30 words) commenting on a specific grounded fact from the prospect's perspective.
  - `Opener Model` (@n8n/n8n-nodes-langchain.lmChatOpenRouter): OpenRouter chat model (`openai/gpt-5.6-luna`) configured with a temperature of `0.4`. Requires OpenRouter API credentials.
  - `Opener Schema` (@n8n/n8n-nodes-langchain.outputParserStructured): Enforces JSON output parsing containing `opener`, `fact_id`, and `angle`.
  - `Check Opener` (n8n-nodes-base.code): Audits the generated opener against word count limits (<45 words), banned phrases (e.g., "hope this email finds you"), em dashes, exclamation points, and ensures any numbers mentioned appear verbatim in the cited fact.

---

#### Block 1.6: Write-Back and Reporting
- **Overview:** Assigns terminal statuses to processed leads, updates Google Sheets rows, aggregates run performance metrics, formats an HTML email report, and dispatches the execution digest via Gmail.
- **Nodes Involved:** `Finalize Row`, `Write Row Back`, `Summarize Run`, `Format Run Report`, `Email Run Report`
- **Node Details:**
  - `Finalize Row` (n8n-nodes-base.code): Compiles final row data, merges error logs, and assigns terminal statuses (`enriched`, `no_email`, `no_domain`, `no_person`, `skipped`, or `error`).
  - `Write Row Back` (n8n-nodes-base.googleSheets): Updates the corresponding row in Google Sheets using `row_id` as the matching column. Requires Google Sheets OAuth2 credentials.
  - `Summarize Run` (n8n-nodes-base.code): Aggregates run metrics across all loop iterations, calculating total processed rows, exact versus accept-all email yields, provider hit rates, API vendor costs in USD, and top ICP fits.
  - `Format Run Report` (n8n-nodes-base.code): Generates a structured HTML email layout summarizing waterfall yields, run economics, opener performance, and top target accounts.
  - `Email Run Report` (n8n-nodes-base.gmail): Sends the formatted HTML digest to the configured recipient address (`report_email`). Requires Gmail OAuth2 credentials.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Run Table Now | n8n-nodes-base.manualTrigger | Manual execution entry point | None | Table Settings | Clay Waterfall Enrichment |
| Every 30 Minutes | n8n-nodes-base.scheduleTrigger | Scheduled execution trigger (30m) | None | Table Settings | Clay Waterfall Enrichment |
| Table Settings | n8n-nodes-base.set | Sets global configuration variables | Run Table Now, Every 30 Minutes | Read Pending Rows | Clay Waterfall Enrichment |
| Read Pending Rows | n8n-nodes-base.googleSheets | Reads pending leads from Google Sheets | Table Settings | Pick Batch | Clay Waterfall Enrichment |
| Pick Batch | n8n-nodes-base.code | Caps batch size and skips invalid rows | Read Pending Rows | Loop Rows | Clay Waterfall Enrichment<br>A. Table Intake |
| Loop Rows | n8n-nodes-base.splitInBatches | Iterates batch rows sequentially | Pick Batch | Check Run Budget, Summarize Run | Clay Waterfall Enrichment<br>A. Table Intake |
| Check Run Budget | n8n-nodes-base.code | Calculates cumulative run credit usage | Loop Rows | Row Needs Work | Clay Waterfall Enrichment<br>A. Table Intake |
| Row Needs Work | n8n-nodes-base.if | Filters out skipped rows | Check Run Budget | Has Website, Skip To Write-Back | Clay Waterfall Enrichment<br>A. Table Intake |
| Skip To Write-Back | n8n-nodes-base.noOp | Pass-through for skipped/domain-less rows | Row Needs Work, Domain Found | Finalize Row | Clay Waterfall Enrichment<br>A. Table Intake |
| Has Website | n8n-nodes-base.if | Checks if website input is provided | Row Needs Work | Normalize Website, Search Company Domain | Clay Waterfall Enrichment<br>B. Company Research |
| Normalize Website | n8n-nodes-base.code | Normalizes input website URL | Has Website | Domain Found | Clay Waterfall Enrichment<br>B. Company Research |
| Search Company Domain | @mendable/n8n-nodes-firecrawl.firecrawl | Searches web for official company domain | Has Website | Pick Domain | Clay Waterfall Enrichment<br>B. Company Research |
| Pick Domain | n8n-nodes-base.code | Filters search results for valid domain | Search Company Domain | Domain Found | Clay Waterfall Enrichment<br>B. Company Research |
| Domain Found | n8n-nodes-base.if | Verifies domain presence | Normalize Website, Pick Domain | Scrape Homepage, Skip To Write-Back | Clay Waterfall Enrichment<br>B. Company Research |
| Scrape Homepage | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes homepage markdown and links | Domain Found | Find About Page | Clay Waterfall Enrichment<br>B. Company Research |
| Find About Page | n8n-nodes-base.code | Identifies About/Team subpage links | Scrape Homepage | About Page Exists | Clay Waterfall Enrichment<br>B. Company Research |
| About Page Exists | n8n-nodes-base.if | Checks if About page was found | Find About Page | Scrape About Page, Build Research Context | Clay Waterfall Enrichment<br>B. Company Research |
| Scrape About Page | @mendable/n8n-nodes-firecrawl.firecrawl | Scrapes About page markdown | About Page Exists | Build Research Context | Clay Waterfall Enrichment<br>B. Company Research |
| Build Research Context | n8n-nodes-base.code | Combines pages and extracts domain emails | About Page Exists, Scrape About Page | Research Company | Clay Waterfall Enrichment<br>B. Company Research |
| Research Company | @n8n/n8n-nodes-langchain.chainLlm | AI chain extracting business intelligence | Build Research Context, Research Model, Research Schema | Check Research Grounding | Clay Waterfall Enrichment<br>B. Company Research |
| Research Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | OpenRouter chat model for research | None | Research Company | Clay Waterfall Enrichment<br>B. Company Research |
| Research Schema | @n8n/n8n-nodes-langchain.outputParserStructured | Structured schema for research extraction | None | Research Company | Clay Waterfall Enrichment<br>B. Company Research |
| Check Research Grounding | n8n-nodes-base.code | Verifies quote grounding in source text | Research Company | Has Person | Clay Waterfall Enrichment<br>B. Company Research |
| Has Person | n8n-nodes-base.if | Checks if prospect name is provided | Check Research Grounding | Plan Waterfall, Find Decision Maker | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Find Decision Maker | n8n-nodes-base.hunter | Searches Hunter for decision-maker | Has Person | Pick Decision Maker | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Pick Decision Maker | n8n-nodes-base.code | Ranks and selects best decision-maker | Find Decision Maker | Person Found | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Person Found | n8n-nodes-base.if | Checks if decision-maker was found | Pick Decision Maker | Plan Waterfall, Waterfall Stopped | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Waterfall Stopped | n8n-nodes-base.noOp | Collects early waterfall exits | Person Found, Settled After Website, Settled After Hunter, Hunter Budget Left, Dropcontact Budget Left | Assemble Email Result | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Plan Waterfall | n8n-nodes-base.code | Plans email waterfall and checks site emails | Has Person, Person Found | Website Email Found | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Website Email Found | n8n-nodes-base.if | Checks if website email candidate exists | Plan Waterfall | Verify Website Email, Settled After Website | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Verify Website Email | n8n-nodes-base.hunter | Verifies website candidate email via Hunter | Website Email Found | Judge Website Email | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Judge Website Email | n8n-nodes-base.code | Evaluates website email verification result | Verify Website Email | Settled After Website | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Settled After Website | n8n-nodes-base.if | Checks if email is resolved after website | Judge Website Email | Waterfall Stopped, Hunter Budget Left | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Hunter Budget Left | n8n-nodes-base.if | Checks budget before querying Hunter | Settled After Website | Hunter Email Finder, Waterfall Stopped | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Hunter Email Finder | n8n-nodes-base.hunter | Finds email via Hunter domain search | Hunter Budget Left | Hunter Found Email | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Hunter Found Email | n8n-nodes-base.if | Checks if Hunter returned an email | Hunter Email Finder | Verify Hunter Email, Judge Hunter Email | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Verify Hunter Email | n8n-nodes-base.hunter | Verifies Hunter email via verifier | Hunter Found Email | Judge Hunter Email | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Judge Hunter Email | n8n-nodes-base.code | Evaluates Hunter email verification result | Verify Hunter Email, Hunter Found Email | Settled After Hunter | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Settled After Hunter | n8n-nodes-base.if | Checks if email is resolved after Hunter | Judge Hunter Email | Waterfall Stopped, Dropcontact Budget Left | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Dropcontact Budget Left | n8n-nodes-base.if | Checks budget before querying Dropcontact | Settled After Hunter | Dropcontact Enrich, Waterfall Stopped | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Dropcontact Enrich | n8n-nodes-base.dropcontact | Enriches lead via Dropcontact | Dropcontact Budget Left | Dropcontact Found Email | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Dropcontact Found Email | n8n-nodes-base.if | Checks if Dropcontact returned an email | Dropcontact Enrich | Verify Dropcontact Email, Judge Dropcontact Email | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Verify Dropcontact Email | n8n-nodes-base.hunter | Verifies Dropcontact email via Hunter | Dropcontact Found Email | Judge Dropcontact Email | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Judge Dropcontact Email | n8n-nodes-base.code | Evaluates Dropcontact email and verification | Verify Dropcontact Email | Assemble Email Result | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Assemble Email Result | n8n-nodes-base.code | Assembles final email waterfall results | Waterfall Stopped, Judge Dropcontact Email | Worth An Opener | Clay Waterfall Enrichment<br>C. Email Waterfall |
| Worth An Opener | n8n-nodes-base.if | Checks qualification for cold opener | Assemble Email Result | Write Opener, Finalize Row | Clay Waterfall Enrichment<br>D. Grounded Opener |
| Write Opener | @n8n/n8n-nodes-langchain.chainLlm | AI chain generating personalized opener | Worth An Opener, Opener Model, Opener Schema | Check Opener | Clay Waterfall Enrichment<br>D. Grounded Opener |
| Opener Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | OpenRouter chat model for opener generation | None | Write Opener | Clay Waterfall Enrichment<br>D. Grounded Opener |
| Opener Schema | @n8n/n8n-nodes-langchain.outputParserStructured | Structured schema for opener generation | None | Write Opener | Clay Waterfall Enrichment<br>D. Grounded Opener |
| Check Opener | n8n-nodes-base.code | Audits opener against anti-hallucination rules | Write Opener | Finalize Row | Clay Waterfall Enrichment<br>D. Grounded Opener |
| Finalize Row | n8n-nodes-base.code | Compiles final row status and errors | Skip To Write-Back, Worth An Opener, Check Opener | Write Row Back | Clay Waterfall Enrichment<br>E. Write Back |
| Write Row Back | n8n-nodes-base.googleSheets | Updates row status and data in Google Sheets | Finalize Row | Loop Rows | Clay Waterfall Enrichment<br>E. Write Back |
| Summarize Run | n8n-nodes-base.code | Aggregates overall batch execution metrics | Loop Rows | Format Run Report | Clay Waterfall Enrichment<br>Run Report |
| Format Run Report | n8n-nodes-base.code | Formats HTML digest for email notification | Summarize Run | Email Run Report | Clay Waterfall Enrichment<br>Run Report |
| Email Run Report | n8n-nodes-base.gmail | Sends run report digest via Gmail | Format Run Report | None | Clay Waterfall Enrichment<br>Run Report |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Triggers & Configuration Setup
1. Create a **Manual Trigger** (`Run Table Now`) and a **Schedule Trigger** (`Every 30 Minutes`, set interval to 30 minutes).
2. Create a **Set** node (`Table Settings`). Configure assignments:
   - `batch_size` (Number): `25`
   - `max_credits_per_run` (Number): `40`
   - `min_verifier_score` (Number): `80`
   - `min_icp_fit_for_opener` (Number): `50`
   - `usd_per_hunter_credit` (Number): `0.0245`
   - `usd_per_dropcontact_credit` (Number): `0.158`
   - `icp_description` (String): `B2B SaaS companies selling to mid-market and enterprise with 20-500 employees`
   - `your_offer` (String): `We build custom outbound enrichment and lead qualification workflows that cut data vendor spend by 70%.`
   - `report_email` (String): `user@example.com`
   - `target_departments` (String): `sales,executive,marketing`
   - `target_titles` (String): `head of sales, vp sales, sales director, revenue, growth, founder, ceo`

#### Step 2: Intake & Batching
1. Add a **Google Sheets** node (`Read Pending Rows`). Set Operation to `Lookup`, Sheet Name to `Leads`, Lookup Column to `status`, and Lookup Value to `pending`. Connect Google Sheets OAuth2 credentials.
2. Add a **Code** node (`Pick Batch`) to slice incoming rows by `batch_size` and flag missing required company names as `skipped`.
3. Add a **Split In Batches** node (`Loop Rows`) configured with default options.
4. Add a **Code** node (`Check Run Budget`) to calculate cumulative spent credits from prior loop passes.
5. Add an **If** node (`Row Needs Work`) checking that `status` does not equal `skipped`.

#### Step 3: Domain Resolution & Web Scraping
1. Add an **If** node (`Has Website`) checking if the `website` field is not empty.
2. If true, add a **Code** node (`Normalize Website`) to clean the URL. If false, add a **Firecrawl** node (`Search Company Domain`) configured for MapSearch / Search operation (`query: "{{ $json.company }}" official website`) using Firecrawl API credentials.
3. If search is used, add a **Code** node (`Pick Domain`) to filter out banned directories and validate matching name tokens.
4. Add an **If** node (`Domain Found`) checking if `domain` is present. Route missing domains to a **NoOp** node (`Skip To Write-Back`).
5. Add a **Firecrawl** node (`Scrape Homepage`) using URL `https://{{ $json.domain }}` with markdown and links format. Set error handling to `Continue Regular Output`.
6. Add a **Code** node (`Find About Page`) to scan scraped links for About/Team pages.
7. Add an **If** node (`About Page Exists`) checking `has_about_page`.
8. If true, add a **Firecrawl** node (`Scrape About Page`) targeting `{{ $json.about_url }}` with error handling set to `Continue Regular Output`.
9. Add a **Code** node (`Build Research Context`) to combine homepage and About page markdown (capped at 12,000 characters) and extract same-domain emails.

#### Step 4: AI Research & Grounding
1. Add an **Advanced AI LangChain** node (`Research Company`) configured as a Question and Answer Chain.
2. Connect an **OpenAI Chat Model** node (`Research Model`) set to model `openai/gpt-5.6-luna` with temperature `0` using OpenRouter credentials.
3. Connect a **Structured Output Parser** node (`Research Schema`) with a JSON schema defining `what_they_sell`, `customer_type`, `recent_signal`, `icp_fit`, `icp_reason`, and a `facts` array (`id`, `fact`, `quote`).
4. Add a **Code** node (`Check Research Grounding`) to verify every extracted quote appears verbatim as a case-insensitive substring in the raw research context.

#### Step 5: Prospect Discovery & Email Waterfall
1. Add an **If** node (`Has Person`) checking if `first_name` and `last_name` are populated.
2. If false, add a **Hunter** node (`Find Decision Maker`) configured for Domain Search (`domain: "{{ $json.domain }}"`) using Hunter credentials.
3. Add a **Code** node (`Pick Decision Maker`) to score and select the best decision-maker contact based on title and department relevance.
4. Add an **If** node (`Person Found`) checking `person_found`.
5. Add a **Code** node (`Plan Waterfall`) to check website email candidates against naming patterns.
6. Build the email evaluation loop (`Website Email Found` -> **Hunter** `Verify Website Email` -> `Judge Website Email` -> `Settled After Website`).
7. If not settled and budget allows (`Hunter Budget Left`), execute **Hunter** `Email Finder` -> `Verify Hunter Email` -> `Judge Hunter Email` -> `Settled After Hunter`.
8. If still not settled and budget allows (`Dropcontact Budget Left`), execute **Dropcontact** `Enrich` -> **Hunter** `Verify Dropcontact Email` -> `Judge Dropcontact Email`.
9. Add a **NoOp** node (`Waterfall Stopped`) to collect early exits, and a **Code** node (`Assemble Email Result`) to compile final email waterfall metrics.

#### Step 6: Grounded Opener & Write-Back
1. Add an **If** node (`Worth An Opener`) validating that email status is verified/accept-all, research status is not failed, and `icp_fit` meets the minimum threshold.
2. Add a **LangChain** node (`Write Opener`) connected to an **OpenAI Chat Model** node (`Opener Model`, temperature `0.4`) and a **Structured Output Parser** (`Opener Schema`, defining `opener`, `fact_id`, `angle`).
3. Add a **Code** node (`Check Opener`) to audit length, banned fluff words, em dashes, number accuracy, and ensure no verbatim echoing of the company's offer.
4. Add a **Code** node (`Finalize Row`) to compile terminal status (`enriched`, `error`, `no_domain`, `no_person`, `no_email`, `skipped`) and aggregate error logs.
5. Add a **Google Sheets** node (`Write Row Back`) configured to update rows where matching column is `row_id`. Connect Google Sheets OAuth2 credentials.
6. Connect `Write Row Back` back to `Loop Rows` to process the next item.

#### Step 7: Run Reporting
1. Connect `Loop Rows` completion output to a **Code** node (`Summarize Run`) to calculate aggregate batch metrics, vendor spend in USD, and cost per verified email.
2. Add a **Code** node (`Format Run Report`) to generate the HTML email layout.
3. Add a **Gmail** node (`Email Run Report`) configured to send HTML content to `report_email`. Connect Gmail OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Clay Waterfall Enrichment Pipeline Overview | Outbound list enrichment pipeline that researches company websites, waterfalls across email providers, and writes personalized, grounded cold openers. |
| Setup and Configuration Guidelines | Ensure all 6 API credentials (Google Sheets, Firecrawl, OpenRouter, Hunter, Dropcontact, Gmail) are active and configured before running test batches. Set `max_credits_per_run` conservatively during initial testing. |
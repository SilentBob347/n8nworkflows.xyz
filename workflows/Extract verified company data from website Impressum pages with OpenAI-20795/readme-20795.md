Extract verified company data from website Impressum pages with OpenAI

https://n8nworkflows.xyz/workflows/extract-verified-company-data-from-website-impressum-pages-with-openai-20795


# Extract verified company data from website Impressum pages with OpenAI

### 1. Workflow Overview

This workflow is designed to automate the extraction and validation of official company information from a website's Impressum (legal notice) page using OpenAI. It is optimized for use cases such as CRM enrichment, lead qualification, and automated compliance checking. 

The execution logic is divided into three primary functional blocks:
- **1.1 Input Reception & URL Validation:** Receives incoming POST webhook requests, normalizes the domain, and blocks private or local IP addresses before checking input validity.
- **1.2 Impressum Discovery & Extraction:** Fetches the website homepage, determines the correct Impressum URL using multi-tier fallback strategies, downloads the target page, sanitizes the HTML into clean plain text, and submits the content to OpenAI (`gpt-4o-mini`) for structured JSON extraction.
- **1.3 Verification & Response:** Evaluates every extracted field against the raw page text to eliminate hallucinations, logs dropped fields, and returns a verified JSON payload via webhook response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & URL Validation
**Overview:** This block acts as the entry point, receiving HTTP POST requests containing target company parameters, ensuring proper URL formatting, and filtering out unsafe local or private network destinations.

**Nodes Involved:**
- `Receive Website URL`
- `Validate Input`
- `Input Valid?`
- `Respond with Input Error`

**Node Details:**
- **Receive Website URL**
  - *Type & Technical Role:* `n8n-nodes-base.webhook` (Trigger)
  - *Configuration:* Listens for incoming HTTP POST requests on path `impressum-enrichment` with response mode set to use the response node.
  - *Inputs / Outputs:* Inputs: None (Entrypoint). Outputs: `Validate Input`.
  - *Edge Cases:* Unauthenticated endpoints may receive unsolicited public requests (mitigate via header authentication).

- **Validate Input**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data transformation)
  - *Configuration:* Executes custom JavaScript to parse the request body, prepend `https://` if missing, extract the domain origin, and block private/local hosts (`localhost`, `127.*`, `10.*`, `192.168.*`, etc.).
  - *Key Expressions:* Reads `$input.first().json.body`. Returns an object containing `valid`, `website`, `origin`, `organization`, and `reference_id`.
  - *Inputs / Outputs:* Inputs: `Receive Website URL`. Outputs: `Input Valid?`.
  - *Edge Cases:* Malformed URLs return `{ valid: false, error: '...' }`.

- **Input Valid?**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Flow control)
  - *Configuration:* Evaluates whether `$json.valid` evaluates to `true`.
  - *Inputs / Outputs:* Inputs: `Validate Input`. Outputs: True branch goes to `Fetch Homepage`; False branch goes to `Respond with Input Error`.

- **Respond with Input Error**
  - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Response handler)
  - *Configuration:* Responds with an HTTP status code of `400` and a JSON body containing the validation error message.
  - *Inputs / Outputs:* Inputs: `Input Valid?` (False branch). Outputs: None (Terminal node).

---

#### 1.2 Impressum Discovery & Extraction
**Overview:** This block navigates the target website to locate the legal notice page, cleans the raw HTML content, and queries OpenAI to extract structured corporate data.

**Nodes Involved:**
- `Fetch Homepage`
- `Find Impressum URL`
- `Fetch Impressum Page`
- `Clean Page Text`
- `Page Has Content?`
- `Extract Company Data with AI`

**Node Details:**
- **Fetch Homepage**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Network request)
  - *Configuration:* Performs an HTTP GET request to the validated website URL with a custom `User-Agent`, a 10-second timeout, and a maximum of 3 redirects. Set to continue regular output on error.
  - *Key Expressions:* URL set to `={{ $json.website }}`.
  - *Inputs / Outputs:* Inputs: `Input Valid?` (True branch). Outputs: `Find Impressum URL`.
  - *Edge Cases:* Target site downtime or SSL handshake failures will return empty data payloads.

- **Find Impressum URL**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data transformation)
  - *Configuration:* Parses HTML via regex to find href links containing keywords (`impressum`, `imprint`, `legal-notice`, `rechtliches`), scans SPA JSON-embedded links near the word "Impressum", and falls back to `${origin}/impressum` if needed.
  - *Inputs / Outputs:* Inputs: `Fetch Homepage`. Outputs: `Fetch Impressum Page`.

- **Fetch Impressum Page**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (Network request)
  - *Configuration:* Downloads the HTML content of the located Impressum page using a 10-second timeout and up to 3 redirects. Set to continue regular output on error.
  - *Key Expressions:* URL set to `={{ $json.impressum_url }}`.
  - *Inputs / Outputs:* Inputs: `Find Impressum URL`. Outputs: `Clean Page Text`.

- **Clean Page Text**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data transformation)
  - *Configuration:* Removes script tags, style tags, and HTML tags, normalizes whitespace and HTML entities, and validates that the resulting text length is at least 50 characters (capping content at 8,000 characters).
  - *Inputs / Outputs:* Inputs: `Fetch Impressum Page`. Outputs: `Page Has Content?`.

- **Page Has Content?**
  - *Type & Technical Role:* `n8n-nodes-base.if` (Flow control)
  - *Configuration:* Evaluates whether `$json.has_content` is `true`.
  - *Inputs / Outputs:* Inputs: `Clean Page Text`. Outputs: True branch proceeds to `Extract Company Data with AI`; False branch bypasses AI and routes directly to `Verify Against Page Text`.

- **Extract Company Data with AI**
  - *Type & Technical Role:* `n8n-nodes-base.httpRequest` (AI integration via REST API)
  - *Configuration:* Sends a POST request to OpenAI (`https://api.openai.com/v1/chat/completions`) utilizing `gpt-4o-mini` with strict JSON mode enabled (`response_format: { type: 'json_object' }`).
  - *Credentials Required:* OpenAI API (`openAiApi`).
  - *Key Expressions:* Dynamically passes the organization hint, website, Impressum URL, and cleaned text to the system and user prompts.
  - *Inputs / Outputs:* Inputs: `Page Has Content?` (True branch). Outputs: `Verify Against Page Text`.
  - *Edge Cases:* API rate limits, authentication failures, or upstream token limits.

---

#### 1.3 Verification & Response
**Overview:** This block validates every extracted data point against the raw text of the Impressum page to eliminate AI hallucinations, drops unverified attributes, and returns the final structured JSON payload.

**Nodes Involved:**
- `Verify Against Page Text`
- `Respond with Company Data`

**Node Details:**
- **Verify Against Page Text**
  - *Type & Technical Role:* `n8n-nodes-base.code` (Data validation)
  - *Configuration:* Executes custom JavaScript comparing AI-extracted fields (company name, legal form, address components, emails, phone numbers, VAT IDs, register details, and representatives) against normalized versions of the raw Impressum text. Unverified values are filtered into a `dropped_fields` array.
  - *Inputs / Outputs:* Inputs: `Extract Company Data with AI` or `Page Has Content?` (False branch). Outputs: `Respond with Company Data`.

- **Respond with Company Data**
  - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Response handler)
  - *Configuration:* Returns a JSON response containing the status, reference ID, website, detected Impressum URL, verified `data` object, and `dropped_fields`.
  - *Key Expressions:* Response body set to `={{ JSON.stringify($json) }}`.
  - *Inputs / Outputs:* Inputs: `Verify Against Page Text`. Outputs: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive Website URL | n8n-nodes-base.webhook | Webhook trigger for incoming requests | None | Validate Input | ## Extract company data from a website's Impressum (legal notice)<br><br>**How it works**<br>- A webhook receives a `website` and optionally `organization` and `reference_id`... |
| Validate Input | n8n-nodes-base.code | Normalizes URLs and filters local addresses | Receive Website URL | Input Valid? | ## 1. Validate and find the Impressum<br>Check the URL, block local addresses, load the homepage and locate the Impressum link with three fallback strategies. |
| Input Valid? | n8n-nodes-base.if | Checks if input validation passed | Validate Input | Fetch Homepage, Respond with Input Error | ## 1. Validate and find the Impressum<br>Check the URL, block local addresses, load the homepage and locate the Impressum link with three fallback strategies. |
| Respond with Input Error | n8n-nodes-base.respondToWebhook | Returns 400 error response on invalid input | Input Valid? | None | ## 1. Validate and find the Impressum<br>Check the URL, block local addresses, load the homepage and locate the Impressum link with three fallback strategies. |
| Fetch Homepage | n8n-nodes-base.httpRequest | Fetches website homepage HTML | Input Valid? | Find Impressum URL | ## 1. Validate and find the Impressum<br>Check the URL, block local addresses, load the homepage and locate the Impressum link with three fallback strategies. |
| Find Impressum URL | n8n-nodes-base.code | Locates Impressum link via regex/fallback | Fetch Homepage | Fetch Impressum Page | ## 1. Validate and find the Impressum<br>Check the URL, block local addresses, load the homepage and locate the Impressum link with three fallback strategies. |
| Fetch Impressum Page | n8n-nodes-base.httpRequest | Downloads Impressum page HTML | Find Impressum URL | Clean Page Text | ## 2. Read and extract<br>Fetch the page, strip it to plain text and let the LLM return strict JSON. Empty or unreadable pages skip the AI call. |
| Clean Page Text | n8n-nodes-base.code | Converts HTML to sanitized plain text | Fetch Impressum Page | Page Has Content? | ## 2. Read and extract<br>Fetch the page, strip it to plain text and let the LLM return strict JSON. Empty or unreadable pages skip the AI call. |
| Page Has Content? | n8n-nodes-base.if | Checks if scraped text is sufficient | Clean Page Text | Extract Company Data with AI, Verify Against Page Text | ## 2. Read and extract<br>Fetch the page, strip it to plain text and let the LLM return strict JSON. Empty or unreadable pages skip the AI call. |
| Extract Company Data with AI | n8n-nodes-base.httpRequest | Queries OpenAI for structured company details | Page Has Content? | Verify Against Page Text | ## 2. Read and extract<br>Fetch the page, strip it to plain text and let the LLM return strict JSON. Empty or unreadable pages skip the AI call. |
| Verify Against Page Text | n8n-nodes-base.code | Validates extracted data against raw text | Extract Company Data with AI, Page Has Content? | Respond with Company Data | ## 3. Verify and respond<br>Every extracted value must appear in the page text, otherwise it is dropped and listed. Only verified data is returned. |
| Respond with Company Data | n8n-nodes-base.respondToWebhook | Returns verified company JSON data | Verify Against Page Text | None | ## 3. Verify and respond<br>Every extracted value must appear in the page text, otherwise it is dropped and listed. Only verified data is returned. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entrypoint:**
   - Add a **Webhook** node named `Receive Website URL`. Set HTTP Method to `POST`, path to `impressum-enrichment`, and Response Mode to `Using 'Respond to Webhook' Node`.

2. **Add Input Validation:**
   - Create a **Code** node named `Validate Input` connected to `Receive Website URL`. Paste the input parsing and private IP blocking JavaScript snippet.
   - Add an **If** node named `Input Valid?` connected to `Validate Input`. Set the condition to check if `{{ $json.valid }}` is true.

3. **Handle Validation Errors:**
   - Add a **Respond to Webhook** node named `Respond with Input Error` connected to the `false` output of `Input Valid?`. Set Response Code to `400` and body expression to `={{ JSON.stringify({ status: 'error', error: $json.error }) }}`.

4. **Fetch Homepage & Discover Impressum:**
   - Add an **HTTP Request** node named `Fetch Homepage` connected to the `true` output of `Input Valid?`. Set Method to `GET`, URL to `={{ $json.website }}`, timeout to `10000`, max redirects to `3`, and add a header `User-Agent` with value `Mozilla/5.0 (compatible; ImpressumLookup/1.0)`. Enable `On Error -> Continue Regular Output`.
   - Add a **Code** node named `Find Impressum URL` connected to `Fetch Homepage` to execute the regex matching and `/impressum` fallback logic.
   - Add an **HTTP Request** node named `Fetch Impressum Page` connected to `Find Impressum URL`. Set Method to `GET`, URL to `={{ $json.impressum_url }}`, timeout to `10000`, max redirects to `3`, and add the same `User-Agent` header. Enable `On Error -> Continue Regular Output`.

5. **Clean Text & Evaluate Content:**
   - Add a **Code** node named `Clean Page Text` connected to `Fetch Impressum Page` to strip HTML tags and check if text length is `>= 50`.
   - Add an **If** node named `Page Has Content?` connected to `Clean Page Text` with condition `{{ $json.has_content }}` equals true.

6. **Configure AI Extraction:**
   - Add an **HTTP Request** node named `Extract Company Data with AI` connected to the `true` output of `Page Has Content?`.
   - Configure authentication using a predefined **OpenAI API** credential.
   - Set Method to `POST`, URL to `https://api.openai.com/v1/chat/completions`, Content Type to `Raw/Custom (application/json)`, and use the JSON body containing the `gpt-4o-mini` model, `temperature: 0`, `response_format: { type: 'json_object' }`, and the system/user prompts provided in the definition.

7. **Verify Data & Respond:**
   - Add a **Code** node named `Verify Against Page Text` connected to both `Extract Company Data with AI` and the `false` output of `Page Has Content?`. Paste the cross-validation logic.
   - Add a **Respond to Webhook** node named `Respond with Company Data` connected to `Verify Against Page Text`. Set response body to `={{ JSON.stringify($json) }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Business websites in Germany, Austria, and Switzerland must legally publish an Impressum, making it a highly reliable structured data source. | Regional compliance context |
| Websites that render their Impressum exclusively via client-side JavaScript will return `no_content` because raw HTML scraping does not execute scripts. | Technical limitation / Edge case |
| Private and local IP addresses are automatically blocked during input validation to prevent Server-Side Request Forgery (SSRF) vulnerabilities. | Security constraint |
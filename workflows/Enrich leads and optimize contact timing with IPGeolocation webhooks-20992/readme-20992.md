Enrich leads and optimize contact timing with IPGeolocation webhooks

https://n8nworkflows.xyz/workflows/enrich-leads-and-optimize-contact-timing-with-ipgeolocation-webhooks-20992


# Enrich leads and optimize contact timing with IPGeolocation webhooks

### 1. Workflow Overview

The **Lead enrichment and send-time optimization with IPGeolocation** workflow automates the ingestion, validation, geographic enrichment, and send-time optimization of inbound leads. By accepting an HTTP POST request containing lead details and an IP address, the system evaluates the validity of the IP syntax, fetches geolocation and timezone details from IPGeolocation.io, calculates local business hours, and determines the optimal time to contact the lead. 

The workflow logic is categorized into three functional blocks:
- **1.1 Input Reception & Validation:** Ingests the raw webhook payload, extracts lead fields, performs regex validation for IPv4 and IPv6 formats, and routes execution based on IP syntax accuracy.
- **1.2 Enrichment & Profile Assembly:** Queries external geolocation and timezone APIs for valid IP addresses, processes local time attributes via custom JavaScript, evaluates business hour compliance (09:00–18:00 local time), and builds a structured lead profile.
- **1.3 Response Routing:** Returns either the fully enriched lead profile with send-time recommendations via a successful JSON response or rejects invalid IP submissions early with a 400 Bad Request error payload.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
- **Overview:** This block acts as the entry point, receiving inbound lead payloads via webhook, sanitizing fields, checking the structural integrity of the submitted IP address, and executing conditional routing.
- **Nodes Involved:** 
  - `Receive Lead via Webhook`
  - `Check IP Address Validity`
  - `Validate IP Syntax`

- **Node Details:**
  - **Receive Lead via Webhook**
    - *Type & Technical Role:* `n8n-nodes-base.webhook` (Trigger)
    - *Configuration:* Listens for HTTP `POST` requests on path `lead-geo-router` using response mode `responseNode`.
    - *Key Expressions:* None (triggers on raw payload reception).
    - *Input/Output:* Output connects to `Check IP Address Validity`.
    - *Edge Cases / Failures:* Timeouts or unhandled payload structures if callers transmit malformed JSON headers.

  - **Check IP Address Validity**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Processing)
    - *Configuration:* Runs once per item in `runOnceForEachItem` mode using JavaScript. Extracts lead fields (`ip`, `name`, `email`, `company`) from `$json.body` and applies regular expressions to evaluate IPv4 and IPv6 validity.
    - *Key Expressions:* `$json.body.ip`, `$json.body.name`, `$json.body.email`, `$json.body.company`.
    - *Input/Output:* Input connects from `Receive Lead via Webhook`; output connects to `Validate IP Syntax`.
    - *Edge Cases / Failures:* Missing body keys default to empty strings to prevent downstream reference errors.

  - **Validate IP Syntax**
    - *Type & Technical Role:* `n8n-nodes-base.if` (Flow Control)
    - *Configuration:* Evaluates whether `{{ $json.ipIsValid }}` strictly equals `true`.
    - *Key Expressions:* `={{ $json.ipIsValid }}`.
    - *Input/Output:* Input connects from `Check IP Address Validity`. True branch connects to `Fetch IP Geolocation Data`; false branch connects to `Error Response for Invalid IP`.
    - *Edge Cases / Failures:* Strict type validation mismatch if boolean evaluation fails.

---

#### 2.2 Enrichment & Profile Assembly
- **Overview:** Queries the IPGeolocation service for geographic organization details and timezone data, parses string timestamps safely, computes local business hour statuses, and structures the consolidated lead record.
- **Nodes Involved:**
  - `Fetch IP Geolocation Data`
  - `Fetch Timezone Details`
  - `Assemble Lead Profile`

- **Node Details:**
  - **Fetch IP Geolocation Data**
    - *Type & Technical Role:* `n8n-nodes-ipgeolocation.ipgeolocation` (External API Integration)
    - *Configuration:* Default geolocation operation utilizing expressions for target IP evaluation.
    - *Key Expressions:* `={{ $json.ip }}`.
    - *Credentials:* `ipgeolocationApi` (API Key).
    - *Input/Output:* Input connects from `Validate IP Syntax` (True branch); output connects to `Fetch Timezone Details`.
    - *Edge Cases / Failures:* API rate limiting, network timeouts, or invalid API keys.

  - **Fetch Timezone Details**
    - *Type & Technical Role:* `n8n-nodes-ipgeolocation.ipgeolocation` (External API Integration)
    - *Configuration:* Executes the `getTimezone` operation queried by IP timezone.
    - *Key Expressions:* `={{ $('Validate IP Syntax').item.json.ip }}`.
    - *Credentials:* `ipgeolocationApi` (ID: `6EcexKRgFyKdpkt7`, Name: `IPGeolocation API Key`).
    - *Input/Output:* Input connects from `Fetch IP Geolocation Data`; output connects to `Assemble Lead Profile`.
    - *Edge Cases / Failures:* API authentication failure or lack of timezone dataset entitlements for specific IPs.

  - **Assemble Lead Profile**
    - *Type & Technical Role:* `n8n-nodes-base.code` (Data Processing)
    - *Configuration:* Runs once per item (`runOnceForEachItem`) combining data from `Fetch IP Geolocation Data`, `Check IP Address Validity`, and current node output (`time_zone`). Extracts local hours via regex matching to prevent timezone parsing errors, evaluates business hours (09:00–18:00), and calculates next contact windows.
    - *Key Expressions:* Uses node linkage references such as `$('Fetch IP Geolocation Data').item.json` and `$('Check IP Address Validity').item.json`.
    - *Input/Output:* Input connects from `Fetch Timezone Details`; output connects to `Reply with Lead Details`.
    - *Edge Cases / Failures:* Missing time string properties defaulting to empty values or null local hour metrics.

---

#### 2.3 Response Routing
- **Overview:** Formulates and transmits the final HTTP response back to the webhook caller, returning either a successfully enriched lead profile or a 400 Bad Request error payload for invalid IP entries.
- **Nodes Involved:**
  - `Reply with Lead Details`
  - `Error Response for Invalid IP`

- **Node Details:**
  - **Reply with Lead Details**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Webhook Response)
    - *Configuration:* Responds with JSON data utilizing the incoming item payload.
    - *Key Expressions:* `={{ $json }}`.
    - *Input/Output:* Input connects from `Assemble Lead Profile`; terminal node (no output).
    - *Edge Cases / Failures:* Connection drops if the webhook caller closes the HTTP connection prematurely.

  - **Error Response for Invalid IP**
    - *Type & Technical Role:* `n8n-nodes-base.respondToWebhook` (Webhook Response)
    - *Configuration:* Responds with JSON data returning HTTP status code `400` and an explicit error message structure.
    - *Key Expressions:* `={{ { "error": "invalid_ip", "message": "The 'ip' field must be a valid IPv4 or IPv6 address.", "ip": $json.ip } }}`.
    - *Input/Output:* Input connects from `Validate IP Syntax` (False branch); terminal node (no output).
    - *Edge Cases / Failures:* Output payload generation failure if `$json.ip` is undefined.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Sticky Note** | `n8n-nodes-base.stickyNote` | Workflow documentation and configuration overview. | None | None | ## Lead enrichment and send-time optimization with IPGeolocation<br><br>### How it works<br><br>This workflow receives a lead via webhook, normalizes and validates the submitted IP address, then branches based on whether the IP looks valid. Valid leads are enriched with IPGeolocation location and timezone data, combined into a clean lead profile that can include send-time optimization details, and returned to the caller. Invalid IP submissions are stopped early with a 400-style webhook response.<br><br>### Setup steps<br><br>- Configure the Webhook trigger URL and ensure the caller sends the expected lead fields, especially the IP address field used by the validation code.<br>- Add valid IPGeolocation credentials or API key configuration to the IPGeolocation nodes.<br>- Review the Validate IP Address code node so its payload field names match the incoming webhook body.<br>- Confirm the Respond to Webhook nodes use the intended status codes and response body format for your API clients.<br><br>### Customization<br><br>Adjust the Build Lead Profile code to change which geolocation fields are returned, how timezone data is converted into recommended send times, or what fallback behavior should apply when location data is incomplete. |
| **Sticky Note1** | `n8n-nodes-base.stickyNote` | Documentation block for ingestion and validation. | None | None | ## Receive and validate lead<br><br>Accepts the incoming lead webhook, normalizes the payload, checks IPv4/IPv6 syntax, and routes the workflow based on whether the IP address appears valid. |
| **Sticky Note2** | `n8n-nodes-base.stickyNote` | Documentation block for valid lead enrichment. | None | None | ## Enrich valid lead<br><br>Uses IPGeolocation to retrieve location and timezone details for valid IPs, then combines those results into a structured enriched lead profile before returning it to the webhook caller. |
| **Sticky Note3** | `n8n-nodes-base.stickyNote` | Documentation block for invalid IP handling. | None | None | ## Reject invalid IP<br><br>Handles the invalid-IP branch with a dedicated webhook response, returning an error without attempting enrichment. |
| **Receive Lead via Webhook** | `n8n-nodes-base.webhook` | Ingests incoming lead HTTP POST payloads. | None | Check IP Address Validity | |
| **Check IP Address Validity** | `n8n-nodes-base.code` | Normalizes incoming payload and performs IPv4/IPv6 regex validation. | Receive Lead via Webhook | Validate IP Syntax | |
| **Validate IP Syntax** | `n8n-nodes-base.if` | Routes workflow execution based on IP validity boolean status. | Check IP Address Validity | Fetch IP Geolocation Data, Error Response for Invalid IP | |
| **Fetch IP Geolocation Data** | `n8n-nodes-ipgeolocation.ipgeolocation` | Retrieves location and ASN organization data via IPGeolocation. | Validate IP Syntax | Fetch Timezone Details | |
| **Fetch Timezone Details** | `n8n-nodes-ipgeolocation.ipgeolocation` | Retrieves local timezone and current time details for the IP address. | Fetch IP Geolocation Data | Assemble Lead Profile | |
| **Assemble Lead Profile** | `n8n-nodes-base.code` | Combines lookup results, evaluates local business hours, and determines contact windows. | Fetch Timezone Details | Reply with Lead Details | |
| **Reply with Lead Details** | `n8n-nodes-base.respondToWebhook` | Returns the fully structured and enriched lead profile in JSON format. | Assemble Lead Profile | None | |
| **Error Response for Invalid IP** | `n8n-nodes-base.respondToWebhook` | Returns a 400 Bad Request JSON response for invalid IP addresses. | Validate IP Syntax | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node named `Receive Lead via Webhook`.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `lead-geo-router`.
   - Set **Response Mode** to `Response Node`.

2. **Add IP Normalization & Syntax Check:**
   - Add a **Code** node named `Check IP Address Validity` connected after `Receive Lead via Webhook`.
   - Set mode to **Run Once for Each Item**.
   - Paste JavaScript to extract `$json.body.ip`, `$json.body.name`, `$json.body.email`, and `$json.body.company`, evaluating IPv4 and IPv6 regex patterns to output an `ipIsValid` boolean.

3. **Configure Conditional Routing:**
   - Add an **If** node named `Validate IP Syntax` connected after `Check IP Address Validity`.
   - Configure a condition where Left Value equals `={{ $json.ipIsValid }}` with operator **Boolean** equal to `true`.

4. **Implement Valid IP Enrichment Pipeline:**
   - **Step A (Geolocation):** Add an **IPGeolocation** node named `Fetch IP Geolocation Data` connected to the `true` output of `Validate IP Syntax`. Set parameter `IP Geolocation` to `={{ $json.ip }}`. Configure credentials for `ipgeolocationApi`.
   - **Step B (Timezone):** Add a second **IPGeolocation** node named `Fetch Timezone Details` connected after `Fetch IP Geolocation Data`. Set operation to `Get Timezone`, query by `ipTimezone`, and pass expression `={{ $('Validate IP Syntax').item.json.ip }}`. Attach the `IPGeolocation API Key` credential.
   - **Step C (Profile Assembly):** Add a **Code** node named `Assemble Lead Profile` connected after `Fetch Timezone Details`. Use **Run Once for Each Item** mode to combine geolocation, lead metadata, and timezone records, parse the local hour via regular expressions, calculate business hour compliance (09:00–18:00), and structure the final profile object.
   - **Step D (Webhook Response):** Add a **Respond to Webhook** node named `Reply with Lead Details` connected after `Assemble Lead Profile`. Set **Respond With** to `JSON` and response body to `={{ $json }}`.

5. **Implement Invalid IP Error Handling:**
   - Add a **Respond to Webhook** node named `Error Response for Invalid IP` connected to the `false` (bottom) output of `Validate IP Syntax`.
   - Set **Response Code** to `400`.
   - Set **Respond With** to `JSON`.
   - Set **Response Body** expression to `={{ { "error": "invalid_ip", "message": "The 'ip' field must be a valid IPv4 or IPv6 address.", "ip": $json.ip } }}`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| IPGeolocation API Documentation | [https://ipgeolocation.io/documentation.html](https://ipgeolocation.io/documentation.html) |
| n8n Webhook Node Guide | [https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.webhook/) |
Verify phone numbers via SMS Vert Pro OTP and log verified numbers to Google Sheets

https://n8nworkflows.xyz/workflows/verify-phone-numbers-via-sms-vert-pro-otp-and-log-verified-numbers-to-google-sheets-20862


# Verify phone numbers via SMS Vert Pro OTP and log verified numbers to Google Sheets

### 1. Workflow Overview

This workflow provides a complete backend service for phone number verification via SMS One-Time Passwords (OTP). It exposes two independent webhook endpoints to handle code generation and code verification without requiring external application infrastructure. 

The logic is grouped into the following functional blocks:
- **1.1 Input Reception & Normalization:** Captures incoming HTTP POST requests from either endpoint, loads configuration variables, and normalizes and validates phone numbers and OTP codes against security criteria.
- **1.2 Request Routing & Dispatch:** Evaluates validation results and routes valid requests to the SMS Vert Pro Model Context Protocol (MCP) server for SMS generation or code verification.
- **1.3 Storage & Response Handling:** Handles API responses, logs successfully verified numbers to Google Sheets, and returns standardized JSON responses with appropriate HTTP status codes (200, 400, 401, 429, or 502) to the calling client.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Normalization
This block listens for incoming requests, initializes configuration settings, parses the raw phone number payload into a standardized international format, and filters out unauthorized country codes.

- **Request a code**
  - **Type & Technical Role:** `n8n-nodes-base.webhook` (Trigger). Exposes a POST endpoint at `/sms-otp/send` to initiate OTP generation.
  - **Configuration Choices:** Responds via the response node; expects JSON payloads containing a phone number.
  - **Key Expressions / Variables:** Webhook path configured as `sms-otp/send`.
  - **Input / Output Connections:** Input: None (Trigger). Output: Connected to *OTP settings*.
  - **Edge Cases / Failure Types:** Invalid HTTP method, network timeout, or unreachable endpoint URL.

- **Check a code**
  - **Type & Technical Role:** `n8n-nodes-base.webhook` (Trigger). Exposes a POST endpoint at `/sms-otp/verify` to validate submitted OTP codes.
  - **Configuration Choices:** Responds via the response node; expects JSON payloads containing a phone number and a 6-digit code.
  - **Key Expressions / Variables:** Webhook path configured as `sms-otp/verify`.
  - **Input / Output Connections:** Input: None (Trigger). Output: Connected to *OTP settings*.
  - **Edge Cases / Failure Types:** Invalid HTTP method or malformed payload.

- **OTP settings**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Data transformation / Configuration). Defines global settings used downstream in the workflow.
  - **Configuration Choices:** Assigns string values for sender identity, spreadsheet URL, default country code, and allowed country codes.
  - **Key Expressions / Variables:** 
    - `sender`: `"MyApp"`
    - `sheet_url`: `"https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID/edit#gid=0"`
    - `default_country_code`: `"33"`
    - `allowed_country_codes`: `"33,32,41"`
  - **Input / Output Connections:** Input: *Request a code* or *Check a code*. Output: Connected to *Read request*.
  - **Edge Cases / Failure Types:** Missing configuration variables or misformatted sheet URLs.

- **Read request**
  - **Type & Technical Role:** `n8n-nodes-base.code` (JavaScript execution). Normalizes phone numbers, handles regional prefix conversions, and validates numbers and codes against security rules.
  - **Configuration Choices:** Executes custom JavaScript to strip non-numeric characters, prepend default country codes where applicable, and check against whitelist arrays.
  - **Key Expressions / Variables:** 
    - Determines action via `$('Check a code').isExecuted ? 'verify' : 'send'`
    - Uses regex `/[^0-9+]/g` for phone sanitization and `/^[0-9]{6}$/` for code validation.
  - **Input / Output Connections:** Input: *OTP settings*. Output: Connected to *Valid request?*.
  - **Edge Cases / Failure Types:** Unhandled payload structures resulting in empty strings or regex matching errors.

- **Valid request?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow control). Evaluates whether the normalized request meets validation standards.
  - **Configuration Choices:** Evaluates boolean condition.
  - **Key Expressions / Variables:** `={{ $json.valid }}`
  - **Input / Output Connections:** Input: *Read request*. Outputs: True branch connects to *Send or check?*; False branch connects to *Respond: invalid request*.
  - **Edge Cases / Failure Types:** Expression evaluation failure if data types do not resolve cleanly to booleans.

- **Respond: invalid request**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Response node). Returns a 400 Bad Request HTTP response when validation fails.
  - **Configuration Choices:** Responds with JSON containing error metadata.
  - **Key Expressions / Variables:** `={{ JSON.stringify({ ok: false, error: $json.error }) }}`
  - **Input / Output Connections:** Input: *Valid request?* (False branch). Output: None (Terminal node).
  - **Edge Cases / Failure Types:** Connection drop before response transmission.

---

#### 2.2 Request Routing & Dispatch
This block routes valid requests to the correct SMS Vert Pro MCP tool based on whether the action is set to send or verify.

- **Send or check?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow control). Splits execution based on whether the request action is to send an OTP or verify it.
  - **Configuration Choices:** String comparison condition.
  - **Key Expressions / Variables:** `={{ $json.action }}` equals `"send"`
  - **Input / Output Connections:** Input: *Valid request?* (True branch). Outputs: True branch connects to *Send code by SMS*; False branch connects to *Check code*.
  - **Edge Cases / Failure Types:** Unexpected action string value causing misrouting.

- **Send code by SMS**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.mcpClient` (Integration / MCP tool execution). Calls the SMS Vert Pro MCP server to generate and send an SMS OTP.
  - **Configuration Choices:** Configured to use HTTP streamable transport, Bearer Auth, and the `generate_otp` tool ID. Error handling set to continue regular output.
  - **Key Expressions / Variables:** `={{ JSON.stringify({ to: $json.phone, sender: $('OTP settings').first().json.sender }) }}`
  - **Input / Output Connections:** Input: *Send or check?* (True branch). Output: Connected to *Code sent?*.
  - **Edge Cases / Failure Types:** Authentication failures, rate limits, network timeouts, or SMS provider downtime.

- **Check code**
  - **Type & Technical Role:** `@n8n/n8n-nodes-langchain.mcpClient` (Integration / MCP tool execution). Calls the SMS Vert Pro MCP server to validate a submitted OTP code.
  - **Configuration Choices:** Configured to use HTTP streamable transport, Bearer Auth, and the `verify_otp` tool ID. Error handling set to continue regular output.
  - **Key Expressions / Variables:** `={{ JSON.stringify({ to: $json.phone, code: $json.code }) }}`
  - **Input / Output Connections:** Input: *Send or check?* (False branch). Output: Connected to *Code valid?*.
  - **Edge Cases / Failure Types:** Invalid credentials, malformed JSON inputs, or MCP server unavailability.

---

#### 2.3 Storage & Response Handling
This block inspects MCP server execution results, logs successful verifications to Google Sheets, and returns appropriate success or error responses to the caller.

- **Code sent?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow control). Evaluates if the SMS Vert Pro `generate_otp` operation succeeded.
  - **Configuration Choices:** Boolean evaluation of structured content.
  - **Key Expressions / Variables:** `={{ $json.structuredContent?.ok }}`
  - **Input / Output Connections:** Input: *Send code by SMS*. Outputs: True branch connects to *Respond: code sent*; False branch connects to *Respond: code not sent*.
  - **Edge Cases / Failure Types:** Missing structured content properties due to upstream API errors.

- **Respond: code sent**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Response node). Returns a 200 OK HTTP response confirming that the SMS OTP was successfully dispatched.
  - **Configuration Choices:** Responds with JSON payload and expiration timer data.
  - **Key Expressions / Variables:** `={{ JSON.stringify({ ok: true, phone: $('Read request').item.json.phone, expires_in_seconds: 300 }) }}`
  - **Input / Output Connections:** Input: *Code sent?* (True branch). Output: None (Terminal node).
  - **Edge Cases / Failure Types:** Client disconnects before receiving the HTTP response.

- **Respond: code not sent**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Response node). Returns an error HTTP response (429 Too Many Requests if an OTP already exists, or 502 Bad Gateway otherwise).
  - **Configuration Choices:** Dynamic HTTP status code evaluation and JSON error body formatting.
  - **Key Expressions / Variables:** 
    - Response code: `={{ $json.structuredContent?.status === 'OTP_ALREADY_EXIST' ? 429 : 502 }}`
    - Response body: `={{ JSON.stringify({ ok: false, error: $json.structuredContent?.status || 'SEND_ERROR', message: ... }) }}`
  - **Input / Output Connections:** Input: *Code sent?* (False branch). Output: None (Terminal node).
  - **Edge Cases / Failure Types:** Unrecognized error status codes defaulting to generic error messages.

- **Code valid?**
  - **Type & Technical Role:** `n8n-nodes-base.if` (Flow control). Evaluates whether the submitted OTP code was verified successfully by the MCP server.
  - **Configuration Choices:** Boolean check on structured content validity.
  - **Key Expressions / Variables:** `={{ $json.structuredContent?.valid }}`
  - **Input / Output Connections:** Input: *Check code*. Outputs: True branch connects to *Log verified number*; False branch connects to *Respond: wrong or expired code*.
  - **Edge Cases / Failure Types:** Null pointer exceptions if the MCP response structure changes unexpectedly.

- **Log verified number**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Data storage). Appends the verified phone number and current timestamp as a new row in Google Sheets.
  - **Configuration Choices:** Append operation using URL mapping mode for document and sheet specification. Types conversion disabled.
  - **Key Expressions / Variables:** 
    - Spreadsheet URL: `={{ $('OTP settings').first().json.sheet_url }}`
    - Phone: `={{ $('Read request').item.json.phone }}`
    - Verified timestamp: `={{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}`
  - **Input / Output Connections:** Input: *Code valid?* (True branch). Output: Connected to *Respond: number verified*.
  - **Edge Cases / Failure Types:** Google Sheets API rate limits, permission errors, or missing sheet columns (`phone`, `verified_at`).

- **Respond: number verified**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Response node). Returns a 200 OK HTTP response confirming successful phone number verification.
  - **Configuration Choices:** Responds with a success JSON payload.
  - **Key Expressions / Variables:** `={{ JSON.stringify({ ok: true, verified: true, phone: $('Read request').item.json.phone }) }}`
  - **Input / Output Connections:** Input: *Log verified number*. Output: None (Terminal node).
  - **Edge Cases / Failure Types:** Intermittent webhook delivery failures.

- **Respond: wrong or expired code**
  - **Type & Technical Role:** `n8n-nodes-base.respondToWebhook` (Response node). Returns a 401 Unauthorized HTTP response when the submitted code is incorrect or has expired.
  - **Configuration Choices:** Responds with 401 status code and failure JSON payload.
  - **Key Expressions / Variables:** `={{ JSON.stringify({ ok: false, verified: false, error: $json.structuredContent?.status || 'VERIFY_ERROR' }) }}`
  - **Input / Output Connections:** Input: *Code valid?* (False branch). Output: None (Terminal node).
  - **Edge Cases / Failure Types:** Client parsing errors if unexpected error objects are returned.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Request a code | n8n-nodes-base.webhook | Exposes POST endpoint to request an SMS OTP. | None (Trigger) | OTP settings | ## Verify phone numbers with a one-time SMS code (OTP)<br><br>### How it works<br>Two webhooks give your sign-up or checkout form a phone verification step, with no code to host:<br>1. **Request a code** (`POST /sms-otp/send` with `{ \"phone\": \"...\" }`) checks the number and sends a 6-digit code by SMS with the `generate_otp` tool of the SMS Vert Pro MCP server. The code is valid for 5 minutes.<br>2. **Check a code** (`POST /sms-otp/verify` with `{ \"phone\": \"...\", \"code\": \"123456\" }`) checks it with `verify_otp`. A code works only once.<br>3. Each verified number is added to a Google Sheet with the date, and the form gets a clear JSON answer: 200, 400 (invalid number), 401 (wrong or expired code), 429 (a code was already sent a moment ago).<br><br>### Setup steps<br>1. Create a Google Sheet with the columns `phone` and `verified_at`, and connect your Google account in **Log verified number**.<br>2. Create a **Bearer Auth** credential with your SMS Vert Pro API V2 token (log in at smsvertpro.com, then open https://www.smsvertpro.com/api/doc-v2.php) and select it in the two MCP Client nodes.<br>3. In **OTP settings**, set your sender name (11 characters max), your sheet URL, your default country code (numbers starting with 0 are converted) and the country codes you accept.<br>4. Activate the workflow and call the two production webhook URLs from your form.<br><br>### Customization<br>Replace the Google Sheet with your CRM or database, or add a step that unlocks the user account once the number is verified. |
| Check a code | n8n-nodes-base.webhook | Exposes POST endpoint to verify an SMS OTP. | None (Trigger) | OTP settings | ## Verify phone numbers with a one-time SMS code (OTP)<br><br>### How it works<br>Two webhooks give your sign-up or checkout form a phone verification step, with no code to host:<br>1. **Request a code** (`POST /sms-otp/send` with `{ \"phone\": \"...\" }`) checks the number and sends a 6-digit code by SMS with the `generate_otp` tool of the SMS Vert Pro MCP server. The code is valid for 5 minutes.<br>2. **Check a code** (`POST /sms-otp/verify` with `{ \"phone\": \"...\", \"code\": \"123456\" }`) checks it with `verify_otp`. A code works only once.<br>3. Each verified number is added to a Google Sheet with the date, and the form gets a clear JSON answer: 200, 400 (invalid number), 401 (wrong or expired code), 429 (a code was already sent a moment ago).<br><br>### Setup steps<br>1. Create a Google Sheet with the columns `phone` and `verified_at`, and connect your Google account in **Log verified number**.<br>2. Create a **Bearer Auth** credential with your SMS Vert Pro API V2 token (log in at smsvertpro.com, then open https://www.smsvertpro.com/api/doc-v2.php) and select it in the two MCP Client nodes.<br>3. In **OTP settings**, set your sender name (11 characters max), your sheet URL, your default country code (numbers starting with 0 are converted) and the country codes you accept.<br>4. Activate the workflow and call the two production webhook URLs from your form.<br><br>### Customization<br>Replace the Google Sheet with your CRM or database, or add a step that unlocks the user account once the number is verified. |
| OTP settings | n8n-nodes-base.set | Assigns configuration parameters for sender, sheet URL, and country codes. | Request a code, Check a code | Read request | ## Verify phone numbers with a one-time SMS code (OTP)<br><br>### How it works<br>Two webhooks give your sign-up or checkout form a phone verification step, with no code to host:<br>1. **Request a code** (`POST /sms-otp/send` with `{ \"phone\": \"...\" }`) checks the number and sends a 6-digit code by SMS with the `generate_otp` tool of the SMS Vert Pro MCP server. The code is valid for 5 minutes.<br>2. **Check a code** (`POST /sms-otp/verify` with `{ \"phone\": \"...\", \"code\": \"123456\" }`) checks it with `verify_otp`. A code works only once.<br>3. Each verified number is added to a Google Sheet with the date, and the form gets a clear JSON answer: 200, 400 (invalid number), 401 (wrong or expired code), 429 (a code was already sent a moment ago).<br><br>### Setup steps<br>1. Create a Google Sheet with the columns `phone` and `verified_at`, and connect your Google account in **Log verified number**.<br>2. Create a **Bearer Auth** credential with your SMS Vert Pro API V2 token (log in at smsvertpro.com, then open https://www.smsvertpro.com/api/doc-v2.php) and select it in the two MCP Client nodes.<br>3. In **OTP settings**, set your sender name (11 characters max), your sheet URL, your default country code (numbers starting with 0 are converted) and the country codes you accept.<br>4. Activate the workflow and call the two production webhook URLs from your form.<br><br>### Customization<br>Replace the Google Sheet with your CRM or database, or add a step that unlocks the user account once the number is verified. |
| Read request | n8n-nodes-base.code | Normalizes phone numbers, handles country code rules, and validates payloads. | OTP settings | Valid request? | ## 1. Read the request<br>Both webhooks share the same settings. The phone number is normalized and only the country codes you accept are allowed, which protects your credits against SMS pumping. |
| Valid request? | n8n-nodes-base.if | Checks if the parsed phone number and code are valid. | Read request | Send or check?, Respond: invalid request | |
| Respond: invalid request | n8n-nodes-base.respondToWebhook | Returns a 400 HTTP response when validation fails. | Valid request? | None | |
| Send or check? | n8n-nodes-base.if | Routes execution flow to either SMS sending or code verification. | Valid request? | Send code by SMS, Check code | |
| Send code by SMS | @n8n/n8n-nodes-langchain.mcpClient | Calls SMS Vert Pro MCP `generate_otp` tool to dispatch an SMS. | Send or check? | Code sent? | ## 2. Send the code<br>`generate_otp` sends the SMS. If a code was already sent to this number a moment ago, the form gets a 429 answer instead of a second SMS. |
| Code sent? | n8n-nodes-base.if | Verifies if the SMS dispatch operation succeeded. | Send code by SMS | Respond: code sent, Respond: code not sent | ## 2. Send the code<br>`generate_otp` sends the SMS. If a code was already sent to this number a moment ago, the form gets a 429 answer instead of a second SMS. |
| Respond: code sent | n8n-nodes-base.respondToWebhook | Returns a 200 HTTP response confirming SMS code dispatch. | Code sent? | None | |
| Respond: code not sent | n8n-nodes-base.respondToWebhook | Returns a 429 or 502 HTTP error response if SMS sending failed. | Code sent? | None | |
| Check code | @n8n/n8n-nodes-langchain.mcpClient | Calls SMS Vert Pro MCP `verify_otp` tool to validate the code. | Send or check? | Code valid? | ## 3. Check the code and log the number<br>`verify_otp` checks the code; verified numbers are added to the Google Sheet. |
| Code valid? | n8n-nodes-base.if | Evaluates whether the verification code was valid. | Check code | Log verified number, Respond: wrong or expired code | ## 3. Check the code and log the number<br>`verify_otp` checks the code; verified numbers are added to the Google Sheet. |
| Log verified number | n8n-nodes-base.googleSheets | Appends verified phone number and timestamp to Google Sheets. | Code valid? | Respond: number verified | ## 3. Check the code and log the number<br>`verify_otp` checks the code; verified numbers are added to the Google Sheet. |
| Respond: number verified | n8n-nodes-base.respondToWebhook | Returns a 200 HTTP success response after successful verification. | Log verified number | None | |
| Respond: wrong or expired code | n8n-nodes-base.respondToWebhook | Returns a 401 HTTP error response for incorrect or expired codes. | Code valid? | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Triggers:**
   - Add a **Webhook** node named `Request a code`. Set HTTP Method to `POST`, Path to `sms-otp/send`, and Response Mode to `Response Node`.
   - Add a second **Webhook** node named `Check a code`. Set HTTP Method to `POST`, Path to `sms-otp/verify`, and Response Mode to `Response Node`.

2. **Configure Global Settings:**
   - Add a **Set** node named `OTP settings`. Connect both webhook triggers to this node.
   - Configure assignments with string values:
     - `sender`: `"MyApp"`
     - `sheet_url`: `"https://docs.google.com/spreadsheets/d/YOUR_SHEET_ID/edit#gid=0"`
     - `default_country_code`: `"33"`
     - `allowed_country_codes`: `"33,32,41"`
     - Ensure "Include Other Fields" is enabled.

3. **Add Request Reader and Normalizer:**
   - Add a **Code** node named `Read request`. Connect *OTP settings* to it.
   - Insert the JavaScript code snippet to sanitize phone numbers, apply default country prefixes, filter allowed country codes, and validate 6-digit OTP codes.

4. **Validate Request Payload:**
   - Add an **If** node named `Valid request?`. Connect *Read request* to it.
   - Set condition: Left Value `={{ $json.valid }}` equals `true`.
   - Connect the False output to a **Respond to Webhook** node named `Respond: invalid request` configured with Response Code `400` and response body `={{ JSON.stringify({ ok: false, error: $json.error }) }}`.

5. **Route Requests (Send vs. Verify):**
   - Connect the True output of *Valid request?* to an **If** node named `Send or check?`.
   - Set condition: Left Value `={{ $json.action }}` equals `"send"`.

6. **Implement SMS Sending Branch:**
   - Connect the True output of *Send or check?* to an **MCP Client** node named `Send code by SMS`.
   - Set Authentication to `Bearer Auth` (create a credential using your SMS Vert Pro API V2 token).
   - Set Endpoint URL to `https://www.smsvertpro.com/mcp/`, Server Transport to `HTTP Streamable`, Input Mode to `JSON`, Tool Mode to `ID`, Tool Value to `generate_otp`, and On Error behavior to `Continue Regular Output`.
   - Set JSON Input to `={{ JSON.stringify({ to: $json.phone, sender: $('OTP settings').first().json.sender }) }}`.
   - Connect *Send code by SMS* to an **If** node named `Code sent?`.
   - Set condition: `={{ $json.structuredContent?.ok }}` equals `true`.
   - Connect True output to a **Respond to Webhook** node named `Respond: code sent` (Response Code `200`, body: `={{ JSON.stringify({ ok: true, phone: $('Read request').item.json.phone, expires_in_seconds: 300 }) }}`).
   - Connect False output to a **Respond to Webhook** node named `Respond: code not sent` (Response Code: `={{ $json.structuredContent?.status === 'OTP_ALREADY_EXIST' ? 429 : 502 }}`, body: error JSON structure).

7. **Implement Code Verification Branch:**
   - Connect the False output of *Send or check?* to an **MCP Client** node named `Check code`.
   - Set Authentication to `Bearer Auth` (using the same SMS Vert Pro credential).
   - Set Endpoint URL to `https://www.smsvertpro.com/mcp/`, Server Transport to `HTTP Streamable`, Input Mode to `JSON`, Tool Mode to `ID`, Tool Value to `verify_otp`, and On Error behavior to `Continue Regular Output`.
   - Set JSON Input to `={{ JSON.stringify({ to: $json.phone, code: $json.code }) }}`.
   - Connect *Check code* to an **If** node named `Code valid?`.
   - Set condition: `={{ $json.structuredContent?.valid }}` equals `true`.

8. **Implement Google Sheets Logging & Success Responses:**
   - Connect the True output of *Code valid?* to a **Google Sheets** node named `Log verified number`.
   - Configure credentials (Google account), Operation to `Append`, Document ID/Sheet Name mode to URL, using `={{ $('OTP settings').first().json.sheet_url }}`.
   - Map columns: `phone` to `={{ $('Read request').item.json.phone }}` and `verified_at` to `={{ $now.toFormat('yyyy-MM-dd HH:mm:ss') }}`.
   - Connect *Log verified number* to a **Respond to Webhook** node named `Respond: number verified` (Response Code `200`, body: `={{ JSON.stringify({ ok: true, verified: true, phone: $('Read request').item.json.phone }) }}`).
   - Connect the False output of *Code valid?* to a **Respond to Webhook** node named `Respond: wrong or expired code` (Response Code `401`, body: error JSON structure).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| SMS Vert Pro API V2 Documentation | [SMS Vert Pro API Documentation V2](https://www.smsvertpro.com/api/doc-v2.php) |
| SMS Vert Pro Platform | [SMS Vert Pro Website](https://www.smsvertpro.com) |
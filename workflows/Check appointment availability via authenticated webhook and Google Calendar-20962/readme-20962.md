Check appointment availability via authenticated webhook and Google Calendar

https://n8nworkflows.xyz/workflows/check-appointment-availability-via-authenticated-webhook-and-google-calendar-20962


# Check appointment availability via authenticated webhook and Google Calendar

### 1. Workflow Overview

The purpose of this workflow is to expose a secure, authenticated webhook endpoint that validates proposed appointment time slots and queries Google Calendar to determine their availability. It is designed for use cases such as external booking forms, voice assistants, chat interfaces, and internal scheduling tools where checking calendar availability prior to booking is required.

The workflow logic is grouped into the following functional blocks:
- **1.1 Input Reception & Validation:** Receives incoming HTTP POST requests, enforces header authentication, and validates timestamp format, chronological order, and duration constraints.
- **1.2 Calendar Availability Check:** Evaluates the request validity and, if passed, queries Google Calendar for free/busy status within the specified interval.
- **1.3 Response Formatting & Delivery:** Normalizes the availability output or handles validation and provider errors, returning appropriate HTTP status codes (200, 400, or 502) to the caller.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Validation
**Overview:** This block ingests the incoming scheduling request, secures it using header authentication, and applies strict programmatic checks to ensure timestamps conform to RFC3339 standards, occur in the future, and span no longer than 24 hours.

**Nodes Involved:**
- `Receive Request`
- `Validate Request`
- `Request Is Valid`

**Node Details:**
- **Receive Request**
  - *Type and Technical Role:* `n8n-nodes-base.webhook` (v2). Acts as the workflow entry point, exposing an HTTP POST webhook secured via Header Authentication.
  - *Configuration Choices:* Configured to listen on path `template-calendar-availability` with method `POST`, using `headerAuth` for authentication and `responseNode` mode.
  - *Key Expressions or Variables:* None (Webhook trigger).
  - *Input and Output Connections:* Input: External HTTP request. Output: Connects to `Validate Request`.
  - *Version-specific Requirements:* Version 2.
  - *Edge Cases / Potential Failures:* Unauthorized calls will fail if headers do not match credentials.
- **Validate Request**
  - *Type and Technical Role:* `n8n-nodes-base.code` (v2). Executes custom JavaScript to parse and validate request body parameters (`start` and `end`).
  - *Configuration Choices:* Custom JS script checking for RFC3339 format, future timestamps, positive intervals, and a maximum duration of 24 hours.
  - *Key Expressions or Variables:* Reads `$input.first().json.body`.
  - *Input and Output Connections:* Input: `Receive Request`. Output: Connects to `Request Is Valid`.
  - *Version-specific Requirements:* Version 2.
  - *Edge Cases / Potential Failures:* Malformed JSON bodies or missing parameters are caught and populated into an `errors` array.
- **Request Is Valid**
  - *Type and Technical Role:* `n8n-nodes-base.if` (v2.2). Branches the workflow based on the validation outcome.
  - *Configuration Choices:* Evaluates whether `={{ $json.valid }}` equals `true`.
  - *Key Expressions or Variables:* `={{ $json.valid }}`.
  - *Input and Output Connections:* Input: `Validate Request`. Output (True): Connects to `Check Calendar`. Output (False): Connects to `Return Invalid`.
  - *Version-specific Requirements:* Version 2.2.
  - *Edge Cases / Potential Failures:* Strict type validation enabled.

---

#### Block 1.2: Calendar Availability Check
**Overview:** This block interacts with Google Calendar to query free/busy availability for validated time intervals, handling potential upstream provider errors gracefully.

**Nodes Involved:**
- `Check Calendar`

**Node Details:**
- **Check Calendar**
  - *Type and Technical Role:* `n8n-nodes-base.googleCalendar` (v1.3). Queries Google Calendar API for availability data.
  - *Configuration Choices:* Resource set to `calendar`, operation set to `availability`, output format set to `availability`. Configured with target calendar ID and error output continuation (`onError: continueErrorOutput`).
  - *Key Expressions or Variables:* 
    - `timeMin`: `={{ $('Validate Request').first().json.request.start }}`
    - `timeMax`: `={{ $('Validate Request').first().json.request.end }}`
    - Calendar ID: `YOUR_CALENDAR_ID`
  - *Input and Output Connections:* Input: `Request Is Valid` (True branch). Output 0 (Main): Connects to `Format Availability`. Output 1 (Error): Connects to `Return Provider Error`.
  - *Version-specific Requirements:* Version 1.3. Requires Google Calendar OAuth2 credentials.
  - *Edge Cases / Potential Failures:* API rate limits, permission failures, or invalid calendar IDs trigger the error output branch (`Return Provider Error`).

---

#### Block 1.3: Response Formatting & Delivery
**Overview:** This block formats the successful calendar availability check into a structured JSON payload or constructs error responses for invalid client requests and provider failures.

**Nodes Involved:**
- `Format Availability`
- `Return Availability`
- `Return Invalid`
- `Return Provider Error`

**Node Details:**
- **Format Availability**
  - *Type and Technical Role:* `n8n-nodes-base.code` (v2). Processes the raw calendar availability output into a standardized schema.
  - *Configuration Choices:* Validates that a boolean `available` property exists and constructs a clean JSON response object with status, availability flag, and normalized UTC times. Error output continuation enabled (`onError: continueErrorOutput`).
  - *Key Expressions or Variables:* Reads `$json.available` and references `$('Validate Request').first().json.request`.
  - *Input and Output Connections:* Input: `Check Calendar`. Output 0 (Main): Connects to `Return Availability`. Output 1 (Error): Connects to `Return Provider Error`.
  - *Version-specific Requirements:* Version 2.
  - *Edge Cases / Potential Failures:* Missing or unexpected data structures throw an error, routing execution to the provider error handler.
- **Return Availability**
  - *Type and Technical Role:* `n8n-nodes-base.respondToWebhook` (v1.4). Sends an HTTP 200 response containing the availability status.
  - *Configuration Choices:* Response code `200`, respond with `json`, response body `={{ $json }}`.
  - *Key Expressions or Variables:* `={{ $json }}`.
  - *Input and Output Connections:* Input: `Format Availability`. Output: None (Terminal node).
  - *Version-specific Requirements:* Version 1.4.
- **Return Invalid**
  - *Type and Technical Role:* `n8n-nodes-base.respondToWebhook` (v1.4). Returns an HTTP 400 response for client-side validation errors.
  - *Configuration Choices:* Response code `400`, respond with `json`, response body returning status `invalid_request` and error list.
  - *Key Expressions or Variables:* `={{ { status: 'invalid_request', errors: $json.errors } }}`.
  - *Input and Output Connections:* Input: `Request Is Valid` (False branch). Output: None (Terminal node).
  - *Version-specific Requirements:* Version 1.4.
- **Return Provider Error**
  - *Type and Technical Role:* `n8n-nodes-base.respondToWebhook` (v1.4). Returns an HTTP 502 response when Google Calendar cannot be reached or queried successfully.
  - *Configuration Choices:* Response code `502`, respond with `json`, response body indicating an unavailable service state without making false availability claims.
  - *Key Expressions or Variables:* `={{ { status: 'unavailable', message: 'The calendar could not be checked. No availability claim was made.' } }}`.
  - *Input and Output Connections:* Input: Connected from error outputs of `Check Calendar` and `Format Availability`. Output: None (Terminal node).
  - *Version-specific Requirements:* Version 1.4.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Setup Instructions | `n8n-nodes-base.stickyNote` | Workflow documentation and setup guide. | None | None | Check appointment availability in Google Calendar<br><br>Check a proposed appointment interval through an authenticated webhook. This workflow validates the requested times, reads Google Calendar availability, and returns a structured available or busy result. It suits booking forms, voice assistants, chat assistants, and internal scheduling tools that already have a proposed start and end time.<br><br>### Setup<br>1. Import the workflow and leave it inactive while configuring it.<br>2. In Receive Request, create a Header Auth credential with your chosen header name and secret value. Set a unique webhook path. Keep the secret in credentials and in your calling application, never in the workflow JSON.<br>3. In Check Calendar, connect your Google Calendar OAuth credential and replace YOUR_CALENDAR_ID with a calendar that credential can access. Start with a dedicated test calendar.<br>4. Send Content-Type: application/json and the configured authentication header. The request body needs start and end, such as `{"start":"2030-01-15T14:00:00-06:00","end":"2030-01-15T14:30:00-06:00"}`. Replace these example dates with a future interval you intend to test.<br>5. Test one free interval and one busy interval. Test invalid times and a provider permission failure. Verify that a provider failure never returns an available result.<br>6. Activate only after these checks, then use the production webhook URL shown by n8n.<br><br>HTTP 200 returns status available or busy, a boolean available, and the normalized UTC start and end. HTTP 400 returns invalid_request and validation errors. HTTP 502 returns unavailable without claiming that the interval is free. This is a snapshot for one interval. It neither generates a slot catalogue nor holds or creates an appointment. Calendar state can change after the check. Successful, error, and manual execution saving are disabled in the export; review your instance logging and retention before using real requests. Prepared by Benian Technologies: https://benian.ai/integrations/n8n-automation |
| Receive Request | `n8n-nodes-base.webhook` | Receives incoming POST webhook requests with header auth. | None | Validate Request | |
| Validate Request | `n8n-nodes-base.code` | Validates RFC3339 timestamps, future dates, and 24-hour max duration. | Receive Request | Request Is Valid | Validate the interval<br><br>Send start and end in the JSON body. Use complete timestamps with seconds and Z or a numeric UTC offset. Start must be in the future; duration must be greater than zero and at most 24 hours. Invalid requests stop before Google Calendar. |
| Request Is Valid | `n8n-nodes-base.if` | Routes execution based on request validation results. | Validate Request | Check Calendar, Return Invalid | Validate the interval<br><br>Send start and end in the JSON body. Use complete timestamps with seconds and Z or a numeric UTC offset. Start must be in the future; duration must be greater than zero and at most 24 hours. Invalid requests stop before Google Calendar. |
| Check Calendar | `n8n-nodes-base.googleCalendar` | Queries Google Calendar availability for the specified time interval. | Request Is Valid | Format Availability, Return Provider Error | Configure the calendar<br><br>Select a Google Calendar OAuth credential and replace YOUR_CALENDAR_ID. Use a dedicated test calendar first. This node reads availability only; it does not reserve or create an event. |
| Format Availability | `n8n-nodes-base.code` | Formats availability output into status, boolean, and normalized times. | Check Calendar | Return Availability, Return Provider Error | |
| Return Availability | `n8n-nodes-base.respondToWebhook` | Returns HTTP 200 with availability status. | Format Availability | None | |
| Return Invalid | `n8n-nodes-base.respondToWebhook` | Returns HTTP 400 with validation errors. | Request Is Valid | None | |
| Return Provider Error | `n8n-nodes-base.respondToWebhook` | Returns HTTP 502 when calendar checks fail. | Check Calendar, Format Availability | None | Interpret the response<br><br>200: available or busy with a boolean result.<br>400: invalid_request.<br>502: unavailable, with no availability claim.<br><br>A free result is a snapshot. Recheck before booking and account for concurrent calendar changes. |
| Validate the interval | `n8n-nodes-base.stickyNote` | Documentation note for validation logic. | None | None | Validate the interval<br><br>Send start and end in the JSON body. Use complete timestamps with seconds and Z or a numeric UTC offset. Start must be in the future; duration must be greater than zero and at most 24 hours. Invalid requests stop before Google Calendar. |
| Configure the calendar | `n8n-nodes-base.stickyNote` | Documentation note for calendar configuration. | None | None | Configure the calendar<br><br>Select a Google Calendar OAuth credential and replace YOUR_CALENDAR_ID. Use a dedicated test calendar first. This node reads availability only; it does not reserve or create an event. |
| Interpret the response | `n8n-nodes-base.stickyNote` | Documentation note for HTTP responses. | None | None | Interpret the response<br><br>200: available or busy with a boolean result.<br>400: invalid_request.<br>502: unavailable, with no availability claim.<br><br>A free result is a snapshot. Recheck before booking and account for concurrent calendar changes. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger:**
   - Add a **Webhook** node named `Receive Request`.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `template-calendar-availability` (or your custom path).
   - Set **Authentication** to `Header Auth`. Create and assign a Header Auth credential with your chosen header name and secret.
   - Set **Response Mode** to `Response Node`.

2. **Add Request Validation Logic:**
   - Add a **Code** node named `Validate Request`, connected after `Receive Request`.
   - Paste the validation JavaScript code into the **JavaScript** parameter to parse RFC3339 timestamps, check future constraints, and validate maximum 24-hour durations.

3. **Add Conditional Routing:**
   - Add an **IF** node named `Request Is Valid`, connected after `Validate Request`.
   - Configure condition: Left Value `={{ $json.valid }}` equal to Boolean `true`.

4. **Add Google Calendar Integration:**
   - Add a **Google Calendar** node named `Check Calendar`, connected to the **True** output of `Request Is Valid`.
   - Set **Resource** to `calendar` and **Operation** to `availability`.
   - Set **Output Format** to `availability`.
   - Configure **Calendar** mode to `ID` and value to your target calendar ID (`YOUR_CALENDAR_ID`).
   - Set **Time Min** expression: `={{ $('Validate Request').first().json.request.start }}`
   - Set **Time Max** expression: `={{ $('Validate Request').first().json.request.end }}`
   - Configure **On Error** to `Continue Error Output`.
   - Connect the error output to a response webhook node.

5. **Format Availability Results:**
   - Add a **Code** node named `Format Availability`, connected to the main output of `Check Calendar`.
   - Paste the formatting JavaScript code to map availability flags and normalized times.
   - Configure **On Error** to `Continue Error Output` and connect its error output to the provider error webhook.

6. **Add Webhook Response Nodes:**
   - Add three **Respond to Webhook** nodes:
     - **Return Availability:** Connected after `Format Availability`. Response Code: `200`. Response Body: `={{ $json }}`.
     - **Return Invalid:** Connected to the **False** output of `Request Is Valid`. Response Code: `400`. Response Body: `={{ { status: 'invalid_request', errors: $json.errors } }}`.
     - **Return Provider Error:** Connected to the error outputs of both `Check Calendar` and `Format Availability`. Response Code: `502`. Response Body: `={{ { status: 'unavailable', message: 'The calendar could not be checked. No availability claim was made.' } }}`.

7. **Finalize Credentials & Testing:**
   - Configure your Google Calendar OAuth2 credentials.
   - Test locally with sample JSON payloads (`Content-Type: application/json` with valid `start` and `end` keys) before activating the workflow.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Prepared by Benian Technologies | https://benian.ai/integrations/n8n-automation |
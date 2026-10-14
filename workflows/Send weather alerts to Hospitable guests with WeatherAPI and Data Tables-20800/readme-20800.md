Send weather alerts to Hospitable guests with WeatherAPI and Data Tables

https://n8nworkflows.xyz/workflows/send-weather-alerts-to-hospitable-guests-with-weatherapi-and-data-tables-20800


# Send weather alerts to Hospitable guests with WeatherAPI and Data Tables

### 1. Workflow Overview

This workflow automates severe weather alerts for properties managed via Hospitable. It runs daily at 17:00 to query WeatherAPI.com for upcoming rain or strong wind forecasts within a 24-hour window. If adverse weather conditions are detected, the workflow identifies the current or arriving guest through the Hospitable Public API, verifies that a message has not already been sent within the past 48 hours (using an n8n Data Table), and posts a personalized weather advisory directly to the booking conversation. In `DRY_RUN` mode, or whenever automated messaging is unsafe or unconfirmed, the workflow gracefully bypasses guest messaging and notifies the property host via SMTP email instead.

The workflow logic is grouped into the following functional blocks:

- **1.1 Input Reception & Configuration:** Initializes execution parameters, validates user configuration settings, and establishes the operational mode (`DRY_RUN` or `LIVE`).
- **1.2 Forecast Evaluation:** Fetches weather forecast data from WeatherAPI.com and evaluates whether rain or wind thresholds are met in the target 24-hour window.
- **1.3 Guest Identification:** Queries the Hospitable API for active reservations, filters eligible guests based on stay status, check-in/check-out dates, and communication channel availability.
- **1.4 Message Processing & Data Governance:** Ensures the tracking Data Table exists, retrieves recent communication logs, enforces the 48-hour anti-spam rule, and constructs contextual messages with natural time phrases.
- **1.5 Dispatch & Logging:** Routes execution based on operating mode (`LIVE` vs `DRY_RUN`), claims transaction log rows, sends messages to guests via the Hospitable API, and handles fallback email notifications for hosts.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception & Configuration
- **Overview:** Triggers the workflow on a daily schedule at 17:00 and injects user-defined properties, API credentials, and operational thresholds into the execution pipeline.
- **Nodes Involved:** `Every day at 17:00`, `Config`
- **Node Details:**
  - **Every day at 17:00**
    - Type and technical role: `n8n-nodes-base.scheduleTrigger` (Trigger node)
    - Configuration choices: Configured to fire daily at hour 17 (5:00 PM).
    - Key expressions or variables: None.
    - Input and output connections: Output connects to `Config`.
    - Version-specific requirements: Version 1.2.
    - Edge cases or potential failure types: Failure to align workflow time zone in n8n settings with the property's local time zone will cause the trigger to fire at the wrong local hour.
  - **Config**
    - Type and technical role: `n8n-nodes-base.code` (Code node)
    - Configuration choices: Executes JavaScript to define configuration constants including `property_id`, geographic `location`, `time_zone`, weather `thresholds`, messaging `templates`, and operational `MODE` (`DRY_RUN` or `LIVE`). Performs validation checks to ensure placeholder strings (e.g., `YOUR_HOSPITABLE_PROPERTY_ID`) are replaced before execution.
    - Key expressions or variables: Reads system time via `new Date().toISOString()`.
    - Input and output connections: Input from `Every day at 17:00`; output connects to `Fetch Forecast`.
    - Version-specific requirements: Version 2.
    - Edge cases or potential failure types: Throws an explicit error if required configuration keys contain unedited template values or if an invalid operational mode is specified.

---

#### Block 1.2: Forecast Evaluation
- **Overview:** Requests hourly weather forecast data from WeatherAPI.com and programmatically determines if rain or wind metrics exceed configured safety thresholds within the next 24 hours.
- **Nodes Involved:** `Fetch Forecast`, `Decide Weather`, `Weather Needed?`
- **Node Details:**
  - **Fetch Forecast**
    - Type and technical role: `n8n-nodes-base.httpRequest` (Action node)
    - Configuration choices: Calls `https://api.weatherapi.com/v1/forecast.json` with a 30-second timeout, requesting a 2-day forecast while omitting AQI and alerts. Configured with a maximum of 3 retries and a 5-second wait interval on failure.
    - Key expressions or variables: Uses `={{ $('Config').first().json.location }}` for geographic coordinates.
    - Input and output connections: Input from `Config`; output connects to `Decide Weather`.
    - Version-specific requirements: Version 4.2.
    - Edge cases or potential failure types: Authentication failures due to invalid WeatherAPI keys, network timeouts, or rate limits.
    - Credentials required: WeatherAPI.com (HTTP Query Auth with parameter `key`).
  - **Decide Weather**
    - Type and technical role: `n8n-nodes-base.code` (Code node)
    - Configuration choices: Processes hourly forecast arrays against rain probability, precipitation millimeters, and wind gust thresholds. Calculates natural language time references (`this evening`, `tomorrow morning`, etc.).
    - Key expressions or variables: Accesses parameters from `Config` and raw payload from `Fetch Forecast`.
    - Input and output connections: Input from `Fetch Forecast`; output connects to `Weather Needed?`.
    - Version-specific requirements: Version 2.
    - Edge cases or potential failure types: Throws errors if the forecast payload returns fewer than 20 valid hours or contains non-numeric data fields.
  - **Weather Needed?**
    - Type and technical role: `n8n-nodes-base.if` (Routing node)
    - Configuration choices: Evaluates boolean condition `needed` to determine if adverse weather criteria were met.
    - Key expressions or variables: `={{ $json.needed }}`
    - Input and output connections: Input from `Decide Weather`; true branch connects to `Get Stays`, false branch terminates execution.
    - Version-specific requirements: Version 2.2.
    - Edge cases or potential failure types: Strict type validation failure if forecast data structure changes unexpectedly.

---

#### Block 1.3: Guest Identification
- **Overview:** Queries the Hospitable Public API to list active reservations for the property and filters them to pinpoint the single eligible in-house or arriving guest.
- **Nodes Involved:** `Get Stays`, `Pick Guest`
- **Node Details:**
  - **Get Stays**
    - Type and technical role: `n8n-nodes-base.httpRequest` (Action node)
    - Configuration choices: Sends a GET request to `https://public.api.hospitable.com/v2/reservations` querying check-outs starting today through 120 days out, including guest details with a pagination limit of 100 items. Configured with up to 2 retries.
    - Key expressions or variables: Uses `={{ $('Config').first().json.property_id }}` and `={{ $('Decide Weather').first().json.today }}`.
    - Input and output connections: Input from `Weather Needed?` (true branch); output connects to `Pick Guest`.
    - Version-specific requirements: Version 4.2.
    - Edge cases or potential failure types: Expired or revoked Hospitable personal access tokens resulting in 401 Unauthorized errors.
    - Credentials required: Hospitable (HTTP Bearer token).
  - **Pick Guest**
    - Type and technical role: `n8n-nodes-base.code` (Code node)
    - Configuration choices: Iterates through reservation records, validating stay status (`accepted`), property IDs, and timing constraints (in-house now vs. arriving today). Applies exclusion rules for check-outs today or tomorrow (if configured). Handles ambiguous states (multiple eligible guests) or missing conversation identifiers.
    - Key expressions or variables: Accesses outputs from `Config` and `Decide Weather`.
    - Input and output connections: Input from `Get Stays`; output connects to `Ensure Weather Table`.
    - Version-specific requirements: Version 2.
    - Edge cases or potential failure types: Throws an error if the Hospitable API response lacks a valid `data` array.

---

#### Block 1.4: Message Processing & Data Governance
- **Overview:** Manages the n8n Data Table (`weather_messages`) to verify logging structures, query past interaction history, and enforce a strict 48-hour no-repeat messaging rule.
- **Nodes Involved:** `Ensure Weather Table`, `Read Recent Sends`, `Render Message`
- **Node Details:**
  - **Ensure Weather Table**
    - Type and technical role: `n8n-nodes-base.dataTable` (Data Table management node)
    - Configuration choices: Automatically creates the `weather_messages` data table if it does not already exist, defining schema columns for reservation codes, execution IDs, states, timestamps, and HTTP statuses.
    - Key expressions or variables: None.
    - Input and output connections: Input from `Pick Guest`; output connects to `Read Recent Sends`.
    - Version-specific requirements: Version 1.1.
    - Edge cases or potential failure types: Insufficient database permissions or unsupported n8n storage backend configurations.
  - **Read Recent Sends**
    - Type and technical role: `n8n-nodes-base.dataTable` (Data Table read node)
    - Configuration choices: Queries records in `weather_messages` matching the current reservation code to evaluate communication history. Configured to execute once per run and always output data.
    - Key expressions or variables: `={{ ($('Pick Guest').first().json.reservation || {}).reservation_code || '__none__' }}`
    - Input and output connections: Input from `Ensure Weather Table`; output connects to `Render Message`.
    - Version-specific requirements: Version 1.1.
    - Edge cases or potential failure types: Missing table indices or database query timeouts.
  - **Render Message**
    - Type and technical role: `n8n-nodes-base.code` (Code node)
    - Configuration choices: Applies the 48-hour anti-spam rule against recent log entries, formats personalized message templates with natural time phrases, compiles detailed operational metadata, and establishes routing paths (`SEND`, `NOTIFY`, or `NONE`).
    - Key expressions or variables: Combines variables from `Config`, `Decide Weather`, `Pick Guest`, and `Read Recent Sends`.
    - Input and output connections: Input from `Read Recent Sends`; output connects to `Send?`.
    - Version-specific requirements: Version 2.
    - Edge cases or potential failure types: Throws an error if any template placeholder remains unfulfilled after substitution.

---

#### Block 1.5: Dispatch & Logging
- **Overview:** Routes execution based on operating mode (`LIVE` vs `DRY_RUN`), securely claims transaction log rows in the Data Table, posts messages to guest conversations via the Hospitable API, classifies delivery status, and triggers fallback email notifications to the host.
- **Nodes Involved:** `Send?`, `Claim Row`, `Claimed?`, `Send Guest Message`, `Classify Send`, `Not Sent?`, `Claim Failed`, `Email Send Problem`, `Notify host?`, `Log Row?`, `Insert Log Row`, `Email host`
- **Node Details:**
  - **Send?**
    - Type and technical role: `n8n-nodes-base.if` (Routing node)
    - Configuration choices: Evaluates whether the execution route is set to `SEND`.
    - Key expressions or variables: `={{ $json.route === 'SEND' }}`
    - Input and output connections: Input from `Render Message`; true branch connects to `Claim Row`, false branch connects to `Notify host?`.
    - Version-specific requirements: Version 2.2.
    - Edge cases or potential failure types: Strict type mismatch if route string evaluation fails.
  - **Claim Row**
    - Type and technical role: `n8n-nodes-base.dataTable` (Data Table write node)
    - Configuration choices: Inserts a row into `weather_messages` with a `CLAIMED` state to lock the reservation and prevent race conditions. Always outputs data.
    - Key expressions or variables: Mapped using properties from `Render Message` (`claim_row`).
    - Input and output connections: Input from `Send?` (true branch); output connects to `Claimed?`.
    - Version-specific requirements: Version 1.1.
    - Edge cases or potential failure types: Table insertion failures due to constraint violations.
  - **Claimed?**
    - Type and technical role: `n8n-nodes-base.if` (Routing node)
    - Configuration choices: Verifies whether a valid database row ID was successfully returned from the table insert operation.
    - Key expressions or variables: `={{ !!$json.id }}`
    - Input and output connections: Input from `Claim Row`; true branch connects to `Send Guest Message`, false branch connects to `Claim Failed`.
    - Version-specific requirements: Version 2.2.
    - Edge cases or potential failure types: Null ID returns if table write fails silently.
  - **Send Guest Message**
    - Type and technical role: `n8n-nodes-base.httpRequest` (Action node)
    - Configuration choices: Sends a POST request to `https://public.api.hospitable.com/v2/reservations/{reservation_id}/messages` with a JSON payload containing the rendered message body. Configured with `onError: continueRegularOutput` and a full response structure with error suppression (`neverError: true`) to capture HTTP status codes safely.
    - Key expressions or variables: Uses `={{ $('Render Message').first().json.reservation_id }}` and `={{ JSON.stringify({ body: $('Render Message').first().json.message_body }) }}`.
    - Input and output connections: Input from `Claimed?` (true branch); output connects to `Classify Send`.
    - Version-specific requirements: Version 4.2.
    - Edge cases or potential failure types: API rate limiting (Hospitable allows 2 messages per minute per reservation) or network timeouts.
    - Credentials required: Hospitable (HTTP Bearer token).
  - **Classify Send**
    - Type and technical role: `n8n-nodes-base.code` (Code node)
    - Configuration choices: Interprets the HTTP response status code from Hospitable, categorizing the outcome as `SENT` (2xx), `FAILED` (4xx excluding 408/429), or `SEND_UNKNOWN` (timeouts, 5xx, 408, 429).
    - Key expressions or variables: Reads status codes from `Send Guest Message`.
    - Input and output connections: Input from `Send Guest Message`; output connects to `Record Send` and `Not Sent?`.
    - Version-specific requirements: Version 2.
    - Edge cases or potential failure types: Unhandled non-standard HTTP status responses.
  - **Record Send**
    - Type and technical role: `n8n-nodes-base.dataTable` (Data Table update node)
    - Configuration choices: Updates the previously claimed log row in `weather_messages` with the finalized delivery state and HTTP status code using the row ID.
    - Key expressions or variables: Mapped from `Classify Send` output (`row_id`, `state`, `http_status`).
    - Input and output connections: Input from `Classify Send`; output terminates branch.
    - Version-specific requirements: Version 1.1.
    - Edge cases or potential failure types: Failure to locate row ID during update operations.
  - **Not Sent?**
    - Type and technical role: `n8n-nodes-base.if` (Routing node)
    - Configuration choices: Evaluates whether the message send operation was unsuccessful (`!sent`).
    - Key expressions or variables: `={{ !$json.sent }}`
    - Input and output connections: Input from `Classify Send`; true branch connects to `Email Send Problem`.
    - Version-specific requirements: Version 2.2.
    - Edge cases or potential failure types: Incorrect boolean evaluation of send status flags.
  - **Claim Failed**
    - Type and technical role: `n8n-nodes-base.code` (Code node)
    - Configuration choices: Generates fallback email content for situations where transaction row claiming failed in the Data Table.
    - Key expressions or variables: References data from `Render Message`.
    - Input and output connections: Input from `Claimed?` (false branch); output connects to `Email Send Problem`.
    - Version-specific requirements: Version 2.
    - Edge cases or potential failure types: Missing variable scopes during email body compilation.
  - **Email Send Problem**
    - Type and technical role: `n8n-nodes-base.emailSend` (Action node)
    - Configuration choices: Sends plain-text alert emails to the host regarding message delivery failures or unconfirmed sends.
    - Key expressions or variables: `={{ $json.email_body }}`, `={{ $json.email_subject }}`, `={{ $json.email_to }}`, `={{ $('Config').first().json.from_email }}`.
    - Input and output connections: Input from `Not Sent?` or `Claim Failed`; output terminates execution.
    - Version-specific requirements: Version 2.1.
    - Edge cases or potential failure types: SMTP authentication errors or connection refusals.
    - Credentials required: SMTP.
  - **Notify host?**
    - Type and technical role: `n8n-nodes-base.if` (Routing node)
    - Configuration choices: Evaluates whether the operational route requires host notification (`route === 'NOTIFY'`).
    - Key expressions or variables: `={{ $json.route === 'NOTIFY' }}`
    - Input and output connections: Input from `Send?` (false branch); true branch connects to `Log Row?`.
    - Version-specific requirements: Version 2.2.
    - Edge cases or potential failure types: Route string mismatches.
  - **Log Row?**
    - Type and technical role: `n8n-nodes-base.if` (Routing node)
    - Configuration choices: Checks if a log row object exists to be recorded for dry runs or manual interventions.
    - Key expressions or variables: `={{ !!$json.log_row }}`
    - Input and output connections: Input from `Notify host?` (true branch); true branch connects to `Insert Log Row`, false branch connects directly to `Email host`.
    - Version-specific requirements: Version 2.2.
    - Edge cases or potential failure types: Undefined property access errors.
  - **Insert Log Row**
    - Type and technical role: `n8n-nodes-base.dataTable` (Data Table write node)
    - Configuration choices: Inserts dry run or manual hand-off log entries into `weather_messages`.
    - Key expressions or variables: Mapped from `Render Message` (`log_row` properties).
    - Input and output connections: Input from `Log Row?` (true branch); output connects to `Email host`.
    - Version-specific requirements: Version 1.1.
    - Edge cases or potential failure types: Database schema validation errors.
  - **Email host**
    - Type and technical role: `n8n-nodes-base.emailSend` (Action node)
    - Configuration choices: Sends informational dry-run summaries or manual send alerts to the host via SMTP in plain text format.
    - Key expressions or variables: `={{ $('Render Message').first().json.email_body }}`, `={{ $('Render Message').first().json.email_subject }}`, `={{ $('Render Message').first().json.email_to }}`, `={{ $('Config').first().json.from_email }}`.
    - Input and output connections: Inputs from `Log Row?` (false branch) and `Insert Log Row`; output terminates execution.
    - Version-specific requirements: Version 2.1.
    - Edge cases or potential failure types: Invalid SMTP credentials or recipient address rejection.
    - Credentials required: SMTP.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Documentation and setup guide | None | None | ## Weather heads-up for your Hospitable guests<br>Sends the guest who is staying one short, friendly message when rain or strong wind is on the way, so the outdoor cushions and umbrella come inside in time. Most evenings it checks, finds nothing and stays quiet.<br><br>### How it works<br>1. Every day at 17:00 it reads the next 24 hours of forecast for your property from WeatherAPI.com.<br>2. If an hour meets your rain or gust threshold, it asks Hospitable who is staying, or arriving today.<br>3. It writes the message with a natural time ("tonight", "tomorrow morning") and posts it into the guest's booking conversation on Airbnb, Booking.com or Vrbo.<br>4. A Data Table records every send, so a guest hears from you at most once in 48 hours, and an unclear send is never repeated.<br>5. Anything it cannot do safely (two overlapping stays, a stay with no conversation, an unconfirmed send) comes to you by email instead.<br><br>### Setup steps<br>1. Create three credentials: WeatherAPI.com (Query Auth, parameter `key`), a Hospitable personal access token (Bearer Auth), and SMTP for the emails. Select them in Fetch Forecast, Get Stays, Send Guest Message and the two email nodes.<br>2. Fill in the **Config** node: property ID, location, time zone, your email and the sign-off.<br>3. Set the workflow's time zone in Workflow settings to the property's.<br>4. Leave MODE on DRY_RUN for a few days and read the emails, then switch it to LIVE.<br>5. Optional: set an error workflow in Workflow settings, so a failed run reaches you too.<br><br>### Customising<br>Thresholds and message wording are in Config. Needs an n8n version with Data Tables. |
| Section: forecast | n8n-nodes-base.stickyNote | Visual section grouping for forecast nodes | None | None | ## 1. Check the forecast |
| Section: guest | n8n-nodes-base.stickyNote | Visual section grouping for guest lookup nodes | None | None | ## 2. Find who is staying |
| Section: send | n8n-nodes-base.stickyNote | Visual section grouping for message dispatch nodes | None | None | ## 3. Send once, and record it |
| Section: notify | n8n-nodes-base.stickyNote | Visual section grouping for notification and email nodes | None | None | ## 4. Tell the host when it cannot |
| Every day at 17:00 | n8n-nodes-base.scheduleTrigger | Triggers workflow daily at 17:00 | None | Config | |
| Config | n8n-nodes-base.code | Defines property settings, thresholds, and operational mode | Every day at 17:00 | Fetch Forecast | |
| Fetch Forecast | n8n-nodes-base.httpRequest | Retrieves 2-day hourly weather forecast from WeatherAPI | Config | Decide Weather | |
| Decide Weather | n8n-nodes-base.code | Evaluates forecast against rain/wind thresholds | Fetch Forecast | Weather Needed? | |
| Weather Needed? | n8n-nodes-base.if | Checks if adverse weather conditions require action | Decide Weather | Get Stays | |
| Get Stays | n8n-nodes-base.httpRequest | Fetches active reservations from Hospitable API | Weather Needed? | Pick Guest | |
| Pick Guest | n8n-nodes-base.code | Identifies single eligible in-house or arriving guest | Get Stays | Ensure Weather Table | |
| Ensure Weather Table | n8n-nodes-base.dataTable | Creates weather messages data table if missing | Pick Guest | Read Recent Sends | |
| Read Recent Sends | n8n-nodes-base.dataTable | Queries recent sends for anti-spam enforcement | Ensure Weather Table | Render Message | |
| Render Message | n8n-nodes-base.code | Enforces 48-hour rule and formats message text | Read Recent Sends | Send? | |
| Send? | n8n-nodes-base.if | Routes execution based on LIVE send eligibility | Render Message | Claim Row, Notify host? | |
| Notify host? | n8n-nodes-base.if | Routes execution for dry-run or manual notifications | Send? | Log Row? | |
| Log Row? | n8n-nodes-base.if | Checks if log row needs insertion | Notify host? | Insert Log Row, Email host | |
| Insert Log Row | n8n-nodes-base.dataTable | Inserts dry-run or hand-off record into data table | Log Row? | Email host | |
| Email host | n8n-nodes-base.emailSend | Sends summary or alert emails to property host | Log Row?, Insert Log Row | None | |
| Claim Row | n8n-nodes-base.dataTable | Claims transaction row in data table to prevent races | Send? | Claimed? | |
| Claimed? | n8n-nodes-base.if | Verifies successful row claim before messaging | Claim Row | Send Guest Message, Claim Failed | |
| Send Guest Message | n8n-nodes-base.httpRequest | Posts weather alert message to Hospitable conversation | Claimed? | Classify Send | |
| Classify Send | n8n-nodes-base.code | Evaluates API response status for delivery confirmation | Send Guest Message | Record Send, Not Sent? | |
| Record Send | n8n-nodes-base.dataTable | Updates data table log with final delivery state | Classify Send | None | |
| Not Sent? | n8n-nodes-base.if | Checks if message delivery failed or is unknown | Classify Send | Email Send Problem | |
| Claim Failed | n8n-nodes-base.code | Generates notification payload for failed row claims | Claimed? | Email Send Problem | |
| Email Send Problem | n8n-nodes-base.emailSend | Emails host regarding delivery failures or claim issues | Not Sent?, Claim Failed | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a New Workflow:** Open n8n and create a blank workflow. Set the workflow's global time zone in **Workflow Settings** to match your property's local time zone (e.g., `Europe/London`).
2. **Add Trigger Node (`Every day at 17:00`):**
   - Add a **Schedule Trigger** node.
   - Configure rule: Interval set to trigger at hour `17`.
3. **Add Configuration Node (`Config`):**
   - Add a **Code** node connected downstream of the schedule trigger.
   - Paste the JavaScript configuration snippet (defining `property_id`, `location`, `time_zone`, `thresholds`, `templates`, and `MODE` as `'DRY_RUN'`).
   - Replace placeholder strings (`YOUR_HOSPITABLE_PROPERTY_ID`, `YOUR_LATITUDE,YOUR_LONGITUDE`, `you@example.com`, etc.) with your actual property data.
4. **Add Forecast Fetch Node (`Fetch Forecast`):**
   - Add an **HTTP Request** node.
   - Set Method to `GET`, URL to `https://api.weatherapi.com/v1/forecast.json`.
   - Add Query Parameters: `q` (`={{ $('Config').first().json.location }}`), `days` (`2`), `aqi` (`no`), `alerts` (`no`).
   - Configure Authentication: Generic Credential Type -> HTTP Query Auth (parameter name: `key`). Set retry settings to 3 retries with a 5000ms wait.
5. **Add Weather Decision Node (`Decide Weather`):**
   - Add a **Code** node to process forecast arrays against rain/wind thresholds and generate natural time strings. Connect input from `Fetch Forecast`.
6. **Add Weather Evaluation Conditional Node (`Weather Needed?`):**
   - Add an **If** node connected to `Decide Weather`.
   - Set condition: Left value `={{ $json.needed }}` equals boolean `true`.
7. **Add Reservation Fetch Node (`Get Stays`):**
   - Add an **HTTP Request** node connected to the true branch of `Weather Needed?`.
   - Set Method to `GET`, URL to `https://public.api.hospitable.com/v2/reservations`.
   - Add Query Parameters: `properties[]` (`={{ $('Config').first().json.property_id }}`), `date_query` (`checkout`), `start_date` (`={{ $('Decide Weather').first().json.today }}`), `end_date` (`={{ DateTime.fromISO($('Decide Weather').first().json.today).plus({ days: 120 }).toISODate() }}`), `include` (`guest`), `per_page` (`100`).
   - Add Header Parameter: `Accept` (`application/json`).
   - Configure Authentication: Generic Credential Type -> HTTP Bearer Auth (using your Hospitable Personal Access Token).
8. **Add Guest Selection Node (`Pick Guest`):**
   - Add a **Code** node connected to `Get Stays` to filter active guest stays, evaluate check-in/check-out boundaries, and handle platform messageability.
9. **Add Data Table Setup Nodes (`Ensure Weather Table`, `Read Recent Sends`):**
   - Add a **Data Table** node (`Ensure Weather Table`) configured to create table `weather_messages` if it does not exist, with columns: `reservation_code`, `reservation_id`, `platform`, `mode`, `kind`, `first_hour`, `when_text`, `state`, `sent_at`, `http_status`, `execution_id`.
   - Add a second **Data Table** node (`Read Recent Sends`) configured to `Get` rows from `weather_messages` filtered where `reservation_code` matches the selected guest's reservation code. Enable "Always Output Data".
10. **Add Message Rendering Node (`Render Message`):**
    - Add a **Code** node connected to `Read Recent Sends` to enforce the 48-hour anti-spam rule, compile template text, and determine operational routing paths (`SEND`, `NOTIFY`, `NONE`).
11. **Add Routing and Notification Nodes (`Send?`, `Notify host?`, `Log Row?`, `Insert Log Row`, `Email host`):**
    - Add an **If** node (`Send?`) checking `={{ $json.route === 'SEND' }}`.
    - For the false branch, add an **If** node (`Notify host?`) checking `={{ $json.route === 'NOTIFY' }}`.
    - For its true branch, add an **If** node (`Log Row?`) checking `={{ !!$json.log_row }}`.
    - If true, add a **Data Table** node (`Insert Log Row`) to write to `weather_messages`.
    - Connect both branches to an **Email Send** node (`Email host`) configured with SMTP credentials, mapping To to `={{ $('Render Message').first().json.email_to }}` and From to `={{ $('Config').first().json.from_email }}`.
12. **Add Transaction Claim & Messaging Nodes (`Claim Row`, `Claimed?`, `Send Guest Message`, `Classify Send`):**
    - From the true branch of `Send?`, add a **Data Table** node (`Claim Row`) inserting a row into `weather_messages` with state `CLAIMED`.
    - Add an **If** node (`Claimed?`) checking `={{ !!$json.id }}`.
    - For the true branch, add an **HTTP Request** node (`Send Guest Message`) with Method `POST` to `=https://public.api.hospitable.com/v2/reservations/{{ $('Render Message').first().json.reservation_id }}/messages`. Set body type to JSON with expression `={{ JSON.stringify({ body: $('Render Message').first().json.message_body }) }}`. Configure HTTP Bearer authentication and set error handling to `continueRegularOutput` with full response mapping (`neverError: true`).
    - Add a **Code** node (`Classify Send`) to categorize the HTTP status response into `SENT`, `FAILED`, or `SEND_UNKNOWN`.
13. **Add Record Update & Error Handling Nodes (`Record Send`, `Not Sent?`, `Claim Failed`, `Email Send Problem`):**
    - Connect `Classify Send` to a **Data Table** node (`Record Send`) updating the log row by `row_id` with state and HTTP status.
    - Also connect `Classify Send` to an **If** node (`Not Sent?`) checking `={{ !$json.sent }}`. If true, route to an **Email Send** node (`Email Send Problem`).
    - From the false branch of `Claimed?`, add a **Code** node (`Claim Failed`) connected to the same **Email Send** node (`Email Send Problem`).
14. **Credentials Setup:** Ensure credentials are created and assigned:
    - WeatherAPI.com (HTTP Query Auth)
    - Hospitable API (HTTP Bearer Token)
    - SMTP (Email Service)
15. **Testing & Activation:** Run initial tests in `DRY_RUN` mode. Review generated emails and table logs, then switch `MODE` in the `Config` node to `'LIVE'` for production deployment.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| WeatherAPI.com | External weather forecast data provider service | https://www.weatherapi.com/ |
| Hospitable Public API v2 | Property management platform integration for reservations and guest messaging | https://public.api.hospitable.com/v2/docs |
| n8n Data Tables | Native n8n storage feature used for logging and anti-spam rate limiting | n8n documentation on Data Tables |
Send domain expiry and WHOIS change alerts by email with Apify

https://n8nworkflows.xyz/workflows/send-domain-expiry-and-whois-change-alerts-by-email-with-apify-21012


# Send domain expiry and WHOIS change alerts by email with Apify

### 1. Workflow Overview

This workflow is designed to automate domain portfolio management by monitoring domain expiration dates and WHOIS/RDAP changes on a weekly basis. Its primary target use cases include agencies, freelancers, and IT administrators who manage multiple domains across various registrars and want centralized, proactive alerts without manual checks.

The logical flow is structured into the following distinct blocks:
- **1.1 Schedule & Input Initialization:** Triggers the workflow automatically every Monday morning and injects static configuration variables (domain list, recipient email, warning threshold).
- **1.2 Domain Health & Change Verification:** Interacts with an external cloud service (Apify) to query registry data and track historical changes for the supplied domains.
- **1.3 Data Filtering & Formatting:** Processes the raw output items, evaluates urgency and change status, halts the workflow if no events require attention, and compiles a concise plaintext notification body.
- **1.4 Notification Dispatch:** Delivers the compiled alert summary via an SMTP mail server.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Input Initialization
This block initiates the execution cycle on a recurring schedule and declares the necessary input variables required for the monitoring sequence.

* **Nodes Involved:** 
  - `Every Monday 9:00`
  - `Your domains`

* **Node Details:**
  - **`Every Monday 9:00`**
    - *Type and technical role:* `scheduleTrigger` (Trigger Node). Starts the workflow on a time-based schedule.
    - *Configuration choices:* Interval-based execution set to trigger every week on Mondays at 09:00 AM.
    - *Key expressions or variables:* None.
    - *Input and output connections:* No inputs; outputs to `Your domains`.
    - *Version-specific requirements:* Type version 1.2.
    - *Edge cases or potential failure types:* System downtime during the scheduled execution window (will be skipped until the next occurrence).
  
  - **`Your domains`**
    - *Type and technical role:* `set` (Data Transformation / Initialization Node). Defines hardcoded static inputs for downstream requests.
    - *Configuration choices:* Assigns values to three distinct fields: `domains` (string), `alertEmail` (string), and `expiryWarningDays` (number).
    - *Key expressions or variables:* 
      - `domains`: `"example.com, yourcompany.com"`
      - `alertEmail`: `"you@example.com"`
      - `expiryWarningDays`: `30`
    - *Input and output connections:* Input from `Every Monday 9:00`; outputs to `Check domains on Apify`.
    - *Version-specific requirements:* Type version 3.4.
    - *Edge cases or potential failure types:* Invalid data types if manually edited (e.g., text in a numeric field).

---

#### 2.2 Domain Health & Change Verification
This block communicates with the Apify platform using a synchronous HTTP call to execute an actor that queries domain registries.

* **Nodes Involved:** 
  - `Check domains on Apify`

* **Node Details:**
  - **`Check domains on Apify`**
    - *Type and technical role:* `httpRequest` (Integration / API Call Node). Calls the Apify API endpoint for the Domain Expiry & WHOIS Monitor actor.
    - *Configuration choices:* POST method to `https://api.apify.com/v2/acts/zaden~domain-expiry-whois-monitor/run-sync-get-dataset-items?timeout=280` with a 300,000ms timeout. Uses generic HTTP Header Authentication.
    - *Key expressions or variables:* 
      - Request body JSON: 
        ```javascript
        ={{ JSON.stringify({ domains: $json.domains.split(',').map(d => d.trim()).filter(Boolean), expiryWarningDays: $json.expiryWarningDays, trackChanges: true, onlyChanges: true }) }}
        ```
    - *Input and output connections:* Input from `Your domains`; outputs to `Build alert (stops if nothing changed)`.
    - *Version-specific requirements:* Type version 4.2.
    - *Edge cases or potential failure types:* API timeout (if registry RDAP servers are unresponsive), authentication failure due to expired or incorrect Apify tokens, or incorrect payload parsing.

---

#### 2.3 Data Filtering & Formatting
This block aggregates the returned domain records, sorts them by priority, filters out uninteresting results, and constructs a clean plaintext notification payload.

* **Nodes Involved:** 
  - `Build alert (stops if nothing changed)`

* **Node Details:**
  - **`Build alert (stops if nothing changed)`**
    - *Type and technical role:* `code` (Scripting Node). Executes custom JavaScript in "Run Once for All Items" mode to process and filter dataset outputs.
    - *Configuration choices:* JavaScript execution mode evaluating all incoming items. If zero relevant domain items are returned, the execution array is returned empty (`[]`), terminating the workflow branch.
    - *Key expressions or variables:* 
      - Sorts by severity (`critical`, `warning`, `ok`) and days until expiry.
      - Assembles dynamic text blocks depending on whether a domain lookup failed, is unregistered, expired, or changed.
    - *Input and output connections:* Input from `Check domains on Apify`; outputs to `Email me`.
    - *Version-specific requirements:* Type version 2.
    - *Edge cases or potential failure types:* Script termination if unexpected data structures are returned from the API; empty array returns properly stop execution to prevent sending blank emails.

---

#### 2.4 Notification Dispatch
This block handles the physical delivery of the alert summary to the designated recipient.

* **Nodes Involved:** 
  - `Email me`

* **Node Details:**
  - **`Email me`**
    - *Type and technical role:* `emailSend` (Communication Node). Sends an email via SMTP.
    - *Configuration choices:* Plain text email format.
    - *Key expressions or variables:* 
      - `subject`: `={{ $json.subject }}`
      - `text`: `={{ $json.text }}`
      - `toEmail`: `={{ $('Your domains').first().json.alertEmail }}`
      - `fromEmail`: `"user@example.com"`
    - *Input and output connections:* Input from `Build alert (stops if nothing changed)`; no downstream outputs.
    - *Version-specific requirements:* Type version 2.1.
    - *Edge cases or potential failure types:* SMTP connection timeouts, incorrect credential details, rate-limiting or blacklisting by the mail server, or an invalid recipient address structure.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Every Monday 9:00` | `scheduleTrigger` | Triggers workflow weekly on Mondays at 09:00 | None | `Your domains` | Send domain expiry and WHOIS change alerts to email with Apify<br><br>Never lose a domain to a forgotten renewal. Every Monday this workflow checks your domains and emails you **only when something needs attention**.<br><br>### Who's it for<br>Agencies, freelancers and IT teams who look after domains across several registrars and want one early warning instead of logging into each one.<br><br>### How it works<br>1. **Every Monday 9:00** starts the run.<br>2. **Your domains** holds your list and alert address.<br>3. **Check domains on Apify** runs the Domain Expiry & WHOIS Monitor Actor, which reads each domain's record from its registry over RDAP (WHOIS fallback) and compares it with last week's.<br>4. **Build alert** keeps only domains that are expiring soon, expired, unregistered, or whose registrar, nameservers or expiry date changed. A quiet week stops here, so no empty email.<br>5. **Email me** sends one summary.<br><br>### How to set up<br>1. In **Your domains**, enter your domains (comma-separated) and email.<br>2. In **Check domains on Apify**, create a *Header Auth* credential: name `Authorization`, value `Bearer YOUR_APIFY_TOKEN` (Apify Console > Settings > API & Integrations).<br>3. In **Email me**, choose your SMTP account.<br>4. Activate the workflow.<br><br>### Requirements<br>An Apify account (the free plan works) and an SMTP email account. Cost: $0.0025 per domain per run; failed lookups are free.<br><br>### How to customize the workflow<br>Change `expiryWarningDays`, edit the schedule, or replace **Email me** with Slack, Telegram or Discord using the `subject` and `text` fields.<br><br>### 1. Schedule and your domains<br>Runs every Monday at 9:00. Edit **Your domains**: a comma-separated list, the email to alert, and how many days ahead to warn (default 30). |
| `Your domains` | `set` | Defines input variables (domains list, recipient, and warning threshold) | `Every Monday 9:00` | `Check domains on Apify` | Send domain expiry and WHOIS change alerts to email with Apify<br><br>Never lose a domain to a forgotten renewal. Every Monday this workflow checks your domains and emails you **only when something needs attention**.<br><br>### Who's it for<br>Agencies, freelancers and IT teams who look after domains across several registrars and want one early warning instead of logging into each one.<br><br>### How it works<br>1. **Every Monday 9:00** starts the run.<br>2. **Your domains** holds your list and alert address.<br>3. **Check domains on Apify** runs the Domain Expiry & WHOIS Monitor Actor, which reads each domain's record from its registry over RDAP (WHOIS fallback) and compares it with last week's.<br>4. **Build alert** keeps only domains that are expiring soon, expired, unregistered, or whose registrar, nameservers or expiry date changed. A quiet week stops here, so no empty email.<br>5. **Email me** sends one summary.<br><br>### How to set up<br>1. In **Your domains**, enter your domains (comma-separated) and email.<br>2. In **Check domains on Apify**, create a *Header Auth* credential: name `Authorization`, value `Bearer YOUR_APIFY_TOKEN` (Apify Console > Settings > API & Integrations).<br>3. In **Email me**, choose your SMTP account.<br>4. Activate the workflow.<br><br>### Requirements<br>An Apify account (the free plan works) and an SMTP email account. Cost: $0.0025 per domain per run; failed lookups are free.<br><br>### How to customize the workflow<br>Change `expiryWarningDays`, edit the schedule, or replace **Email me** with Slack, Telegram or Discord using the `subject` and `text` fields.<br><br>### 1. Schedule and your domains<br>Runs every Monday at 9:00. Edit **Your domains**: a comma-separated list, the email to alert, and how many days ahead to warn (default 30). |
| `Check domains on Apify` | `httpRequest` | Executes the Apify Domain Expiry & WHOIS Monitor Actor API | `Your domains` | `Build alert (stops if nothing changed)` | Send domain expiry and WHOIS change alerts to email with Apify<br><br>Never lose a domain to a forgotten renewal. Every Monday this workflow checks your domains and emails you **only when something needs attention**.<br><br>### Who's it for<br>Agencies, freelancers and IT teams who look after domains across several registrars and want one early warning instead of logging into each one.<br><br>### How it works<br>1. **Every Monday 9:00** starts the run.<br>2. **Your domains** holds your list and alert address.<br>3. **Check domains on Apify** runs the Domain Expiry & WHOIS Monitor Actor, which reads each domain's record from its registry over RDAP (WHOIS fallback) and compares it with last week's.<br>4. **Build alert** keeps only domains that are expiring soon, expired, unregistered, or whose registrar, nameservers or expiry date changed. A quiet week stops here, so no empty email.<br>5. **Email me** sends one summary.<br><br>### How to set up<br>1. In **Your domains**, enter your domains (comma-separated) and email.<br>2. In **Check domains on Apify**, create a *Header Auth* credential: name `Authorization`, value `Bearer YOUR_APIFY_TOKEN` (Apify Console > Settings > API & Integrations).<br>3. In **Email me**, choose your SMTP account.<br>4. Activate the workflow.<br><br>### Requirements<br>An Apify account (the free plan works) and an SMTP email account. Cost: $0.0025 per domain per run; failed lookups are free.<br><br>### How to customize the workflow<br>Change `expiryWarningDays`, edit the schedule, or replace **Email me** with Slack, Telegram or Discord using the `subject` and `text` fields.<br><br>### 2. Check on Apify<br>Needs a Header Auth credential: `Authorization: Bearer <token>`. The Actor remembers each domain between runs. |
| `Build alert (stops if nothing changed)` | `code` | Filters items and formats email subject and body content | `Check domains on Apify` | `Email me` | Send domain expiry and WHOIS change alerts to email with Apify<br><br>Never lose a domain to a forgotten renewal. Every Monday this workflow checks your domains and emails you **only when something needs attention**.<br><br>### Who's it for<br>Agencies, freelancers and IT teams who look after domains across several registrars and want one early warning instead of logging into each one.<br><br>### How it works<br>1. **Every Monday 9:00** starts the run.<br>2. **Your domains** holds your list and alert address.<br>3. **Check domains on Apify** runs the Domain Expiry & WHOIS Monitor Actor, which reads each domain's record from its registry over RDAP (WHOIS fallback) and compares it with last week's.<br>4. **Build alert** keeps only domains that are expiring soon, expired, unregistered, or whose registrar, nameservers or expiry date changed. A quiet week stops here, so no empty email.<br>5. **Email me** sends one summary.<br><br>### How to set up<br>1. In **Your domains**, enter your domains (comma-separated) and email.<br>2. In **Check domains on Apify**, create a *Header Auth* credential: name `Authorization`, value `Bearer YOUR_APIFY_TOKEN` (Apify Console > Settings > API & Integrations).<br>3. In **Email me**, choose your SMTP account.<br>4. Activate the workflow.<br><br>### Requirements<br>An Apify account (the free plan works) and an SMTP email account. Cost: $0.0025 per domain per run; failed lookups are free.<br><br>### How to customize the workflow<br>Change `expiryWarningDays`, edit the schedule, or replace **Email me** with Slack, Telegram or Discord using the `subject` and `text` fields.<br><br>### 3. Alert only when needed<br>Builds one email from changed or urgent domains and stops on a quiet week. Swap **Email me** for Slack or Telegram if you prefer. |
| `Email me` | `emailSend` | Sends the compiled plaintext summary via SMTP | `Build alert (stops if nothing changed)` | None | Send domain expiry and WHOIS change alerts to email with Apify<br><br>Never lose a domain to a forgotten renewal. Every Monday this workflow checks your domains and emails you **only when something needs attention**.<br><br>### Who's it for<br>Agencies, freelancers and IT teams who look after domains across several registrars and want one early warning instead of logging into each one.<br><br>### How it works<br>1. **Every Monday 9:00** starts the run.<br>2. **Your domains** holds your list and alert address.<br>3. **Check domains on Apify** runs the Domain Expiry & WHOIS Monitor Actor, which reads each domain's record from its registry over RDAP (WHOIS fallback) and compares it with last week's.<br>4. **Build alert** keeps only domains that are expiring soon, expired, unregistered, or whose registrar, nameservers or expiry date changed. A quiet week stops here, so no empty email.<br>5. **Email me** sends one summary.<br><br>### How to set up<br>1. In **Your domains**, enter your domains (comma-separated) and email.<br>2. In **Check domains on Apify**, create a *Header Auth* credential: name `Authorization`, value `Bearer YOUR_APIFY_TOKEN` (Apify Console > Settings > API & Integrations).<br>3. In **Email me**, choose your SMTP account.<br>4. Activate the workflow.<br><br>### Requirements<br>An Apify account (the free plan works) and an SMTP email account. Cost: $0.0025 per domain per run; failed lookups are free.<br><br>### How to customize the workflow<br>Change `expiryWarningDays`, edit the schedule, or replace **Email me** with Slack, Telegram or Discord using the `subject` and `text` fields.<br><br>### 3. Alert only when needed<br>Builds one email from changed or urgent domains and stops on a quiet week. Swap **Email me** for Slack or Telegram if you prefer. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Set the interval rule to trigger every week on day `1` (Monday) at hour `9`.
   - Rename the node to `Every Monday 9:00`.

2. **Create the Data Input Node:**
   - Add a **Set** node (`n8n-nodes-base.set`).
   - Connect `Every Monday 9:00`'s output to this node's input.
   - Configure three assignments under values:
     - Name: `domains`, Type: `String`, Value: `example.com, yourcompany.com`
     - Name: `alertEmail`, Type: `String`, Value: `you@example.com`
     - Name: `expiryWarningDays`, Type: `Number`, Value: `30`
   - Rename the node to `Your domains`.

3. **Create the Apify Integration Node:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Connect `Your domains`' output to this node's input.
   - Set Method to `POST`.
   - Set URL to `https://api.apify.com/v2/acts/zaden~domain-expiry-whois-monitor/run-sync-get-dataset-items?timeout=280`.
   - Configure Options: Set Timeout to `300000` ms.
   - Configure Authentication: Select `Generic Credential Type` -> `Header Auth`. Set up credentials using header name `Authorization` and value `Bearer <YOUR_APIFY_TOKEN>`.
   - Configure Body: Specify Body as `JSON` and insert the expression:
     ```javascript
     ={{ JSON.stringify({ domains: $json.domains.split(',').map(d => d.trim()).filter(Boolean), expiryWarningDays: $json.expiryWarningDays, trackChanges: true, onlyChanges: true }) }}
     ```
   - Rename the node to `Check domains on Apify`.

4. **Create the Data Processing Code Node:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Connect `Check domains on Apify`'s output to this node's input.
   - Set Mode to `Run Once for All Items`.
   - Paste the following script into the JavaScript editor:
     ```javascript
     const rows = $input.all().map((i) => i.json).filter((r) => r && r.domain);
     if (rows.length === 0) return [];

     const day = (s) => (s ? String(s).slice(0, 10) : "unknown");
     const rank = { critical: 0, warning: 1, ok: 2 };
     rows.sort((a, b) => (rank[a.severity] ?? 3) - (rank[b.severity] ?? 3) || (a.daysUntilExpiry ?? 1e9) - (b.daysUntilExpiry ?? 1e9));

     const line = (r) => {
       let head;
       if (r.lookupFailed) head = "LOOKUP FAILED";
       else if (!r.found) head = "NOT REGISTERED";
       else if (r.isExpired) head = `EXPIRED ${Math.abs(r.daysUntilExpiry)} days ago`;
       else if (r.isExpiringSoon) head = `expires in ${r.daysUntilExpiry} days (${day(r.expiryDate)})`;
       else head = `expires ${day(r.expiryDate)}`;
       const parts = [`${r.domain}: ${head}`];
       if (r.changeSummary) parts.push(`  changed: ${r.changeSummary}`);
       if (r.registrar) parts.push(`  registrar: ${r.registrar}`);
       if (r.lookupFailed && r.issues?.[0]) parts.push(`  ${r.issues[0]}`);
       return parts.join("\n");
     };

     const urgent = rows.filter((r) => r.isExpired || r.isExpiringSoon || !r.found || r.lookupFailed).length;
     const changed = rows.filter((r) => r.changed).length;
     const subject = urgent
       ? `Domain alert: ${urgent} domain(s) need attention`
       : `Domain watch: ${changed} domain(s) changed`;

     return [{
       json: {
         subject,
         text: [
           subject + ".",
           "",
           ...rows.map(line),
           "",
           "Checked by the Domain Expiry & WHOIS Monitor on Apify. Only changes and alerts are listed; unchanged, healthy domains are left out.",
         ].join("\n"),
         count: rows.length,
       },
     }];
     ```
   - Rename the node to `Build alert (stops if nothing changed)`.

5. **Create the Email Dispatch Node:**
   - Add a **Send Email** node (`n8n-nodes-base.emailSend`).
   - Connect `Build alert (stops if nothing changed)`'s output to this node's input.
   - Configure Credentials: Set up an **SMTP** account credential with your mail server host, port, user, and password settings.
   - Configure Parameters:
     - `Email Format`: `Text`
     - `From Email`: Set to your sending email address (e.g., `user@example.com`).
     - `To Email`: Use expression `={{ $('Your domains').first().json.alertEmail }}`.
     - `Subject`: Use expression `={{ $json.subject }}`.
     - `Text`: Use expression `={{ $json.text }}`.
   - Rename the node to `Email me`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Apify Domain Expiry & WHOIS Monitor Actor | External Apify actor used to track domain registry records via RDAP/WHOIS. |
| Cost Consideration | Costs approximately $0.0025 per domain per execution run; failed lookups are free of charge. |
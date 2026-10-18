Audit client websites weekly with Apify, Google Sheets, and email summaries

https://n8nworkflows.xyz/workflows/audit-client-websites-weekly-with-apify--google-sheets--and-email-summaries-20869


# Audit client websites weekly with Apify, Google Sheets, and email summaries

### 1. Workflow Overview

This workflow automates a weekly website auditing process for a defined list of websites. It executes every Monday, evaluates multiple performance and security factors via Apify, records the structured findings in Google Sheets for historical tracking, and delivers an aggregated HTML summary via email highlighting priority fixes, starting with the lowest-scoring site.

The logic is divided into the following functional blocks:
- **1.1 Trigger & Configuration:** Initiates the workflow on a weekly schedule and provides configuration parameters such as target URLs, agency branding, and email recipients.
- **1.2 Execution & Transformation:** Triggers the external Apify auditing actor, waits for results, and flattens/maps the raw JSON output into clean, structured data rows.
- **1.3 Data Logging & Distribution:** Splits the transformed data concurrently to log scores into Google Sheets and compile an HTML email summary ordered by lowest performance scores first.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Configuration
- **Overview:** Initializes the workflow execution on a scheduled basis and defines the runtime variables, including target website URLs, agency branding name, and notification email addresses.
- **Nodes Involved:** `Every Monday at 6am`, `Client sites and report settings`

##### Node Details:
- **Every Monday at 6am**
  - *Type and Technical Role:* `n8n-nodes-base.scheduleTrigger` (Trigger Node)
  - *Configuration Choices:* Configured to trigger weekly on Mondays at 06:00 AM.
  - *Key Expressions or Variables:* None.
  - *Input Connections:* None (Entry point).
  - *Output Connections:* `Client sites and report settings`
  - *Edge Cases / Potential Failures:* None typical for scheduled triggers.

- **Client sites and report settings**
  - *Type and Technical Role:* `n8n-nodes-base.set` (Data Transformation / Initialization Node)
  - *Configuration Choices:* Sets fixed JSON assignments containing an array of target URLs, agency name, recipient email, and sender email.
  - *Key Expressions or Variables:* 
    - `urls`: `=[\"https://www.example.com/\", \"https://www.example.org/\"]`
    - `agencyName`: `"Your Agency"`
    - `emailTo`: `"you@example.com"`
    - `emailFrom`: `"user@example.com"`
  - *Input Connections:* `Every Monday at 6am`
  - *Output Connections:* `Audit every site`
  - *Edge Cases / Potential Failures:* Invalid JSON array syntax inside assignments.

---

#### 1.2 Execution & Transformation
- **Overview:** Executes the website audit actor on Apify using the provided configuration, then transforms the raw payload into structured data models for downstream consumption.
- **Nodes Involved:** `Audit every site`, `One row per site`

##### Node Details:
- **Audit every site**
  - *Type and Technical Role:* `@apify/n8n-nodes-apify.apify` (External API Integration Node)
  - *Configuration Choices:* Runs the `pewterdesk/website-audit-report` actor, allocates 512MB memory, enforces a maximum spend cap of $1 USD per run, and passes dynamic custom body arguments.
  - *Key Expressions or Variables:* 
    - Custom Body: `={{ JSON.stringify({ urls: $json.urls, agencyName: $json.agencyName, maxPagesPerSite: 10, checkBrokenLinks: true, includeEmailDomainChecks: true }) }}`
  - *Input Connections:* `Client sites and report settings`
  - *Output Connections:* `One row per site`
  - *Credential Requirements:* Apify Account API token.
  - *Edge Cases / Potential Failures:* Apify API rate limits, insufficient account credits, exceeding the $1 run cap, or target websites blocking automated audit crawlers.

- **One row per site**
  - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Data Transformation Node)
  - *Configuration Choices:* Processes input items from Apify and extracts relevant metrics (scores, issues, grades, report URLs) into a clean, flat object structure.
  - *Key Expressions or Variables:* JavaScript mapping function parsing `r.scores?.seo`, `r.issueCount`, `r.topFixesText`, etc.
  - *Input Connections:* `Audit every site`
  - *Output Connections:* `Log scores to Google Sheets`, `Build email with top fixes`
  - *Edge Cases / Potential Failures:* Missing nested properties within the Apify response payload resulting in `undefined` values if optional chaining fails.

---

#### 1.3 Data Logging & Distribution
- **Overview:** Persists audit metrics row-by-row into a Google Sheet while simultaneously compiling and sending an email report prioritizing low-scoring websites.
- **Nodes Involved:** `Log scores to Google Sheets`, `Build email with top fixes`, `Email the summary`

##### Node Details:
- **Log scores to Google Sheets**
  - *Type and Technical Role:* `n8n-nodes-base.googleSheets` (Spreadsheet Integration Node)
  - *Configuration Choices:* Appends incoming rows into a sheet named `Audit log` using automatic data mapping.
  - *Input Connections:* `One row per site`
  - *Output Connections:* None (Terminal node for this branch).
  - *Credential Requirements:* Google Sheets OAuth2 Account.
  - *Edge Cases / Potential Failures:* Missing `Audit log` tab in the destination spreadsheet, authentication token expiration, or schema mismatch.

- **Build email with top fixes**
  - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Aggregation Node)
  - *Configuration Choices:* Runs once for all items, pulls data across node scopes (`$('Audit every site')`), separates successful audits from failed ones, sorts successful sites by lowest overall score first, and constructs an HTML-formatted email body.
  - *Input Connections:* `One row per site`
  - *Output Connections:* `Email the summary`
  - *Edge Cases / Potential Failures:* Malformed URLs or missing properties when parsing hostnames from failed or successful sites.

- **Email the summary**
  - *Type and Technical Role:* `n8n-nodes-base.emailSend` (Email Dispatch Node)
  - *Configuration Choices:* Sends an HTML email using parameters derived from the preceding code node and the initial settings node.
  - *Key Expressions or Variables:*
    - Subject: `={{ $json.subject }}`
    - HTML Body: `={{ $json.html }}`
    - To Email: `={{ $('Client sites and report settings').first().json.emailTo }}`
    - From Email: `={{ $('Client sites and report settings').first().json.emailFrom }}`
  - *Input Connections:* `Build email with top fixes`
  - *Output Connections:* None (Terminal node).
  - *Credential Requirements:* SMTP account (or compatible email relay).
  - *Edge Cases / Potential Failures:* SMTP authentication failures, incorrect sender/recipient configurations, or email delivery rejections.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **About this workflow** | `n8n-nodes-base.stickyNote` | Documentation note | None | None | ## Weekly website audit: scores to Google Sheets, top fixes by email<br><br>Every Monday this workflow audits a list of client (or your own) websites, logs each site's scores to Google Sheets so you can track them week over week, and emails you one summary with each site's top 3 fixes and a link to its white-label HTML report.<br><br>**What you get per site:** a 0–100 overall score and grade, scores for SEO, performance, security, accessibility and email/domain health, issue counts, a prioritized fix list, and an HTML report branded with your agency name.<br><br>**How it works**<br>1. Schedule trigger fires weekly.<br>2. You list the sites, your agency name and where to send the email.<br>3. The Apify Actor audits each site (homepage plus sampled sitemap pages).<br>4. One row per site goes to Sheets; one email lists the sites lowest score first.<br><br>**Actor:** [Website Audit Report: SEO, Speed, Security & Email Score](https://apify.com/pewterdesk/website-audit-report) on the Apify Store. See its Store page for current pricing; the Apify node caps spend at $1 per run. |
| **Setup** | `n8n-nodes-base.stickyNote` | Configuration instructions note | None | None | ## Setup (about 5 minutes)<br>1. **Apify:** create a free account, copy your API token from Console → Settings → Integrations, and add it as an Apify credential.<br>2. **Edit "Client sites and report settings":** site URLs, your agency name (shown on the HTML reports), and the email from/to addresses.<br>3. **Google Sheets:** paste your sheet's URL and create a tab named `Audit log`.<br>4. **Email:** add an SMTP credential (or swap the node for Gmail/Outlook).<br>5. **Optional:** add `googleApiKey` (free PageSpeed Insights key) to the JSON in the Apify node for Lighthouse scores and real-user Core Web Vitals. |
| **Every Monday at 6am** | `n8n-nodes-base.scheduleTrigger` | Schedule trigger | None | Client sites and report settings | |
| **Client sites and report settings** | `n8n-nodes-base.set` | Define variables | Every Monday at 6am | Audit every site | |
| **Audit every site** | `@apify/n8n-nodes-apify.apify` | Run Apify actor | Client sites and report settings | One row per site | |
| **One row per site** | `n8n-nodes-base.code` | Transform audit results | Audit every site | Log scores to Google Sheets, Build email with top fixes | |
| **Log scores to Google Sheets** | `n8n-nodes-base.googleSheets` | Append to spreadsheet | One row per site | None | |
| **Build email with top fixes** | `n8n-nodes-base.code` | Format email payload | One row per site | Email the summary | |
| **Email the summary** | `n8n-nodes-base.emailSend` | Send email notification | Build email with top fixes | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Configure the trigger rule to fire weekly on Mondays at 06:00 AM.
   - Name the node: `Every Monday at 6am`.

2. **Create the Settings Node:**
   - Add a **Set** node (`n8n-nodes-base.set`).
   - Connect the output of `Every Monday at 6am` to this node.
   - Configure assignments as:
     - `urls` (Type: Array): `["https://www.example.com/", "https://www.example.org/"]`
     - `agencyName` (Type: String): `Your Agency`
     - `emailTo` (Type: String): `you@example.com`
     - `emailFrom` (Type: String): `user@example.com`
   - Name the node: `Client sites and report settings`.

3. **Create the Apify Integration Node:**
   - Add an **Apify** node (`@apify/n8n-nodes-apify.apify`).
   - Connect `Client sites and report settings` to this node.
   - Set Operation to `Run actor and get dataset`.
   - Set Actor ID to `pewterdesk~website-audit-report`.
   - Set Memory to `512` and Max Total Charge to `1`.
   - Configure custom JSON body parameters:
     ```json
     ={{ JSON.stringify({ urls: $json.urls, agencyName: $json.agencyName, maxPagesPerSite: 10, checkBrokenLinks: true, includeEmailDomainChecks: true }) }}
     ```
   - Configure credentials for your Apify account.
   - Name the node: `Audit every site`.

4. **Create the Data Transformation Node:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Connect `Audit every site` to this node.
   - Paste the following JavaScript code:
     ```javascript
     return $input.all().map(({ json: r }) => ({ json: {
       auditedAt: r.auditedAt,
       site: r.finalUrl || r.url,
       overallScore: r.overallScore,
       grade: r.grade,
       seo: r.scores?.seo ?? null,
       performance: r.scores?.performance ?? null,
       security: r.scores?.security ?? null,
       accessibility: r.scores?.accessibility ?? null,
       emailDomain: r.scores?.emailDomain ?? null,
       issues: r.issueCount ?? null,
       highOrCritical: r.criticalOrHighIssues ?? null,
       topFixes: r.topFixesText || '',
       report: r.reportUrl || '',
       error: r.error || '',
     }}));
     ```
   - Name the node: `One row per site`.

5. **Create the Google Sheets Logging Node:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`).
   - Connect `One row per site` to this node.
   - Set Operation to `Append`.
   - Specify the Document ID (URL of your target sheet) and set Sheet Name to `Audit log`.
   - Set Mapping Mode to `Auto map input data`.
   - Configure Google Sheets OAuth2 credentials.
   - Name the node: `Log scores to Google Sheets`.

6. **Create the Email Builder Node:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Connect `One row per site` to this node.
   - Set mode to `Run Once for All Items`.
   - Paste the following JavaScript code:
     ```javascript
     const esc = s => String(s ?? '').replace(/[&<>\"]/g, c => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', '\"': '&quot;' }[c]));
     const sites = $('Audit every site').all().map(i => i.json);
     const ok = sites.filter(s => !s.error).sort((a, b) => (a.overallScore ?? 101) - (b.overallScore ?? 101));
     const failed = sites.filter(s => s.error);
     const block = s => {
       const fixes = (s.topFixes || []).slice(0, 3).map(f => `<li><b>[${esc(f.severity)}] ${esc(f.issue)}</b><br>${esc(f.fix)}</li>`).join('');
       return `<h3>${esc(s.finalUrl)}: ${esc(s.overallScore)}/100 (grade ${esc(s.grade)})</h3>\n<p>SEO ${esc(s.scores?.seo ?? 'n/a')} · Performance ${esc(s.scores?.performance ?? 'n/a')} · Security ${esc(s.scores?.security ?? 'n/a')} · Accessibility ${esc(s.scores?.accessibility ?? 'n/a')} · Email &amp; domain ${esc(s.scores?.emailDomain ?? 'n/a')}<br>\n${esc(s.issueCount)} issues, ${esc(s.criticalOrHighIssues)} high or critical.${s.reportUrl ? ` <a href=\"${esc(s.reportUrl)}\">Full HTML report</a>` : ''}</p>\n${fixes ? `<ol>${fixes}</ol>` : '<p>No significant issues found by the automated checks.</p>'}`;
     };
     const html = `<p>Weekly audit of ${sites.length} site(s), lowest score first.</p>${ok.map(block).join('')}`
       + (failed.length ? `<h3>Could not audit</h3><ul>${failed.map(s => `<li>${esc(s.url)}: ${esc(s.error)}</li>`).join('')}</ul>` : '');
     const worst = ok[0];
     const subject = `Weekly site audit: ${ok.length} audited` + (worst ? `, lowest ${worst.overallScore}/100 (${new URL(worst.finalUrl).hostname})` : '') + (failed.length ? `, ${failed.length} failed` : '');
     return [{ json: { subject, html } }];
     ```
   - Name the node: `Build email with top fixes`.

7. **Create the Email Dispatch Node:**
   - Add a **Send Email** node (`n8n-nodes-base.emailSend`).
   - Connect `Build email with top fixes` to this node.
   - Configure parameters:
     - **Email Format:** `HTML`
     - **Subject:** `={{ $json.subject }}`
     - **HTML:** `={{ $json.html }}`
     - **To Email:** `={{ $('Client sites and report settings').first().json.emailTo }}`
     - **From Email:** `={{ $('Client sites and report settings').first().json.emailFrom }}`
   - Configure SMTP credentials.
   - Name the node: `Email the summary`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Website Audit Report Actor on Apify Store | [Website Audit Report Actor](https://apify.com/pewterdesk/website-audit-report) |
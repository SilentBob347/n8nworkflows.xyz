Back up Cloudflare DNS zones to Amazon S3 with retention and email reports

https://n8nworkflows.xyz/workflows/back-up-cloudflare-dns-zones-to-amazon-s3-with-retention-and-email-reports-20916


# Back up Cloudflare DNS zones to Amazon S3 with retention and email reports

### 1. Workflow Overview

This workflow automates the daily backup of Cloudflare DNS zones into Amazon S3 storage using standard BIND zone file formats. It provides built-in retention management to clean up outdated backup files while preserving a minimum number of recent copies, and concludes by sending a comprehensive plain-text email report summarizing successes, failures, and deleted backups.

The workflow is organized into three sequential logical blocks:
- **1.1 Input Reception & Validation:** Triggers daily on a cron schedule, initializes user configurations, and validates inputs against missing parameters or placeholders.
- **1.2 Zone Iteration & S3 Backup:** Queries Cloudflare for accessible zones, loops through them individually to export BIND zone files, verifies record counts against Cloudflare's API, uploads files to Amazon S3, and catalogs the outcomes.
- **1.3 Retention Cleanup & Email Reporting:** Evaluates existing backups against retention policies, deletes expired files, aggregates run statistics, and emails a detailed report via SMTP.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
- **Overview:** Initializes the backup job on a schedule and ensures that all user-defined configuration settings (such as S3 parameters, retention days, and notification emails) are valid before performing any external requests.
- **Nodes Involved:**
  - `Every day at 03:00` (Schedule Trigger)
  - `Backup settings` (Set)
  - `Check your settings` (Code)
  - `List your zones` (HTTP Request)
  - `Choose the zones` (Code)

- **Node Details:**
  - **Every day at 03:00**
    - *Type and Technical Role:* Schedule Trigger node that starts the execution daily at 03:00 UTC using a cron expression (`0 3 * * *`).
    - *Configuration choices:* Cron expression configured for daily execution.
    - *Input/Output:* No inputs; outputs execution start timestamp to `Backup settings`.
    - *Failure types:* None.
  - **Backup settings**
    - *Type and Technical Role:* Set node used to define global execution parameters.
    - *Configuration choices:* Sets `zones` (string), `s3_bucket` (string), `s3_prefix` (string), `retention_days` (number), `keep_at_least` (number), `notify_email` (string), and `notify_from` (string).
    - *Input/Output:* Input from trigger; outputs configuration variables to `Check your settings`.
    - *Failure types:* Missing data types.
  - **Check your settings**
    - *Type and Technical Role:* Code node performing defensive validation ("fail fast") to catch placeholder values (e.g., `you@example.com`, `your-backup-bucket`) or invalid numeric retention ranges.
    - *Key expressions or variables:* Validates `$input.first().json` properties. Throws a formatted error if conditions fail.
    - *Input/Output:* Input from `Backup settings`; outputs validated settings to `List your zones`.
    - *Failure types:* Throws early errors if settings contain default placeholders or violate constraints.
  - **List your zones**
    - *Type and Technical Role:* HTTP Request node fetching all DNS zones visible to the configured Cloudflare token, utilizing automatic pagination.
    - *Configuration choices:* Calls `https://api.cloudflare.com/client/v4/zones` with pagination parameter `page` and query parameter `per_page=50`. Uses predefined Cloudflare API credentials.
    - *Input/Output:* Input from `Check your settings`; outputs paginated responses to `Choose the zones`.
    - *Version-specific requirements:* Uses n8n HTTP Request v4.2 pagination settings.
    - *Failure types:* API authentication failure, rate limiting, network timeout. Handled via `continueRegularOutput` with up to 3 retries.
  - **Choose the zones**
    - *Type and Technical Role:* Code node filtering the retrieved zone list against the optional comma-separated `zones` setting, or reporting errors if the token cannot access any zones.
    - *Key expressions or variables:* Reads `$input.all()` and compares against `zones` criteria from `Backup settings`.
    - *Input/Output:* Input from `List your zones`; outputs individual zone objects to `One zone at a time` and `Plan the clean-up`.
    - *Failure types:* Parsing errors on malformed API responses.

#### 2.2 Zone Iteration & S3 Backup
- **Overview:** Iterates through the discovered zones one by one, exports their BIND zone files, checks record validity against the Cloudflare API, uploads the files to Amazon S3, and catalogs successful backups.
- **Nodes Involved:**
  - `One zone at a time` (Split In Batches)
  - `Can this zone be backed up?` (If)
  - `Export the zone file` (HTTP Request)
  - `Count records in Cloudflare` (HTTP Request)
  - `Check the zone file` (Code)
  - `Upload to S3` (AWS S3)
  - `List this zone's backups` (AWS S3)
  - `Zone backed up` (Code)
  - `Zone not backed up` (Code)
  - `Record result` (Code)

- **Node Details:**
  - **One zone at a time**
    - *Type and Technical Role:* Split In Batches node that processes zones sequentially to isolate failures.
    - *Input/Output:* Input from `Choose the zones`; outputs one zone object per iteration to `Plan the clean-up` (on loop initiation) and `Can this zone be backed up?`.
  - **Can this zone be backed up?**
    - *Type and Technical Role:* If node verifying whether a valid `zone_id` exists for the current iteration.
    - *Configuration choices:* Checks if `{{ $json.zone_id }}` is not empty.
    - *Input/Output:* Input from `One zone at a time`; true branch leads to `Export the zone file`, false branch leads to `Zone not backed up`.
  - **Export the zone file**
    - *Type and Technical Role:* HTTP Request node exporting the DNS zone in BIND format.
    - *Configuration choices:* Calls `https://api.cloudflare.com/client/v4/zones/{{ $json.zone_id }}/dns_records/export`. Response format configured as text (`zone_file`). Uses Cloudflare API credentials.
    - *Input/Output:* Input from `Can this zone be backed up?` (true branch); outputs binary text data to `Count records in Cloudflare` or `Zone not backed up` on error (`continueErrorOutput`).
    - *Failure types:* Cloudflare API errors, timeouts. Handled via retries.
  - **Count records in Cloudflare**
    - *Type and Technical Role:* HTTP Request node querying the active DNS record count for validation purposes.
    - *Configuration choices:* Calls `https://api.cloudflare.com/client/v4/zones/{{ $('One zone at a time').first().json.zone_id }}/dns_records` with query parameter `per_page=1`. Uses Cloudflare API credentials.
    - *Input/Output:* Input from `Export the zone file`; outputs metadata containing `total_count` to `Check the zone file` or `Zone not backed up` on error.
    - *Failure types:* API request failures.
  - **Check the zone file**
    - *Type and Technical Role:* Code node analyzing the exported BIND file, stripping comments safely (avoiding false positives on quoted TXT records), adjusting for Cloudflare-injected SOA/NS records, and generating binary file data for S3 upload.
    - *Key expressions or variables:* Constructs S3 object key using `s3_prefix`, `zone_name`, and current UTC date (`YYYY-MM-DD.zone`).
    - *Input/Output:* Input from `Count records in Cloudflare`; outputs zone metadata and binary file data to `Upload to S3` or `Zone not backed up` on validation failure.
    - *Failure types:* Empty export files or malformed zone structures trigger errors.
  - **Upload to S3**
    - *Type and Technical Role:* AWS S3 node uploading the generated zone file to the target storage bucket.
    - *Configuration choices:* Operation: `upload`. Bucket name bound to `{{ $('Backup settings').first().json.s3_bucket }}`. File name bound to `{{ $json.key }}`. Uses AWS credentials.
    - *Input/Output:* Input from `Check the zone file`; outputs upload confirmation to `List this zone's backups` or `Zone not backed up` on error.
    - *Failure types:* Missing bucket permissions, incorrect region, or network failures.
  - **List this zone's backups**
    - *Type and Technical Role:* AWS S3 node listing all existing backup objects for the specific zone to evaluate retention policies.
    - *Configuration choices:* Operation: `getAll`. Folder key set to `{{ $('Backup settings').first().json.s3_prefix }}/{{ $('One zone at a time').first().json.zone_name }}/`. Uses AWS credentials.
    - *Input/Output:* Input from `Upload to S3`; outputs file lists to `Zone backed up`.
    - *Failure types:* Missing `s3:ListBucket` permissions. Handled via `continueRegularOutput`.
  - **Zone backed up**
    - *Type and Technical Role:* Code node formatting a unified success object containing file keys, record metrics, and existing backup lists.
    - *Input/Output:* Input from `List this zone's backups`; outputs success state to `Record result`.
  - **Zone not backed up**
    - *Type and Technical Role:* Code node intercepting failure states from various upstream nodes, translating error descriptions, and standardizing error output objects.
    - *Input/Output:* Receives error flows from multiple upstream nodes; outputs structured failure records to `Record result`.
  - **Record result**
    - *Type and Technical Role:* Code node acting as a collection point for each loop iteration, storing zone status metrics for final reporting.
    - *Configuration choices:* `alwaysOutputData: true`.
    - *Input/Output:* Input from `Zone backed up` and `Zone not backed up`; loops back to `One zone at a time` to process the next zone, or flows onward once complete.

#### 1.3 Retention Cleanup & Email Reporting
- **Overview:** Reviews all backups cataloged across the loop iterations, identifies files exceeding the retention window (while preserving the minimum required copies and protecting zones that failed during the current run), deletes expired files from S3, compiles a report, and sends it via SMTP.
- **Nodes Involved:**
  - `Plan the clean-up` (Code)
  - `Anything to delete?` (If)
  - `Delete expired backup` (AWS S3)
  - `Build the report` (Code)
  - `Email the report` (Email Send)

- **Node Details:**
  - **Plan the clean-up**
    - *Type and Technical Role:* Code node calculating retention expirations across all completed zone iterations.
    - *Configuration choices:* Enforces rules: ignores foreign S3 objects outside the naming convention, preserves the newest `keep_at_least` backups regardless of age, and skips cleanup for any zone that failed during the current run.
    - *Input/Output:* Input reads iteration results from `Record result`; outputs expiration items or a fallback `nothingExpired` object to `Anything to delete?`.
  - **Anything to delete?**
    - *Type and Technical Role:* If node checking if any files were marked for deletion.
    - *Configuration choices:* Evaluates `{{ $json.key }}`.
    - *Input/Output:* Input from `Plan the clean-up`; true branch routes to `Delete expired backup`, false branch routes directly to `Build the report`.
  - **Delete expired backup**
    - *Type and Technical Role:* AWS S3 node deleting expired backup files.
    - *Configuration choices:* Operation: `delete`. Bucket name bound to `s3_bucket`, file key bound to `{{ $json.key }}`. Uses AWS credentials.
    - *Input/Output:* Input from `Anything to delete?`; outputs deletion results to `Build the report`.
    - *Failure types:* Missing `s3:DeleteObject` permissions. Handled via `continueRegularOutput`.
  - **Build the report**
    - *Type and Technical Role:* Code node aggregating statistics from all loop iterations (successes, failures, mismatches, and deletions) and assembling a plain-text email body and subject line.
    - *Input/Output:* Input from `Delete expired backup` or `Anything to delete?`; outputs report payload (`subject`, `body`, metrics) to `Email the report`.
  - **Email the report**
    - *Type and Technical Role:* Email Send node dispatching the final execution report via SMTP.
    - *Configuration choices:* Email format set to text. Recipient bound to `notify_email`, sender bound to `notify_from`. Subject and body bound to incoming data properties. Uses SMTP credentials.
    - *Input/Output:* Input from `Build the report`; terminates workflow execution.
    - *Failure types:* SMTP authentication errors, invalid sender/recipient addresses, or relay refusal.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| About this template | n8n-nodes-base.stickyNote | Documentation note | None | None | ## Back up Cloudflare DNS zones to Amazon S3... |
| Section: settings and zones | n8n-nodes-base.stickyNote | Section divider note | None | None | ## 1. Settings and zones... |
| Section: back up each zone | n8n-nodes-base.stickyNote | Section divider note | None | None | ## 2. Back up each zone... |
| Section: retention and report | n8n-nodes-base.stickyNote | Section divider note | None | None | ## 3. Retention and report... |
| Every day at 03:00 | n8n-nodes-base.scheduleTrigger | Starts workflow daily | None | Backup settings | |
| Backup settings | n8n-nodes-base.set | Defines global configuration variables | Every day at 03:00 | Check your settings | |
| Check your settings | n8n-nodes-base.code | Validates configuration parameters | Backup settings | List your zones | |
| List your zones | n8n-nodes-base.httpRequest | Fetches Cloudflare zones | Check your settings | Choose the zones | A refused zone list is passed on as data rather than failing the run, so the report can say why. |
| Choose the zones | n8n-nodes-base.code | Filters zones based on settings | List your zones | One zone at a time, Plan the clean-up | |
| One zone at a time | n8n-nodes-base.splitInBatches | Iterates zones sequentially | Choose the zones, Record result | Plan the clean-up, Can this zone be backed up? | |
| Can this zone be backed up? | n8n-nodes-base.if | Validates zone ID presence | One zone at a time | Export the zone file, Zone not backed up | |
| Export the zone file | n8n-nodes-base.httpRequest | Exports zone BIND file | Can this zone be backed up? | Count records in Cloudflare, Zone not backed up | |
| Count records in Cloudflare | n8n-nodes-base.httpRequest | Queries Cloudflare record count | Export the zone file | Check the zone file, Zone not backed up | |
| Check the zone file | n8n-nodes-base.code | Analyzes and packages zone file | Count records in Cloudflare | Upload to S3, Zone not backed up | |
| Upload to S3 | n8n-nodes-base.awsS3 | Uploads zone file to S3 | Check the zone file | List this zone's backups, Zone not backed up | |
| List this zone's backups | n8n-nodes-base.awsS3 | Lists existing zone backups in S3 | Upload to S3 | Zone backed up | |
| Zone backed up | n8n-nodes-base.code | Formats successful backup result | List this zone's backups | Record result | |
| Zone not backed up | n8n-nodes-base.code | Formats failure and error state | Check the zone file, Count records in Cloudflare, Export the zone file, Upload to S3, Can this zone be backed up? | Record result | |
| Record result | n8n-nodes-base.code | Collects iteration results | Zone backed up, Zone not backed up | One zone at a time | |
| Plan the clean-up | n8n-nodes-base.code | Determines expired backups to delete | Choose the zones, Record result | Anything to delete? | |
| Anything to delete? | n8n-nodes-base.if | Checks for files to delete | Plan the clean-up | Delete expired backup, Build the report | |
| Delete expired backup | n8n-nodes-base.awsS3 | Deletes expired backup from S3 | Anything to delete? | Build the report | |
| Build the report | n8n-nodes-base.code | Compiles email summary report | Delete expired backup, Anything to delete? | Email the report | |
| Email the report | n8n-nodes-base.emailSend | Sends report via SMTP | Build the report | None | |
| Warning: deletes files | n8n-nodes-base.stickyNote | Warning notice on file deletion | None | None | ### Deletes files... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node named `Every day at 03:00`.
   - Set trigger interval to Cron Expression: `0 3 * * *`.

2. **Configure Global Settings:**
   - Add a **Set** node named `Backup settings`.
   - Configure assignments:
     - `zones` (string): `""` (leave empty for all zones, or comma-separated names).
     - `s3_bucket` (string): `your-backup-bucket` (update with your bucket name).
     - `s3_prefix` (string): `cloudflare-dns`.
     - `retention_days` (number): `30`.
     - `keep_at_least` (number): `3`.
     - `notify_email` (string): `you@example.com` (update with recipient email).
     - `notify_from` (string): `""` (update with sender email).

3. **Add Validation Logic:**
   - Add a **Code** node named `Check your settings` and populate it with the validation JavaScript checking for placeholders and empty required fields. Connect `Backup settings` to it.

4. **Retrieve Cloudflare Zones:**
   - Add an **HTTP Request** node named `List your zones`.
   - Method: `GET`, URL: `https://api.cloudflare.com/client/v4/zones`.
   - Query parameters: `per_page = 50`.
   - Pagination: Configure custom pagination checking `response.body.result_info.page >= response.body.result_info.total_pages` with `page = $pageCount + 1`.
   - Credential: Configure **Cloudflare API** credential (`cloudflareApi`). Set `onError` to `continueRegularOutput`.
   - Add a **Code** node named `Choose the zones` to parse and filter the zone list. Connect `List your zones` to it.

5. **Set Up the Zone Loop:**
   - Add a **Split In Batches** node named `One zone at a time`. Connect `Choose the zones` and `Record result` to it.
   - Add an **If** node named `Can this zone be backed up?`. Condition: `{{ $json.zone_id }}` is not empty. Connect `One zone at a time` to it.

6. **Process Zone Export and Verification:**
   - Add an **HTTP Request** node named `Export the zone file`.
   - URL: `=https://api.cloudflare.com/client/v4/zones/{{ $json.zone_id }}/dns_records/export`.
   - Response Format: Text (`zone_file`). Credential: Cloudflare API. Set `onError` to `continueErrorOutput`.
   - Add an **HTTP Request** node named `Count records in Cloudflare`.
   - URL: `=https://api.cloudflare.com/client/v4/zones/{{ $('One zone at a time').first().json.zone_id }}/dns_records`.
   - Query parameters: `per_page = 1`. Credential: Cloudflare API. Set `onError` to `continueErrorOutput`.
   - Add a **Code** node named `Check the zone file` to parse the BIND file, count records, and generate binary data. Set `onError` to `continueErrorOutput`.

7. **Upload and Retention Storage Handling:**
   - Add an **AWS S3** node named `Upload to S3`.
   - Operation: `upload`, Bucket Name: `={{ $('Backup settings').first().json.s3_bucket }}`, File Name: `={{ $json.key }}`. Credential: AWS. Set `onError` to `continueErrorOutput`.
   - Add an **AWS S3** node named `List this zone's backups`.
   - Operation: `getAll`, Bucket Name: `={{ $('Backup settings').first().json.s3_bucket }}`, Folder Key: `={{ $('Backup settings').first().json.s3_prefix }}/{{ $('One zone at a time').first().json.zone_name }}/`. Credential: AWS. Set `onError` to `continueRegularOutput`.
   - Add a **Code** node named `Zone backed up` to format success data.
   - Add a **Code** node named `Zone not backed up` to catch failures. Connect error outputs from export, count, check, and upload nodes to this node.
   - Add a **Code** node named `Record result` with `alwaysOutputData: true` to aggregate loop iterations. Connect both `Zone backed up` and `Zone not backed up` here, looping back to `One zone at a time`.

8. **Retention Cleanup & Reporting Execution:**
   - Add a **Code** node named `Plan the clean-up`. Connect `Choose the zones` (or initial loop trigger) to it.
   - Add an **If** node named `Anything to delete?`. Condition: `{{ $json.key }}` is not empty.
   - Add an **AWS S3** node named `Delete expired backup`. Operation: `delete`, Bucket Name: `={{ $('Backup settings').first().json.s3_bucket }}`, File Key: `={{ $json.key }}`. Credential: AWS. Set `onError` to `continueRegularOutput`.
   - Add a **Code** node named `Build the report` to aggregate execution outcomes into subject and body strings.
   - Add an **Email Send** node named `Email the report`.
   - Configuration: Text format. Recipient: `={{ $('Backup settings').first().json.notify_email }}`, Sender: `={{ $('Backup settings').first().json.notify_from }}`, Subject: `={{ $json.subject }}`, Text: `={{ $json.body }}`. Credential: SMTP.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Full setup and restore guide | [datadrifter.io/cloudflare-dns-backup-restore/](https://datadrifter.io/cloudflare-dns-backup-restore/) |
| Project Creator | Made by DataDrifter |
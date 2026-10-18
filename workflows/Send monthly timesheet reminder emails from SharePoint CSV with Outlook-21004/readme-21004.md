Send monthly timesheet reminder emails from SharePoint CSV with Outlook

https://n8nworkflows.xyz/workflows/send-monthly-timesheet-reminder-emails-from-sharepoint-csv-with-outlook-21004


# Send monthly timesheet reminder emails from SharePoint CSV with Outlook

### 1. Workflow Overview

This workflow automates monthly time-tracking reminders by extracting employee data from a CSV file stored in Microsoft SharePoint, sending customized reminder emails via Microsoft Outlook, and subsequently clearing recipient tracking fields in the source file before uploading the updated version back to SharePoint.

The logic is divided into four functional blocks:
- **1.1 Trigger and Schedule Evaluation:** Initiates execution on the first three calendar days of each month and verifies whether the current date represents the first official business day (Monday through Friday) of that month.
- **1.2 Data Acquisition and Processing:** Downloads the employee tracking CSV from Microsoft SharePoint, parses its contents, and prepares recipient data while computing the previous month's label for localization.
- **1.3 Iterative Email Dispatch:** Loops through all valid employee records to dispatch personalized reminder emails with managers placed in the carbon copy (CC) field.
- **1.4 File Reset and Persistence:** Clears recipient columns in parallel to the email loop, converts the dataset back into binary CSV format, and replaces the file on Microsoft SharePoint.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Schedule Evaluation
- **Overview:** Initializes the execution schedule and applies a business-day guard clause to prevent premature or incorrect execution on weekends.
- **Nodes Involved:** `Schedule Trigger`, `First business day?`

##### Node Details
- **Schedule Trigger**
  - *Type & Role:* `n8n-nodes-base.scheduleTrigger` — Triggers execution based on a cron expression.
  - *Configuration Choices:* Executes at 07:00 AM on the 1st, 2nd, and 3rd days of every month (`0 7 1,2,3 * *`).
  - *Key Expressions:* Cron expression: `0 7 1,2,3 * *`
  - *Connections:* Output connects to `First business day?`.
  - *Version-specific Requirements:* Version 1.3.
  - *Edge Cases & Failures:* Timezone misconfigurations on the n8n host instance can cause jobs to fire at unintended local times.

- **First business day?**
  - *Type & Role:* `n8n-nodes-base.code` — Custom JavaScript logic validating whether the execution day is the first Monday–Friday of the month.
  - *Configuration Choices:* Evaluates `new Date()` against a dynamic check that skips Saturdays (6) and Sundays (0). Returns input items if true, or an empty array `[]` if false to halt downstream execution.
  - *Key Expressions:* JavaScript Date manipulation verifying `isFirstWorkday`.
  - *Connections:* Input from `Schedule Trigger`; output connects to `Download file`.
  - *Version-specific Requirements:* Version 2.
  - *Edge Cases & Failures:* Public holidays are not factored into the calculation and are treated as standard business days.

---

#### 2.2 Data Acquisition and Processing
- **Overview:** Downloads the remote SharePoint asset, translates the raw binary payload into structured tabular records, and shapes the recipient schema.
- **Nodes Involved:** `Download file`, `Read CSV`, `Employee List`

##### Node Details
- **Download file**
  - *Type & Role:* `n8n-nodes-base.microsoftSharePoint` — Downloads files from Microsoft SharePoint document libraries.
  - *Configuration Choices:* Configured via Microsoft SharePoint OAuth2 credentials targeting a selected site and file.
  - *Key Expressions:* None (uses static UI parameter selections).
  - *Connections:* Input from `First business day?`; output connects to `Read CSV`.
  - *Version-specific Requirements:* Version 1.
  - *Edge Cases & Failures:* SharePoint API rate limits, authentication token expiration, or missing target files throw connection/not-found errors.

- **Read CSV**
  - *Type & Role:* `n8n-nodes-base.extractFromFile` — Parses raw file binary data into structured JSON objects.
  - *Configuration Choices:* UTF-8 encoding, comma delimiter, Byte Order Mark (BOM) enabled, and header row mapping enabled.
  - *Key Expressions:* None.
  - *Connections:* Input from `Download file`; outputs connect to both `Employee List` and `Clear recipients in data.csv`.
  - *Version-specific Requirements:* Version 1.1.
  - *Edge Cases & Failures:* Malformed CSV rows or unexpected delimiters cause parsing anomalies or missing columns.

- **Employee List**
  - *Type & Role:* `n8n-nodes-base.code` — Transforms parsed CSV items into a sanitized, standardized recipient array and computes the previous month's name.
  - *Configuration Choices:* JavaScript map and filter routines isolating `MA` (name), `CROSS_MA.email` (employee email), and `CROSS_MA.VG_email` (manager email), while filtering out records lacking names or emails.
  - *Key Expressions:* Dynamic calculation of previous month names (`Januar`, `Februar`, etc.) combined with the preceding year.
  - *Connections:* Input from `Read CSV`; output connects to `Loop Over Employees`.
  - *Version-specific Requirements:* Version 2.
  - *Edge Cases & Failures:* Schema changes in source column headers (e.g., renaming `CROSS_MA.email`) break mapping outputs and result in empty recipient arrays.

---

#### 2.3 Iterative Email Dispatch
- **Overview:** Manages iteration over the sanitized employee array, sending individual email notifications via Outlook.
- **Nodes Involved:** `Loop Over Employees`, `Send Reminder Email`

##### Node Details
- **Loop Over Employees**
  - *Type & Role:* `n8n-nodes-base.splitInBatches` — Controls iterative execution loops over item collections.
  - *Configuration Choices:* Default batch processing configuration (batch size 1).
  - *Key Expressions:* None.
  - *Connections:* Input from `Employee List`. Output 1 (done) is empty; Output 2 (loop) connects to `Send Reminder Email`.
  - *Version-specific Requirements:* Version 3.
  - *Edge Cases & Failures:* Large datasets can lead to prolonged execution durations; ensure adequate n8n execution timeout configurations.

- **Send Reminder Email**
  - *Type & Role:* `n8n-nodes-base.microsoftOutlook` — Dispatches emails using a connected Microsoft Outlook mailbox.
  - *Configuration Choices:* Plain text body format, dynamic subject interpolation referencing the calculated month, and manager address assigned to CC recipients.
  - *Key Expressions:* 
    - Subject: `=Erinnerung: Zeitbuchung {{ $json.monat }}`
    - Body: `=Hi {{ $json.name }}, vermutlich ist es untergegangen, die Zeit für {{ $json.monat }} zu buchen. Bitte hol dies bis heute um 16:30 Uhr nach. Schöne Grüße, crossnative Controlling Team`
    - To: `={{ $json.email }}`
    - CC: `={{ $json.vgEmail }}`
  - *Connections:* Input from `Loop Over Employees` (loop output); output loops back to `Loop Over Employees` to fetch the next batch.
  - *Version-specific Requirements:* Version 2.
  - *Edge Cases & Failures:* Invalid email formats, Microsoft Graph API sending limits, or mailbox permission blocks cause delivery failures.

---

#### 2.4 File Reset and Persistence
- **Overview:** Clears tracking columns in the dataset read from SharePoint, serializes the updated data back into a file structure, and uploads the revision to SharePoint.
- **Nodes Involved:** `Clear recipients in data.csv`, `Convert to File`, `Upload file`

##### Node Details
- **Clear recipients in data.csv**
  - *Type & Role:* `n8n-nodes-base.set` — Modifies and assigns values to item properties.
  - *Configuration Choices:* Overwrites `MA`, `CROSS_MA.email`, and `CROSS_MA.VG_email` with empty assignment values (`=`).
  - *Key Expressions:* `=` (assigns empty/null string values to designated keys).
  - *Connections:* Input from `Read CSV`; output connects to `Convert to File`.
  - *Version-specific Requirements:* Version 3.4.
  - *Edge Cases & Failures:* Overwriting unintended properties if property keys match other columns in the dataset.

- **Convert to File**
  - *Type & Role:* `n8n-nodes-base.convertToFile` — Converts JSON item datasets into a binary file format.
  - *Configuration Choices:* File name set to `data`.
  - *Key Expressions:* None.
  - *Connections:* Input from `Clear recipients in data.csv`; output connects to `Upload file`.
  - *Version-specific Requirements:* Version 1.1.
  - *Edge Cases & Failures:* Empty datasets produce empty files, which may disrupt downstream SharePoint schema requirements.

- **Upload file**
  - *Type & Role:* `n8n-nodes-base.microsoftSharePoint` — Uploads binary files back to a Microsoft SharePoint destination.
  - *Configuration Choices:* Set to `upload` operation targeting `data.csv`, using `data` as the binary property source.
  - *Key Expressions:* None.
  - *Connections:* Input from `Convert to File`; terminal node of the reset branch.
  - *Version-specific Requirements:* Version 1.
  - *Edge Cases & Failures:* Write-permission restrictions on the SharePoint library or file-locking conflicts cause upload rejection.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | `n8n-nodes-base.scheduleTrigger` | Triggers workflow execution on specified calendar days | None | First business day? | Runs at 7:00 a.m. on the first three calendar days of each month.<br>Stops running if the current date is not the first business day (Mon–Fri) of the month. |
| First business day? | `n8n-nodes-base.code` | Evaluates if today is the first business day of the month | Schedule Trigger | Download file | Runs at 7:00 a.m. on the first three calendar days of each month.<br>Stops running if the current date is not the first business day (Mon–Fri) of the month. |
| Download file | `n8n-nodes-base.microsoftSharePoint` | Downloads employee CSV from SharePoint | First business day? | Read CSV | Imports a CSV file containing employee data from a SharePoint library and parses the file into individual records. |
| Read CSV | `n8n-nodes-base.extractFromFile` | Parses raw CSV file into structured JSON | Download file | Employee List, Clear recipients in data.csv | Imports a CSV file containing employee data from a SharePoint library and parses the file into individual records. |
| Employee List | `n8n-nodes-base.code` | Sanitizes records and calculates previous month label | Read CSV | Loop Over Employees | - Filters names and email addresses, and calculates the name of the previous month for the email text.<br>- Iterates through the filtered list and sends a reminder email to each employee, with a CC to the respective manager. |
| Loop Over Employees | `n8n-nodes-base.splitInBatches` | Iterates through employee records sequentially | Employee List | Send Reminder Email (Loop), None (Done) | - Filters names and email addresses, and calculates the name of the previous month for the email text.<br>- Iterates through the filtered list and sends a reminder email to each employee, with a CC to the respective manager. |
| Send Reminder Email | `n8n-nodes-base.microsoftOutlook` | Sends reminder email via Outlook with manager in CC | Loop Over Employees | Loop Over Employees | - Filters names and email addresses, and calculates the name of the previous month for the email text.<br>- Iterates through the filtered list and sends a reminder email to each employee, with a CC to the respective manager. |
| Clear recipients in data.csv | `n8n-nodes-base.set` | Clears recipient fields in the dataset | Read CSV | Convert to File | At the same time: clear the relevant columns in the source file and reload them so that they are updated for the next month. |
| Convert to File | `n8n-nodes-base.convertToFile` | Converts modified JSON records back to binary file format | Clear recipients in data.csv | Upload file | At the same time: clear the relevant columns in the source file and reload them so that they are updated for the next month. |
| Upload file | `n8n-nodes-base.microsoftSharePoint` | Uploads the updated CSV file back to SharePoint | Convert to File | None | At the same time: clear the relevant columns in the source file and reload them so that they are updated for the next month. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**
   - Add a **Schedule Trigger** node.
   - Set trigger interval rule to Cron Expression: `0 7 1,2,3 * *`.
2. **Create First Business Day Validation Node:**
   - Add a **Code** node named `First business day?`.
   - Connect `Schedule Trigger` output to this node.
   - Insert JavaScript snippet to evaluate whether the execution date matches the first Monday–Friday of the month, returning an empty array if false.
3. **Create SharePoint Download Node:**
   - Add a **Microsoft SharePoint** node named `Download file`.
   - Connect `First business day?` output to this node.
   - Configure **Microsoft SharePoint OAuth2 API** credentials, select the appropriate site, and locate the target employee CSV file.
4. **Create CSV Parsing Node:**
   - Add an **Extract From File** node named `Read CSV`.
   - Connect `Download file` binary output to this node.
   - Set options: Encoding `utf-8`, Delimiter `,`, Enable BOM `true`, Header Row `true`.
5. **Create Employee List Processing Node:**
   - Add a **Code** node named `Employee List`.
   - Connect `Read CSV` main output to this node.
   - Add code to map `MA`, `CROSS_MA.email`, and `CROSS_MA.VG_email` fields, filter out empty rows, and compute the previous month label string.
6. **Create Iteration Loop Node:**
   - Add a **Split In Batches** node named `Loop Over Employees`.
   - Connect `Employee List` output to this node.
7. **Create Outlook Email Node:**
   - Add a **Microsoft Outlook** node named `Send Reminder Email`.
   - Connect the loop output (second output port) of `Loop Over Employees` to this node.
   - Configure **Microsoft Outlook OAuth2 API** credentials.
   - Set Subject: `=Erinnerung: Zeitbuchung {{ $json.monat }}`.
   - Set Body Content: `=Hi {{ $json.name }},\n\nvermutlich ist es untergegangen, die Zeit für {{ $json.monat }} zu buchen. Bitte hol dies bis heute um 16:30 Uhr nach.\n\nSchöne Grüße,\ncrossnative Controlling Team`.
   - Set To Recipients: `={{ $json.email }}`.
   - Under Additional Fields, set CC Recipients: `={{ $json.vgEmail }}` and Body Content Type to `Text`.
   - Connect the output of `Send Reminder Email` back to the input of `Loop Over Employees` to sustain the loop.
8. **Create Data Reset Node:**
   - Add a **Set (Edit Fields)** node named `Clear recipients in data.csv`.
   - Connect the secondary output of `Read CSV` to this node.
   - Add assignments to clear `MA`, `CROSS_MA.email`, and `CROSS_MA.VG_email` by setting their string values to `=`.
9. **Create File Conversion Node:**
   - Add a **Convert to File** node named `Convert to File`.
   - Connect `Clear recipients in data.csv` output to this node.
   - Set destination file name option to `data`.
10. **Create SharePoint Upload Node:**
    - Add a **Microsoft SharePoint** node named `Upload file`.
    - Connect `Convert to File` output to this node.
    - Reuse the configured **Microsoft SharePoint OAuth2 API** credentials, select the target SharePoint site, set operation to `upload`, set file name to `data.csv`, and reference `data` as the binary property.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Send monthly timesheet reminder emails from a SharePoint CSV with Outlook overview and architecture notes. | Internal Workflow Documentation & Overview |
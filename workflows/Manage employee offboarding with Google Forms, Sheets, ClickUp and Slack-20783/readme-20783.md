Manage employee offboarding with Google Forms, Sheets, ClickUp and Slack

https://n8nworkflows.xyz/workflows/manage-employee-offboarding-with-google-forms--sheets--clickup-and-slack-20783


# Manage employee offboarding with Google Forms, Sheets, ClickUp and Slack

### 1. Workflow Overview

This workflow automates the employee offboarding lifecycle, spanning initial exit request submission, data lookup, task generation in project management software, team notifications, and periodic tracking with management escalations.

The logical blocks are organized as follows:
- **1.1 Input Reception & Date Validation:** Captures employee exit details via an n8n form and ensures the submitted last working date is valid.
- **1.2 Employee Data Lookup & Task Preparation:** Queries Google Sheets for the employee record and uses custom JavaScript logic to compile standard and department-specific IT and HR offboarding tasks.
- **1.3 Task Creation & Team Notification:** Splits the generated checklist into parallel branches to create ClickUp tasks, aggregates execution data, and posts kickoff alerts to respective IT and HR Slack channels.
- **1.4 Periodic Monitoring & Management Escalation:** Runs on a weekly schedule to audit ClickUp task completion status, processes task metrics via JavaScript, and notifies management via Slack if incomplete items remain.

---

### 2. Block-by-Block Analysis

---

### Block 1.1: Input Reception & Date Validation

#### Overview
This block serves as the workflow's primary entry point. It captures offboarding form submissions from users and verifies that the provided last working date is set to today or a future date before proceeding.

#### Nodes Involved
- `On form submission`
- `If`

#### Node Details

##### 1. On form submission
- **Type and Technical Role:** `n8n-nodes-base.formTrigger` (Trigger Node)
- **Configuration Choices:** Configured to expose an interactive web form collecting Employee ID, Employee Name, Department (Mobile, PHP, DotNet, AI/ML, FullStack, IT, HR), Manager Email, Last Working Date, Exit Type (Resignation, Layoff), and Reason.
- **Key Expressions or Variables:** None (Generates initial payload).
- **Input and Output Connections:** Trigger output connects to the `If` node.
- **Version-Specific Requirements:** Version 2.5.
- **Edge Cases / Potential Failure Types:** Form submissions lacking required fields will fail validation at the form level.

##### 2. If
- **Type and Technical Role:** `n8n-nodes-base.if` (Flow Control Node)
- **Configuration Choices:** Evaluates whether the last working date is greater than or equal to the current date using Luxon date parsing.
- **Key Expressions or Variables:** 
  - Left Value: `={{ DateTime.fromISO($json['Last Working Date']).startOf('day').toMillis() }}`
  - Right Value: `={{ $now.startOf('day').toMillis() }}`
- **Input and Output Connections:** Input from `On form submission`; true branch connects to `Get row(s) in sheet`.
- **Version-Specific Requirements:** Version 2.3.
- **Edge Cases / Potential Failure Types:** Invalid date strings will cause ISO parsing errors.

---

### Block 1.2: Employee Data Lookup & Task Preparation

#### Overview
This block retrieves the employee's master record from Google Sheets using their Employee ID and processes the metadata alongside department attributes to build a structured JSON checklist of IT and HR offboarding duties.

#### Nodes Involved
- `Get row(s) in sheet`
- `Code in JavaScript`

#### Node Details

##### 1. Get row(s) in sheet
- **Type and Technical Role:** `n8n-nodes-base.googleSheets` (Data Integration Node)
- **Configuration Choices:** Performs a row lookup filtering by Employee ID against the "Employee Master" sheet (`1YfxVptI6-eac9ty3lkWEoIJuEy2Zdj8chCJYQvV-Rvg`).
- **Key Expressions or Variables:** Lookup Value: `={{$json['Employee ID']}}` against column `=Employee ID`.
- **Input and Output Connections:** Input from `If`; output connects to `Code in JavaScript`.
- **Version-Specific Requirements:** Version 4.7. Requires active Google Sheets OAuth2 credentials.
- **Edge Cases / Potential Failure Types:** Failure occurs if the Employee ID does not exist in the target sheet (resulting in empty arrays) or due to API rate limits/auth expiration.

##### 2. Code in JavaScript
- **Type and Technical Role:** `n8n-nodes-base.code` (Data Transformation Node)
- **Configuration Choices:** Extracts employee details and constructs standardized offboarding arrays for IT and HR. Appends conditional tasks based on department values (e.g., Engineering items for GitLab/development environment access, Finance items for accounting system access).
- **Key Expressions or Variables:** Accesses incoming payload properties such as `$input.first().json`.
- **Input and Output Connections:** Input from `Get row(s) in sheet`; output splits into two parallel branches (`Split Out` and `Split Out1`).
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Errors will throw if input properties are null or missing expected schema keys.

---

### Block 1.3: Task Creation & Team Notification

#### Overview
This block splits the consolidated IT and HR task arrays into individual items, creates corresponding task cards in ClickUp in parallel, aggregates the results, and broadcasts offboarding initiation messages to dedicated IT and HR Slack channels.

#### Nodes Involved
- `Split Out`
- `Create a task`
- `Split Out1`
- `Create a task1`
- `Aggregate`
- `Send a message`
- `Send a message1`

#### Node Details

##### 1. Split Out
- **Type and Technical Role:** `n8n-nodes-base.splitOut` (Data Transformation Node)
- **Configuration Choices:** Splits items based on the `it_tasks` field array.
- **Key Expressions or Variables:** Field to split out: `=it_tasks`.
- **Input and Output Connections:** Input from `Code in JavaScript`; output connects to `Create a task`.
- **Version-Specific Requirements:** Version 1.

##### 2. Create a task
- **Type and Technical Role:** `n8n-nodes-base.clickUp` (Project Management Node)
- **Configuration Choices:** Creates tasks in ClickUp under team `90161771152`, space `90167968621`, and list `130107+1234567890`. Applies priority `1` and tags corresponding to the task department.
- **Key Expressions or Variables:** 
  - Task Name: `={{ $json.task_name }}`
  - Additional Fields Tags: `={{ [$json.department] }}`
  - Additional Fields Content: `=Employee Offboarding Task  \nTask: {{ $json.task_name }}\nPriority:{{ $json.priority }}\nDepartment: {{ $json.department }}`
- **Input and Output Connections:** Input from `Split Out`.
- **Version-Specific Requirements:** Version 1. Requires valid ClickUp API credentials.
- **Edge Cases / Potential Failure Types:** Invalid List IDs or insufficient workspace permissions will trigger API authorization errors.

##### 3. Split Out1
- **Type and Technical Role:** `n8n-nodes-base.splitOut` (Data Transformation Node)
- **Configuration Choices:** Splits items based on the `hr_tasks` field array.
- **Key Expressions or Variables:** Field to split out: `=hr_tasks`.
- **Input and Output Connections:** Input from `Code in JavaScript`; output connects to `Create a task1`.
- **Version-Specific Requirements:** Version 1.

##### 4. Create a task1
- **Type and Technical Role:** `n8n-nodes-base.clickUp` (Project Management Node)
- **Configuration Choices:** Creates HR offboarding tasks in ClickUp under the same project space configuration.
- **Key Expressions or Variables:** Identical dynamic mappings to `Create a task` referencing `task_name`, `priority`, and `department`.
- **Input and Output Connections:** Input from `Split Out1`; output connects to `Aggregate`.
- **Version-Specific Requirements:** Version 1.

##### 5. Aggregate
- **Type and Technical Role:** `n8n-nodes-base.aggregate` (Data Transformation Node)
- **Configuration Choices:** Collects and aggregates all item data from preceding branches.
- **Key Expressions or Variables:** Aggregate method set to `aggregateAllItemData`.
- **Input and Output Connections:** Input from `Create a task1`; output connects to `Send a message`.
- **Version-Specific Requirements:** Version 1.

##### 6. Send a message
- **Type and Technical Role:** `n8n-nodes-base.slack` (Notification Node)
- **Configuration Choices:** Posts an IT offboarding initiation notification to channel ID `C0C0ZL6TP27` (`n8n-it-offboarding`).
- **Key Expressions or Variables:** Uses item expressions referencing `$('On form submission').item.json` to extract employee name, ID, department, and last working date.
- **Input and Output Connections:** Input from `Aggregate`; output connects to `Send a message1`.
- **Version-Specific Requirements:** Version 2.5. Requires Slack OAuth2 or Bot token credentials.
- **Edge Cases / Potential Failure Types:** Bot missing post permissions in the target channel will cause request failures.

##### 7. Send a message1
- **Type and Technical Role:** `n8n-nodes-base.slack` (Notification Node)
- **Configuration Choices:** Posts an HR offboarding notification to channel ID `C0C0ZKASXEX` (`n8n-hr-offboarding`).
- **Key Expressions or Variables:** Dynamic template referencing form submission properties.
- **Input and Output Connections:** Input from `Send a message`.
- **Version-Specific Requirements:** Version 2.5.

---

### Block 1.4: Periodic Monitoring & Management Escalation

#### Overview
This block operates independently on a weekly schedule to pull all tasks from ClickUp, normalize their statuses via JavaScript, determine completion metrics, and trigger an escalation message to management via Slack if pending tasks exist.

#### Nodes Involved
- `Schedule Trigger`
- `Get many tasks1`
- `Code in JavaScript3`
- `If2`
- `Code in JavaScript4`
- `Send a message3`

#### Node Details

##### 1. Schedule Trigger
- **Type and Technical Role:** `n8n-nodes-base.scheduleTrigger` (Trigger Node)
- **Configuration Choices:** Configured to trigger periodically based on weekly interval rules.
- **Input and Output Connections:** Output connects to `Get many tasks1`.
- **Version-Specific Requirements:** Version 1.3.

##### 2. Get many tasks1
- **Type and Technical Role:** `n8n-nodes-base.clickUp` (Project Management Node)
- **Configuration Choices:** Retrieves all tasks from ClickUp team `90161771152`, space `90167968621`, and list `130107+1234567890`, filtering by statuses (`not started`, `in progress`, `completed`).
- **Input and Output Connections:** Input from `Schedule Trigger`; output connects to `Code in JavaScript3`.
- **Version-Specific Requirements:** Version 1.

##### 3. Code in JavaScript3
- **Type and Technical Role:** `n8n-nodes-base.code` (Data Transformation Node)
- **Configuration Choices:** Normalizes ClickUp task status fields (handling objects, strings, or missing values safely), filters tasks into completed and pending lists, calculates completion percentages, and establishes overall workflow statuses.
- **Input and Output Connections:** Input from `Get many tasks1`; output connects to `If2`.
- **Version-Specific Requirements:** Version 2.
- **Edge Cases / Potential Failure Types:** Errors if ClickUp API schema updates unexpected status object properties.

##### 4. If2
- **Type and Technical Role:** `n8n-nodes-base.if` (Flow Control Node)
- **Configuration Choices:** Evaluates whether the number of pending tasks is greater than zero.
- **Key Expressions or Variables:** Left Value: `={{ $json.pending_tasks }}`, Operator: Greater than `0`.
- **Input and Output Connections:** Input from `Code in JavaScript3`; true branch connects to `Code in JavaScript4`.
- **Version-Specific Requirements:** Version 2.3.

##### 5. Code in JavaScript4
- **Type and Technical Role:** `n8n-nodes-base.code` (Data Transformation Node)
- **Configuration Choices:** Compiles pending tasks into a formatted markdown text list and constructs a comprehensive Slack message payload summarizing total tasks, completed tasks, pending tasks, and percentage metrics.
- **Input and Output Connections:** Input from `If2`; output connects to `Send a message3`.
- **Version-Specific Requirements:** Version 2.

##### 6. Send a message3
- **Type and Technical Role:** `n8n-nodes-base.slack` (Notification Node)
- **Configuration Choices:** Posts the formatted escalation summary to Slack channel ID `C0C0R6H2VST` (`n8n-management`).
- **Key Expressions or Variables:** Text expression: `={{ $json.slack_message }}`.
- **Input and Output Connections:** Input from `Code in JavaScript4`.
- **Version-Specific Requirements:** Version 2.5.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| On form submission | `n8n-nodes-base.formTrigger` | Captures employee exit requests via web form | None | If | From field, Sheet: 1. Submit employee form with basic details 2. If node check if last working day true then get code node extract data from sheet |
| If | `n8n-nodes-base.if` | Validates that last working date is current or future | On form submission | Get row(s) in sheet | From field, Sheet: 1. Submit employee form with basic details 2. If node check if last working day true then get code node extract data from sheet |
| Get row(s) in sheet | `n8n-nodes-base.googleSheets` | Fetches employee master record from Google Sheets | If | Code in JavaScript | From field, Sheet: 1. Submit employee form with basic details 2. If node check if last working day true then get code node extract data from sheet |
| Code in JavaScript | `n8n-nodes-base.code` | Compiles IT and HR offboarding task checklists | Get row(s) in sheet | Split Out, Split Out1 | From field, Sheet: 1. Submit employee form with basic details 2. If node check if last working day true then get code node extract data from sheet |
| Split Out | `n8n-nodes-base.splitOut` | Splits IT tasks array for individual iteration | Code in JavaScript | Create a task | Split task IT and HR task parallel: 1. Split out task between IT and HR predefined and create in click up portal 2. Click up portal create task with tag hr and IT |
| Create a task | `n8n-nodes-base.clickUp` | Creates IT offboarding tasks in ClickUp | Split Out | None | Split task IT and HR task parallel: 1. Split out task between IT and HR predefined and create in click up portal 2. Click up portal create task with tag hr and IT |
| Split Out1 | `n8n-nodes-base.splitOut` | Splits HR tasks array for individual iteration | Code in JavaScript | Create a task1 | Split task IT and HR task parallel: 1. Split out task between IT and HR predefined and create in click up portal 2. Click up portal create task with tag hr and IT |
| Create a task1 | `n8n-nodes-base.clickUp` | Creates HR offboarding tasks in ClickUp | Split Out1 | Aggregate | Split task IT and HR task parallel: 1. Split out task between IT and HR predefined and create in click up portal 2. Click up portal create task with tag hr and IT |
| Aggregate | `n8n-nodes-base.aggregate` | Aggregates completed task creation items | Create a task1 | Send a message | Aggregate Slack: 1. After create task both create slack notification 2 different channel IT and HR with general message |
| Send a message | `n8n-nodes-base.slack` | Sends IT offboarding kickoff notification to Slack | Aggregate | Send a message1 | Aggregate Slack: 1. After create task both create slack notification 2 different channel IT and HR with general message |
| Send a message1 | `n8n-nodes-base.slack` | Sends HR offboarding kickoff notification to Slack | Send a message | None | Aggregate Slack: 1. After create task both create slack notification 2 different channel IT and HR with general message |
| Schedule Trigger | `n8n-nodes-base.scheduleTrigger` | Triggers periodic monitoring on a weekly interval | None | Get many tasks1 | Schedule, Get All task, Code: 1. After send slack message wait node will check every day how many task finishes 2. Code node extract pending task and completed task with count and detail |
| Get many tasks1 | `n8n-nodes-base.clickUp` | Fetches all offboarding tasks from ClickUp list | Schedule Trigger | Code in JavaScript3 | Schedule, Get All task, Code: 1. After send slack message wait node will check every day how many task finishes 2. Code node extract pending task and completed task with count and detail |
| Code in JavaScript3 | `n8n-nodes-base.code` | Normalizes task statuses and calculates completion metrics | Get many tasks1 | If2 | Schedule, Get All task, Code: 1. After send slack message wait node will check every day how many task finishes 2. Code node extract pending task and completed task with count and detail |
| If2 | `n8n-nodes-base.if` | Checks if pending task count is greater than zero | Code in JavaScript3 | Code in JavaScript4 | If Code and Slack: 1. If pending task count is greater than zero then send slack message to management like finish task which are unfinished |
| Code in JavaScript4 | `n8n-nodes-base.code` | Generates formatted markdown pending-task escalation text | If2 | Send a message3 | If Code and Slack: 1. If pending task count is greater than zero then send slack message to management like finish task which are unfinished |
| Send a message3 | `n8n-nodes-base.slack` | Sends task escalation alert to management Slack channel | Code in JavaScript4 | None | If Code and Slack: 1. If pending task count is greater than zero then send slack message to management like finish task which are unfinished |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**
   - Create an **On form submission** node (`n8n-nodes-base.formTrigger`).
   - Configure form title as `Employee Offboarding Form`.
   - Add required form fields: `Employee ID` (Text), `Employee Name` (Text), `Department` (Dropdown with options: Mobile, PHP, DotNet, AI/ML, FullStack, IT, HR), `Manager Email` (Text), `Last Working Date` (Date), `Exit Type` (Dropdown with options: Resignation, Layoff), and `Reason` (Text Area).

2. **Create Date Validation Node:**
   - Create an **If** node (`n8n-nodes-base.if`).
   - Connect **On form submission** output to this node.
   - Configure a number condition where left value is `={{ DateTime.fromISO($json['Last Working Date']).startOf('day').toMillis() }}` and right value is `={{ $now.startOf('day').toMillis() }}` with operation `gte`.

3. **Create Google Sheets Lookup Node:**
   - Create a **Get row(s) in sheet** node (`n8n-nodes-base.googleSheets`).
   - Connect the true output of the **If** node here.
   - Configure credentials for Google Sheets. Set document ID to your target employee master sheet and sheet tab to your target sheet name.
   - Set filter lookup column to `Employee ID` and lookup value to `={{$json['Employee ID']}}`.

4. **Create Task Preparation JavaScript Node:**
   - Create a **Code in JavaScript** node (`n8n-nodes-base.code`).
   - Connect the **Get row(s) in sheet** output here.
   - Paste the JavaScript logic that defines standard IT tasks (disable email, collect laptop, remove Slack/VPN access, and conditional Engineering/Finance items) and HR tasks (resignation letter, exit interview, final settlement, assets, experience letter, records update), combining them into structured JSON output containing `it_tasks`, `hr_tasks`, and `all_tasks`.

5. **Create Parallel Task Split Nodes:**
   - Create two **Split Out** nodes (`n8n-nodes-base.splitOut`).
   - Connect the output of the **Code in JavaScript** node to both split nodes.
   - Configure the first split node's field to split out as `=it_tasks`.
   - Configure the second split node's field to split out as `=hr_tasks`.

6. **Create ClickUp Integration Nodes:**
   - Create two **ClickUp** nodes (`n8n-nodes-base.clickUp`).
   - Connect the first **Split Out** node to the first ClickUp node, and the second **Split Out** node to the second ClickUp node.
   - Configure ClickUp credentials, Team ID, Space ID, and List ID.
   - Set operation to create task, task name to `={{ $json.task_name }}`, additional tags to `={{ [$json.department] }}`, priority to `1`, and content description using dynamic item markup.

7. **Create Slack Notification Chain:**
   - Create an **Aggregate** node (`n8n-nodes-base.aggregate`) connected to the output of the second ClickUp task creation node, set to `aggregateAllItemData`.
   - Create a **Send a message** node (`n8n-nodes-base.slack`) connected to the **Aggregate** node output. Configure Slack credentials, select channel mode, and target the IT offboarding channel ID (`C0C0ZL6TP27`) using dynamic properties from the trigger payload.
   - Create a second **Send a message** node (`n8n-nodes-base.slack`) connected in series to the first Slack node. Configure it to post to the HR offboarding channel ID (`C0C0ZKASXEX`).

8. **Create Scheduled Monitoring Workflow Branch:**
   - Create a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`) configured with a weekly interval rule.
   - Create a **Get many tasks** ClickUp node (`n8n-nodes-base.clickUp`) connected to the schedule trigger, set to operation `getAll` for the target team, space, and list, filtering statuses by `not started`, `in progress`, and `completed`.
   - Create a **Code in JavaScript** node (`n8n-nodes-base.code`) connected to the ClickUp task retrieval node. Paste the normalization script that maps status objects safely to strings, filters completed vs. pending arrays, and calculates completion percentages.
   - Create an **If** node (`n8n-nodes-base.if`) connected to the monitoring code node, configured to check if `pending_tasks > 0`.
   - Create a **Code in JavaScript** node (`n8n-nodes-base.code`) connected to the true branch of the monitoring If node, configured to assemble a formatted markdown text list and summary payload.
   - Create a **Send a message** node (`n8n-nodes-base.slack`) connected to the final code node, configured to post the constructed escalation message to the management channel ID (`C0C0R6H2VST`).

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Employee Offboarding Automation Overview | Automates validation, task creation, team notifications, and periodic task escalation for employee departures. |
| WeblineIndia Services | For assistance with custom automation, advanced extensions, department-specific rules, dashboards, and production improvements. |
Migrate ClickUp users, spaces, lists, custom fields, and tasks to Asana

https://n8nworkflows.xyz/workflows/migrate-clickup-users--spaces--lists--custom-fields--and-tasks-to-asana-21078


# Migrate ClickUp users, spaces, lists, custom fields, and tasks to Asana

### 1. Workflow Overview

This workflow automates the comprehensive migration of a ClickUp workspace to Asana, transferring users, spaces, folders, lists, custom fields, tasks, subtasks, comments, checklists, and attachments. It is structured into an **orchestrator** workflow that sequentially executes five distinct **migration stages** while managing rate limits, preventing duplicate creation via n8n data tables, and providing progress/error reporting via Gmail.

The migration logic is grouped into the following functional blocks:

- **1.1 Initialization and Validation (Orchestrator):** Configures initial migration parameters, tests ClickUp and Asana API credentials and workspace IDs, and dispatches a start notification email.
- **1.2 Sequential Stage Execution & CSV Generation (Orchestrator):** Sequentially calls the five sub-workflows (Stages 1 through 5), aggregates mapping data tables into CSV files, emails the final summary with attachments, and cleans up temporary data tables.
- **1.3 Stage 1 – User Migration:** Identifies ClickUp users missing from Asana, invites them to the Asana workspace, and updates their display names.
- **1.4 Stage 2 – Space Migration:** Maps ClickUp spaces to Asana teams, skipping already migrated spaces using a persistent lookup table.
- **1.5 Stage 3 – Folders and Lists Migration:** Converts ClickUp folders into Asana portfolios (if supported by the Asana plan) and lists into Asana projects with status-based sections.
- **1.6 Stage 4 – Custom Fields Migration:** Discovers and migrates custom fields across workspace, space, folder, and list scopes, handling multiple data types (text, number, currency, date, enums, multi-enums, people, checkboxes, emojis, task references) and space priorities.
- **1.7 Stage 5 – Tasks Migration:** Recursively reads tasks and subtasks, creates them in their respective Asana projects and sections, maps custom field values, priorities, comments, checklists (as subtasks), and downloads/uploads attachments.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Initialization and Validation (Orchestrator)
- **Overview:** Sets up environment variables containing API tokens and workspace identifiers, validates them against external APIs, and sends a start notification or stops execution with an error email if validation fails.
- **Nodes Involved:** `When clicking ‘Execute workflow’`, `Set Initial Workflow Environment`, `Check Asana Credentials`, `Check Asana Workspace ID`, `Check ClickUp Credentials`, `Check ClickUp Workspace ID`, `Send Asana Credentials Error`, `Send Asana Workspace ID Error`, `Send ClickUp Credentials Error`, `Send ClickUp Workspace ID Error`, `Send Start Notice`.
- **Node Details:**
  - `When clicking ‘Execute workflow’`: Manual Trigger initiating the orchestrator.
  - `Set Initial Workflow Environment`: Set node defining `FeedBackEmail`, `ClickUpWorkspaceID`, `ClickUpPAK`, `AsanaWorkspaceID`, `AsanaPlan`, and `AsanaPAK`.
  - `Check Asana Credentials`, `Check Asana Workspace ID`, `Check ClickUp Credentials`, `Check ClickUp Workspace ID`: HTTP Request nodes validating API tokens against `https://app.asana.com/api/1.0/users/me`, workspaces, and ClickUp endpoints. Configured with `onError: continueErrorOutput`.
  - `Send Asana Credentials Error`, `Send Asana Workspace ID Error`, `Send ClickUp Credentials Error`, `Send ClickUp Workspace ID Error`, `Send Start Notice`: Gmail nodes utilizing OAuth2 to transmit failure diagnostics or startup summaries to the feedback email.

#### Block 1.2: Sequential Stage Execution & CSV Generation (Orchestrator)
- **Overview:** Executes migration stages 1 through 5 in order, compiles data mapping tables into CSV attachments, emails the completion summary, and deletes temporary data tables.
- **Nodes Involved:** `Call '1. Users Migration'`, `Call '2. Spaces Migration'`, `Call '3. Folders & Lists Migration'`, `Call '4. Custom Fields Migration'`, `Call '5. Tasks Migration'`, `Get Spaces Mapping Rows`, `Get Lists Mapping Rows`, `Get Fields Mapping Rows`, `Get Tasks Mapping Rows`, `Convert Spaces To CSV`, `Convert Lists To CSV (orchestrator)`, `Convert Fields To CSV (orchestrator)`, `Convert Tasks To CSV (orchestrator)`, `Merge CSV Files`, `Send Success Message (orchestrator)`, `Delete Spaces Table`, `Delete Lists Table`, `Delete Fields Table`, `Delete Tasks Table`, `Wait for all lookup tables to be deleted`, `Set Workflow Environment (orchestrator)`.
- **Node Details:**
  - `Set Workflow Environment (orchestrator)`: Computes dynamic rate limits based on plan tiers (e.g., ClickUp Enterprise limits vs Standard) and creates a unified `TableSuffix`.
  - `Call '1. Users Migration'` through `Call '5. Tasks Migration'`: Execute Workflow nodes configured with `waitForSubWorkflow: true` to run migration stages sequentially.
  - `Get Spaces Mapping Rows` to `Get Tasks Mapping Rows`: Data Table nodes retrieving mapping records.
  - `Convert Spaces To CSV` to `Convert Tasks To CSV (orchestrator)`: Convert to File nodes generating downloadable CSV payloads.
  - `Merge CSV Files`: Merge node combining all four CSV binary streams.
  - `Send Success Message (orchestrator)`: Gmail node dispatching final notification with attached CSVs.
  - `Delete Spaces Table` to `Delete Tasks Table`: Data Table nodes performing cleanup operations.

#### Block 1.3: Stage 1 – User Migration
- **Overview:** Fetches all users from both platforms, compares their normalized email addresses, and invites missing ClickUp members to the Asana workspace.
- **Nodes Involved:** `Set Workflow Environment (stage 1)`, `Get All ClickUp Users`, `Create ClickUp Teams List`, `Create ClickUp User List`, `Normalise ClickUp User Data`, `Get All Asana Users`, `Create Asana Users List`, `Normalise Asana User Data`, `Get Unmatched ClickUp Users`, `Loop Over Users`, `Add User To Asana`, `If Asana User Created`, `Update Asana User`, `Send Success Message (stage 1)`, `Send Error Message (stage 1)`, `Stop Migration (stage 1)`.
- **Node Details:**
  - `Set Workflow Environment (stage 1)`: Set node initializing stage parameters.
  - `Get All ClickUp Users`, `Get All Asana Users`: HTTP Request nodes fetching member directories.
  - `Normalise ClickUp User Data`, `Normalise Asana User Data`: Set nodes trimming and lowercasing email strings (`{{ $json.user.email.trim().toLowerCase() }}`).
  - `Get Unmatched ClickUp Users`: Merge node configured with `joinMode: keepNonMatches` matching on email fields.
  - `Add User To Asana`: HTTP Request posting new users to Asana workspace via `/addUser`.

#### Block 1.4: Stage 2 – Space Migration
- **Overview:** Retrieves ClickUp spaces, checks against a local mapping table to prevent duplicates, and provisions missing spaces as Asana teams.
- **Nodes Involved:** `Set Workflow Environment (stage 2)`, `Get ClickUp Spaces (stage 2)`, `Create ClickUp Spaces Array (stage 2)`, `Search ClickUpSpace-AsanaTeam Lookup Table`, `If Table Exists (stage 2)`, `Get row(s)`, `Create Lookup Table (stage 2)`, `Get Unmatched ClickUp Spaces`, `Loop Over Spaces (stage 2)`, `Create Asana Team`, `If Asana Team Created`, `Update Lookup Table (stage 2)`, `Collect all spaces`, `Get Lookup Table Rows (stage 2)`, `Convert Teams To CSV`, `Send Success Message (stage 2)`, `Send Error Message (stage 2)`, `Stop Migration (stage 2)`.
- **Node Details:**
  - `Create Asana Team`: HTTP Request posting to `https://app.asana.com/api/1.0/teams` with organization scope and name suffix ` (ClickUp)`.
  - `Update Lookup Table (stage 2)`: Data Table node storing mapping relationships between `clickup_space_id` and `asana_team_gid`.

#### Block 1.5: Stage 3 – Folders and Lists Migration
- **Overview:** Processes ClickUp folders and lists across all spaces, creates portfolios for folders (if supported by the Asana plan), creates projects for lists, generates status-based sections, and links projects to teams and portfolios.
- **Nodes Involved:** `Set Workflow Environment (stage 3)`, `Get ClickUp Spaces (stage 3) (Find lists that are not migrated)`, `Create ClickUp Spaces Array (stage 3) (Find lists that are not migrated)`, `Get ClickUp Folders (stage 3) (Find lists that are not migrated)`, `Create ClickUp Folders Array (stage 3) (Find lists that are not migrated)`, `Get ClickUp Lists`, `Create ClickUp Lists Array (stage 3)`, `Get ClickUp Folderless Lists (stage 3)`, `Create ClickUp Folderless Lists Array (stage 3)`, `Join ClickUp Lists (stage 3)`, `Get All Lists`, `Get Unmatched ClickUp Lists`, `Loop Over Lists (stage 3)`, `If Asana Plan Advanced Or Above (Create portfolios for folders)`, `Get ClickUp Spaces (stage 3) (Create portfolios for folders)`, `Create ClickUp Spaces Array (stage 3) (Create portfolios for folders)`, `Get ClickUp Folders (stage 3) (Create portfolios for folders)`, `Create ClickUp Folders Array (stage 3) (Create portfolios for folders)`, `Get Unmatched ClickUp Folders`, `Get All Folders`, `Any Folders To Create`, `Create Asana Portfolio`, `Update Lookup Table With Portfolio`, `If Asana Plan Advanced Or Above (Create the Asana projects)`, `Create Asana Project With Folder Name`, `Create Asana Project`, `If Asana Project Created`, `Get Asana Team`, `Assign Team To Project`, `Get Portfolio Data`, `If Has Portfolio`, `Connect Project To Portfolio`, `Get ClickUp List Data`, `Create ClickUp List Statuses Array`, `Sort Statuses`, `Create Section In Project`, `Wait for team, section and portfolio updates`, `Update Lookup Table (stage 3)`, `Collect all lists`, `Get Lookup Table Rows (stage 3)`, `Convert Lists To CSV (stage 3)`, `Send Success Message (stage 3)`, `Send Error Message (stage 3)`, `Stop Migration (stage 3)`.
- **Node Details:**
  - `If Asana Plan Advanced Or Above`: If node inspecting whether `AsanaPlan` contains `advanced` or `enterprise` to decide on portfolio creation.
  - `Create Asana Portfolio`: HTTP Request provisioning portfolios (`/portfolios`).
  - `Sort Statuses`: Sort node ordering status options by `orderindex` descending.
  - `Create Section In Project`: HTTP Request creating sections in Asana projects corresponding to ClickUp list statuses.

#### Block 1.6: Stage 4 – Custom Fields Migration
- **Overview:** Reads custom fields across Workspace, Space, Folder, and List scopes, handles space priorities, maps enum/multi-enum colors using a nearest-color algorithm, creates custom fields in Asana, and attaches them to relevant projects.
- **Nodes Involved:** `Set Workflow Environment (stage 4)`, `Search ClickUpField-AsanaField Lookup Table`, `If Table Doesn't Exists`, `Create ClickUpField-AsanaField Lookup Table`, `Get ClickUp Workspace Custom Fields`, `Create Workspace Fields Array`, `Loop over workspace fields`, `Get ClickUp Workspaces`, `Set Workspace Field Data`, `Lookup Migrated Fields Data (Read workspace fields)`, `If Field Not Migrated (Read workspace fields)`, `Get ClickUp Spaces (stage 4)`, `Create ClickUp Spaces Array (stage 4)`, `Loop Over Spaces (stage 4) (Read space fields)`, `Wait For ClickUp Rate-limit (Read space fields)`, `Get Spaces Custom Fields`, `Set Space Data`, `If Space Custom Field Exist`, `Lookup Migrated Fields Data (Read space fields)`, `If Field Not Migrated (Read space fields)`, `Set Space Field Data`, `Loop Over Spaces (stage 4) (Migrate space priorities)`, `If Space Priorities Enabled (Migrate space priorities)`, `Lookup Priority Fields Data`, `If Priority Not Migrated`, `Create Workspace Priorities In Asana`, `Split Priority options (workspace field)`, `Update ClickUpField-AsanaField Table (Migrate space priorities)`, `Get ClickUp Folders (stage 4)`, `Create ClickUp Folders Array (stage 4)`, `Loop Over Folders`, `Wait For ClickUp Rate-limit (Read folder fields)`, `Get Folder Custom Fields`, `Set Folder Data`, `If Folder Custom Field Exist`, `Lookup Migrated Fields Data (Read folder fields)`, `If Field Not Migrated (Read folder fields)`, `Set Folder Field Data`, `Get All Lists`, `Create ClickUp Lists Array (stage 4)`, `Get ClickUp Folderless Lists (stage 4)`, `Create ClickUp Folderless Lists Array (stage 4)`, `Join ClickUp Lists (stage 4)`, `Loop Over Lists (stage 4)`, `Wait For ClickUp Rate-limit (Add fields to Asana projects)`, `Set List Data`, `Get List Custom Fields`, `If List Custom Fields Exist`, `Lookup Migrated Fields Data (Add fields to Asana projects)`, `If Field Not Migrated (Add fields to Asana projects)`, `If Upper Scope Field`, `Get List's Space Data`, `If Space Priorities Enabled (Add fields to Asana projects)`, `Lookup Priority Field Data`, `Look up Asana project to add priority field`, `Add Priority Field To Asana Project`, `Check priority field already added`, `Set Priority Flag`, `Look up Asana project to add custom field`, `Add Custom Field To Asana Project`, `Check custom field already added`, `Field Type Switch (stage 4)`, `Check scope for Text/Number/Currency/Date/Enum/MultiEnum/People/Checkbox/Emoji/TaskReference fields`, `Map To Asana Enum Options`, `Map To Asana Multi Enum Options`, `Map Emoji To Asana Enum Options`, `Create Asana Workspace/Project * Fields`, `Split * options`, `Join * Options Data`, `Set * Options Data`, `Update ClickUpField-AsanaField Table (...)`, `Wait for all field loops to finish`, `Collect all field results`, `Get Lookup Table Rows (stage 4)`, `Convert Fields To CSV (stage 4)`, `Send Success Message (stage 4)`.
- **Node Details:**
  - `Map To Asana Enum Options` / `Map To Asana Multi Enum Options`: Code nodes executing JavaScript algorithms to convert hexadecimal color codes from ClickUp to the nearest valid Asana color palette (`ASANA_COLORS`).
  - `Field Type Switch (stage 4)`: Switch node routing custom fields based on type (`text`, `number`, `currency`, `date`, `drop_down`, `labels`, `users`, `checkbox`, `emoji`, `tasks`).

#### Block 1.7: Stage 5 – Tasks Migration
- **Overview:** Recursively fetches tasks and subtasks from ClickUp lists, handles hierarchy sorting, creates Asana tasks/subtasks, sets custom field values, assigns priorities, adds comments, converts checklists to subtasks, and transfers attachments.
- **Nodes Involved:** `Stage 5 trigger`, `Set Workflow Environment (stage 5)`, `Search ClickUpTask-AsanaTask Lookup Table`, `If Table Exists (stage 5)`, `Create Lookup Table (stage 5)`, `Get All Rows`, `Get Unmatched ClickUp Tasks`, `Get ClickUp Spaces (stage 5)`, `Create ClickUp Spaces Array (stage 5)`, `Get ClickUp Folders (stage 5)`, `Create ClickUp Folders Array (stage 5)`, `Get ClickUp Folder Lists (stage 5)`, `Create ClickUp Lists Array (stage 5)`, `Get ClickUp Folderless Lists (stage 5)`, `Create ClickUp Folderless Lists Array (stage 5)`, `Join ClickUp Lists (stage 5)`, `Get All ClickUp Tasks And Subtasks`, `Filter Empty ClickUp Lists`, `Create Tasks Array`, `Sort Subtasks After Tasks`, `Loop Over Tasks`, `If No Parent Task`, `Lookup Asana Project`, `Get Sections In Asana Project`, `Get Parent Asana Task`, `Set Essential Asana Task Data`, `Create Asana Task`, `If Not Subtask`, `Add Task To Section`, `Get ClickUp Task Data`, `Set Asana Task ID`, `If Any Field Values`, `Create Fields Array`, `Filter Empty Fields`, `Field Type Switch (stage 5)`, `Get Asana * Field ID / Option ID`, `Set Asana * Field`, `Create Label Array`, `Create Asana Multi Enum Options Array`, `Merge Field Values`, `Collect task field values`, `Add Asana Task Field Values`, `If Priority Exists`, `Get Asana Priority Field ID`, `Set Asana Priority Field`, `Set Asana Task Priority`, `Get ClickUp Task Comments`, `If Any Comments`, `Create Comments Array`, `Set Asana Comment Data`, `Create Asana Task Comment`, `If Checklists Exist`, `Set ClickUp Checklist Data`, `Create Checklists Array`, `Create Checklist Items Array`, `Set Asana Checklist Subtasks Data`, `Creat Asana Checklist Subtask`, `If Attachments Exist`, `Set ClickUp Attachment Data`, `Create Attachments Array`, `Download Attachment`, `Upload Attachment To Asana Task`, `Update Lookup Table (stage 5)`, `Aggregate Tasks`, `Get Lookup Table Rows (stage 5)`, `Convert Tasks To CSV (stage 5)`, `Send Success Message (stage 5)`.
- **Node Details:**
  - `Sort Subtasks After Tasks`: Code node sorting tasks by hierarchy depth (`depth`) to ensure parent tasks are created before subtasks.
  - `Create Asana Task`: HTTP Request dynamically creating tasks or subtasks based on the presence of `ParentAsanaGid`.
  - `Upload Attachment To Asana Task`: HTTP Request with `multipart-form-data` payload uploading binary attachments downloaded from ClickUp.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Documentation overview | None | None | Migrate users, spaces, lists, custom fields and tasks from ClickUp to Asana |
| Call '1. Users Migration' | n8n-nodes-base.executeWorkflow | Executes Stage 1 sub-workflow | Start Notice | Call '2. Spaces Migration' | Run the five stages |
| Set Workflow Environment (orchestrator) | n8n-nodes-base.set | Prepares orchestrator environment | Start Notice | Call '1. Users Migration' | Start the run |
| Check Asana Credentials | n8n-nodes-base.httpRequest | Validates Asana API token | Set Initial Workflow Environment | Check Asana Workspace ID, Send Asana Credentials Error | Check tokens and workspace IDs |
| Check Asana Workspace ID | n8n-nodes-base.httpRequest | Validates Asana workspace ID | Check Asana Credentials | Check ClickUp Credentials, Send Asana Workspace ID Error | Check tokens and workspace IDs |
| Send Asana Credentials Error | n8n-nodes-base.gmail | Emails Asana auth failure | Check Asana Credentials | None | Check tokens and workspace IDs |
| Send Asana Workspace ID Error | n8n-nodes-base.gmail | Emails Asana workspace ID failure | Check Asana Credentials | None | Check tokens and workspace IDs |
| Send ClickUp Credentials Error | n8n-nodes-base.gmail | Emails ClickUp auth failure | Check ClickUp Credentials | None | Check tokens and workspace IDs |
| Check ClickUp Workspace ID | n8n-nodes-base.httpRequest | Validates ClickUp workspace | Check Asana Workspace ID | Set Workflow Environment (orchestrator), Send ClickUp Workspace ID Error | Check tokens and workspace IDs |
| Send ClickUp Workspace ID Error | n8n-nodes-base.gmail | Emails ClickUp workspace failure | Check ClickUp Workspace ID | None | Check tokens and workspace IDs |
| Send Start Notice | n8n-nodes-base.gmail | Emails migration start notice | Set Initial Workflow Environment | Call '1. Users Migration' | Start the run |
| Call '2. Spaces Migration' | n8n-nodes-base.executeWorkflow | Executes Stage 2 sub-workflow | Call '1. Users Migration' | Call '3. Folders & Lists Migration' | Run the five stages |
| Call '4. Custom Fields Migration' | n8n-nodes-base.executeWorkflow | Executes Stage 4 sub-workflow | Call '3. Folders & Lists Migration' | Call '5. Tasks Migration' | Run the five stages |
| Call '5. Tasks Migration' | n8n-nodes-base.executeWorkflow | Executes Stage 5 sub-workflow | Call '4. Custom Fields Migration' | Get * Mapping Rows | Run the five stages |
| Call '3. Folders & Lists Migration' | n8n-nodes-base.executeWorkflow | Executes Stage 3 sub-workflow | Call '2. Spaces Migration' | Call '4. Custom Fields Migration' | Run the five stages |
| Send Success Message (orchestrator) | n8n-nodes-base.gmail | Emails final summary with CSVs | Merge CSV Files | Delete * Table | Build the mapping files |
| Check ClickUp Credentials | n8n-nodes-base.httpRequest | Validates ClickUp token | Check Asana Workspace ID | Check ClickUp Workspace ID, Send ClickUp Credentials Error | Check tokens and workspace IDs |
| Get Spaces Mapping Rows | n8n-nodes-base.dataTable | Reads space mapping data | Call '5. Tasks Migration' | Convert Spaces To CSV | Build the mapping files |
| Get Lists Mapping Rows | n8n-nodes-base.dataTable | Reads list mapping data | Call '5. Tasks Migration' | Convert Lists To CSV (orchestrator) | Build the mapping files |
| Get Fields Mapping Rows | n8n-nodes-base.dataTable | Reads field mapping data | Call '5. Tasks Migration' | Convert Fields To CSV (orchestrator) | Build the mapping files |
| Get Tasks Mapping Rows | n8n-nodes-base.dataTable | Reads task mapping data | Call '5. Tasks Migration' | Convert Tasks To CSV (orchestrator) | Build the mapping files |
| Convert Spaces To CSV | n8n-nodes-base.convertToFile | Converts space mapping to CSV | Get Spaces Mapping Rows | Merge CSV Files | Build the mapping files |
| Convert Lists To CSV (orchestrator) | n8n-nodes-base.convertToFile | Converts list mapping to CSV | Get Lists Mapping Rows | Merge CSV Files | Build the mapping files |
| Convert Fields To CSV (orchestrator) | n8n-nodes-base.convertToFile | Converts field mapping to CSV | Get Fields Mapping Rows | Merge CSV Files | Build the mapping files |
| Convert Tasks To CSV (orchestrator) | n8n-nodes-base.convertToFile | Converts task mapping to CSV | Get Tasks Mapping Rows | Merge CSV Files | Build the mapping files |
| Merge CSV Files | n8n-nodes-base.merge | Merges all CSV files | Convert * To CSV | Send Success Message (orchestrator) | Build the mapping files |
| Delete Spaces Table | n8n-nodes-base.dataTable | Deletes temporary space table | Send Success Message (orchestrator) | Wait for all lookup tables to be deleted | Clean up |
| Delete Lists Table | n8n-nodes-base.dataTable | Deletes temporary list table | Send Success Message (orchestrator) | Wait for all lookup tables to be deleted | Clean up |
| Delete Fields Table | n8n-nodes-base.dataTable | Deletes temporary field table | Send Success Message (orchestrator) | Wait for all lookup tables to be deleted | Clean up |
| Delete Tasks Table | n8n-nodes-base.dataTable | Deletes temporary task table | Send Success Message (orchestrator) | Wait for all lookup tables to be deleted | Clean up |
| Wait for all lookup tables to be deleted | n8n-nodes-base.merge | Synchronizes table deletions | Delete * Table | None | Clean up |
| Set Initial Workflow Environment | n8n-nodes-base.set | Sets orchestrator parameters | When clicking ‘Execute workflow’ | Check Asana Credentials | Enter your settings |
| Section - Enter your settings (orchestrator) | n8n-nodes-base.stickyNote | Documentation note | None | None | Enter your settings |
| Section - Check tokens and workspace IDs (orchestrator) | n8n-nodes-base.stickyNote | Documentation note | None | None | Check tokens and workspace IDs |
| Section - Start the run (orchestrator) | n8n-nodes-base.stickyNote | Documentation note | None | None | Start the run |
| Section - Run the five stages (orchestrator) | n8n-nodes-base.stickyNote | Documentation note | None | None | Run the five stages |
| Section - Build the mapping files (orchestrator) | n8n-nodes-base.stickyNote | Documentation note | None | None | Build the mapping files |
| Section - Clean up (orchestrator) | n8n-nodes-base.stickyNote | Documentation note | None | None | Clean up |
| When clicking ‘Execute workflow’ | n8n-nodes-base.manualTrigger | Triggers orchestrator | None | Set Initial Workflow Environment | Enter your settings |
| Get Unmatched ClickUp Users | n8n-nodes-base.merge | Filters missing users | Normalise * User Data | Loop Over Users | Find users that are missing in Asana |
| Add User To Asana | n8n-nodes-base.httpRequest | Invites user to Asana | Loop Over Users | If Asana User Created | Add the missing users |
| Get All ClickUp Users | n8n-nodes-base.httpRequest | Fetches ClickUp members | Set Workflow Environment (stage 1) | Create ClickUp Teams List | Find users that are missing in Asana |
| Get All Asana Users | n8n-nodes-base.httpRequest | Fetches Asana users | Set Workflow Environment (stage 1) | Create Asana Users List | Find users that are missing in Asana |
| Create ClickUp Teams List | n8n-nodes-base.splitOut | Splits ClickUp teams | Get All ClickUp Users | Create ClickUp User List | Find users that are missing in Asana |
| Create ClickUp User List | n8n-nodes-base.splitOut | Splits ClickUp members | Create ClickUp Teams List | Normalise ClickUp User Data | Find users that are missing in Asana |
| Create Asana Users List | n8n-nodes-base.splitOut | Splits Asana users | Get All Asana Users | Normalise Asana User Data | Find users that are missing in Asana |
| Normalise ClickUp User Data | n8n-nodes-base.set | Normalizes ClickUp emails | Create ClickUp User List | Get Unmatched ClickUp Users | Find users that are missing in Asana |
| Normalise Asana User Data | n8n-nodes-base.set | Normalizes Asana emails | Create Asana Users List | Get Unmatched ClickUp Users | Find users that are missing in Asana |
| Update Asana User | n8n-nodes-base.httpRequest | Updates Asana user name | If Asana User Created | Loop Over Users | Add the missing users |
| Send Error Message (stage 1) | n8n-nodes-base.gmail | Emails Stage 1 error | If Asana User Created | Stop Migration (stage 1) | Report an error |
| If Asana User Created | n8n-nodes-base.if | Checks if user invite succeeded | Add User To Asana | Update Asana User, Send Error Message (stage 1) | Add the missing users |
| Loop Over Users | n8n-nodes-base.splitInBatches | Batches user processing | Get Unmatched ClickUp Users | Send Success Message (stage 1), Add User To Asana | Add the missing users |
| Stop Migration (stage 1) | n8n-nodes-base.stopAndError | Stops Stage 1 on error | Send Error Message (stage 1) | None | Report an error |
| Section - Receive the settings (stage 1) | n8n-nodes-base.stickyNote | Documentation note | None | None | Receive the settings |
| Section - Find users that are missing in Asana (stage 1) | n8n-nodes-base.stickyNote | Documentation note | None | None | Find users that are missing in Asana |
| Section - Add the missing users (stage 1) | n8n-nodes-base.stickyNote | Documentation note | None | None | Add the missing users |
| Section - Report success (stage 1) | n8n-nodes-base.stickyNote | Documentation note | None | None | Report success |
| Section - Report an error (stage 1) | n8n-nodes-base.stickyNote | Documentation note | None | None | Report an error |
| Send Success Message (stage 1) | n8n-nodes-base.gmail | Emails Stage 1 completion | Loop Over Users | None | Report success |
| Set Workflow Environment (stage 1) | n8n-nodes-base.set | Sets Stage 1 environment | Trigger | Get All ClickUp/Asana Users | Receive the settings |
| Get ClickUp Spaces (stage 2) | n8n-nodes-base.httpRequest | Fetches ClickUp spaces | Set Workflow Environment (stage 2) | Create ClickUp Spaces Array (stage 2) | Find spaces that are not migrated |
| Search ClickUpSpace-AsanaTeam Lookup Table | n8n-nodes-base.dataTable | Queries space mapping table | Set Workflow Environment (stage 2) | If Table Exists (stage 2) | Find spaces that are not migrated |
| Get row(s) | n8n-nodes-base.dataTable | Reads mapping table rows | If Table Exists (stage 2) | Get Unmatched ClickUp Spaces | Find spaces that are not migrated |
| Create Lookup Table (stage 2) | n8n-nodes-base.dataTable | Creates space mapping table | If Table Exists (stage 2) | Get Unmatched ClickUp Spaces | Find spaces that are not migrated |
| If Table Exists (stage 2) | n8n-nodes-base.if | Checks if space table exists | Search ClickUpSpace-AsanaTeam Lookup Table | Get row(s), Create Lookup Table (stage 2) | Find spaces that are not migrated |
| Get Unmatched ClickUp Spaces | n8n-nodes-base.merge | Filters unmigrated spaces | Create ClickUp Spaces Array (stage 2), Get row(s) | Loop Over Spaces (stage 2) | Find spaces that are not migrated |
| Create Asana Team | n8n-nodes-base.httpRequest | Creates Asana team | Loop Over Spaces (stage 2) | If Asana Team Created | Create the Asana teams |
| Update Lookup Table (stage 2) | n8n-nodes-base.dataTable | Saves space/team mapping | If Asana Team Created | Loop Over Spaces (stage 2) | Create the Asana teams |
| If Asana Team Created | n8n-nodes-base.if | Verifies team creation | Create Asana Team | Update Lookup Table (stage 2), Send Error Message (stage 2) | Create the Asana teams |
| Create ClickUp Spaces Array (stage 2) | n8n-nodes-base.splitOut | Splits ClickUp spaces | Get ClickUp Spaces (stage 2) | Get Unmatched ClickUp Spaces | Find spaces that are not migrated |
| Loop Over Spaces (stage 2) | n8n-nodes-base.splitInBatches | Batches space processing | Get Unmatched ClickUp Spaces | Collect all spaces, Create Asana Team | Create the Asana teams |
| Convert Teams To CSV | n8n-nodes-base.convertToFile | Converts team mapping to CSV | Get Lookup Table Rows (stage 2) | Send Success Message (stage 2) | Send the mapping file |
| Get Lookup Table Rows (stage 2) | n8n-nodes-base.dataTable | Reads all space mapping rows | Collect all spaces | Convert Teams To CSV | Send the mapping file |
| Collect all spaces | n8n-nodes-base.aggregate | Collects processed spaces | Loop Over Spaces (stage 2) | Get Lookup Table Rows (stage 2) | Send the mapping file |
| Section - Find spaces that are not migrated (stage 2) | n8n-nodes-base.stickyNote | Documentation note | None | None | Find spaces that are not migrated |
| Section - Create the Asana teams (stage 2) | n8n-nodes-base.stickyNote | Documentation note | None | None | Create the Asana teams |
| Section - Send the mapping file (stage 2) | n8n-nodes-base.stickyNote | Documentation note | None | None | Send the mapping file |
| Set Workflow Environment (stage 2) | n8n-nodes-base.set | Sets Stage 2 environment | Trigger | Search Table, Get Spaces | Receive the settings |
| Send Error Message (stage 2) | n8n-nodes-base.gmail | Emails Stage 2 error | If Asana Team Created | Stop Migration (stage 2) | Report an error |
| Send Success Message (stage 2) | n8n-nodes-base.gmail | Emails Stage 2 completion | Convert Teams To CSV | None | Send the mapping file |
| Stop Migration (stage 2) | n8n-nodes-base.stopAndError | Stops Stage 2 on error | Send Error Message (stage 2) | None | Report an error |
| Section - Receive the settings (stage 2) | n8n-nodes-base.stickyNote | Documentation note | None | None | Receive the settings |
| Section - Report an error (stage 2) | n8n-nodes-base.stickyNote | Documentation note | None | None | Report an error |
| Get ClickUp Folders (stage 3) (Find lists that are not migrated) | n8n-nodes-base.httpRequest | Fetches ClickUp folders | Get ClickUp Spaces (stage 3)... | Create ClickUp Folders Array (stage 3)... | Find lists that are not migrated |
| Search ClickUpList-AsanaProject Lookup Table | n8n-nodes-base.dataTable | Queries list mapping table | Set Workflow Environment (stage 3) | If Table Exists (stage 3) | Find lists that are not migrated |
| Get ClickUp Lists | n8n-nodes-base.httpRequest | Fetches lists in folders | Create ClickUp Folders Array (stage 3)... | Create ClickUp Lists Array (stage 3) | Find lists that are not migrated |
| If Asana Project Created | n8n-nodes-base.if | Checks if project creation succeeded | Create Asana Project / With Folder Name | Get Asana Team, Get List Data, If Has Portfolio | Create the Asana projects |
| Get ClickUp Folderless Lists (stage 3) | n8n-nodes-base.httpRequest | Fetches folderless lists | Get ClickUp Spaces (stage 3)... | Create ClickUp Folderless Lists Array (stage 3) | Find lists that are not migrated |
| Join ClickUp Lists (stage 3) | n8n-nodes-base.merge | Combines folder & folderless lists | Create ClickUp Lists / Folderless Arrays | Get Unmatched ClickUp Lists | Find lists that are not migrated |
| Create ClickUp Folders Array (stage 3) (Find lists that are not migrated) | n8n-nodes-base.splitOut | Splits ClickUp folders | Get ClickUp Folders (stage 3)... | Get ClickUp Lists | Find lists that are not migrated |
| Create ClickUp Lists Array (stage 3) | n8n-nodes-base.splitOut | Splits list items | Get ClickUp Lists | Join ClickUp Lists (stage 3) | Find lists that are not migrated |
| Create ClickUp Folderless Lists Array (stage 3) | n8n-nodes-base.splitOut | Splits folderless lists | Get ClickUp Folderless Lists (stage 3) | Join ClickUp Lists (stage 3) | Find lists that are not migrated |
| Get Unmatched ClickUp Lists | n8n-nodes-base.merge | Filters unmigrated lists | Join ClickUp Lists (stage 3), Get All Lists | Loop Over Lists (stage 3) | Find lists that are not migrated |
| Get Asana Team | n8n-nodes-base.dataTable | Retrieves assigned team | If Asana Project Created | Assign Team To Project | Assign team and portfolio |
| Assign Team To Project | n8n-nodes-base.httpRequest | Assigns project to team | Get Asana Team | Wait for team, section and portfolio updates | Assign team and portfolio |
| Loop Over Lists (stage 3) | n8n-nodes-base.splitInBatches | Batches list processing | Get Unmatched ClickUp Lists | Collect all lists, If Asana Plan Advanced Or Above | Create the Asana projects |
| Create Section In Project | n8n-nodes-base.httpRequest | Creates project section | Sort Statuses | Wait for team, section and portfolio updates | Create sections from statuses |
| Get ClickUp List Data | n8n-nodes-base.httpRequest | Fetches list details/statuses | If Asana Project Created | Create ClickUp List Statuses Array | Create sections from statuses |
| Create ClickUp List Statuses Array | n8n-nodes-base.splitOut | Splits statuses | Get ClickUp List Data | Sort Statuses | Create sections from statuses |
| Sort Statuses | n8n-nodes-base.sort | Sorts statuses by orderindex | Create ClickUp List Statuses Array | Create Section In Project | Create sections from statuses |
| Get All Lists | n8n-nodes-base.dataTable | Reads all migrated lists | If Table Exists (stage 3) | Get Unmatched ClickUp Lists | Find lists that are not migrated |
| If Asana Plan Advanced Or Above (Create portfolios for folders) | n8n-nodes-base.if | Checks plan for portfolios | If Table Exists (stage 3) | Get ClickUp Spaces (stage 3) (Create portfolios for folders), Get ClickUp Spaces (stage 3) (Find lists that are not migrated) | Create portfolios for folders |
| Get ClickUp Spaces (stage 3) (Create portfolios for folders) | n8n-nodes-base.httpRequest | Fetches spaces for portfolios | If Asana Plan Advanced... | Create ClickUp Spaces Array (stage 3)... | Create portfolios for folders |
| Get ClickUp Folders (stage 3) (Create portfolios for folders) | n8n-nodes-base.httpRequest | Fetches folders for portfolios | Create ClickUp Spaces Array (stage 3)... | Create ClickUp Folders Array (stage 3)... | Create portfolios for folders |
| Create ClickUp Spaces Array (stage 3) (Create portfolios for folders) | n8n-nodes-base.splitOut | Splits spaces for portfolios | Get ClickUp Spaces (stage 3)... | Get ClickUp Folders (stage 3) (Create portfolios for folders) | Create portfolios for folders |
| Create ClickUp Folders Array (stage 3) (Create portfolios for folders) | n8n-nodes-base.splitOut | Splits folders for portfolios | Get ClickUp Folders (stage 3)... | Get Unmatched ClickUp Folders | Create portfolios for folders |
| Create Asana Portfolio | n8n-nodes-base.httpRequest | Creates Asana portfolio | Any Folders To Create | Update Lookup Table With Portfolio | Create portfolios for folders |
| Any Folders To Create | n8n-nodes-base.if | Checks if folders exist | Get Unmatched ClickUp Folders | Create Asana Portfolio, Get ClickUp Spaces (stage 3) (Find lists that are not migrated) | Create portfolios for folders |
| Create Asana Project With Folder Name | n8n-nodes-base.httpRequest | Creates project with folder prefix | If Asana Plan Advanced... (Create the Asana projects) | If Asana Project Created | Create the Asana projects |
| If Asana Plan Advanced Or Above (Create the Asana projects) | n8n-nodes-base.if | Checks plan for folder naming | Loop Over Lists (stage 3) | Get Portfolio Data, Create Asana Project With Folder Name | Create the Asana projects |
| Get Unmatched ClickUp Folders | n8n-nodes-base.merge | Filters unmigrated folders | Create ClickUp Folders Array (stage 3)..., Get All Folders | Any Folders To Create | Create portfolios for folders |
| Get All Folders | n8n-nodes-base.dataTable | Reads existing portfolios | If Asana Plan Advanced... | Get Unmatched ClickUp Folders | Create portfolios for folders |
| Create Asana Project | n8n-nodes-base.httpRequest | Creates standard project | Get Portfolio Data | If Asana Project Created | Create the Asana projects |
| Connect Project To Portfolio | n8n-nodes-base.httpRequest | Adds project to portfolio | If Has Portfolio | Wait for team, section and portfolio updates | Assign team and portfolio |
| Update Lookup Table With Portfolio | n8n-nodes-base.dataTable | Saves portfolio mapping | Create Asana Portfolio | Get ClickUp Spaces (stage 3) (Find lists that are not migrated) | Create portfolios for folders |
| Get Portfolio Data | n8n-nodes-base.dataTable | Retrieves portfolio GID | If Asana Plan Advanced... (Create the Asana projects) | Create Asana Project | Create the Asana projects |
| If Has Portfolio | n8n-nodes-base.if | Checks portfolio existence | If Asana Project Created | Connect Project To Portfolio, Wait for team, section and portfolio updates | Assign team and portfolio |
| Section - Create portfolios for folders (stage 3) | n8n-nodes-base.stickyNote | Documentation note | None | None | Create portfolios for folders |
| Section - Find lists that are not migrated (stage 3) | n8n-nodes-base.stickyNote | Documentation note | None | None | Find lists that are not migrated |
| Section - Create the Asana projects (stage 3) | n8n-nodes-base.stickyNote | Documentation note | None | None | Create the Asana projects |
| Section - Create sections from statuses (stage 3) | n8n-nodes-base.stickyNote | Documentation note | None | None | Create sections from statuses |
| Section - Assign team and portfolio (stage 3) | n8n-nodes-base.stickyNote | Documentation note | None | None | Assign team and portfolio |
| Section - Send the mapping file (stage 3) | n8n-nodes-base.stickyNote | Documentation note | None | None | Send the mapping file |
| Section - Report an error (stage 3) | n8n-nodes-base.stickyNote | Documentation note | None | None | Report an error |
| Get ClickUp Spaces (stage 3) (Find lists that are not migrated) | n8n-nodes-base.httpRequest | Fetches spaces for list migration | Any Folders To Create, If Asana Plan Advanced... | Create ClickUp Spaces Array (stage 3) (Find lists that are not migrated) | Find lists that are not migrated |
| Create Lookup Table (stage 3) | n8n-nodes-base.dataTable | Creates list mapping table | If Table Exists (stage 3) | If Asana Plan Advanced Or Above (Create portfolios for folders) | Find lists that are not migrated |
| If Table Exists (stage 3) | n8n-nodes-base.if | Checks list table existence | Search ClickUpList-AsanaProject Lookup Table | If Asana Plan Advanced Or Above, Create Lookup Table (stage 3) | Find lists that are not migrated |
| Send Error Message (stage 3) | n8n-nodes-base.gmail | Emails Stage 3 error | If Asana Project Created | Stop Migration (stage 3) | Report an error |
| Send Success Message (stage 3) | n8n-nodes-base.gmail | Emails Stage 3 completion | Convert Lists To CSV (stage 3) | None | Send the mapping file |
| Create ClickUp Spaces Array (stage 3) (Find lists that are not migrated) | n8n-nodes-base.splitOut | Splits spaces for list migration | Get ClickUp Spaces (stage 3)... | Get ClickUp Folders (stage 3)... | Find lists that are not migrated |
| Update Lookup Table (stage 3) | n8n-nodes-base.dataTable | Saves list mapping | Wait for team, section and portfolio updates | Loop Over Lists (stage 3) | Assign team and portfolio |
| Wait for team, section and portfolio updates | n8n-nodes-base.merge | Synchronizes project assignments | Assign Team To Project, Create Section In Project, Connect Project To Portfolio / If Has Portfolio | Update Lookup Table (stage 3) | Assign team and portfolio |
| Collect all lists | n8n-nodes-base.aggregate | Collects processed lists | Loop Over Lists (stage 3) | Get Lookup Table Rows (stage 3) | Send the mapping file |
| Get Lookup Table Rows (stage 3) | n8n-nodes-base.dataTable | Reads all list mappings | Collect all lists | Convert Lists To CSV (stage 3) | Send the mapping file |
| Convert Lists To CSV (stage 3) | n8n-nodes-base.convertToFile | Converts list mapping to CSV | Get Lookup Table Rows (stage 3) | Send Success Message (stage 3) | Send the mapping file |
| Stop Migration (stage 3) | n8n-nodes-base.stopAndError | Stops Stage 3 on error | Send Error Message (stage 3) | None | Report an error |
| Section - Receive the settings (stage 3) | n8n-nodes-base.stickyNote | Documentation note | None | None | Receive the settings |
| Get ClickUp Workspace Custom Fields | n8n-nodes-base.httpRequest | Fetches workspace custom fields | If Table Doesn't Exists | Create Workspace Fields Array | Read workspace fields |
| Get List Custom Fields | n8n-nodes-base.httpRequest | Fetches list custom fields | Wait For ClickUp Rate-limit (Add fields to Asana projects) | Set List Data | Read list fields |
| Create Asana Project Text Field | n8n-nodes-base.httpRequest | Creates project text field | Check scope for Text field | Update ClickUpField-AsanaField Table (...) | Create text, number, currency and date fields |
| Create Asana Project Number Field | n8n-nodes-base.httpRequest | Creates project number field | Check scope for Number field | Update ClickUpField-AsanaField Table (...) | Create text, number, currency and date fields |
| Create Asana Project Currency Field | n8n-nodes-base.httpRequest | Creates project currency field | Check scope for Currency field | Update ClickUpField-AsanaField Table (...) | Create text, number, currency and date fields |
| Create Asana Project Data Field | n8n-nodes-base.httpRequest | Creates project date field | Check scope for Date field | Update ClickUpField-AsanaField Table (...) | Create text, number, currency and date fields |
| Create Asana Project Enum Field | n8n-nodes-base.httpRequest | Creates project enum field | Check scope for Enum field | Split Enum options (project field) | Create dropdown and label fields |
| Update ClickUpField-AsanaField Table (Create dropdown and label fields) | n8n-nodes-base.dataTable | Saves custom field mapping | Set * Options Data | Loop Switch | Create dropdown and label fields |
| Create Asana Project Multi Enum Field | n8n-nodes-base.httpRequest | Creates project multi-enum field | Check scope for Multi Enum field | Split Multi Enum options (project field) | Create dropdown and label fields |
| Map To Asana Enum Options | n8n-nodes-base.code | Maps enum colors | Field Type Switch (stage 4) | Check scope for Enum field | Create dropdown and label fields |
| Map To Asana Multi Enum Options | n8n-nodes-base.code | Maps multi-enum colors | Field Type Switch (stage 4) | Check scope for Multi Enum field | Create dropdown and label fields |
| Create Asana Project People Field | n8n-nodes-base.httpRequest | Creates project people field | Check scope for People field | Update ClickUpField-AsanaField Table (...) | Create people, checkbox, emoji and task fields |
| Create Asana Project Enum-CheckBox Field | n8n-nodes-base.httpRequest | Creates project checkbox field | Check scope for Enum-CheckBox field | Split Enum-CheckBox options (project field) | Create people, checkbox, emoji and task fields |
| Create Asana Project Emoji -Enum Field | n8n-nodes-base.httpRequest | Creates project emoji field | Check scope for Emoji-Enum field | Split Emoji-Enum options (project field) | Create people, checkbox, emoji and task fields |
| Create Asana Task Reference Field | n8n-nodes-base.httpRequest | Creates project task reference field | Check scope for Task Reference field | Update ClickUpField-AsanaField Table (...) | Create people, checkbox, emoji and task fields |
| Look up Asana project for list field data | n8n-nodes-base.dataTable | Finds Asana project GID | If Field Not Migrated (Add fields to Asana projects) | Set List Field Data | Add fields to Asana projects |
| Split Enum options (project field) | n8n-nodes-base.splitOut | Splits enum options | Create Asana Project Enum Field | Join Enum Options Data | Create dropdown and label fields |
| Set Enum Options Data | n8n-nodes-base.set | Sets enum options data | Join Enum Options Data | Update ClickUpField-AsanaField Table (...) | Create dropdown and label fields |
| Split Dropdown options (project field) | n8n-nodes-base.splitOut | Splits dropdown options | Field Type Switch (stage 4) | Join Enum Options Data | Create dropdown and label fields |
| Split Label options (project field) | n8n-nodes-base.splitOut | Splits label options | Field Type Switch (stage 4) | Join Multi Enum Options Data | Create dropdown and label fields |
| Split Multi Enum options (project field) | n8n-nodes-base.splitOut | Splits multi-enum options | Create Asana Project Multi Enum Field | Join Multi Enum Options Data | Create dropdown and label fields |
| Join Multi Enum Options Data | n8n-nodes-base.merge | Joins multi-enum options | Split Label options (project field), Split Multi Enum options (project field) | Set Multi Enum Options Data | Create dropdown and label fields |
| Join Enum Options Data | n8n-nodes-base.merge | Joins enum options | Split Enum options (project field), Split Dropdown options (project field) | Set Enum Options Data | Create dropdown and label fields |
| Set Multi Enum Options Data | n8n-nodes-base.set | Sets multi-enum options data | Join Multi Enum Options Data | Update ClickUpField-AsanaField Table (...) | Create dropdown and label fields |
| Map Emoji To Asana Enum Options | n8n-nodes-base.code | Generates emoji enum options | Field Type Switch (stage 4) | Check scope for Emoji-Enum field | Create people, checkbox, emoji and task fields |
| Split Emoji-Enum options (project field) | n8n-nodes-base.splitOut | Splits emoji options | Create Asana Project Emoji -Enum Field | Set Emoji -Enum Options Data | Create people, checkbox, emoji and task fields |
| Set Emoji -Enum Options Data | n8n-nodes-base.set | Sets emoji enum options data | Split Emoji-Enum options (project field) | Update ClickUpField-AsanaField Table (...) | Create people, checkbox, emoji and task fields |
| Search ClickUpField-AsanaField Lookup Table | n8n-nodes-base.dataTable | Queries field mapping table | Set Workflow Environment (stage 4) | If Table Doesn't Exists | Receive the settings |
| If Table Doesn't Exists | n8n-nodes-base.if | Checks field table existence | Search ClickUpField-AsanaField Lookup Table | Create ClickUpField-AsanaField Table, Get ClickUp Workspace Custom Fields / Spaces | Receive the settings |
| Create ClickUpField-AsanaField Lookup Table | n8n-nodes-base.dataTable | Creates field mapping table | If Table Doesn't Exists | Get ClickUp Workspace Custom Fields / Spaces | Receive the settings |
| Check scope for Text field | n8n-nodes-base.if | Checks scope for text fields | Field Type Switch (stage 4) | Create Asana Workspace Text Field, Create Asana Project Text Field | Create text, number, currency and date fields |
| Create Asana Workspace Text Field | n8n-nodes-base.httpRequest | Creates workspace text field | Check scope for Text field | Update ClickUpField-AsanaField Table (...) | Create text, number, currency and date fields |
| Check scope for Number field | n8n-nodes-base.if | Checks scope for number fields | Field Type Switch (stage 4) | Create Asana Workspace Number Field, Create Asana Project Number Field | Create text, number, currency and date fields |
| Create Asana Workspace Number Field | n8n-nodes-base.httpRequest | Creates workspace number field | Check scope for Number field | Update ClickUpField-AsanaField Table (...) | Create text, number, currency and date fields |
| Check scope for Currency field | n8n-nodes-base.if | Checks scope for currency fields | Field Type Switch (stage 4) | Create Asana Workspace Currency Field, Create Asana Project Currency Field | Create text, number, currency and date fields |
| Create Asana Workspace Currency Field | n8n-nodes-base.httpRequest | Creates workspace currency field | Check scope for Currency field | Update ClickUpField-AsanaField Table (...) | Create text, number, currency and date fields |
| Check scope for Enum field | n8n-nodes-base.if | Checks scope for enum fields | Map To Asana Enum Options | Create Asana Workspace Enum Field, Create Asana Project Enum Field | Create dropdown and label fields |
| Split Enum options (workspace field) | n8n-nodes-base.splitOut | Splits workspace enum options | Create Asana Workspace Enum Field | Join Enum Options Data | Create dropdown and label fields |
| Create Asana Workspace Enum Field | n8n-nodes-base.httpRequest | Creates workspace enum field | Check scope for Enum field | Split Enum options (workspace field) | Create dropdown and label fields |
| Check scope for Date field | n8n-nodes-base.if | Checks scope for date fields | Field Type Switch (stage 4) | Create Asana Workspace Date Field, Create Asana Project Data Field | Create text, number, currency and date fields |
| Create Asana Workspace Date Field | n8n-nodes-base.httpRequest | Creates workspace date field | Check scope for Date field | Update ClickUpField-AsanaField Table (...) | Create text, number, currency and date fields |
| Check scope for Multi Enum field | n8n-nodes-base.if | Checks scope for multi-enum fields | Map To Asana Multi Enum Options | Create Asana Workspace Multi Enum Field, Create Asana Project Multi Enum Field | Create dropdown and label fields |
| Create Asana Workspace Multi Enum Field | n8n-nodes-base.httpRequest | Creates workspace multi-enum field | Check scope for Multi Enum field | Split Multi Enum options (workspace field) | Create dropdown and label fields |
| Split Multi Enum options (workspace field) | n8n-nodes-base.splitOut | Splits workspace multi-enum options | Create Asana Workspace Multi Enum Field | Join Multi Enum Options Data | Create dropdown and label fields |
| Check scope for People field | n8n-nodes-base.if | Checks scope for people fields | Field Type Switch (stage 4) | Create Asana Workspace People Field, Create Asana Project People Field | Create people, checkbox, emoji and task fields |
| Create Asana Workspace People Field | n8n-nodes-base.httpRequest | Creates workspace people field | Check scope for People field | Update ClickUpField-AsanaField Table (...) | Create people, checkbox, emoji and task fields |
| Check scope for Enum-CheckBox field | n8n-nodes-base.if | Checks scope for checkbox fields | Field Type Switch (stage 4) | Create Asana Workspace Enum-CheckBox Field, Create Asana Project Enum-CheckBox Field | Create people, checkbox, emoji and task fields |
| Create Asana Workspace Enum-CheckBox Field | n8n-nodes-base.httpRequest | Creates workspace checkbox field | Check scope for Enum-CheckBox field | Split Enum-CheckBox options (workspace field) | Create people, checkbox, emoji and task fields |
| Check scope for Task Reference field | n8n-nodes-base.if | Checks scope for task reference fields | Field Type Switch (stage 4) | Create Asana Workspace Task Reference Field, Create Asana Task Reference Field | Create people, checkbox, emoji and task fields |
| Create Asana Workspace Task Reference Field | n8n-nodes-base.httpRequest | Creates workspace task reference field | Check scope for Task Reference field | Update ClickUpField-AsanaField Table (...) | Create people, checkbox, emoji and task fields |
| Check scope for Emoji-Enum field | n8n-nodes-base.if | Checks scope for emoji fields | Map Emoji To Asana Enum Options | Create Asana Workspace Emoji -Enum Field, Create Asana Project Emoji -Enum Field | Create people, checkbox, emoji and task fields |
| Split Emoji-Enum options (workspace field) | n8n-nodes-base.splitOut | Splits workspace emoji options | Create Asana Workspace Emoji -Enum Field | Set Emoji -Enum Options Data | Create people, checkbox, emoji and task fields |
| Loop Over Folders | n8n-nodes-base.splitInBatches | Batches folder processing | Create ClickUp Folders Array (stage 4) | Create Folder Fields Array, Wait For ClickUp Rate-limit (Read folder fields) | Read folder fields |
| Create Workspace Fields Array | n8n-nodes-base.splitOut | Splits workspace fields | Get ClickUp Workspace Custom Fields | Loop over workspace fields | Read workspace fields |
| Loop over workspace fields | n8n-nodes-base.splitInBatches | Batches workspace fields | Create Workspace Fields Array | Wait for all field loops to finish, If Workspace Custom Field Exist | Read workspace fields |
| Set Workspace Field Data | n8n-nodes-base.set | Formats workspace field data | If Field Not Migrated (Read workspace fields) | Field Type Switch (stage 4) | Read workspace fields |
| Get Folder Custom Fields | n8n-nodes-base.httpRequest | Fetches folder custom fields | Wait For ClickUp Rate-limit (Read folder fields) | Set Folder Data | Read folder fields |
| Create Folder Fields Array | n8n-nodes-base.splitOut | Splits folder fields | Loop Over Folders | Loop over folder fields | Read folder fields |
| Loop over folder fields | n8n-nodes-base.splitInBatches | Batches folder fields | Create Folder Fields Array | Wait for all field loops to finish, If Folder Custom Field Exist | Read folder fields |
| Set Folder Field Data | n8n-nodes-base.set | Formats folder field data | If Field Not Migrated (Read folder fields) | Field Type Switch (stage 4) | Read folder fields |
| Create List Fields Array | n8n-nodes-base.splitOut | Splits list fields | Loop Over Lists (stage 4) | Loop Over List Fields | Read list fields |
| If List Custom Fields Exist | n8n-nodes-base.if | Checks list custom fields | Loop Over List Fields | Lookup Migrated Fields Data (Add fields to Asana projects), Loop Over List Fields | Read list fields |
| Loop Over List Fields | n8n-nodes-base.splitInBatches | Batches list custom fields | Create List Fields Array, Add Custom Field To Asana Project, If Upper Scope Field, Check custom field already added | Wait for all field loops to finish, If List Custom Fields Exist | Read list fields |
| Set List Field Data | n8n-nodes-base.set | Formats list field data | Look up Asana project for list field data | Field Type Switch (stage 4) | Read list fields |
| Loop Switch | n8n-nodes-base.switch | Routes loop iterations | Update ClickUpField-AsanaField Table (...), Field Type Switch (stage 4) | Loop over workspace/space/folder fields, Loop Over List Fields | Create dropdown and label fields |
| Field Type Switch (stage 4) | n8n-nodes-base.switch | Routes custom field types | Set * Field Data | Check scope for *, Map To Asana *, Loop Switch | Create text, number, currency and date fields |
| Create Asana Workspace Emoji -Enum Field | n8n-nodes-base.httpRequest | Creates workspace emoji field | Check scope for Emoji-Enum field | Split Emoji-Enum options (workspace field) | Create people, checkbox, emoji and task fields |
| Lookup Migrated Fields Data (Read workspace fields) | n8n-nodes-base.dataTable | Queries workspace field mapping | If Workspace Custom Field Exist | If Field Not Migrated (Read workspace fields) | Read workspace fields |
| Lookup Migrated Fields Data (Read folder fields) | n8n-nodes-base.dataTable | Queries folder field mapping | If Folder Custom Field Exist | If Field Not Migrated (Read folder fields) | Read folder fields |
| Lookup Migrated Fields Data (Add fields to Asana projects) | n8n-nodes-base.dataTable | Queries list field mapping | If List Custom Fields Exist | If Field Not Migrated (Add fields to Asana projects) | Read list fields |
| If Field Not Migrated (Read workspace fields) | n8n-nodes-base.if | Checks if workspace field migrated | Lookup Migrated Fields Data (Read workspace fields) | Get ClickUp Workspaces, Loop over workspace fields | Read workspace fields |
| If Field Not Migrated (Read folder fields) | n8n-nodes-base.if | Checks if folder field migrated | Lookup Migrated Fields Data (Read folder fields) | Set Folder Field Data, Loop over folder fields | Read folder fields |
| If Field Not Migrated (Add fields to Asana projects) | n8n-nodes-base.if | Checks if list field migrated | Lookup Migrated Fields Data (Add fields to Asana projects) | Look up Asana project for list field data, If Upper Scope Field | Read list fields |
| Wait For ClickUp Rate-limit (Read space fields) | n8n-nodes-base.wait | Throttles ClickUp requests | Loop Over Spaces (stage 4)... | Get Spaces Custom Fields | Read space fields |
| Lookup Migrated Fields Data (Read space fields) | n8n-nodes-base.dataTable | Queries space field mapping | If Space Custom Field Exist | If Field Not Migrated (Read space fields) | Read space fields |
| If Field Not Migrated (Read space fields) | n8n-nodes-base.if | Checks if space field migrated | Lookup Migrated Fields Data (Read space fields) | Set Space Field Data, Loop over space fields | Read space fields |
| Get Spaces Custom Fields | n8n-nodes-base.httpRequest | Fetches space custom fields | Wait For ClickUp Rate-limit (Read space fields) | Set Space Data | Read space fields |
| Create Space Fields Array | n8n-nodes-base.splitOut | Splits space fields | Loop Over Spaces (stage 4) (Read space fields) | Loop over space fields | Read space fields |
| Loop over space fields | n8n-nodes-base.splitInBatches | Batches space fields | Create Space Fields Array | Wait for all field loops to finish, If Space Custom Field Exist | Read space fields |
| Set Space Field Data | n8n-nodes-base.set | Formats space field data | If Field Not Migrated (Read space fields) | Field Type Switch (stage 4) | Read space fields |
| Get ClickUp Workspaces | n8n-nodes-base.httpRequest | Fetches ClickUp team info | If Field Not Migrated (Read workspace fields) | Set Workspace Field Data | Read workspace fields |
| Wait For ClickUp Rate-limit (Read folder fields) | n8n-nodes-base.wait | Throttles ClickUp requests | Loop Over Folders | Get Folder Custom Fields | Read folder fields |
| Set Space Data | n8n-nodes-base.set | Formats space data | Get Spaces Custom Fields | Loop Over Spaces (stage 4) (Read space fields) | Read space fields |
| Set Folder Data | n8n-nodes-base.set | Formats folder data | Get Folder Custom Fields | Loop Over Folders | Read folder fields |
| If Space Custom Field Exist | n8n-nodes-base.if | Checks space custom fields | Loop over space fields | Lookup Migrated Fields Data (Read space fields), Loop over space fields | Read space fields |
| If Folder Custom Field Exist | n8n-nodes-base.if | Checks folder custom fields | Loop over folder fields | Lookup Migrated Fields Data (Read folder fields), Loop over folder fields | Read folder fields |
| If Workspace Custom Field Exist | n8n-nodes-base.if | Checks workspace custom fields | Loop over workspace fields | Lookup Migrated Fields Data (Read workspace fields), Loop over workspace fields | Read workspace fields |
| Wait For ClickUp Rate-limit (Add fields to Asana projects) | n8n-nodes-base.wait | Throttles ClickUp requests | Loop Over Lists (stage 4) | Get List Custom Fields | Read list fields |
| Set List Data | n8n-nodes-base.set | Formats list data | Get List Custom Fields | Loop Over Lists (stage 4) | Read list fields |
| Get ClickUp Folder Lists (stage 4) | n8n-nodes-base.httpRequest | Fetches folder lists | Create ClickUp Folders Array (stage 4) | Create ClickUp Lists Array (stage 4) | Read list fields |
| If Upper Scope Field | n8n-nodes-base.if | Checks if field scope > list | If Field Not Migrated (Add fields to Asana projects) | Look up Asana project to add custom field, Loop Over List Fields | Add fields to Asana projects |
| Add Custom Field To Asana Project | n8n-nodes-base.httpRequest | Adds custom field to project | Look up Asana project to add custom field | Loop Over List Fields, Check custom field already added | Add fields to Asana projects |
| Look up Asana project to add custom field | n8n-nodes-base.dataTable | Finds Asana project GID | If Upper Scope Field | Add Custom Field To Asana Project | Add fields to Asana projects |
| Check custom field already added | n8n-nodes-base.if | Checks if field attached | Add Custom Field To Asana Project | Loop Over List Fields | Add fields to Asana projects |
| If Space Priorities Enabled (Migrate space priorities) | n8n-nodes-base.if | Checks space priority setting | Loop Over Spaces (stage 4) (Migrate space priorities) | Lookup Priority Fields Data, Loop Over Spaces (stage 4) (Migrate space priorities) | Migrate space priorities |
| Lookup Priority Fields Data | n8n-nodes-base.dataTable | Queries priority field mapping | If Space Priorities Enabled (Migrate space priorities) | If Priority Not Migrated | Migrate space priorities |
| If Priority Not Migrated | n8n-nodes-base.if | Checks if priority migrated | Lookup Priority Fields Data | Create Workspace Priorities In Asana, Loop Over Spaces (stage 4) (Migrate space priorities) | Migrate space priorities |
| Create Workspace Priorities In Asana | n8n-nodes-base.httpRequest | Creates priority field in Asana | If Priority Not Migrated | Split Priority options (workspace field) | Migrate space priorities |
| Loop Over Spaces (stage 4) (Migrate space priorities) | n8n-nodes-base.splitInBatches | Batches space priority migration | Create ClickUp Spaces Array (stage 4) | Wait for all field loops to finish, If Space Priorities Enabled (Migrate space priorities) | Migrate space priorities |
| Update ClickUpField-AsanaField Table (Migrate space priorities) | n8n-nodes-base.dataTable | Saves priority field mapping | Split Priority options (workspace field) | Loop Over Spaces (stage 4) (Migrate space priorities) | Migrate space priorities |
| Split Priority options (workspace field) | n8n-nodes-base.splitOut | Splits priority options | Create Workspace Priorities In Asana | Update ClickUpField-AsanaField Table (Migrate space priorities) | Migrate space priorities |
| Split Enum-CheckBox options (workspace field) | n8n-nodes-base.splitOut | Splits workspace checkbox options | Create Asana Workspace Enum-CheckBox Field | Set Enum-CheckBox Options Data | Create people, checkbox, emoji and task fields |
| Split Enum-CheckBox options (project field) | n8n-nodes-base.splitOut | Splits project checkbox options | Create Asana Project Enum-CheckBox Field | Set Enum-CheckBox Options Data | Create people, checkbox, emoji and task fields |
| Set Enum-CheckBox Options Data | n8n-nodes-base.set | Formats checkbox option data | Split Enum-CheckBox options (*) | Update ClickUpField-AsanaField Table (...) | Create people, checkbox, emoji and task fields |
| Get List's Space Data | n8n-nodes-base.httpRequest | Fetches list space details | Loop Over Lists (stage 4) | If Space Priorities Enabled (Add fields to Asana projects) | Read list fields |
| If Space Priorities Enabled (Add fields to Asana projects) | n8n-nodes-base.if | Checks space priorities | Get List's Space Data | Lookup Priority Field Data, Set Priority Flag | Add fields to Asana projects |
| Check priority field already added | n8n-nodes-base.if | Checks if priority attached | Add Priority Field To Asana Project | Set Priority Flag | Add fields to Asana projects |
| Lookup Priority Field Data | n8n-nodes-base.dataTable | Queries priority field mapping | If Space Priorities Enabled (Add fields to Asana projects) | Look up Asana project to add priority field | Add fields to Asana projects |
| Look up Asana project to add priority field | n8n-nodes-base.dataTable | Finds Asana project GID | Lookup Priority Field Data | Add Priority Field To Asana Project | Add fields to Asana projects |
| Add Priority Field To Asana Project | n8n-nodes-base.httpRequest | Attaches priority to project | Look up Asana project to add priority field | Set Priority Flag, Check priority field already added | Add fields to Asana projects |
| Set Priority Flag | n8n-nodes-base.set | Sets priority flag | If Space Priorities Enabled..., Add Priority Field To Asana Project, Check priority field already added | Wait For ClickUp Rate-limit (Add fields to Asana projects) | Add fields to Asana projects |
| Section - Read workspace fields (stage 4) | n8n-nodes-base.stickyNote | Documentation note | None | None | Read workspace fields |
| Section - Read space fields (stage 4) | n8n-nodes-base.stickyNote | Documentation note | None | None | Read space fields |
| Section - Migrate space priorities (stage 4) | n8n-nodes-base.stickyNote | Documentation note | None | None | Migrate space priorities |
| Section - Read folder fields (stage 4) | n8n-nodes-base.stickyNote | Documentation note | None | None | Read folder fields |
| Section - Read list fields (stage 4) | n8n-nodes-base.stickyNote | Documentation note | None | None | Read list fields |
| Section - Add fields to Asana projects (stage 4) | n8n-nodes-base.stickyNote | Documentation note | None | None | Add fields to Asana projects |
| Section - Create text, number, currency and date fields (stage 4) | n8n-nodes-base.stickyNote | Documentation note | None | None | Create text, number, currency and date fields |
| Section - Create dropdown and label fields (stage 4) | n8n-nodes-base.stickyNote | Documentation note | None | None | Create dropdown and label fields |
| Section - Create people, checkbox, emoji and task fields (stage 4) | n8n-nodes-base.stickyNote | Documentation note | None | None | Create people, checkbox, emoji and task fields |
| Section - Send the mapping file (stage 4) | n8n-nodes-base.stickyNote | Documentation note | None | None | Send the mapping file |
| Get ClickUp Spaces (stage 4) | n8n-nodes-base.httpRequest | Fetches spaces for custom fields | If Table Doesn't Exists | Create ClickUp Spaces Array (stage 4) | Read workspace fields |
| Get ClickUp Folders (stage 4) | n8n-nodes-base.httpRequest | Fetches folders for custom fields | Create ClickUp Spaces Array (stage 4) | Create ClickUp Folders Array (stage 4) | Read folder fields |
| Get ClickUp Folderless Lists (stage 4) | n8n-nodes-base.httpRequest | Fetches folderless lists for fields | Create ClickUp Spaces Array (stage 4) | Create ClickUp Folderless Lists Array (stage 4) | Read list fields |
| Create ClickUp Spaces Array (stage 4) | n8n-nodes-base.splitOut | Splits spaces for fields | Get ClickUp Spaces (stage 4) | Loop Over Spaces (stage 4)... | Read workspace fields |
| Create ClickUp Folders Array (stage 4) | n8n-nodes-base.splitOut | Splits folders for fields | Get ClickUp Folders (stage 4) | Loop Over Folders | Read folder fields |
| Create ClickUp Lists Array (stage 4) | n8n-nodes-base.splitOut | Splits list items for fields | Get ClickUp Folder Lists (stage 4) | Join ClickUp Lists (stage 4) | Read list fields |
| Create ClickUp Folderless Lists Array (stage 4) | n8n-nodes-base.splitOut | Splits folderless lists for fields | Get ClickUp Folderless Lists (stage 4) | Join ClickUp Lists (stage 4) | Read list fields |
| Join ClickUp Lists (stage 4) | n8n-nodes-base.merge | Combines list arrays for fields | Create ClickUp Lists / Folderless Arrays | Loop Over Lists (stage 4) | Read list fields |
| Loop Over Lists (stage 4) | n8n-nodes-base.splitInBatches | Batches list field processing | Join ClickUp Lists (stage 4) | Create List Fields Array, Get List's Space Data | Read list fields |
| Set Workflow Environment (stage 4) | n8n-nodes-base.set | Sets Stage 4 environment | Trigger | Search Table | Receive the settings |
| Send Success Message (stage 4) | n8n-nodes-base.gmail | Emails Stage 4 completion | Convert Fields To CSV (stage 4) | None | Send the mapping file |
| Wait for all field loops to finish | n8n-nodes-base.merge | Synchronizes field loops | Loop over workspace/space/folder fields, Loop Over List Fields, Loop Over Spaces (stage 4)... | Collect all field results | Add fields to Asana projects |
| Loop Over Spaces (stage 4) (Read space fields) | n8n-nodes-base.splitInBatches | Batches space field processing | Create ClickUp Spaces Array (stage 4) | Create Space Fields Array, Wait For ClickUp Rate-limit (Read space fields) | Read space fields |
| Collect all field results | n8n-nodes-base.aggregate | Collects field processing results | Wait for all field loops to finish | Get Lookup Table Rows (stage 4) | Send the mapping file |
| Get Lookup Table Rows (stage 4) | n8n-nodes-base.dataTable | Reads field mapping rows | Collect all field results | Convert Fields To CSV (stage 4) | Send the mapping file |
| Convert Fields To CSV (stage 4) | n8n-nodes-base.convertToFile | Converts field mapping to CSV | Get Lookup Table Rows (stage 4) | Send Success Message (stage 4) | Send the mapping file |
| Section - Receive the settings (stage 4) | n8n-nodes-base.stickyNote | Documentation note | None | None | Receive the settings |
| Get All Rows | n8n-nodes-base.dataTable | Reads all task mapping rows | If Table Exists (stage 5) | Get Unmatched ClickUp Tasks | Prepare the mapping table |
| Search ClickUpTask-AsanaTask Lookup Table | n8n-nodes-base.dataTable | Queries task mapping table | Set Workflow Environment (stage 5) | If Table Exists (stage 5) | Prepare the mapping table |
| Create Tasks Array | n8n-nodes-base.splitOut | Splits tasks for filtering | Get All ClickUp Tasks And Subtasks | Get Unmatched ClickUp Tasks | Read the ClickUp tasks |
| Get Unmatched ClickUp Tasks | n8n-nodes-base.merge | Filters unmigrated tasks | Create Tasks Array, Get All Rows | Sort Subtasks After Tasks | Read the ClickUp tasks |
| Add Task To Section | n8n-nodes-base.httpRequest | Moves task to section | If Not Subtask | Get ClickUp Task Data | Create the Asana tasks |
| Loop Over Tasks | n8n-nodes-base.splitInBatches | Batches task processing | Sort Subtasks After Tasks | Aggregate Tasks, If No Parent Task | Read the ClickUp tasks |
| Get Sections In Asana Project | n8n-nodes-base.httpRequest | Fetches project sections | Lookup Asana Project | Set Essential Asana Task Data | Create the Asana tasks |
| Get ClickUp Task Data | n8n-nodes-base.httpRequest | Fetches task details/subtasks | Add Task To Section / If Not Subtask | Set Asana Task ID | Create the Asana tasks |
| Create Fields Array | n8n-nodes-base.splitOut | Splits custom field values | If Any Field Values | Filter Empty Fields | Copy custom field values |
| Get Asana Text Field ID | n8n-nodes-base.dataTable | Retrieves text field GID | Field Type Switch (stage 5) | Set Asana Text Field | Copy custom field values |
| Set Asana Text Field | n8n-nodes-base.set | Formats text field value | Get Asana Text Field ID | Merge Field Values | Copy custom field values |
| Get Asana Number Field ID | n8n-nodes-base.dataTable | Retrieves number field GID | Field Type Switch (stage 5) | Set Asana Number Field | Copy custom field values |
| Set Asana Number Field | n8n-nodes-base.set | Formats number field value | Get Asana Number Field ID | Merge Field Values | Copy custom field values |
| Get Asana Date Field ID | n8n-nodes-base.dataTable | Retrieves date field GID | Field Type Switch (stage 5) | Set Asana Date Field | Copy custom field values |
| Set Asana Date Field | n8n-nodes-base.set | Formats date field value | Get Asana Date Field ID | Merge Field Values | Copy custom field values |
| Get Asana Currency Field ID | n8n-nodes-base.dataTable | Retrieves currency field GID | Field Type Switch (stage 5) | Set Asana Currency Field | Copy custom field values |
| Set Asana Currency Field | n8n-nodes-base.set | Formats currency field value | Get Asana Currency Field ID | Merge Field Values | Copy custom field values |
| Set Asana Enum Field | n8n-nodes-base.set | Formats enum field value | Get Asana Enum Option ID | Merge Field Values | Copy custom field values |
| Set Asana Multi Enum Field | n8n-nodes-base.set | Formats multi-enum value | Get Asana Multi Enum Field ID | Merge Field Values | Copy custom field values |
| Create Label Array | n8n-nodes-base.splitOut | Splits label values | Field Type Switch (stage 5) | Get Asana Multi Enum Option ID | Copy custom field values |
| Get Asana Multi Enum Option ID | n8n-nodes-base.dataTable | Retrieves multi-enum option GID | Create Label Array | Create Asana Multi Enum Options Array | Copy custom field values |
| Get Asana Enum Option ID | n8n-nodes-base.dataTable | Retrieves enum option GID | Field Type Switch (stage 5) | Set Asana Enum Field | Copy custom field values |
| Create Asana Multi Enum Options Array | n8n-nodes-base.aggregate | Aggregates multi-enum option IDs | Get Asana Multi Enum Option ID | Get Asana Multi Enum Field ID | Copy custom field values |
| Get Asana Multi Enum Field ID | n8n-nodes-base.dataTable | Retrieves multi-enum field GID | Create Asana Multi Enum Options Array | Set Asana Multi Enum Field | Copy custom field values |
| Get Asana People Field ID | n8n-nodes-base.dataTable | Retrieves people field GID | Field Type Switch (stage 5) | Set Asana People Field | Copy custom field values |
| Set Asana People Field | n8n-nodes-base.set | Formats people field value | Get Asana People Field ID | Merge Field Values | Copy custom field values |
| Get Asana Checkbox Enum Field ID | n8n-nodes-base.dataTable | Retrieves checkbox option GID | Field Type Switch (stage 5) | Set Asana Checkbox Enum Field | Copy custom field values |
| Add Asana Task Field Values | n8n-nodes-base.httpRequest | Writes custom field values to Asana task | Collect task field values | If Priority Exists | Copy custom field values |
| Get Asana Emoji-Enum Option ID | n8n-nodes-base.dataTable | Retrieves emoji option GID | Field Type Switch (stage 5) | Set Asana Emoji-Enum Field | Copy custom field values |
| Set Asana Emoji-Enum Field | n8n-nodes-base.set | Formats emoji field value | Get Asana Emoji-Enum Option ID | Merge Field Values | Copy custom field values |
| Set Asana Checkbox Enum Field | n8n-nodes-base.set | Formats checkbox field value | Get Asana Checkbox Enum Field ID | Merge Field Values | Copy custom field values |
| Filter Empty Fields | n8n-nodes-base.filter | Filters out empty custom fields | Create Fields Array | Field Type Switch (stage 5) | Copy custom field values |
| If Any Field Values | n8n-nodes-base.if | Checks if task has custom fields | Set Asana Task ID | Create Fields Array, If Priority Exists | Copy custom field values |
| Get Asana Priority Field ID | n8n-nodes-base.dataTable | Retrieves priority field GID | If Priority Exists | Set Asana Priority Field | Copy the priority |
| Set Asana Priority Field | n8n-nodes-base.set | Formats priority field value | Get Asana Priority Field ID | Set Asana Task Priority | Copy the priority |
| Get ClickUp Task Comments | n8n-nodes-base.httpRequest | Fetches task comments | If Priority Exists / Set Asana Task Priority | If Any Comments | Copy comments |
| If Any Comments | n8n-nodes-base.if | Checks if comments exist | Get ClickUp Task Comments | Create Comments Array, If Checklists Exist | Copy comments |
| Create Comments Array | n8n-nodes-base.splitOut | Splits comment items | If Any Comments | Set Asana Comment Data | Copy comments |
| Create Asana Task Comment | n8n-nodes-base.httpRequest | Creates comment in Asana | Set Asana Comment Data | If Checklists Exist | Copy comments |
| If Checklists Exist | n8n-nodes-base.if | Checks if checklists exist | If Any Comments / Create Asana Task Comment | Set ClickUp Checklist Data, If Attachments Exist | Copy checklists |
| Create Checklists Array | n8n-nodes-base.splitOut | Splits checklist items | Set ClickUp Checklist Data | Create Checklist Items Array | Copy checklists |
| Create Checklist Items Array | n8n-nodes-base.splitOut | Splits checklist sub-items | Create Checklists Array | Set Asana Checklist Subtasks Data | Copy checklists |
| If Attachments Exist | n8n-nodes-base.if | Checks if attachments exist | If Checklists Exist / Creat Asana Checklist Subtask | Set ClickUp Attachment Data, Update Lookup Table (stage 5) | Copy attachments |
| Create Attachments Array | n8n-nodes-base.splitOut | Splits attachment items | If Attachments Exist | Download Attachment | Copy attachments |
| Download Attachment | n8n-nodes-base.httpRequest | Downloads attachment from ClickUp | Create Attachments Array | Upload Attachment To Asana Task | Copy attachments |
| Upload Attachment To Asana Task | n8n-nodes-base.httpRequest | Uploads attachment to Asana task | Download Attachment | Update Lookup Table (stage 5) | Copy attachments |
| If Priority Exists | n8n-nodes-base.if | Checks if task has priority | Add Asana Task Field Values | Get Asana Priority Field ID, Get ClickUp Task Comments | Copy the priority |
| Set Asana Task Priority | n8n-nodes-base.httpRequest | Sets priority on Asana task | Set Asana Priority Field | Get ClickUp Task Comments | Copy the priority |
| Merge Field Values | n8n-nodes-base.merge | Combines custom field values | Set Asana * Field | Collect task field values | Copy custom field values |
| Set Essential Asana Task Data | n8n-nodes-base.set | Prepares core task parameters | Get Sections In Asana Project | Create Asana Task | Create the Asana tasks |
| Set Asana Task ID | n8n-nodes-base.set | Stores created Asana task GID | Get ClickUp Task Data | If Any Field Values | Create the Asana tasks |
| Set Asana Comment Data | n8n-nodes-base.set | Formats comment text | Create Comments Array | Create Asana Task Comment | Copy comments |
| Set Asana Checklist Subtasks Data | n8n-nodes-base.set | Formats checklist subtask data | Create Checklist Items Array | Creat Asana Checklist Subtask | Copy checklists |
| Creat Asana Checklist Subtask | n8n-nodes-base.httpRequest | Creates checklist item as subtask | Set Asana Checklist Subtasks Data | If Attachments Exist | Copy checklists |
| Set ClickUp Checklist Data | n8n-nodes-base.set | Extracts checklist array | If Checklists Exist | Create Checklists Array | Copy checklists |
| Set ClickUp Attachment Data | n8n-nodes-base.set | Extracts attachment array | If Attachments Exist | Create Attachments Array | Copy attachments |
| If No Parent Task | n8n-nodes-base.if | Checks if task is a subtask | Loop Over Tasks | Lookup Asana Project, Get Parent Asana Task | Read the ClickUp tasks |
| Get Parent Asana Task | n8n-nodes-base.dataTable | Retrieves parent task GID | If No Parent Task | Set Essential Asana Task Data | Read the ClickUp tasks |
| Sort Subtasks After Tasks | n8n-nodes-base.code | Sorts tasks by hierarchy depth | Get Unmatched ClickUp Tasks | Loop Over Tasks | Read the ClickUp tasks |
| Get All ClickUp Tasks And Subtasks | n8n-nodes-base.httpRequest | Fetches tasks and subtasks | Join ClickUp Lists (stage 5) | Filter Empty ClickUp Lists | Read the ClickUp tasks |
| Filter Empty ClickUp Lists | n8n-nodes-base.filter | Filters empty task lists | Get All ClickUp Tasks And Subtasks | Create Tasks Array | Read the ClickUp tasks |
| Create Asana Task | n8n-nodes-base.httpRequest | Creates task in Asana | Set Essential Asana Task Data | If Not Subtask | Create the Asana tasks |
| If Not Subtask | n8n-nodes-base.if | Checks if task is top-level | Create Asana Task | Add Task To Section, Get ClickUp Task Data | Create the Asana tasks |
| Stage 5 trigger | n8n-nodes-base.executeWorkflowTrigger | Trigger for Stage 5 sub-workflow | None | Set Workflow Environment (stage 5) | Receive the settings |
| Aggregate Tasks | n8n-nodes-base.aggregate | Collects processed tasks | Loop Over Tasks | Get Lookup Table Rows (stage 5) | Send the mapping file |
| Section - Prepare the mapping table (stage 5) | n8n-nodes-base.stickyNote | Documentation note | None | None | Prepare the mapping table |
| Section - Read the ClickUp tasks (stage 5) | n8n-nodes-base.stickyNote | Documentation note | None | None | Read the ClickUp tasks |
| Section - Create the Asana tasks (stage 5) | n8n-nodes-base.stickyNote | Documentation note | None | None | Create the Asana tasks |
| Section - Copy custom field values (stage 5) | n8n-nodes-base.stickyNote | Documentation note | None | None | Copy custom field values |
| Section - Copy the priority (stage 5) | n8n-nodes-base.stickyNote | Documentation note | None | None | Copy the priority |
| Section - Copy comments (stage 5) | n8n-nodes-base.stickyNote | Documentation note | None | None | Copy comments |
| Section - Copy checklists (stage 5) | n8n-nodes-base.stickyNote | Documentation note | None | None | Copy checklists |
| Section - Copy attachments (stage 5) | n8n-nodes-base.stickyNote | Documentation note | None | None | Copy attachments |
| Section - Save the task mapping (stage 5) | n8n-nodes-base.stickyNote | Documentation note | None | None | Save the task mapping |
| Create Lookup Table (stage 5) | n8n-nodes-base.dataTable | Creates task mapping table | If Table Exists (stage 5) | Get Unmatched ClickUp Tasks | Prepare the mapping table |
| If Table Exists (stage 5) | n8n-nodes-base.if | Checks task table existence | Search ClickUpTask-AsanaTask Lookup Table | Get All Rows, Create Lookup Table (stage 5) | Prepare the mapping table |
| Get ClickUp Spaces (stage 5) | n8n-nodes-base.httpRequest | Fetches spaces for task migration | Set Workflow Environment (stage 5) | Create ClickUp Spaces Array (stage 5) | Read the ClickUp tasks |
| Get ClickUp Folders (stage 5) | n8n-nodes-base.httpRequest | Fetches folders for task migration | Create ClickUp Spaces Array (stage 5) | Create ClickUp Folders Array (stage 5) | Read the ClickUp tasks |
| Get ClickUp Folderless Lists (stage 5) | n8n-nodes-base.httpRequest | Fetches folderless lists for tasks | Create ClickUp Spaces Array (stage 5) | Create ClickUp Folderless Lists Array (stage 5) | Read the ClickUp tasks |
| Create ClickUp Spaces Array (stage 5) | n8n-nodes-base.splitOut | Splits spaces for tasks | Get ClickUp Spaces (stage 5) | Get ClickUp Folders (stage 5), Get ClickUp Folderless Lists (stage 5) | Read the ClickUp tasks |
| Create ClickUp Folders Array (stage 5) | n8n-nodes-base.splitOut | Splits folders for tasks | Get ClickUp Folders (stage 5) | Get ClickUp Folder Lists (stage 5) | Read the ClickUp tasks |
| Create ClickUp Lists Array (stage 5) | n8n-nodes-base.splitOut | Splits lists for tasks | Get ClickUp Folder Lists (stage 5) | Join ClickUp Lists (stage 5) | Read the ClickUp tasks |
| Create ClickUp Folderless Lists Array (stage 5) | n8n-nodes-base.splitOut | Splits folderless lists for tasks | Get ClickUp Folderless Lists (stage 5) | Join ClickUp Lists (stage 5) | Read the ClickUp tasks |
| Join ClickUp Lists (stage 5) | n8n-nodes-base.merge | Combines lists for tasks | Create ClickUp Lists / Folderless Arrays | Get All ClickUp Tasks And Subtasks | Read the ClickUp tasks |
| Get ClickUp Folder Lists (stage 5) | n8n-nodes-base.httpRequest | Fetches lists inside folders | Create ClickUp Folders Array (stage 5) | Create ClickUp Lists Array (stage 5) | Read the ClickUp tasks |
| Set Workflow Environment (stage 5) | n8n-nodes-base.set | Sets Stage 5 environment | Stage 5 trigger | Get ClickUp Spaces (stage 5), Search Table | Receive the settings |
| Lookup Asana Project | n8n-nodes-base.dataTable | Finds Asana project GID for task | If No Parent Task | Get Sections In Asana Project | Read the ClickUp tasks |
| Send Success Message (stage 5) | n8n-nodes-base.gmail | Emails Stage 5 completion | Convert Tasks To CSV (stage 5) | None | Send the mapping file |
| Field Type Switch (stage 5) | n8n-nodes-base.switch | Routes task custom field types | Filter Empty Fields | Get Asana * Field ID / Option ID | Copy custom field values |
| Update Lookup Table (stage 5) | n8n-nodes-base.dataTable | Saves task mapping | If Attachments Exist, Upload Attachment... | Loop Over Tasks | Save the task mapping |
| Get Asana Currency Field ID | n8n-nodes-base.dataTable | Retrieves currency field GID | Field Type Switch (stage 5) | Set Asana Currency Field | Copy custom field values |
| Get Asana Priority Field ID | n8n-nodes-base.dataTable | Retrieves priority field GID | If Priority Exists | Set Asana Priority Field | Copy the priority |
| Set Asana Priority Field | n8n-nodes-base.set | Formats priority field value | Get Asana Priority Field ID | Set Asana Task Priority | Copy the priority |
| Set Workflow Environment (stage 3) | n8n-nodes-base.set | Sets Stage 3 environment | Trigger | Search Table | Receive the settings |
| Set Workflow Environment (stage 5) | n8n-nodes-base.set | Sets Stage 5 environment | Trigger | Search Table, Get Spaces | Receive the settings |
| Call '4. Custom Fields Migration' | n8n-nodes-base.executeWorkflow | Executes Stage 4 sub-workflow | Call '3. Folders & Lists Migration' | Call '5. Tasks Migration' | Run the five stages |
| Get ClickUp Workspace Custom Fields | n8n-nodes-base.httpRequest | Fetches workspace custom fields | If Table Doesn't Exists | Create Workspace Fields Array | Read workspace fields |
| Send Success Message (orchestrator) | n8n-nodes-base.gmail | Emails final summary | Merge CSV Files | Delete * Table | Build the mapping files |
| Call '3. Folders & Lists Migration' | n8n-nodes-base.executeWorkflow | Executes Stage 3 sub-workflow | Call '2. Spaces Migration' | Call '4. Custom Fields Migration' | Run the five stages |
| Call '2. Spaces Migration' | n8n-nodes-base.executeWorkflow | Executes Stage 2 sub-workflow | Call '1. Users Migration' | Call '3. Folders & Lists Migration' | Run the five stages |

---

### 4. Reproducing the Workflow from Scratch

To rebuild this multi-workflow migration tool manually in n8n, follow these step-by-step instructions:

#### Step 1: Create the Orchestrator Workflow
1. Create a new workflow named `Migrate ClickUp users, spaces, lists, custom fields and tasks from ClickUp to Asana`.
2. Add a **Manual Trigger** (`When clicking ‘Execute workflow’`).
3. Add a **Set** node (`Set Initial Workflow Environment`) connected to the trigger. Configure string and number assignments for `FeedBackEmail`, `ClickUpWorkspaceID`, `ClickUpPAK`, `AsanaWorkspaceID`, `AsanaPlan`, and `AsanaPAK`.
4. Add HTTP Request nodes (`Check Asana Credentials`, `Check Asana Workspace ID`, `Check ClickUp Credentials`, `Check ClickUp Workspace ID`) to validate tokens and workspace IDs. Set `onError` to `continueErrorOutput`.
5. Connect Gmail nodes (`Send Asana Credentials Error`, `Send Asana Workspace ID Error`, `Send ClickUp Credentials Error`, `Send ClickUp Workspace ID Error`, `Send Start Notice`) to handle failure outputs and successful initialization.
6. Add 5 **Execute Workflow** nodes (`Call '1. Users Migration'`, `Call '2. Spaces Migration'`, `Call '3. Folders & Lists Migration'`, `Call '4. Custom Fields Migration'`, `Call '5. Tasks Migration'`) linked in series, with `waitForSubWorkflow` enabled.
7. Add Data Table reader nodes (`Get Spaces Mapping Rows`, `Get Lists Mapping Rows`, `Get Fields Mapping Rows`, `Get Tasks Mapping Rows`) and Convert to File nodes (`Convert Spaces To CSV`, etc.) to generate CSV exports.
8. Merge the CSV files using a **Merge** node, send the summary email via Gmail, and execute Data Table deletion nodes to clean up temporary tables.

#### Step 2: Create Stage 1 (Users Migration Sub-Workflow)
1. Create a sub-workflow named `1. Users Migration`.
2. Add an **Execute Workflow Trigger** defining input parameters (`FeedbackEmail`, `ClickUpWorkspaceID`, `ClickUpPAK`, `ClickUpRateLimit`, `AsanaWorkspaceID`, `AsanaPAK`, `AsanaRateLimit`).
3. Add a **Set** node (`Set Workflow Environment (stage 1)`) to propagate parameters.
4. Add HTTP Request nodes to fetch ClickUp teams/members (`https://api.clickup.com/api/v2/team`) and Asana workspace users (`https://app.asana.com/api/1.0/workspaces/{workspace_id}/users`).
5. Normalize email formats using **Set** nodes and filter missing users using a **Merge** node (`joinMode: keepNonMatches`).
6. Loop through unmatched users using a **Split In Batches** node (`Loop Over Users`) and invite them via HTTP Request (`POST https://app.asana.com/api/1.0/workspaces/{workspace_id}/addUser`).
7. Handle success/error states with Gmail and `Stop and Error` nodes.

#### Step 3: Create Stage 2 (Spaces Migration Sub-Workflow)
1. Create a sub-workflow named `2. Spaces Migration`.
2. Add an **Execute Workflow Trigger** and environment **Set** node.
3. Check for the existence of the lookup table `ClickUpSpace-AsanaTeam-{{ $json.TableSuffix }}` using a Data Table node.
4. Fetch ClickUp spaces (`GET https://api.clickup.com/api/v2/team/{workspace_id}/space`) and filter out already migrated spaces.
5. Loop through unmigrated spaces and create corresponding Asana teams (`POST https://app.asana.com/api/1.0/teams`) with organization scope.
6. Save mappings in the Data Table, generate a CSV file, and email progress.

#### Step 4: Create Stage 5 (Tasks Migration Sub-Workflow)
*(Note: Stages 3 and 4 follow the same architectural pattern of environment initialization, table checks, recursive ClickUp traversal, API batching, and mapping storage).*
1. Create a sub-workflow named `5. Tasks Migration`.
2. Add an **Execute Workflow Trigger** with input parameters.
3. Check or create the task mapping table `ClickUpTask-AsanaTask-{{ $json.TableSuffix }}`.
4. Recursively fetch spaces, folders, lists, and tasks (including closed items, subtasks, and time tracking data) from ClickUp.
5. Sort tasks by hierarchy depth using a **Code** node (`Sort Subtasks After Tasks`) to ensure parent tasks are created first.
6. Loop through tasks (`Split In Batches`), resolve parent task GIDs from the mapping table if applicable, and create Asana tasks or subtasks (`POST https://app.asana.com/api/1.0/tasks` or `/subtasks`).
7. Update custom field values, priorities, comments, subtask checklists, and attachments via subsequent HTTP requests.
8. Store task mappings in the data table, convert to CSV, and email completion notices.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Asana Developer Console | [Asana Developer Console](https://app.asana.com/0/my-apps) |
| Asana Workspaces API Reference | [Asana Workspaces API](https://app.asana.com/api/1.0/workspaces) |
Spawn recurring Notion tasks from rules with Data Tables and Slack

https://n8nworkflows.xyz/workflows/spawn-recurring-notion-tasks-from-rules-with-data-tables-and-slack-17549


# Spawn recurring Notion tasks from rules with Data Tables and Slack

### 1. Workflow Overview

This workflow automates the generation of recurring tasks in Notion based on customizable recurrence rules stored in a separate Notion database. It operates on a daily schedule, evaluates which rules are due based on a designated timezone, cross-references an n8n Data Table ledger to prevent duplicate task creation, creates the corresponding task pages in Notion, updates the ledger, and sends a summary notification to Slack.

The workflow logic is grouped into three main functional blocks:
- **1.1 Schedule & Rule Evaluation:** Triggers daily, fetches all defined recurrence rules from Notion, and filters out the rules that match the current date.
- **1.2 Duplicate Prevention:** Retrieves the local execution ledger from an n8n Data Table and filters out any rules that have already been executed for the current date.
- **1.3 Execution & Notification:** Creates the new task pages in Notion iteratively, upserts the execution records back into the Data Table, and posts a consolidated summary message to a Slack channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Rule Evaluation
- **Overview:** Initiates the automation process on a daily interval, queries the source Notion database for active recurrence rules, and processes them programmatically to determine which tasks are due on the current day.
- **Nodes Involved:** 
  - `When Daily Schedule Fires`
  - `Read Recurrence Rules`
  - `Select Rules Due Today`
- **Node Details:**
  - **When Daily Schedule Fires**
    - *Type and Role:* `n8n-nodes-base.scheduleTrigger` (Trigger). Initiates the workflow execution at a fixed hour every day.
    - *Configuration Choices:* Trigger interval configured to run daily at hour 06:00.
    - *Key Expressions/Variables:* None.
    - *Connections:* Input: None (Root trigger). Output: Connects to `Read Recurrence Rules`.
    - *Version Requirements:* v1.3.
    - *Edge Cases / Failures:* Missed executions if the n8n instance is offline at 06:00.
  - **Read Recurrence Rules**
    - *Type and Role:* `n8n-nodes-base.notion` (Action). Fetches all pages from the Notion database containing recurrence configurations.
    - *Configuration Choices:* Resource: `databasePage`, Operation: `getAll`, Return All: `true`, Database ID: `YOUR_NOTION_RULES_DATABASE_ID`.
    - *Key Expressions/Variables:* None.
    - *Connections:* Input: `When Daily Schedule Fires`. Output: Connects to `Select Rules Due Today`.
    - *Version Requirements:* v2.2.
    - *Edge Cases / Failures:* API rate limits, invalid database ID, or revoked Notion credentials.
  - **Select Rules Due Today**
    - *Type and Role:* `n8n-nodes-base.code` (Data transformation/Filtering). Evaluates all fetched rules against the current date across four distinct frequency types (`every_n_days`, `weekly`, `monthly_day`, `monthly_nth_weekday`).
    - *Configuration Choices:* Custom JavaScript execution environment with a configurable `TIMEZONE` variable (set to `'UTC'` by default).
    - *Key Expressions/Variables:* `$input.all()`
    - *Connections:* Input: `Read Recurrence Rules`. Output: Connects to `Get Spawn Ledger`.
    - *Version Requirements:* v2.,
    - *Edge Cases / Failures:* Runtime errors if expected database properties (`rule_id`, `task_title`, `frequency`, etc.) are missing or malformed in Notion.

#### 2.2 Duplicate Prevention
- **Overview:** Queries an internal n8n Data Table ledger to retrieve past execution timestamps and filters out rules that have already spawned tasks for the current calendar date.
- **Nodes Involved:**
  - `Get Spawn Ledger`
  - `Filter Already Spawned`
- **Node Details:**
  - **Get Spawn Ledger**
    - *Type and Role:* `n8n-nodes-base.dataTable` (Data retrieval). Reads all rows from the local tracking ledger.
    - *Configuration Choices:* Operation: `get`, Return All: `true`, Data Table Name: `cadence_last_spawned`, Executed Once: `true`, Always Output Data: `true`.
    - *Key Expressions/Variables:* None.
    - *Connections:* Input: `Select Rules Due Today`. Output: Connects to `Filter Already Spawned`.
    - *Version Requirements:* v1.1.
    - *Edge Cases / Failures:* Missing Data Table schema or uninitialized table.
  - **Filter Already Spawned**
    - *Type and Role:* `n8n-nodes-base.code` (Filtering). Compares the list of rules due today against the ledger timestamps to isolate fresh, unexecuted rules.
    - *Configuration Choices:* Custom JavaScript execution mapping input datasets.
    - *Key Expressions/Variables:* `$('Select Rules Due Today').all()`, `$input.all()`
    - *Connections:* Input: `Get Spawn Ledger`. Output: Connects to `Create Notion Task Page`.
    - *Version Requirements:* v2.
    - *Edge Cases / Failures:* Empty inputs resulting in empty output arrays.

#### 2.3 Execution & Notification
- **Overview:** Iterates through the filtered fresh rules to create corresponding pages in the Notion tasks database, updates the execution ledger via upsert operations, and compiles a summary sent to Slack.
- **Nodes Involved:**
  - `Create Notion Task Page`
  - `Record Spawn In Ledger`
  - `Post Spawn Summary`
- **Node Details:**
  - **Create Notion Task Page**
    - *Type and Role:* `n8n-nodes-base.notion` (Action). Creates a new task page inside the destination Notion database for each due rule.
    - *Configuration Choices:* Resource: `databasePage`, Operation: `create`, Database ID: `YOUR_NOTION_TASKS_DATABASE_ID`, Page Title set dynamically, Property mapping for `Due` date.
    - *Key Expressions/Variables:* `={{ $json.task_title }}`, `={{ $json.spawn_date }}`
    - *Connections:* Input: `Filter Already Spawned`. Output: Connects to `Record Spawn In Ledger`.
    - *Version Requirements:* v2.2.
    - *Edge Cases / Failures:* Schema mismatch in Notion (e.g., missing `Due` date property), property validation errors.
  - **Record Spawn In Ledger**
    - *Type and Role:* `n8n-nodes-base.dataTable` (Data manipulation). Upserts execution logs into the tracking table to record that a rule has spawned for the current date.
    - *Configuration Choices:* Operation: `upsert`, Matching Column: `rule_id`, Mapped columns: `rule_id`, `rule_title`, `last_spawned_on`.
    - *Key Expressions/Variables:* `={{ $('Filter Already Spawned').item.json.rule_id }}`, `={{ $('Filter Already Spawned').item.json.task_title }}`, `={{ $('Filter Already Spawned').item.json.spawn_date }}`
    - *Connections:* Input: `Create Notion Task Page`. Output: Connects to `Post Spawn Summary`.
    - *Version Requirements:* v1.1.
    - *Edge Cases / Failures:* Primary key constraint violations if table schema is misconfigured.
  - **Post Spawn Summary**
    - *Type and Role:* `n8n-nodes-base.slack` (Action). Sends a notification message containing a comma-separated list of newly spawned tasks to a designated Slack channel.
    - *Configuration Choices:* Resource: `message`, Operation: `post`, Select: `channel`, Channel Name: `YOUR_SLACK_CHANNEL`, Execute Once: `true`.
    - *Key Expressions/Variables:* `={{ $('Filter Already Spawned').all().length }}`, `={{ $('Filter Already Spawned').first().json.spawn_date }}`, `={{ $('Filter Already Spawned').all().map(i => i.json.task_title).join(', ') }}`
    - *Connections:* Input: `Record Spawn In Ledger`. Output: None (Terminal node).
    - *Version Requirements:* v2.5.
    - *Edge Cases / Failures:* Invalid Slack token, inaccessible channel ID, or missing bot permissions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When Daily Schedule Fires | scheduleTrigger | Triggers workflow execution daily at 06:00 | None | Read Recurrence Rules | Spawn recurring Notion tasks from a rules database on a schedule<br><br>### How it works<br>1. A daily Schedule Trigger reads a Notion rules database where every row describes one recurring to-do.<br>2. A Code node tests each rule against today's date in a configurable timezone: every N days, weekly on chosen weekdays, monthly on day D, and the nth weekday of the month.<br>3. A Data Table named `cadence_last_spawned` is read, and any rule already stamped with today's date is dropped so a repeat run creates nothing.<br>4. Notion creates one page per due rule, with the due date set to today.<br>5. The ledger is upserted on `rule_id` and Slack gets a one line summary of what was created.<br><br>### Setup steps<br>- [ ] Connect your Notion credential on `Read Recurrence Rules` and `Create Notion Task Page`.<br>- [ ] Replace `YOUR_NOTION_RULES_DATABASE_ID` and `YOUR_NOTION_TASKS_DATABASE_ID` with your own database IDs.<br>- [ ] Give the rules database the properties `rule_id`, `task_title`, `frequency`, `interval_days`, `anchor_date`, `weekdays`, `day_of_month`, `nth`, `nth_weekday`, and `active`.<br>- [ ] Give the tasks database a date property named `Due`.<br>- [ ] Create a Data Table called `cadence_last_spawned` with the columns `rule_id`, `last_spawned_on`, and `rule_title`.<br>- [ ] Connect your Slack credential and pick the channel on `Post Spawn Summary`.<br><br>### Customization<br>Set `TIMEZONE` at the top of `Select Rules Due Today` to the zone your day boundary should follow, and add another branch in that node if you need a pattern the four built in ones cannot express. |
| Read Recurrence Rules | notion | Fetches all pages from the Notion rules database | When Daily Schedule Fires | Select Rules Due Today | Spawn recurring Notion tasks from a rules database on a schedule<br><br>### How it works<br>1. A daily Schedule Trigger reads a Notion rules database where every row describes one recurring to-do.<br>2. A Code node tests each rule against today's date in a configurable timezone: every N days, weekly on chosen weekdays, monthly on day D, and the nth weekday of the month.<br>3. A Data Table named `cadence_last_spawned` is read, and any rule already stamped with today's date is dropped so a repeat run creates nothing.<br>4. Notion creates one page per due rule, with the due date set to today.<br>5. The ledger is upserted on `rule_id` and Slack gets a one line summary of what was created.<br><br>### Setup steps<br>- [ ] Connect your Notion credential on `Read Recurrence Rules` and `Create Notion Task Page`.<br>- [ ] Replace `YOUR_NOTION_RULES_DATABASE_ID` and `YOUR_NOTION_TASKS_DATABASE_ID` with your own database IDs.<br>- [ ] Give the rules database the properties `rule_id`, `task_title`, `frequency`, `interval_days`, `anchor_date`, `weekdays`, `day_of_month`, `nth`, `nth_weekday`, and `active`.<br>- [ ] Give the tasks database a date property named `Due`.<br>- [ ] Create a Data Table called `cadence_last_spawned` with the columns `rule_id`, `last_spawned_on`, and `rule_title`.<br>- [ ] Connect your Slack credential and pick the channel on `Post Spawn Summary`.<br><br>### Customization<br>Set `TIMEZONE` at the top of `Select Rules Due Today` to the zone your day boundary should follow, and add another branch in that node if you need a pattern the four built in ones cannot express.<br><br>Read rules and pick today |
| Select Rules Due Today | code | Evaluates rules against today's date and time zone | Read Recurrence Rules | Get Spawn Ledger | Spawn recurring Notion tasks from a rules database on a schedule<br><br>### How it works<br>1. A daily Schedule Trigger reads a Notion rules database where every row describes one recurring to-do.<br>2. A Code node tests each rule against today's date in a configurable timezone: every N days, weekly on chosen weekdays, monthly on day D, and the nth weekday of the month.<br>3. A Data Table named `cadence_last_spawned` is read, and any rule already stamped with today's date is dropped so a repeat run creates nothing.<br>4. Notion creates one page per due rule, with the due date set to today.<br>5. The ledger is upserted on `rule_id` and Slack gets a one line summary of what was created.<br><br>### Setup steps<br>- [ ] Connect your Notion credential on `Read Recurrence Rules` and `Create Notion Task Page`.<br>- [ ] Replace `YOUR_NOTION_RULES_DATABASE_ID` and `YOUR_NOTION_TASKS_DATABASE_ID` with your own database IDs.<br>- [ ] Give the rules database the properties `rule_id`, `task_title`, `frequency`, `interval_days`, `anchor_date`, `weekdays`, `day_of_month`, `nth`, `nth_weekday`, and `active`.<br>- [ ] Give the tasks database a date property named `Due`.<br>- [ ] Create a Data Table called `cadence_last_spawned` with the columns `rule_id`, `last_spawned_on`, and `rule_title`.<br>- [ ] Connect your Slack credential and pick the channel on `Post Spawn Summary`.<br><br>### Customization<br>Set `TIMEZONE` at the top of `Select Rules Due Today` to the zone your day boundary should follow, and add another branch in that node if you need a pattern the four built in ones cannot express.<br><br>Read rules and pick today |
| Get Spawn Ledger | dataTable | Reads tracking data from the n8n Data Table | Select Rules Due Today | Filter Already Spawned | Spawn recurring Notion tasks from a rules database on a schedule<br><br>### How it works<br>1. A daily Schedule Trigger reads a Notion rules database where every row describes one recurring to-do.<br>2. A Code node tests each rule against today's date in a configurable timezone: every N days, weekly on chosen weekdays, monthly on day D, and the nth weekday of the month.<br>3. A Data Table named `cadence_last_spawned` is read, and any rule already stamped with today's date is dropped so a repeat run creates nothing.<br>4. Notion creates one page per due rule, with the due date set to today.<br>5. The ledger is upserted on `rule_id` and Slack gets a one line summary of what was created.<br><br>### Setup steps<br>- [ ] Connect your Notion credential on `Read Recurrence Rules` and `Create Notion Task Page`.<br>- [ ] Replace `YOUR_NOTION_RULES_DATABASE_ID` and `YOUR_NOTION_TASKS_DATABASE_ID` with your own database IDs.<br>- [ ] Give the rules database the properties `rule_id`, `task_title`, `frequency`, `interval_days`, `anchor_date`, `weekdays`, `day_of_month`, `nth`, `nth_weekday`, and `active`.<br>- [ ] Give the tasks database a date property named `Due`.<br>- [ ] Create a Data Table called `cadence_last_spawned` with the columns `rule_id`, `last_spawned_on`, and `rule_title`.<br>- [ ] Connect your Slack credential and pick the channel on `Post Spawn Summary`.<br><br>### Customization<br>Set `TIMEZONE` at the top of `Select Rules Due Today` to the zone your day boundary should follow, and add another branch in that node if you need a pattern the four built in ones cannot express.<br><br>Skip what already spawned |
| Filter Already Spawned | code | Filters out rules already executed today using the ledger | Get Spawn Ledger | Create Notion Task Page | Spawn recurring Notion tasks from a rules database on a schedule<br><br>### How it works<br>1. A daily Schedule Trigger reads a Notion rules database where every row describes one recurring to-do.<br>2. A Code node tests each rule against today's date in a configurable timezone: every N days, weekly on chosen weekdays, monthly on day D, and the nth weekday of the month.<br>3. A Data Table named `cadence_last_spawned` is read, and any rule already stamped with today's date is dropped so a repeat run creates nothing.<br>4. Notion creates one page per due rule, with the due date set to today.<br>5. The ledger is upserted on `rule_id` and Slack gets a one line summary of what was created.<br><br>### Setup steps<br>- [ ] Connect your Notion credential on `Read Recurrence Rules` and `Create Notion Task Page`.<br>- [ ] Replace `YOUR_NOTION_RULES_DATABASE_ID` and `YOUR_NOTION_TASKS_DATABASE_ID` with your own database IDs.<br>- [ ] Give the rules database the properties `rule_id`, `task_title`, `frequency`, `interval_days`, `anchor_date`, `weekdays`, `day_of_month`, `nth`, `nth_weekday`, and `active`.<br>- [ ] Give the tasks database a date property named `Due`.<br>- [ ] Create a Data Table called `cadence_last_spawned` with the columns `rule_id`, `last_spawned_on`, and `rule_title`.<br>- [ ] Connect your Slack credential and pick the channel on `Post Spawn Summary`.<br><br>### Customization<br>Set `TIMEZONE` at the top of `Select Rules Due Today` to the zone your day boundary should follow, and add another branch in that node if you need a pattern the four built in ones cannot express.<br><br>Skip what already spawned |
| Create Notion Task Page | notion | Creates a new page in the Notion tasks database | Filter Already Spawned | Record Spawn In Ledger | Spawn recurring Notion tasks from a rules database on a schedule<br><br>### How it works<br>1. A daily Schedule Trigger reads a Notion rules database where every row describes one recurring to-do.<br>2. A Code node tests each rule against today's date in a configurable timezone: every N days, weekly on chosen weekdays, monthly on day D, and the nth weekday of the month.<br>3. A Data Table named `cadence_last_spawned` is read, and any rule already stamped with today's date is dropped so a repeat run creates nothing.<br>4. Notion creates one page per due rule, with the due date set to today.<br>5. The ledger is upserted on `rule_id` and Slack gets a one line summary of what was created.<br><br>### Setup steps<br>- [ ] Connect your Notion credential on `Read Recurrence Rules` and `Create Notion Task Page`.<br>- [ ] Replace `YOUR_NOTION_RULES_DATABASE_ID` and `YOUR_NOTION_TASKS_DATABASE_ID` with your own database IDs.<br>- [ ] Give the rules database the properties `rule_id`, `task_title`, `frequency`, `interval_days`, `anchor_date`, `weekdays`, `day_of_month`, `nth`, `nth_weekday`, and `active`.<br>- [ ] Give the tasks database a date property named `Due`.<br>- [ ] Create a Data Table called `cadence_last_spawned` with the columns `rule_id`, `last_spawned_on`, and `rule_title`.<br>- [ ] Connect your Slack credential and pick the channel on `Post Spawn Summary`.<br><br>### Customization<br>Set `TIMEZONE` at the top of `Select Rules Due Today` to the zone your day boundary should follow, and add another branch in that node if you need a pattern the four built in ones cannot express.<br><br>Create tasks and record them |
| Record Spawn In Ledger | dataTable | Upserts rule execution data into the n8n Data Table | Create Notion Task Page | Post Spawn Summary | Spawn recurring Notion tasks from a rules database on a schedule<br><br>### How it works<br>1. A daily Schedule Trigger reads a Notion rules database where every row describes one recurring to-do.<br>2. A Code node tests each rule against today's date in a configurable timezone: every N days, weekly on chosen weekdays, monthly on day D, and the nth weekday of the month.<br>3. A Data Table named `cadence_last_spawned` is read, and any rule already stamped with today's date is dropped so a repeat run creates nothing.<br>4. Notion creates one page per due rule, with the due date set to today.<br>5. The ledger is upserted on `rule_id` and Slack gets a one line summary of what was created.<br><br>### Setup steps<br>- [ ] Connect your Notion credential on `Read Recurrence Rules` and `Create Notion Task Page`.<br>- [ ] Replace `YOUR_NOTION_RULES_DATABASE_ID` and `YOUR_NOTION_TASKS_DATABASE_ID` with your own database IDs.<br>- [ ] Give the rules database the properties `rule_id`, `task_title`, `frequency`, `interval_days`, `anchor_date`, `weekdays`, `day_of_month`, `nth`, `nth_weekday`, and `active`.<br>- [ ] Give the tasks database a date property named `Due`.<br>- [ ] Create a Data Table called `cadence_last_spawned` with the columns `rule_id`, `last_spawned_on`, and `rule_title`.<br>- [ ] Connect your Slack credential and pick the channel on `Post Spawn Summary`.<br><br>### Customization<br>Set `TIMEZONE` at the top of `Select Rules Due Today` to the zone your day boundary should follow, and add another branch in that node if you need a pattern the four built in ones cannot express.<br><br>Create tasks and record them |
| Post Spawn Summary | slack | Posts a summary message to a Slack channel | Record Spawn In Ledger | None | Spawn recurring Notion tasks from a rules database on a schedule<br><br>### How it works<br>1. A daily Schedule Trigger reads a Notion rules database where every row describes one recurring to-do.<br>2. A Code node tests each rule against today's class date in a configurable timezone: every N days, weekly on chosen weekdays, monthly on day D, and the nth weekday of the month.<br>3. A Data Table named `cadence_last_spawned` is read, and any rule already stamped with today's date is dropped so a repeat run creates nothing.<br>4. Notion creates one page per due rule, with the due date set to today.<br>5. The ledger is upserted on `rule_id` and Slack gets a one line summary of what was created.<br><br>### Setup steps<br>- [ ] Connect your Notion credential on `Read Recurrence Rules` and `Create Notion Task Page`.<br>- [ ] Replace `YOUR_NOTION_RULES_DATABASE_ID` and `YOUR_NOTION_TASKS_DATABASE_ID` with your own database IDs.<br>- [ ] Give the rules database the properties `rule_id`, `task_title`, `frequency`, `interval_days`, `anchor_date`, `weekdays`, `day_of_month`, `nth`, `nth_weekday`, and `active`.<br>- [ ] Give the tasks database a date property named `Due`.<br>- [ ] Create a Data Table called `cadence_last_spawned` with the columns `rule_id`, `last_spawned_on`, and `rule_title`.<br>- [ ] Connect your Slack credential and pick the channel on `Post Spawn Summary`.<br><br>### Customization<br>Set `TIMEZONE` at the top of `Select Rules Due Today` to the zone your day boundary should follow, and add another branch in that node if you need a pattern the four built in ones cannot express.<br><br>Create tasks and record them |

---

### 4. Reproducing the Workflow from Scratch

Follow these sequential steps to rebuild the workflow in an n8n canvas:

1. **Create the Data Table Ledger:**
   - Navigate to **Data Tables** in your n8n instance and create a new table named `cadence_last_spawned`.
   - Add three string columns: `rule_id`, `last_spawned_on`, and `rule_title`.

2. **Add Node 1: Schedule Trigger**
   - Type: `n8n-nodes-base.scheduleTrigger`
   - Name: `When Daily Schedule Fires`
   - Configuration: Set trigger interval to run at hour `6`.

3. **Add Node 2: Read Recurrence Rules (Notion)**
   - Type: `n8n-nodes-base.notion`
   - Name: `Read Recurrence Rules`
   - Configuration: Set Resource to `Database Page`, Operation to `Get All`, and Return All to `true`.
   - Database ID: Set to your Notion Rules Database ID (`YOUR_NOTION_RULES_DATABASE_ID`).
   - Credentials: Connect a valid Notion API account credential.
   - Connection: Connect `When Daily Schedule Fires` to `Read Recurrence Rules`.

4. **Add Node 3: Select Rules Due Today (Code)**
   - Type: `n8n-nodes-base.code`
   - Name: `Select Rules Due Today`
   - Configuration: Paste the provided JavaScript code snippet handling date evaluation and rules matching. Ensure the `TIMEZONE` variable matches your target geography.
   - Connection: Connect `Read Recurrence Rules` to `Select Rules Due Today`.

5. **Add Node 4: Get Spawn Ledger (Data Table)**
   - Type: `n8n-nodes-base.dataTable`
   - Name: `Get Spawn Ledger`
   - Configuration: Set Operation to `Get`, Return All to `true`, and select Data Table `cadence_last_spawned`. Set node options to execute once and always output data.
   - Connection: Connect `Select Rules Due Today` to `Get Spawn Ledger`.

6. **Add Node 5: Filter Already Spawned (Code)**
   - Type: `n8n-nodes-base.code`
   - Name: `Filter Already Spawned`
   - Configuration: Paste the provided JavaScript filtering code to reconcile incoming due items against ledger records.
   - Connection: Connect `Get Spawn Ledger` to `Filter Already Spawned`.

7. **Add Node 6: Create Notion Task Page (Notion)**
   - Type: `n8n-nodes-base.notion`
   - Name: `Create Notion Task Page`
   - Configuration: Set Resource to `Database Page`, Operation to `Create`.
   - Database ID: Set to your Notion Tasks Database ID (`YOUR_NOTION_TASKS_DATABASE_ID`).
   - Title Property: `={{ $json.task_title }}`
   - Properties UI: Add property `Due` of type `date`, mapped to `={{ $json.spawn_date }}`.
   - Credentials: Connect your Notion API credential.
   - Connection: Connect `Filter Already Spawned` to `Create Notion Task Page`.

8. **Add Node 7: Record Spawn In Ledger (Data Table)**
   - Type: `n8n-nodes-base.dataTable`
   - Name: `Record Spawn In Ledger`
   - Configuration: Set Operation to `Upsert`, Matching Columns to `rule_id`. Map table columns (`rule_id`, `rule_title`, `last_spawned_on`) to expression references pulling from `$('Filter Already Spawned')`.
   - Connection: Connect `Create Notion Task Page` to `Record Spawn In Ledger`.

9. **Add Node 8: Post Spawn Summary (Slack)**
   - Type: `n8n-nodes-base.slack`
   - Name: `Post Spawn Summary`
   - Configuration: Set Resource to `Message`, Operation to `Post`, Select to `Channel`, and target your preferred Slack channel (`YOUR_SLACK_CHANNEL`).
   - Text Property: Use the expression compiling total count, date, and task titles.
   - Credentials: Connect a valid Slack API credential.
   - Connection: Connect `Record Spawn In Ledger` to `Post Spawn Summary`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Notion Database Property Requirements | Ensure the rules database contains properties: `rule_id`, `task_title`, `frequency`, `interval_days`, `anchor_date`, `weekdays`, `day_of_month`, `nth`, `nth_weekday`, and `active`. Ensure the tasks database contains a date property named `Due`. |
| Timezone Configuration | The day evaluation logic relies on a designated timezone string (default `'UTC'`) defined at the top of the `Select Rules Due Today` script node. Modify this value to align generation with local calendar days. |
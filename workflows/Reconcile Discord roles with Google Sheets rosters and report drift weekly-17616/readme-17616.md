Reconcile Discord roles with Google Sheets rosters and report drift weekly

https://n8nworkflows.xyz/workflows/reconcile-discord-roles-with-google-sheets-rosters-and-report-drift-weekly-17616


# Reconcile Discord roles with Google Sheets rosters and report drift weekly

### 1. Workflow Overview

This workflow automates the weekly auditing and reconciliation process between a Google Sheets membership roster and actual Discord member roles. Its primary purpose is to identify, log, and report discrepancies ("drift") such as members missing entitled roles, members holding unauthorized roles, and unjudgeable roster entries. 

The workflow is strictly read-only regarding Discord (it never assigns or revokes roles), making it completely safe to deploy on production servers.

The logic is organized into the following functional blocks:
- **1.1 Input Reception & Configuration:** Triggers on a weekly schedule and establishes all operational parameters, IDs, and URLs.
- **1.2 Discord Context & Roster Ingestion:** Fetches live Discord server roles and reads the membership roster from Google Sheets in parallel/sequence.
- **1.3 Roster Normalization & Member Inspection:** Validates and normalizes roster rows, resolves role names to live Discord role IDs, and fetches the complete server member list.
- **1.4 Drift Reconciliation:** Compares roster entitlements against live Discord roles, bucketing discrepancies (e.g., missing roles, unexpected roles, grace periods, unusable rows).
- **1.5 Reporting & Notification:** Appends all drift details to a Google Sheets report tab and compiles/posts a concise summary message to a designated Discord channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Configuration
- **Overview:** Initializes the execution flow on a weekly schedule and defines all environment-specific configuration variables (such as target server ID, spreadsheet URLs, and status rules) in a single centralized location.
- **Nodes Involved:** 
  - `Run Weekly Roster Check`
  - `Set Reconcile Settings`

##### Node Details:
- **`Run Weekly Roster Check`**
  - **Type & Technical Role:** `n8n-nodes-base.scheduleTrigger` (Trigger)
  - **Configuration Choices:** Configured to fire weekly on Mondays at 08:00.
  - **Key Expressions:** None.
  - **Connections:** Output connects to `Set Reconcile Settings`.
  - **Failure Types:** None standard; schedule triggers depend on n8n core execution uptime.

- **`Set Reconcile Settings`**
  - **Type & Technical Role:** `n8n-nodes-base.set` (Data Transformation / Constant Definition)
  - **Configuration Choices:** Stores string and numerical variables including `guild_id`, `roster_sheet_url`, `report_sheet_url`, `gated_roles`, `active_status_values`, `inactive_status_values`, `grace_period_days`, and `max_examples_in_summary`.
  - **Key Expressions:** None (static assignment values).
  - **Connections:** Input from `Run Weekly Roster Check`; output connects to `Get Guild Roles`.
  - **Failure Types:** Typo in placeholder IDs or URLs will cause downstream API or integration errors.

---

#### 2.2 Discord Context & Roster Ingestion
- **Overview:** Pulls the server's role definitions from the Discord API and reads the active membership roster from Google Sheets using the parameters configured previously.
- **Nodes Involved:**
  - `Get Guild Roles`
  - `Read Roster Sheet`

##### Node Details:
- **`Get Guild Roles`**
  - **Type & Technical Role:** `n8n-nodes-base.httpRequest` (API Request)
  - **Configuration Choices:** Calls Discord API v10 endpoint `GET /guilds/{guild_id}/roles` using predefined Discord Bot credentials.
  - **Key Expressions:** Uses `={{ $('Set Reconcile Settings').first().json.guild_id }}` to target the correct server.
  - **Connections:** Input from `Set Reconcile Settings`; output connects to `Read Roster Sheet`.
  - **Version-Specific Requirements:** Requires Discord Bot API credentials with valid bot token scope.
  - **Failure Types:** Unauthorized access, invalid guild ID, or API rate limits.

- **`Read Roster Sheet`**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Spreadsheet Reader)
  - **Configuration Choices:** Reads data from a named tab (`Roster`) using a dynamic document URL. Set to execute once (`executeOnce: true`).
  - **Key Expressions:** 
    - Document ID: `={{ $("Set Reconcile Settings").first().json.roster_sheet_url }}`
    - Sheet Name: `={{ $("Set Reconcile Settings").first().json.roster_tab_name }}`
  - **Connections:** Input from `Get Guild Roles`; output connects to `Normalize Roster Rows`.
  - **Failure Types:** Invalid Google Sheets credentials, missing sheet/tab name, or lack of permissions on the target document.

---

#### 2.3 Roster Normalization & Member Inspection
- **Overview:** Parses the raw roster rows, validates Discord snowflakes and status strings, resolves human-readable role names to live Discord role IDs, and queries all server members.
- **Nodes Involved:**
  - `Normalize Roster Rows`
  - `Get Server Members`

##### Node Details:
- **`Normalize Roster Rows`**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Custom JavaScript Processing)
  - **Configuration Choices:** Executes a custom JavaScript function mapping raw spreadsheet rows against live guild roles, validating user IDs via regex (`/^[0-9]{17,20}$/`), and evaluating active/inactive status codes.
  - **Key Expressions:** Reads data from `Set Reconcile Settings`, `Get Guild Roles`, and the incoming Google Sheets items.
  - **Connections:** Input from `Read Roster Sheet`; output connects to `Get Server Members`.
  - **Failure Types:** Malformed JavaScript handling or empty roster tabs (managed via fallback logic yielding a `roster_empty` flag).

- **`Get Server Members`**
  - **Type & Technical Role:** `n8n-nodes-base.discord` (API Integration)
  - **Configuration Choices:** Retrieves all guild members (`resource: member`, `returnAll: true`, `simplify: false`).
  - **Key Expressions:** Guild ID: `={{ $("Set Reconcile Settings").first().json.guild_id }}`
  - **Connections:** Input from `Normalize Roster Rows`; output connects to `Reconcile Roster Against Discord`.
  - **Version-Specific Requirements:** Requires the **Server Members Intent** to be enabled in the Discord Developer Portal for the bot application.
  - **Failure Types:** Missing Server Members Intent returns an empty member list; API timeout on massive guilds (>10,00arded members).

---

#### 2.4 Drift Reconciliation
- **Overview:** Cross-references validated roster entitlements against the actual roles held by Discord members, filtering out bots and ignored IDs, accounting for grace periods, and sorting discrepancies into distinct report buckets.
- **Nodes Involved:**
  - `Reconcile Roster Against Discord`

##### Node Details:
- **`Reconcile Roster Against Discord`**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Custom JavaScript Processing)
  - **Configuration Choices:** Executes complex reconciliation logic mapping server members to roster rows. Buckets include `gated_role_not_found`, `roster_empty`, `no_roster_row`, `missing_role`, `unexpected_role`, `roster_row_unusable`, `in_grace_period`, and `no_drift`. Attaches summary metadata (`__summary`) to each item.
  - **Key Expressions:** Consumes outputs from `Set Reconcile Settings`, `Normalize Roster Rows`, `Get Server Members`, and `Get Guild Roles`. Uses `$now.toISO()` and `$execution.id`.
  - **Connections:** Input from `Get Server Members`; output connects to `Write Drift Report`.
  - **Failure Types:** Script memory limits or execution timeout if handling excessively large datasets with unoptimized structures.

---

#### 2.5 Reporting & Notification
- **Overview:** Writes every identified drift row back to the Google Sheets report tab and compiles a human-readable summary post sent directly to the specified Discord channel.
- **Nodes Involved:**
  - `Write Drift Report`
  - `Build Drift Summary`
  - `Post Drift Summary`

##### Node Details:
- **`Write Drift Report`**
  - **Type & Technical Role:** `n8n-nodes-base.googleSheets` (Spreadsheet Writer)
  - **Configuration Choices:** Appends rows (`operation: append`, `cellFormat: RAW`) containing audit attributes (`checked_at`, `run_id`, `bucket`, `discord_user_id`, `display_name`, `discord_username`, `role`, `note`).
  - **Key Expressions:** 
    - Document ID: `={{ $("Set Reconcile Settings").first().json.report_sheet_url }}`
    - Sheet Name: `={{ $("Set Reconcile Settings").first().json.report_tab_name }}`
    - Mapped columns pull directly from current item JSON fields.
  - **Connections:** Input from `Reconcile Roster Against Discord`; output connects to `Build Drift Summary`.
  - **Failure Types:** Column header mismatch between n8n configuration and the physical Google Sheet.

- **`Build Drift Summary`**
  - **Type & Technical Role:** `n8n-nodes-base.code` (Custom JavaScript Processing)
  - **Configuration Choices:** Aggregates bucketed drift records, constructs sample lists up to `max_examples_in_summary`, formats Markdown output, and enforces a strict character length guard (< 1900 characters).
  - **Key Expressions:** Accesses `Set Reconcile Settings` and the structured items output by `Reconcile Roster Against Discord`. Uses `$now.toFormat('d LLLL yyyy')`.
  - **Connections:** Input from `Write Drift Report`; output connects to `Post Drift Summary`.
  - **Failure Types:** Formatting evaluation errors if expected summary arrays are absent.

- **`Post Drift Summary`**
  - **Type & Technical Role:** `n8n-nodes-base.discord` (API Integration)
  - **Configuration Choices:** Posts message content (`resource: message`) to a specified Discord channel with embed suppression enabled (`flags: ["SUPPRESS_EMBEDS"]`).
  - **Key Expressions:** 
    - Channel ID: `={{ $("Set Reconcile Settings").first().json.report_channel_id }}`
    - Guild ID: `={{ $("Set Reconcile Settings").first().json.guild_id }}`
    - Content: `={{ $json.content }}`
  - **Connections:** Input from `Build Drift Summary`; terminal node in the workflow.
  - **Failure Types:** Invalid channel ID, missing bot permissions to view/send messages in the target channel, or message content exceeding Discord size limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Run Weekly Roster Check` | `scheduleTrigger` | Triggers the workflow execution weekly on Mondays. | None | `Set Reconcile Settings` | Reconcile a Google Sheets roster against Discord roles and report the drift... |
| `Set Reconcile Settings` | `set` | Defines core configuration variables, IDs, and rule parameters. | `Run Weekly Roster Check` | `Get Guild Roles` | Configure once, then read both sides... |
| `Get Guild Roles` | `httpRequest` | Fetches live role definitions from the Discord API. | `Set Reconcile Settings` | `Read Roster Sheet` | Configure once, then read both sides... |
| `Read Roster Sheet` | `googleSheets` | Reads membership roster data from Google Sheets. | `Get Guild Roles` | `Normalize Roster Rows` | Configure once, then read both sides... |
| `Normalize Roster Rows` | `code` | Validates roster rows and maps role names to IDs. | `Read Roster Sheet` | `Get Server Members` | Diff the roster against Discord... / Your sheet needs Discord user IDs... |
| `Get Server Members` | `discord` | Fetches all members and associated roles from the Discord server. | `Normalize Roster Rows` | `Reconcile Roster Against Discord` | Reads the member list, and nothing else... |
| `Reconcile Roster Against Discord` | `code` | Compares roster entitlements against Discord roles and buckets drift. | `Get Server Members` | `Write Drift Report` | Diff the roster against Discord... |
| `Write Drift Report` | `googleSheets` | Appends individual drift records to the report sheet tab. | `Reconcile Roster Against Discord` | `Build Drift Summary` | Report two numbers, not three lists... |
| `Build Drift Summary` | `code` | Compiles formatted Markdown summary text and aggregate metrics. | `Write Drift Report` | `Post Drift Summary` | Report two numbers, not three lists... |
| `Post Drift Summary` | `discord` | Posts the formatted drift summary message to a Discord channel. | `Build Drift Summary` | None | Report two numbers, not three lists... |

---

### 4. Reproducing the Workflow from Scratch

Follow these step-by-step instructions to rebuild the workflow manually in n8n:

1. **Create the Trigger Node:**
   - Add a **Schedule Trigger** node (`n8n-nodes-base.scheduleTrigger`).
   - Name it `Run Weekly Roster Check`.
   - Set interval to weeks, triggering every Monday at `08:00`.

2. **Create the Configuration Node:**
   - Add a **Set** node (`n8n-nodes-base.set`).
   - Name it `Set Reconcile Settings`.
   - Add the following string/number assignments:
     - `guild_id` (string): Your Discord Server ID.
     - `roster_sheet_url` (string): URL of your Google Sheets roster file.
     - `roster_tab_name` (string): Set to `Roster`.
     - `report_sheet_url` (string): URL of your Google Sheets report file (can match roster URL).
     - `report_tab_name` (string): Set to `Drift Report`.
     - `report_channel_id` (string): Target Discord channel ID for the summary post.
     - `gated_roles` (string): Comma-separated list of role names to enforce (e.g., `Volunteer,Committee`).
     - `active_status_values` (string): `active,current,paid`.
     - `inactive_status_values` (string): `inactive,lapsed,expired,cancelled,left`.
     - `grace_period_days` (number): `7`.
     - `ignore_user_ids` (string): Leave blank or list ignored user snowflakes.
     - `max_examples_in_summary` (number): `5`.

3. **Fetch Discord Roles:**
   - Add an **HTTP Request** node (`n8n-nodes-base.httpRequest`).
   - Name it `Get Guild Roles`.
   - Set method to `GET`, URL to `=https://discord.com/api/v10/guilds/{{ $('Set Reconcile Settings').first().json.guild_id }}/roles`.
   - Configure authentication using a predefined **Discord Bot API** credential.
   - Connect `Set Reconcile Settings` output to this node.

4. **Read the Google Sheets Roster:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`).
   - Name it `Read Roster Sheet`.
   - Set Operation to `Get` (or default read operation), Document to URL (`={{ $("Set Reconcile Settings").first().json.roster_sheet_url }}`), and Sheet Name to Name (`={{ $("Set Reconcile Settings").first().json.roster_tab_name }}`).
   - Configure **Google Sheets OAuth2 API** credentials.
   - Enable `Execute Once` in node options.
   - Connect `Get Guild Roles` output to this node.

5. **Normalize Roster Data:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Name it `Normalize Roster Rows`.
   - Paste the custom JavaScript processing code that maps role names, validates user snowflakes (`/^[0-9]{17,20}$/`), and evaluates statuses.
   - Connect `Read Roster Sheet` output to this node.

6. **Fetch Server Members:**
   - Add a **Discord** node (`n8n-nodes-base.discord`).
   - Name it `Get Server Members`.
   - Set Resource to `Member`, Operation to `Get Many` (or `returnAll: true`).
   - Set Guild ID to `={{ $("Set Reconcile Settings").first().json.guild_id }}`.
   - Configure **Discord Bot API** credentials with **Server Members Intent** enabled.
   - Enable `Execute Once`.
   - Connect `Normalize Roster Rows` output to this node.

7. **Reconcile Roster Against Discord:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Name it `Reconcile Roster Against Discord`.
   - Paste the reconciliation logic code that compares active memberships against server role allocations and generates structured drift items.
   - Connect `Get Server Members` output to this node.

8. **Write Drift Report to Google Sheets:**
   - Add a **Google Sheets** node (`n8n-nodes-base.googleSheets`).
   - Name it `Write Drift Report`.
   - Set Operation to `Append`, Document URL and Sheet Name using expressions pointing to settings.
   - Map columns explicitly: `checked_at`, `run_id`, `bucket`, `discord_user_id`, `display_name`, `discord_username`, `role`, and `note` using corresponding `$json` fields.
   - Configure **Google Sheets OAuth2 API** credentials.
   - Connect `Reconcile Roster Against Discord` output to this node.

9. **Build Drift Summary:**
   - Add a **Code** node (`n8n-nodes-base.code`).
   - Name it `Build Drift Summary`.
   - Paste the summary generation script that aggregates counts and builds markdown text.
   - Enable `Execute Once`.
   - Connect `Write Drift Report` output to this node.

10. **Post Drift Summary to Discord:**
    - Add a **Discord** node (`n8n-nodes-base.discord`).
    - Name it `Post Drift Summary`.
    - Set Resource to `Message`, Operation to `Send`.
    - Set Guild ID and Channel ID via expressions referencing the settings node.
    - Set Content to `={{ $json.content }}`.
    - Enable option flags: `SUPPRESS_EMBEDS`.
    - Configure **Discord Bot API** credentials.
    - Enable `Execute Once`.
    - Connect `Build Drift Summary` output to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Server Members Intent Required** | The Discord Bot must have the **Server Members Intent** toggled on within the Discord Developer Portal. Without this intent, member fetching operations will return an empty collection. |
| **Read-Only Safety Guarantee** | This workflow never invokes role assignment or removal actions (`roleAdd`/`roleRemove`). It functions entirely read-only, rendering it safe for execution on production servers of any scale. |
| **Discord Snowflake Requirement** | Roster rows rely strictly on 18-digit Discord user IDs (`discord_user_id`). Names or email addresses are not utilized for joins. Capture snowflakes during initial user onboarding forms rather than attempting manual backfills. |
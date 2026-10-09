Monitor Instagram location posts and build creator leads in Google Sheets with Telegram alerts

https://n8nworkflows.xyz/workflows/monitor-instagram-location-posts-and-build-creator-leads-in-google-sheets-with-telegram-alerts-20557


# Monitor Instagram location posts and build creator leads in Google Sheets with Telegram alerts

### 1. Workflow Overview

This workflow automates the tracking of Instagram locations via the RecentReborn API, logs newly discovered posts to Google Sheets, builds and maintains a local creator lead list, and broadcasts rich Telegram alerts regarding high-engagement posts, creator tier changes, monitoring summaries, and system or API errors.

The architecture is divided into two primary logical pathways:
- **1.1 Input Reception & Setup Flow:** Captures location search inputs via an n8n Form trigger, queries the RecentReborn API, presents matching choices back to the user via an interactive form, and saves the selected locations to Google Sheets.
- **1.2 Scheduled Monitoring & Processing Engine:** Runs periodically (every 30 minutes) or via manual trigger to fetch active locations from Google Sheets, check API posts per location using pagination and watermarks, filter and deduplicate records, write new rows to the "Posts" and "Creators" tables, update monitoring state watermarks, and dispatch consolidated alerts via Telegram.

---

### 2. Block-by-Block Analysis

#### Block 1: Entry Triggers & Configuration
- **Overview:** Initializes workflow execution via a Form submission, a 30-minute interval schedule, or a manual test, and defines global parameters utilized across downstream nodes.
- **Nodes Involved:** `When Location Setup Form Submitted`, `Every 30 Minutes Trigger`, `Manual Workflow Test Trigger`, `Set Configuration Parameters`, `If Setup Form Submission`.
- **Node Details:**
  - **When Location Setup Form Submitted** (`n8n-nodes-base.formTrigger`): 
    - *Technical Role:* Webhook-based form trigger serving as the onboarding entry point.
    - *Configuration:* Path configured to `instagram-location-setup`, collects a single required field (`Location search`).
    - *Connections:* Outputs to `Set Configuration Parameters`.
    - *Edge Cases:* Malformed inputs or webhook accessibility issues.
  - **Every 30 Minutes Trigger** (`n8n-nodes-base.scheduleTrigger`):
    - *Technical Role:* Cron-like recurring trigger firing every 30 minutes.
    - *Configuration:* Interval set to 30 minutes.
    - *Connections:* Outputs to `Set Configuration Parameters`.
  - **Manual Workflow Test Trigger** (`n8n-nodes-base.manualTrigger`):
    - *Technical Role:* Allows manual execution for testing and debugging.
    - *Connections:* Outputs to `Set Configuration Parameters`.
  - **Set Configuration Parameters** (`n8n-nodes-base.set`):
    - *Technical Role:* Sets core environment variables, thresholds, and configuration values.
    - *Configuration:* Assigns variables including `sheetUrl`, `telegramChatId`, `setupFormUrl`, `formats` (`all`), `leadMinLikes` (`50`), `viralLikes` (`500`), `regularThreshold` (`3`), and `sendDigest` (`true`).
    - *Connections:* Input from triggers; output to `If Setup Form Submission`.
  - **If Setup Form Submission** (`n8n-nodes-base.if`):
    - *Technical Role:* Routes execution depending on whether the trigger originated from the setup form or the scheduled monitoring cycle.
    - *Configuration:* Evaluates `={{ $json['Location search'] !== undefined ? 'yes' : 'no' }}`.
    - *Connections:* Input from `Set Configuration Parameters`; True path leads to `Fetch Location Data`, False path leads to `Read Locations from Sheets`.

---

#### Block 2: Location Setup & Discovery
- **Overview:** Searches the external RecentReborn API for matching Instagram locations based on user input, parses the results, and prompts the user to select specific locations for monitoring.
- **Nodes Involved:** `Fetch Location Data`, `Generate Location Choices`, `If Locations Found`, `Select Locations Form`, `Form No Match Acknowledgment`, `Form Lookup Failed Acknowledgment`.
- **Node Details:**
  - **Fetch Location Data** (`n8n-nodes-base.httpRequest`):
    - *Technical Role:* External API call to look up location matches.
    - *Configuration:* GET request to `https://app.recentreborn.com/api/v1/locations` with query parameter `query` mapped from input. Uses Generic Header Authentication (`Bearer`). Error output handled separately via `continueErrorOutput`.
    - *Connections:* Input from `If Setup Form Submission`; outputs to `Generate Location Choices` (success) and `Form Lookup Failed Acknowledgment` (error).
    - *Edge Cases:* API authentication errors, rate limiting, network timeouts.
  - **Generate Location Choices** (`n8n-nodes-base.code`):
    - *Technical Role:* JavaScript code node transforming the API payload into a dropdown-compatible format for form selection.
    - *Configuration:* Filters out null/undefined IDs, limits choices to 30 items, constructs formatted location labels.
    - *Connections:* Input from `Fetch Location Data`; output to `If Locations Found`.
  - **If Locations Found** (`n8n-nodes-base.if`):
    - *Technical Role:* Checks whether locations were successfully matched.
    - *Configuration:* Evaluates `={{ $json.found ? 'yes' : 'no' }}`.
    - *Connections:* Input from `Generate Location Choices`; True path leads to `Select Locations Form`, False path leads to `Form No Match Acknowledgment`.
  - **Select Locations Form** (`n8n-nodes-base.form`):
    - *Technical Role:* Presents an interactive multi-select form to the user for choosing which locations to monitor.
    - *Configuration:* Defined via JSON output mapping form fields dynamically.
    - *Connections:* Input from `If Locations Found`; output to `Prepare Rows for Locations`.
  - **Form No Match Acknowledgment** (`n8n-nodes-base.form`):
    - *Technical Role:* Completion form displayed when no location matches the search query.
    - *Connections:* Input from `If Locations Found`.
  - **Form Lookup Failed Acknowledgment** (`n8n-nodes-base.form`):
    - *Technical Role:* Completion form displayed when the API request fails.
    - *Connections:* Input from `Fetch Location Data` (error branch).

---

#### Block 3: Location Persistence & Configuration Saving
- **Overview:** Processes user selections from the setup form, formats them into rows, upserts them into the Google Sheets "Locations" tab, and displays a completion message.
- **Nodes Involved:** `Prepare Rows for Locations`, `Upsert Locations in Sheets`, `Form Saved Acknowledgment`.
- **Node Details:**
  - **Prepare Rows for Locations** (`n8n-nodes-base.code`):
    - *Technical Role:* JavaScript node mapping selected form labels back to location IDs and constructing row objects.
    - *Configuration:* Validates selections and throws an error if empty.
    - *Connections:* Input from `Select Locations Form`; output to `Upsert Locations in Sheets`.
  - **Upsert Locations in Sheets** (`n8n-nodes-base.googleSheets`):
    - *Technical Role:* Google Sheets integration node to append or update location entries.
    - *Configuration:* Uses "appendOrUpdate" operation on sheet "Locations", matching on column `Location ID`. Document ID pulled from `sheetUrl` configuration parameter.
    - *Connections:* Input from `Prepare Rows for Locations`; output to `Form Saved Acknowledgment`.
    - *Edge Cases:* Google Sheets API write errors, quota limits, authorization failure.
  - **Form Saved Acknowledgment** (`n8n-nodes-base.form`):
    - *Technical Role:* Completion form confirming successful save operations.
    - *Connections:* Input from `Upsert Locations in Sheets`.

---

#### Block 4: Scheduled Location Retrieval & Setup Validation
- **Overview:** Reads configured locations from Google Sheets, filters out inactive ones, establishes cutoff timestamps, and verifies if any locations exist for monitoring.
- **Nodes Involved:** `Read Locations from Sheets`, `Select Active Location Jobs`, `If Setup Required`, `Notify Setup Reminder via Telegram`, `Fetch Creators from Sheets`, `Create Search Jobs`.
- **Node Details:**
  - **Read Locations from Sheets** (`n8n-nodes-base.googleSheets`):
    - *Technical Role:* Retrieves all rows from the "Locations" sheet.
    - *Configuration:* Operation "get" (default) on sheet "Locations". Always outputs data.
    - *Connections:* Input from `If Setup Form Submission` (scheduled path); output to `Select Active Location Jobs`.
  - **Select Active Location Jobs** (`n8n-nodes-base.code`):
    - *Technical Role:* Filters active location rows, computes lookback/cutoff timestamps, and checks whether setup reminders should be sent.
    - *Configuration:* JavaScript logic utilizing workflow static data to rate-limit setup reminders to once every 24 hours.
    - *Connections:* Input from `Read Locations from Sheets`; output to `If Setup Required`.
  - **If Setup Required** (`n8n-nodes-base.if`):
    - *Technical Role:* Branches execution depending on whether active locations are configured.
    - *Configuration:* Evaluates `={{ $json.setupNeeded ? 'yes' : 'no' }}`.
    - *Connections:* Input from `Select Active Location Jobs`; True path leads to `Notify Setup Reminder via Telegram`, False path leads to `Fetch Creators from Sheets`.
  - **Notify Setup Reminder via Telegram** (`n8n-nodes-base.telegram`):
    - *Technical Role:* Sends a setup reminder notification to the designated Telegram chat.
    - *Configuration:* HTML parse mode enabled, sends alert text if no locations are active.
    - *Connections:* Input from `If Setup Required`.
    - *Edge Cases:* Telegram API errors, incorrect chat ID.
  - **Fetch Creators from Sheets** (`n8n-nodes-base.googleSheets`):
    - *Technical Role:* Reads existing creator records from the "Creators" tab to prepare context for deduplication and scoring.
    - *Configuration:* Retrieves rows from sheet "Creators" once per execution.
    - *Connections:* Input from `If Setup Required` (False branch); output to `Create Search Jobs`.
  - **Create Search Jobs** (`n8n-nodes-base.code`):
    - *Technical Role:* Re-emits active location jobs after the Google Sheets read node replaced the item context.
    - *Connections:* Input from `Fetch Creators from Sheets`; output to `Process Locations in Batches`.

---

#### Block 5: Location Post Scraping & Batch Processing
- **Overview:** Iterates through active locations in batches, queries the RecentReborn API for recent location posts with pagination support, and handles search errors.
- **Nodes Involved:** `Process Locations in Batches`, `Fetch Posts by Location`, `Assign Location Tags to Results`, `Log Search Errors`.
- **Node Details:**
  - **Process Locations in Batches** (`n8n-nodes-base.splitInBatches`):
    - *Technical Role:* Loop controller processing active location jobs sequentially.
    - *Connections:* Input from `Create Search Jobs` and `Log Search Errors`; outputs to `Fetch Posts by Location` (loop item) and `Analyze New Instagram Posts` (loop completion).
  - **Fetch Posts by Location** (`n8n-nodes-base.httpRequest`):
    - *Technical Role:* External API request fetching recent Instagram posts for a specific location ID with built-in pagination.
    - *Configuration:* GET request to `https://app.recentreborn.com/api/v1/search/location` with `location_id`. Configured with automatic pagination handling up to 3 pages using `pagination_token`, with completion rules checking post timestamps against the location cutoff. Error output routed via `continueErrorOutput`.
    - *Connections:* Input from `Process Locations in Batches`; outputs to `Assign Location Tags to Results` (success) and `Log Search Errors` (error).
    - *Edge Cases:* API timeouts, rate limits, invalid location IDs.
  - **Assign Location Tags to Results** (`n8n-nodes-base.code`):
    - *Technical Role:* Re-attaches location metadata to each retrieved post item.
    - *Connections:* Input from `Fetch Posts by Location`; output loops back to `Process Locations in Batches`.
  - **Log Search Errors** (`n8n-nodes-base.code`):
    - *Technical Role:* Captures failed location search requests into a structured error marker object.
    - *Connections:* Input from `Fetch Posts by Location` (error branch); output loops back to `Process Locations in Batches`.

---

#### Block 6: Post Analysis, Filtering & Deduplication
- **Overview:** Consolidates all location search results, filters out duplicate and outdated posts, checks media formats, identifies high-engagement (viral) posts, and compiles structured datasets for Google Sheets and Telegram.
- **Nodes Involved:** `Analyze New Instagram Posts`, `If New Posts Exist`.
- **Node Details:**
  - **Analyze New Instagram Posts** (`n8n-nodes-base.code`):
    - *Technical Role:* Core JavaScript analytics engine processing raw posts against configuration thresholds (`leadMinLikes`, `viralLikes`, `regularThreshold`, `formats`), updating creator stats, and building summary payloads.
    - *Connections:* Input from `Process Locations in Batches` (loop completion); output to `If New Posts Exist`.
  - **If New Posts Exist** (`n8n-nodes-base.if`):
    - *Technical Role:* Determines if any new posts were discovered during the run.
    - *Configuration:* Evaluates `={{ $json.postRows.length > 0 ? 'yes' : 'no' }}`.
    - *Connections:* Input from `Analyze New Instagram Posts`; True path leads to `Split Posts into Rows`, False path leads to `If Creator Updates Exist`.

---

#### Block 7: Google Sheets Persistence (Posts, Creators, & Location State)
- **Overview:** Appends new posts to the "Posts" sheet, upserts creator records into the "Creators" sheet, updates location watermarks in the "Locations" sheet, and handles write failures.
- **Nodes Involved:** `Split Posts into Rows`, `Add Posts to Sheets`, `Restore Post Summary Data`, `If Creator Updates Exist`, `Divide Creator Data Rows`, `Update Creators in Sheets`, `Restore Creator Summary Data`, `Separate Location State Rows`, `Modify Location State in Sheets`, `Notify Sheet Write Failure`.
- **Node Details:**
  - **Split Posts into Rows** (`n8n-nodes-base.splitOut`):
    - *Technical Role:* Explodes the `postRows` array into individual items for batch insertion.
    - *Connections:* Input from `If New Posts Exist`; output to `Add Posts to Sheets`.
  - **Add Posts to Sheets** (`n8n-nodes-base.googleSheets`):
    - *Technical Role:* Appends new post records to the "Posts" sheet.
    - *Configuration:* Append operation on sheet "Posts". Error branch handled via `continueErrorOutput`.
    - *Connections:* Input from `Split Posts into Rows`; outputs to `Restore Post Summary Data` (success) and `Notify Sheet Write Failure` (error).
  - **Restore Post Summary Data** (`n8n-nodes-base.code`):
    - *Technical Role:* Restores the original single summary item context after row splitting.
    - *Connections:* Input from `Add Posts to Sheets`; output to `If Creator Updates Exist`.
  - **If Creator Updates Exist** (`n8n-nodes-base.if`):
    - *Technical Role:* Checks if creator records require updating.
    - *Configuration:* Evaluates `={{ $json.creatorRows.length > 0 ? 'yes' : 'no' }}`.
    - *Connections:* Inputs from `If New Posts Exist` (False branch) and `Restore Post Summary Data`; True path leads to `Divide Creator Data Rows`, False path leads to `Separate Location State Rows`.
  - **Divide Creator Data Rows** (`n8n-nodes-base.splitOut`):
    - *Technical Role:* Explodes the `creatorRows` array into individual items for upserting.
    - *Connections:* Input from `If Creator Updates Exist`; output to `Update Creators in Sheets`.
  - **Update Creators in Sheets** (`n8n-nodes-base.googleSheets`):
    - *Technical Role:* Upserts creator lead data into the "Creators" sheet.
    - *Configuration:* Append or Update operation on sheet "Creators", matching on `Author ID`. Error branch handled via `continueErrorOutput`.
    - *Connections:* Input from `Divide Creator Data Rows`; outputs to `Restore Creator Summary Data` (success) and `Notify Sheet Write Failure` (error).
  - **Restore Creator Summary Data** (`n8n-nodes-base.code`):
    - *Technical Role:* Restores the single summary item context after creator row splitting.
    - *Connections:* Input from `Update Creators in Sheets`; output to `Separate Location State Rows`.
  - **Separate Location State Rows** (`n8n-nodes-base.splitOut`):
    - *Technical Role:* Explodes the `locationRows` array into individual items for state updating.
    - *Connections:* Inputs from `If Creator Updates Exist` (False branch) and `Restore Creator Summary Data`; output to `Modify Location State in Sheets`.
  - **Modify Location State in Sheets** (`n8n-nodes-base.googleSheets`):
    - *Technical Role:* Updates location monitoring watermarks and status in the "Locations" sheet.
    - *Configuration:* Append or Update operation on sheet "Locations", matching on `Location ID`. Error branch handled via `continueErrorOutput`.
    - *Connections:* Input from `Separate Location State Rows`; outputs to `Compose Telegram Alerts` (success) and `Notify Sheet Write Failure` (error).
  - **Notify Sheet Write Failure** (`n8n-nodes-base.telegram`):
    - *Technical Role:* Sends a Telegram alert when writing posts, creators, or location state to Google Sheets fails.
    - *Configuration:* HTML parse mode enabled, executes once per failure.
    - *Connections:* Inputs from error error-branches of Google Sheets nodes.

---

#### Block 8: Telegram Notification & Alert Dispatch
- **Overview:** Composes escaped HTML Telegram notifications for viral posts, new regular creators, run digests, and search errors, then dispatches them to the target chat.
- **Nodes Involved:** `Compose Telegram Alerts`, `Dispatch Telegram Notifications`.
- **Node Details:**
  - **Compose Telegram Alerts** (`n8n-nodes-base.code`):
    - *Technical Role:* JavaScript code node generating formatted HTML notification messages for viral posts, regular creators, digests, and location errors.
    - *Configuration:* Sanitizes user input and constructs message objects with preview settings.
    - *Connections:* Input from `Modify Location State in Sheets`; output to `Dispatch Telegram Notifications`.
  - **Dispatch Telegram Notifications** (`n8n-nodes-base.telegram`):
    - *Technical Role:* Sends composed notifications to the configured Telegram chat.
    - *Configuration:* HTML parse mode enabled, preview toggle configured dynamically based on item metadata.
    - *Connections:* Input from `Compose Telegram Alerts`.
    - *Edge Cases:* Telegram API rate limits, message length limits.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation & setup instructions | — | — | Monitor Instagram locations and build a local creator lead list with Google Sheets and Telegram |
| Sticky Note1 | n8n-nodes-base.stickyNote | Groups entry trigger nodes | — | — | Workflow entry triggers |
| Sticky Note2 | n8n-nodes-base.stickyNote | Groups configuration and routing logic | — | — | Configure and route |
| Sticky Note3 | n8n-nodes-base.stickyNote | Groups location lookup nodes | — | — | Lookup location matches |
| Sticky Note4 | n8n-nodes-base.stickyNote | Groups location saving nodes | — | — | Save selected locations |
| Sticky Note5 | n8n-nodes-base.stickyNote | Groups location loading nodes | — | — | Load monitored locations |
| Sticky Note6 | n8n-nodes-base.stickyNote | Groups setup check and reminder nodes | — | — | Check setup requirement |
| Sticky Note7 | n8n-nodes-base.stickyNote | Groups creator context loading nodes | — | — | Prepare creator context |
| Sticky Note8 | n8n-nodes-base.stickyNote | Groups location search iteration nodes | — | — | Search each location |
| Sticky Note9 | n8n-nodes-base.stickyNote | Groups search result normalization nodes | — | — | Normalize search results |
| Sticky Note10 | n8n-nodes-base.stickyNote | Groups post summary analysis nodes | — | — | Process post summary |
| Sticky Note11 | n8n-nodes-base.stickyNote | Groups post persistence nodes | — | — | Append new posts |
| Sticky Note12 | n8n-nodes-base.stickyNote | Groups creator upsert nodes | — | — | Upsert creator records |
| Sticky Note13 | n8n-nodes-base.stickyNote | Groups location state update nodes | — | — | Update location state |
| Sticky Note14 | n8n-nodes-base.stickyNote | Groups Telegram summary notification nodes | — | — | Send Telegram summary |
| Sticky Note15 | n8n-nodes-base.stickyNote | Groups sheet failure reporting nodes | — | — | Report sheet failures |
| When Location Setup Form Submitted | n8n-nodes-base.formTrigger | Triggers workflow on setup form submission | — | Set Configuration Parameters | — |
| Every 30 Minutes Trigger | n8n-nodes-base.scheduleTrigger | Triggers workflow every 30 minutes | — | Set Configuration Parameters | — |
| Manual Workflow Test Trigger | n8n-nodes-base.manualTrigger | Triggers workflow manually for testing | — | Set Configuration Parameters | — |
| Set Configuration Parameters | n8n-nodes-base.set | Sets global configuration variables | When Location Setup Form Submitted, Every 30 Minutes Trigger, Manual Workflow Test Trigger | If Setup Form Submission | — |
| If Setup Form Submission | n8n-nodes-base.if | Routes between setup form handling and scheduled monitoring | Set Configuration Parameters | Fetch Location Data, Read Locations from Sheets | — |
| Fetch Location Data | n8n-nodes-base.httpRequest | Queries RecentReborn API for locations | If Setup Form Submission | Generate Location Choices, Form Lookup Failed Acknowledgment | — |
| Generate Location Choices | n8n-nodes-base.code | Formats location API response into form choices | Fetch Location Data | If Locations Found | — |
| If Locations Found | n8n-nodes-base.if | Checks if location search returned results | Generate Location Choices | Select Locations Form, Form No Match Acknowledgment | — |
| Select Locations Form | n8n-nodes-base.form | Interactive form for selecting locations | If Locations Found | Prepare Rows for Locations | — |
| Prepare Rows for Locations | n8n-nodes-base.code | Maps selections to database rows | Select Locations Form | Upsert Locations in Sheets | — |
| Upsert Locations in Sheets | n8n-nodes-base.googleSheets | Saves monitored locations to Google Sheets | Prepare Rows for Locations | Form Saved Acknowledgment | — |
| Form Saved Acknowledgment | n8n-nodes-base.form | Displays success message on form completion | Upsert Locations in Sheets | — | — |
| Form No Match Acknowledgment | n8n-nodes-base.form | Displays no match message | If Locations Found | — | — |
| Form Lookup Failed Acknowledgment | n8n-nodes-base.form | Displays API failure message | Fetch Location Data | — | — |
| Read Locations from Sheets | n8n-nodes-base.googleSheets | Reads configured locations from Google Sheets | If Setup Form Submission | Select Active Location Jobs | — |
| Select Active Location Jobs | n8n-nodes-base.code | Filters active locations and computes cutoffs | Read Locations from Sheets | If Setup Required | — |
| If Setup Required | n8n-nodes-base.if | Checks if active locations are configured | Select Active Location Jobs | Notify Setup Reminder via Telegram, Fetch Creators from Sheets | — |
| Notify Setup Reminder via Telegram | n8n-nodes-base.telegram | Sends setup reminder via Telegram | If Setup Required | — | — |
| Fetch Creators from Sheets | n8n-nodes-base.googleSheets | Reads existing creator records | If Setup Required | Create Search Jobs | — |
| Create Search Jobs | n8n-nodes-base.code | Re-emits active location search jobs | Fetch Creators from Sheets | Process Locations in Batches | — |
| Process Locations in Batches | n8n-nodes-base.splitInBatches | Iterates over active location jobs | Create Search Jobs, Log Search Errors | Analyze New Instagram Posts, Fetch Posts by Location | — |
| Fetch Posts by Location | n8n-nodes-base.httpRequest | Queries RecentReborn API for location posts | Process Locations in Batches | Assign Location Tags to Results, Log Search Errors | — |
| Assign Location Tags to Results | n8n-nodes-base.code | Attaches location metadata to post items | Fetch Posts by Location | Process Locations in Batches | — |
| Log Search Errors | n8n-nodes-base.code | Captures search errors into marker items | Fetch Posts by Location | Process Locations in Batches | — |
| Analyze New Instagram Posts | n8n-nodes-base.code | Analyzes, filters, and deduplicates posts | Process Locations in Batches | If New Posts Exist | — |
| If New Posts Exist | n8n-nodes-base.if | Checks if new posts were found | Analyze New Instagram Posts | Split Posts into Rows, If Creator Updates Exist | — |
| Split Posts into Rows | n8n-nodes-base.splitOut | Explodes post rows for batch insertion | If New Posts Exist | Add Posts to Sheets | — |
| Add Posts to Sheets | n8n-nodes-base.googleSheets | Appends new post records to Google Sheets | Split Posts into Rows | Restore Post Summary Data, Notify Sheet Write Failure | — |
| Restore Post Summary Data | n8n-nodes-base.code | Restores single summary item context | Add Posts to Sheets | If Creator Updates Exist | — |
| If Creator Updates Exist | n8n-nodes-base.if | Checks if creator records need updating | If New Posts Exist, Restore Post Summary Data | Divide Creator Data Rows, Separate Location State Rows | — |
| Divide Creator Data Rows | n8n-nodes-base.splitOut | Explodes creator rows for upserting | If Creator Updates Exist | Update Creators in Sheets | — |
| Update Creators in Sheets | n8n-nodes-base.googleSheets | Upserts creator leads into Google Sheets | Divide Creator Data Rows | Restore Creator Summary Data, Notify Sheet Write Failure | — |
| Restore Creator Summary Data | n8n-nodes-base.code | Restores summary context after creator upsert | Update Creators in Sheets | Separate Location State Rows | — |
| Separate Location State Rows | n8n-nodes-base.splitOut | Explodes location state updates | If Creator Updates Exist, Restore Creator Summary Data | Modify Location State in Sheets | — |
| Modify Location State in Sheets | n8n-nodes-base.googleSheets | Updates location watermarks in Google Sheets | Separate Location State Rows | Compose Telegram Alerts, Notify Sheet Write Failure | — |
| Notify Sheet Write Failure | n8n-nodes-base.telegram | Sends Telegram alert on sheet write failure | Add Posts to Sheets, Update Creators in Sheets, Modify Location State in Sheets | — | — |
| Compose Telegram Alerts | n8n-nodes-base.code | Composes HTML notification messages | Modify Location State in Sheets | Dispatch Telegram Notifications | — |
| Dispatch Telegram Notifications | n8n-nodes-base.telegram | Sends notifications to Telegram | Compose Telegram Alerts | — | — |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers and Configuration:**
   - Add a **Form Trigger** node (`When Location Setup Form Submitted`), set path to `instagram-location-setup`, and configure a required text field `Location search`.
   - Add a **Schedule Trigger** node (`Every 30 Minutes Trigger`) and set interval to 30 minutes.
   - Add a **Manual Trigger** node (`Manual Workflow Test Trigger`).
   - Add a **Set** node (`Set Configuration Parameters`) and assign string/number/boolean assignments for `sheetUrl`, `telegramChatId`, `setupFormUrl`, `formats` (`all`), `firstRunLookbackHours` (`6`), `overlapMinutes` (`60`), `leadMinLikes` (`50`), `viralLikes` (`500`), `regularThreshold` (`3`), and `sendDigest` (`true`). Connect all three triggers to this node.
   - Add an **If** node (`If Setup Form Submission`) to evaluate whether `{{ $json['Location search'] !== undefined ? 'yes' : 'no' }}` equals `yes`.

2. **Build Location Setup Flow (True Branch):**
   - Add an **HTTP Request** node (`Fetch Location Data`) configured with Generic HTTP Header Auth (`Authorization: Bearer <API_KEY>`), GET method, URL `https://app.recentreborn.com/api/v1/locations`, and query parameter `query` = `={{ $json['Location search'] }}`. Set error handling to continue on error output.
   - Add a **Code** node (`Generate Location Choices`) to parse `body.locations` and build multi-select dropdown options.
   - Add an **If** node (`If Locations Found`) evaluating `={{ $json.found ? 'yes' : 'no' }}`.
   - Add a **Form** node (`Select Locations Form`) configured with JSON output `={{ JSON.stringify($json.formFields) }}`.
   - Add a **Code** node (`Prepare Rows for Locations`) to map selected labels to location IDs and build rows.
   - Add a **Google Sheets** node (`Upsert Locations in Sheets`) with operation `appendOrUpdate`, sheet name `Locations`, matching on `Location ID`.
   - Add completion **Form** nodes for `Form Saved Acknowledgment`, `Form No Match Acknowledgment`, and `Form Lookup Failed Acknowledgment`.

3. **Build Scheduled Monitoring & Processing Engine (False Branch):**
   - Add a **Google Sheets** node (`Read Locations from Sheets`) to read sheet "Locations" with always output data enabled.
   - Add a **Code** node (`Select Active Location Jobs`) to filter active rows and compute cutoffs and recent IDs.
   - Add an **If** node (`If Setup Required`) evaluating `={{ $json.setupNeeded ? 'yes' : 'no' }}`.
   - Add a **Telegram** node (`Notify Setup Reminder via Telegram`) with HTML parse mode for the True path.
   - Add a **Google Sheets** node (`Fetch Creators from Sheets`) for the False path to read sheet "Creators".
   - Add a **Code** node (`Create Search Jobs`) to re-emit active location jobs.
   - Add a **Split In Batches** node (`Process Locations in Batches`) to loop through locations.
   - Add an **HTTP Request** node (`Fetch Posts by Location`) for URL `https://app.recentreborn.com/api/v1/search/location` with query parameter `location_id`, pagination enabled for up to 3 requests with `pagination_token`, and error handling set to continue on error.
   - Add a **Code** node (`Assign Location Tags to Results`) connecting back to the batch splitter, and a **Code** node (`Log Search Errors`) for error handling.

4. **Build Post Analysis and Persistence Pipeline:**
   - Add a **Code** node (`Analyze New Instagram Posts`) after the batch loop completion to process posts, filter formats, calculate engagement scores, and compile summary payloads.
   - Add an **If** node (`If New Posts Exist`) evaluating `={{ $json.postRows.length > 0 ? 'yes' : 'no' }}`.
   - Add a **Split Out** node (`Split Posts into Rows`) for `postRows`.
   - Add a **Google Sheets** node (`Add Posts to Sheets`) with append operation on sheet "Posts" (continue on error enabled).
   - Add a **Code** node (`Restore Post Summary Data`) to restore the summary context.
   - Add an **If** node (`If Creator Updates Exist`) evaluating `={{ $json.creatorRows.length > 0 ? 'yes' : 'no' }}`.
   - Add a **Split Out** node (`Divide Creator Data Rows`) for `creatorRows`.
   - Add a **Google Sheets** node (`Update Creators in Sheets`) with appendOrUpdate operation on sheet "Creators", matching on `Author ID` (continue on error enabled).
   - Add a **Code** node (`Restore Creator Summary Data`).
   - Add a **Split Out** node (`Separate Location State Rows`) for `locationRows`.
   - Add a **Google Sheets** node (`Modify Location State in Sheets`) with appendOrUpdate operation on sheet "Locations", matching on `Location ID` (continue on error enabled).
   - Add a **Telegram** node (`Notify Sheet Write Failure`) connected to the error output of all Google Sheets write nodes.

5. **Build Notification Pipeline:**
   - Add a **Code** node (`Compose Telegram Alerts`) to generate HTML-formatted messages for viral posts, new regulars, digests, and errors.
   - Add a **Telegram** node (`Dispatch Telegram Notifications`) to send the composed alerts using HTML parse mode and dynamic preview configuration.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Monitor Instagram locations and build a local creator lead list with Google Sheets and Telegram | Workflow title and functional overview |
| RecentReborn API Documentation & Account Setup | [RecentReborn Platform](https://app.recentreborn.com) |
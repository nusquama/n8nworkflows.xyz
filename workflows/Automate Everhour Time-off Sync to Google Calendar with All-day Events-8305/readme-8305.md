Automate Everhour Time-off Sync to Google Calendar with All-day Events

https://n8nworkflows.xyz/workflows/automate-everhour-time-off-sync-to-google-calendar-with-all-day-events-8305


# Automate Everhour Time-off Sync to Google Calendar with All-day Events

---

### 1. Workflow Overview

This n8n workflow automates the synchronization of approved time-off records from Everhour into a Google Calendar as all-day events. It is designed for teams who use Everhour for time tracking and Google Calendar for visualizing time off, enabling seamless calendar updates that reflect current approved absences. The workflow runs on a schedule, fetches time-off data, creates or updates corresponding events, and cleans up obsolete calendar events.

**Logical blocks:**

- **1.1 Configuration and Trigger**  
  Setup calendar ID and trigger workflow execution on a time interval.

- **1.2 Everhour Data Retrieval and Filtering**  
  Fetch time-off assignments from Everhour API and filter only approved time-off entries.

- **1.3 Time-Off Item Preparation and Key Generation**  
  Process raw time-off data into per-day items with unique external keys, expanding multi-day periods and deriving all-day event flags.

- **1.4 Calendar Event Upsert Logic**  
  For each time-off day item, search Google Calendar for an existing event by external key, then update if found or create a new all-day event.

- **1.5 Obsolete Event Cleanup**  
  Identify and delete calendar events that no longer correspond to current approved time-off, both on a per-assignment basis and globally.

- **1.6 Auxiliary Logic and Utilities**  
  Build assignment groups and global key sets to assist cleanup and synchronization logic.

---

### 2. Block-by-Block Analysis

#### 1.1 Configuration and Trigger

**Overview:**  
Defines the Google Calendar ID to use and triggers the workflow on a fixed hourly schedule.

**Nodes Involved:**  
- Config  
- Schedule Trigger

**Node Details:**

- **Config**  
  - Type: Set  
  - Role: Stores configuration data, particularly the Google Calendar ID used by calendar nodes.  
  - Configuration: A single string parameter `calendarId` where users insert their calendar ID (e.g., `team@group.calendar.google.com`).  
  - Inputs: None (entry data)  
  - Outputs: Provides config JSON for downstream nodes.  
  - Edge cases: Misconfigured calendar ID will cause Google Calendar API failures.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution on an hourly interval.  
  - Configuration: Runs every hour (interval field set to hours).  
  - Inputs: None  
  - Outputs: Triggers downstream data retrieval.  
  - Edge cases: Missed triggers if n8n is down or throttled.

---

#### 1.2 Everhour Data Retrieval and Filtering

**Overview:**  
Fetches all assignments from Everhour via API using header authentication and filters out only the approved time-off entries.

**Nodes Involved:**  
- Everhour - Fetch Time Off  
- Filter Approved Time-Off

**Node Details:**

- **Everhour - Fetch Time Off**  
  - Type: HTTP Request  
  - Role: Calls Everhour API endpoint `/resource-planner/assignments` to retrieve time off and assignments.  
  - Configuration: Authenticated via Header Auth credential with `X-Api-Key`. No keys are hardcoded in the node, securing credentials in n8n.  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: JSON array of assignment/time-off objects.  
  - Edge cases: API key invalid or expired; network errors; rate limiting.

- **Filter Approved Time-Off**  
  - Type: If node  
  - Role: Filters items where `status` equals `"approved"`.  
  - Configuration: Condition checks `$json.status === "approved"` (case-sensitive).  
  - Inputs: Output from Everhour fetch  
  - Outputs: Only approved time-off entries proceed.  
  - Edge cases: Unexpected status values; missing status fields.

---

#### 1.3 Time-Off Item Preparation and Key Generation

**Overview:**  
Transforms approved time-off data into daily items with unique external keys (`everhour:<id>:<date>`) and determines all-day flags. Also builds assignment groups and a global key set for synchronization.

**Nodes Involved:**  
- Prepare Time-Off Items  
- Build Calendar Payload  
- Build Assignment Groups  
- Build Global Key Set

**Node Details:**

- **Prepare Time-Off Items**  
  - Type: Code  
  - Role: Expands multi-day approved time-off into single-day items, generates unique external keys, marks each as all-day or half-day based on Everhour's original period, and skips weekends if configured.  
  - Configuration: JavaScript code iterates through approved items, yielding one item per included day with fields like `startDate`, `endDateExclusive`, `externalKey`, `isAllDay`.  
  - Inputs: Approved time-off items from the filter node  
  - Outputs: One item per day off with detailed metadata.  
  - Edge cases: Invalid or missing dates; half-day handling is conservative—all events created as all-day but with `isAllDay` flag saved.

- **Build Calendar Payload**  
  - Type: Set  
  - Role: Prepares search query objects keyed by `externalKey` for later Google Calendar event lookup.  
  - Configuration: Sets a property `searchQuery` to the item's `externalKey`.  
  - Inputs: Time-off day items from Prepare Time-Off Items  
  - Outputs: Items enriched with `searchQuery` for calendar searching.  
  - Edge cases: Missing keys would result in failed calendar event searches.

- **Build Assignment Groups**  
  - Type: Code  
  - Role: Aggregates per-day time-off items into groups per `assignmentId`, computing the min and max dates to define a search window for calendar events.  
  - Configuration: Uses JavaScript to collect `currentKeys`, `minDate`, and `maxDate` per assignment.  
  - Inputs: Items enriched with `externalKey` from Build Calendar Payload  
  - Outputs: One item per assignment group with date range and keys.  
  - Edge cases: Missing or malformed external keys affect grouping.

- **Build Global Key Set**  
  - Type: Code  
  - Role: Builds a global set of all current external keys and calculates overall min/max date windows for global cleanup operations.  
  - Configuration: Aggregates keys from all inputs; extends date range by default if none found.  
  - Inputs: All time-off day items with keys  
  - Outputs: Single item containing global `currentKeys` array and min/max search time.  
  - Edge cases: Empty inputs default to current date and one year forward.

---

#### 1.4 Calendar Event Upsert Logic

**Overview:**  
For each daily time-off item, searches Google Calendar for an event matching the external key; updates the event if found or creates a new all-day event otherwise.

**Nodes Involved:**  
- Find Existing Event by External Key  
- Attach Found Event ID  
- Check if Event Exists  
- Update All-Day Event  
- Create All-Day Event

**Node Details:**

- **Find Existing Event by External Key**  
  - Type: HTTP Request  
  - Role: Queries the Google Calendar events API filtering by the external key as the search query (`q`).  
  - Configuration: Uses Google OAuth credentials via predefined credential type. Calendar ID is dynamically fetched from the Config node. Limits results to 1 matching event.  
  - Inputs: Items from Build Calendar Payload  
  - Outputs: Search result JSON from Google API.  
  - Edge cases: API quota errors, no results found, malformed calendar ID.

- **Attach Found Event ID**  
  - Type: Code (Run Once for Each Item)  
  - Role: Extracts the found event ID from search response and attaches fields `existingEventId` and boolean `found` to the original item.  
  - Configuration: References the corresponding original item by index for data consistency.  
  - Inputs: Search results from previous node  
  - Outputs: Enriched item including event ID or null.  
  - Edge cases: Empty or malformed API responses.

- **Check if Event Exists**  
  - Type: If  
  - Role: Routes items based on whether a matching event was found (`found === true`).  
  - Inputs: Enriched items from Attach Found Event ID  
  - Outputs: Two branches — update path if found, create path if not.  
  - Edge cases: Missing `found` field or unexpected values.

- **Update All-Day Event**  
  - Type: Google Calendar  
  - Role: Updates existing event with new start/end dates, marks as all-day, and updates summary and description fields.  
  - Configuration: Uses calendar ID from Config node, event ID from item. Updates include all-day flag set to “yes” and descriptive text including external key and notes.  
  - Inputs: Items routed from Check if Event Exists (true branch)  
  - Outputs: Updated event details.  
  - Edge cases: Event deleted or changed between search and update, permission errors.

- **Create All-Day Event**  
  - Type: Google Calendar  
  - Role: Creates a new all-day event with similar fields as update node for time-off day.  
  - Configuration: Calendar ID from Config node, event data includes all-day flag and detailed description.  
  - Inputs: Items routed from Check if Event Exists (false branch)  
  - Outputs: Created event details.  
  - Edge cases: Quota limits, invalid calendar ID.

---

#### 1.5 Obsolete Event Cleanup

**Overview:**  
Identifies Google Calendar events that no longer correspond to any approved time-off keys and deletes them to keep the calendar clean and accurate.

**Nodes Involved:**  
- List Assignment Events  
- Find Obsolete Events (per assignment)  
- List All Everhour Events  
- Find Obsolete Events (global)  
- Delete Calendar Event

**Node Details:**

- **List Assignment Events**  
  - Type: HTTP Request  
  - Role: Lists Google Calendar events filtered by assignment ID prefix in external key to fetch events related to a particular assignment.  
  - Configuration: Uses calendar ID from Config node. Query filters events containing `everhour:<assignmentId>:`.  
  - Inputs: Assignment groups from Build Assignment Groups  
  - Outputs: Events JSON arrays per assignment.  
  - Edge cases: Large event sets, API limits.

- **Find Obsolete Events (per assignment)**  
  - Type: Code  
  - Role: Compares calendar events per assignment against the current keys for that assignment, outputs events whose keys are no longer current (stale).  
  - Configuration: Extracts keys from event descriptions using regex on `Key: everhour:<id>:<date>`.  
  - Inputs: Event lists from List Assignment Events and assignment groups.  
  - Outputs: List of obsolete event IDs for deletion.  
  - Edge cases: Events without expected key format are skipped.

- **List All Everhour Events**  
  - Type: HTTP Request  
  - Role: Lists all Google Calendar events containing `everhour:` in their description globally.  
  - Configuration: Uses calendar ID from Config node.  
  - Inputs: Output from Build Global Key Set  
  - Outputs: All events with Everhour keys.  
  - Edge cases: Large number of events may require pagination handling (maxResults=2500 is max per Google API call).

- **Find Obsolete Events (global)**  
  - Type: Code  
  - Role: Compares global calendar events to the global set of current keys and outputs events that are no longer valid.  
  - Inputs: Event list from List All Everhour Events and global keys from Build Global Key Set  
  - Outputs: List of obsolete event IDs for deletion.  
  - Edge cases: Same as per-assignment node.

- **Delete Calendar Event**  
  - Type: Google Calendar  
  - Role: Deletes the identified obsolete events from the calendar.  
  - Configuration: Uses calendar ID from Config node. Sends no update notifications (`sendUpdates: none`). Retry enabled on failure.  
  - Inputs: Obsolete event IDs from both Find Obsolete nodes  
  - Outputs: Deletion confirmations.  
  - Edge cases: Deletion failures due to permissions or event already deleted.

---

#### 1.6 Auxiliary Logic and Notes

**Overview:**  
Sticky notes and configuration reminders assist users to understand and maintain the workflow.

**Sticky Notes:**  
- Template Description (Read Me) — Overview, purpose, setup instructions, customization tips.  
- Note: Fetch Everhour — Credential guidance for Everhour API.  
- Note: Build Keys — Explanation of key structure for calendar event matching.  
- Note: Upsert Events — Explanation of search then update or create logic.  
- Note: Cleanup — Explains cleanup of stale calendar events.

---

### 3. Summary Table

| Node Name                     | Node Type              | Functional Role                                    | Input Node(s)                     | Output Node(s)                      | Sticky Note                                                                                                 |
|-------------------------------|------------------------|---------------------------------------------------|----------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger       | Starts workflow hourly                            | None                             | Everhour - Fetch Time Off, Config |                                                                                                             |
| Config                        | Set                    | Stores Google Calendar ID                         | None                             | Everhour - Fetch Time Off          | Set your Google Calendar ID here. Example: team-calendar@group.calendar.google.com                           |
| Everhour - Fetch Time Off     | HTTP Request           | Fetches assignments/time-off from Everhour API   | Schedule Trigger, Config          | Filter Approved Time-Off           | Uses Header Auth credential (add `X-Api-Key`). No hardcoded keys.                                            |
| Filter Approved Time-Off      | If                     | Filters only approved time-off entries            | Everhour - Fetch Time Off         | Prepare Time-Off Items             |                                                                                                             |
| Prepare Time-Off Items        | Code                   | Expands multi-day off to per-day items, sets keys| Filter Approved Time-Off          | Build Calendar Payload             | Generate per-day `externalKey` like `everhour:<id>:<date>`.                                                 |
| Build Calendar Payload        | Set                    | Prepares payload with search query per item       | Prepare Time-Off Items            | Find Existing Event by External Key, Build Assignment Groups, Build Global Key Set |                                                                                                             |
| Find Existing Event by External Key | HTTP Request           | Searches Google Calendar for event by key         | Build Calendar Payload            | Attach Found Event ID              |                                                                                                             |
| Attach Found Event ID         | Code                   | Extracts found event ID for each item             | Find Existing Event by External Key | Check if Event Exists              |                                                                                                             |
| Check if Event Exists         | If                     | Routes to update or create event branch           | Attach Found Event ID             | Update All-Day Event, Create All-Day Event | Search by key → update if found, otherwise create all-day event.                                            |
| Update All-Day Event          | Google Calendar        | Updates existing all-day event                     | Check if Event Exists (true)      | None                              | Calendar ID pulled from the Config node.                                                                    |
| Create All-Day Event          | Google Calendar        | Creates new all-day event                           | Check if Event Exists (false)     | None                              | Calendar ID pulled from the Config node.                                                                    |
| Build Assignment Groups       | Code                   | Groups time-off keys by assignment, sets date range| Build Calendar Payload            | List Assignment Events            |                                                                                                             |
| List Assignment Events        | HTTP Request           | Lists calendar events per assignment               | Build Assignment Groups           | Find Obsolete Events (per assignment) | Calendar ID pulled from the Config node.                                                                    |
| Find Obsolete Events (per assignment) | Code                   | Finds stale events per assignment to delete        | List Assignment Events, Build Assignment Groups | Delete Calendar Event               |                                                                                                             |
| Build Global Key Set          | Code                   | Builds global set of current keys and date range  | Build Calendar Payload            | List All Everhour Events          |                                                                                                             |
| List All Everhour Events      | HTTP Request           | Lists all calendar events with Everhour keys       | Build Global Key Set              | Find Obsolete Events (global)     | Calendar ID pulled from the Config node.                                                                    |
| Find Obsolete Events (global) | Code                   | Finds stale events globally to delete              | List All Everhour Events, Build Global Key Set | Delete Calendar Event               |                                                                                                             |
| Delete Calendar Event         | Google Calendar        | Deletes obsolete calendar events                    | Find Obsolete Events (per assignment), Find Obsolete Events (global) | None                              | Calendar ID pulled from the Config node. Retry enabled on failure.                                          |
| 🗒️ Template Description (Read Me) | Sticky Note            | Explains workflow purpose, setup, and customization| None                             | None                             | Everhour time-off sync to Google Calendar; setup instructions included.                                     |
| Note: Fetch Everhour          | Sticky Note            | Explains Everhour API credential usage              | None                             | None                             | Fetch approved time-off from Everhour using Header Auth (no keys in node).                                  |
| Note: Build Keys             | Sticky Note            | Explains key generation scheme                       | None                             | None                             | Generate per-day `externalKey` like `everhour:<id>:<date>`.                                                 |
| Note: Upsert Events          | Sticky Note            | Explains update vs create logic                      | None                             | None                             | Search by key → update if found, otherwise create all-day event.                                            |
| Note: Cleanup                | Sticky Note            | Explains stale event deletion logic                  | None                             | None                             | List events and delete those not present in current keys.                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Set Node named "Config"**  
   - Add a string field `calendarId`  
   - Set its value to your Google Calendar ID (e.g., `team@group.calendar.google.com`)  
   - Position it near the start of the workflow.

2. **Add a Schedule Trigger node "Schedule Trigger"**  
   - Set interval to run every 1 hour  
   - Connect its output to "Everhour - Fetch Time Off" and "Config" nodes.

3. **Add an HTTP Request node "Everhour - Fetch Time Off"**  
   - URL: `https://api.everhour.com/resource-planner/assignments`  
   - Authentication: Header Auth credential with header `X-Api-Key: <YOUR_EVERHOUR_API_KEY>`  
   - Connect input from "Schedule Trigger" and "Config" nodes.  
   - No body or query parameters required.

4. **Add an If node "Filter Approved Time-Off"**  
   - Condition: `$json.status === "approved"` (strict, case-sensitive)  
   - Connect input from "Everhour - Fetch Time Off" node's output.

5. **Add a Code node "Prepare Time-Off Items"**  
   - Paste the provided JavaScript code that expands multi-day time off to daily items with keys and all-day flags.  
   - Input from the true branch of "Filter Approved Time-Off".

6. **Add a Set node "Build Calendar Payload"**  
   - Assign a string field `searchQuery` to the value of `{{$json.externalKey}}`  
   - Include all other fields from the input.  
   - Input from "Prepare Time-Off Items".

7. **Add an HTTP Request node "Find Existing Event by External Key"**  
   - URL: `https://www.googleapis.com/calendar/v3/calendars/{{$items("Config")[0].json.calendarId}}/events`  
   - Query parameters:  
     - `singleEvents=true`  
     - `maxResults=1`  
     - `q={{$json.externalKey}}`  
   - Authentication: Google API OAuth2 credential  
   - Input from "Build Calendar Payload".

8. **Add a Code node "Attach Found Event ID"**  
   - Mode: Run Once for Each Item  
   - Code extracts event ID from HTTP response and adds `existingEventId` and boolean `found`.  
   - Input from "Find Existing Event by External Key".

9. **Add an If node "Check if Event Exists"**  
   - Condition: `$json.found === true`  
   - Input from "Attach Found Event ID".

10. **Add a Google Calendar node "Update All-Day Event"** (for true branch)  
    - Operation: Update event  
    - Calendar ID: `{{$items("Config")[0].json.calendarId}}`  
    - Event ID: `{{$json.existingEventId}}`  
    - Set start and end dates (`startDate`, `endDateExclusive`)  
    - Mark event as all-day (`allday: yes`)  
    - Summary: `{{$json.employeeName}} - {{$json.type}}`  
    - Description: Template including Everhour PTO key, type, status, note.

11. **Add a Google Calendar node "Create All-Day Event"** (for false branch)  
    - Operation: Create event  
    - Same calendar ID and event fields as update node.

12. **Add a Code node "Build Assignment Groups"**  
    - Groups daily time off by `assignmentId` with date ranges and current keys.  
    - Input from "Build Calendar Payload".

13. **Add an HTTP Request node "List Assignment Events"**  
    - URL: `https://www.googleapis.com/calendar/v3/calendars/{{$items("Config")[0].json.calendarId}}/events`  
    - Query parameters:  
      - `singleEvents=true`  
      - `maxResults=2500`  
      - `orderBy=startTime`  
      - `q=everhour:{{$json.assignmentId}}:`  
    - Authentication: Google API OAuth2  
    - Input from "Build Assignment Groups".

14. **Add a Code node "Find Obsolete Events (per assignment)"**  
    - Compares calendar events per assignment to current keys and outputs obsolete events.  
    - Input from "List Assignment Events" and "Build Assignment Groups".

15. **Add a Code node "Build Global Key Set"**  
    - Aggregates all current keys and global min/max date range.  
    - Input from "Build Calendar Payload".

16. **Add an HTTP Request node "List All Everhour Events"**  
    - URL: `https://www.googleapis.com/calendar/v3/calendars/{{$items("Config")[0].json.calendarId}}/events`  
    - Query parameters:  
      - `singleEvents=true`  
      - `maxResults=2500`  
      - `orderBy=startTime`  
      - `q=everhour:`  
    - Authentication: Google API OAuth2  
    - Input from "Build Global Key Set".

17. **Add a Code node "Find Obsolete Events (global)"**  
    - Compares all calendar events to global current keys and outputs obsolete events.  
    - Input from "List All Everhour Events" and "Build Global Key Set".

18. **Add a Google Calendar node "Delete Calendar Event"**  
    - Operation: Delete event  
    - Calendar ID: `{{$items("Config")[0].json.calendarId}}`  
    - Event ID: `{{$json.eventId}}`  
    - Option `sendUpdates`: none  
    - Retry on failure enabled.  
    - Input from "Find Obsolete Events (per assignment)" and "Find Obsolete Events (global)".

19. **Add Sticky Notes** to document workflow purpose, setup, and node functions:  
    - Template Description  
    - Note: Fetch Everhour  
    - Note: Build Keys  
    - Note: Upsert Events  
    - Note: Cleanup

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                   |
|-------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Workflow designed for teams using Everhour and Google Calendar to keep time-off in sync with visibility in shared calendar.     | Overview and purpose (Sticky Note: Template Description)         |
| Set your Everhour API key securely in n8n Header Auth credentials with header `X-Api-Key`.                                     | Everhour - Fetch Time Off node and Sticky Note: Fetch Everhour   |
| Use Google OAuth credentials for all Google Calendar nodes to ensure proper access and permissions.                            | Google Calendar nodes configuration                              |
| External keys follow the format `everhour:<assignmentId>:<YYYY-MM-DD>` used for event identification and de-duplication.       | Sticky Note: Build Keys, code comments                            |
| The workflow creates all-day events even for half-days but marks original period for reference.                                | Prepare Time-Off Items node explanation                           |
| Cleanup nodes ensure calendar events that no longer correspond to approved time off are deleted to avoid stale data visibility. | Cleanup block and Sticky Note: Cleanup                            |
| Google Calendar API maxResults is set to 2500 (max allowed) to fetch large event sets; pagination is not implemented.          | Potential enhancement for large calendars                         |
| Adjust schedule cadence and filters to customize for team-specific needs (e.g., filtering by users or time-off types).          | Sticky Note: Template Description, Customize section             |

---

**Disclaimer:** The text provided derives exclusively from an automated n8n workflow integration. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.

---
Sync Zendesk tickets to Pipedrive contact owners

https://n8nworkflows.xyz/workflows/sync-zendesk-tickets-to-pipedrive-contact-owners-1807


# Sync Zendesk tickets to Pipedrive contact owners

### 1. Workflow Overview

This workflow synchronizes Zendesk tickets with Pipedrive contact owners to ensure that ticket information and comments in Zendesk are reflected as notes in Pipedrive. It is designed for daily execution and processes only tickets updated since the last run, focusing on tickets coming through the email channel.

**Target Use Cases:**  
- Customer support teams managing tickets in Zendesk but tracking contacts in Pipedrive.  
- Automating the synchronization of ticket comments as notes under Pipedrive contacts.  
- Keeping contact owners in Pipedrive updated with the latest information from Zendesk tickets.

**Logical Blocks:**

- **1.1 Workflow Trigger and Last Execution Timestamp Retrieval**  
  Triggers the workflow daily and retrieves the timestamp of the last execution to fetch only new or updated tickets.

- **1.2 Zendesk Ticket Retrieval and Filtering**  
  Fetches tickets updated after the last execution and filters tickets where the channel is email.

- **1.3 Email Extraction and Duplicate Removal**  
  Extracts the email address from tickets and removes duplicates to optimize downstream searches.

- **1.4 Pipedrive Person Search and Data Mapping**  
  Searches Pipedrive persons by email, renames and filters relevant fields, and merges person IDs with Zendesk tickets.

- **1.5 Zendesk Comments Retrieval and Merging**  
  Fetches comments for the tickets, merges them with ticket data for processing.

- **1.6 Comment Processing and Note Creation in Pipedrive**  
  Splits comments into individual items, filters new comments, and adds them as notes in Pipedrive under the appropriate person.

- **1.7 Workflow Completion and Timestamp Update**  
  Checks if all comments are processed and updates the last execution timestamp accordingly.

---

### 2. Block-by-Block Analysis

#### 2.1 Workflow Trigger and Last Execution Timestamp Retrieval

- **Overview:**  
  This block triggers the workflow daily at 09:00 and retrieves the last execution timestamp from static data to filter only tickets updated after this time. It also prepares the current execution timestamp.

- **Nodes Involved:**  
  - Every day at 09:00 (Cron)  
  - Get last execution timestamp (FunctionItem)

- **Node Details:**

  - **Every day at 09:00**  
    - Type: Cron Trigger  
    - Configuration: Scheduled to trigger daily at 09:00 (hour set to 9).  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Get last execution timestamp"  
    - Edge Cases: If the system timezone is misconfigured, the trigger time may not be accurate.

  - **Get last execution timestamp**  
    - Type: FunctionItem  
    - Role: Reads static workflow data to get the last run timestamp; if none exists, initializes it with current time. Also sets current execution timestamp.  
    - Key Expressions: Uses `getWorkflowStaticData('global')` to store and retrieve timestamps.  
    - Inputs: From "Every day at 09:00"  
    - Outputs: Provides `lastExecution` and `executionTimeStamp` for downstream filtering.  
    - Edge Cases: If static data is corrupted or inaccessible, timestamp retrieval may fail.

---

#### 2.2 Zendesk Ticket Retrieval and Filtering

- **Overview:**  
  Fetches all Zendesk tickets updated after the last recorded execution timestamp and filters to process only those where the ticket channel is email.

- **Nodes Involved:**  
  - Get tickets updated after last execution (Zendesk)  
  - Channel is email (If)

- **Node Details:**

  - **Get tickets updated after last execution**  
    - Type: Zendesk node (getAll operation)  
    - Configuration: Query for tickets updated after `lastExecution` timestamp, sorted descending by update time.  
    - Credentials: Zendesk API credentials required.  
    - Inputs: From "Get last execution timestamp"  
    - Outputs: List of tickets matching the query.  
    - Edge Cases: API rate limits or authentication errors may cause failure.

  - **Channel is email**  
    - Type: If  
    - Configuration: Checks if ticket's `via.channel` field equals `"email"`.  
    - Inputs: From "Get tickets updated after last execution"  
    - Outputs: True branch continues workflow; false branch leads to a NoOp node (skips non-email tickets).  
    - Edge Cases: Tickets without `via.channel` field or other channel types will be ignored.

---

#### 2.3 Email Extraction and Duplicate Removal

- **Overview:**  
  Extracts the sender email address from Zendesk tickets, sets it as a search parameter, and removes duplicate emails to optimize Pipedrive searches.

- **Nodes Involved:**  
  - Set search email (Set)  
  - Remove duplicates to make search efficient (Item Lists)

- **Node Details:**

  - **Set search email**  
    - Type: Set  
    - Configuration: Creates a new field `SearchEmail` from `via.source.from.address` in the ticket JSON.  
    - Inputs: From "Channel is email" (true branch)  
    - Outputs: Passes tickets with `SearchEmail` field set.  
    - Edge Cases: Tickets without a valid `from.address` may lead to empty search fields.

  - **Remove duplicates to make search efficient**  
    - Type: Item Lists (removeDuplicates operation)  
    - Configuration: Removes duplicate items based on the `SearchEmail` field.  
    - Inputs: From "Set search email"  
    - Outputs: Unique list of emails for searching in Pipedrive.  
    - Edge Cases: If `SearchEmail` is missing or malformed, duplicates might not be correctly identified.

---

#### 2.4 Pipedrive Person Search and Data Mapping

- **Overview:**  
  Searches for Pipedrive persons by extracted emails, then renames and filters relevant fields to prepare for merging with Zendesk tickets.

- **Nodes Involved:**  
  - Search persons by email (Pipedrive)  
  - Rename fields and keep only needed fields (Set)  
  - Add Pipedrive person Id to Zendesk tickets (Merge)  
  - Pipedrive person Id found (If)

- **Node Details:**

  - **Search persons by email**  
    - Type: Pipedrive (search operation on person resource)  
    - Configuration: Searches by `SearchEmail` with additional fields limited to email.  
    - Credentials: Pipedrive API credentials required.  
    - Inputs: From "Remove duplicates to make search efficient"  
    - Outputs: Returns matching persons with available fields.  
    - Edge Cases: Search might return no matches or multiple matches; API errors possible.

  - **Rename fields and keep only needed fields**  
    - Type: Set  
    - Configuration: Keeps only `id` (renamed to `PipeDrivePersonId`) and `primary_email`. Drops other fields.  
    - Inputs: From "Search persons by email"  
    - Outputs: Cleaned and simplified person data for merging.  
    - Edge Cases: Missing fields may cause issues downstream.

  - **Add Pipedrive person Id to Zendesk tickets**  
    - Type: Merge (mergeByKey)  
    - Configuration: Merges ticket data with Pipedrive data by matching `via.source.from.address` (ticket) and `primary_email` (person).  
    - Inputs: From "Set search email" (tickets) and "Rename fields and keep only needed fields" (persons)  
    - Outputs: Enriched ticket data including `PipeDrivePersonId`.  
    - Edge Cases: Mismatches lead to missing IDs; merge failure if keys are missing.

  - **Pipedrive person Id found**  
    - Type: If  
    - Configuration: Checks if `PipeDrivePersonId` is not empty, indicating a successful match.  
    - Inputs: From "Add Pipedrive person Id to Zendesk tickets"  
    - Outputs: True branch continues to comment retrieval; false branch leads to NoOp1 (skips unmatched tickets).  
    - Edge Cases: Tickets without matching Pipedrive contacts are skipped.

---

#### 2.5 Zendesk Comments Retrieval and Merging

- **Overview:**  
  Retrieves comments from Zendesk for each ticket with a matched Pipedrive contact and merges comments with the ticket data.

- **Nodes Involved:**  
  - Get Zendesk comments for tickets (HTTP Request)  
  - Add comments to tickets (Merge)  
  - NoOp1 (No Operation)

- **Node Details:**

  - **Get Zendesk comments for tickets**  
    - Type: HTTP Request  
    - Configuration: GET request to Zendesk API endpoint `/api/v2/tickets/{ticket_id}/comments`. Uses predefined Zendesk credentials.  
    - Inputs: From "Pipedrive person Id found" (true branch)  
    - Outputs: Ticket comments JSON for merging.  
    - Edge Cases: API rate limits, authentication errors, or unavailable comments lead to failure.

  - **Add comments to tickets**  
    - Type: Merge (mergeByIndex)  
    - Configuration: Merges comments array with corresponding ticket data by index.  
    - Inputs: From "Get Zendesk comments for tickets" (primary) and "Pipedrive person Id found" (secondary)  
    - Outputs: Combined ticket and comments data for processing.  
    - Edge Cases: Index misalignment can cause incorrect merges.

  - **NoOp1**  
    - Type: No Operation  
    - Purpose: Placeholder for tickets without matched Pipedrive contacts (false branch of If).  
    - Inputs: From "Pipedrive person Id found" (false branch)  
    - Outputs: None (workflow effectively skips these tickets).

---

#### 2.6 Comment Processing and Note Creation in Pipedrive

- **Overview:**  
  Processes comments per ticket by splitting into batches and individual items, filters new comments, and adds them as notes in Pipedrive.

- **Nodes Involved:**  
  - Process commenst per ticket (SplitInBatches)  
  - Split comments to separate items (Item Lists)  
  - New comment (If)  
  - Add comment as a note in Pipedrive (Pipedrive)  
  - NoOp2 (No Operation)  
  - Done processing (If)

- **Node Details:**

  - **Process commenst per ticket**  
    - Type: SplitInBatches  
    - Configuration: Processes tickets one at a time (`batchSize:1`) to handle comments sequentially.  
    - Inputs: From "Add comments to tickets"  
    - Outputs: Single ticket data per batch.  
    - Edge Cases: Large ticket sets may slow processing; batch size can be adjusted.

  - **Split comments to separate items**  
    - Type: Item Lists (split operation)  
    - Configuration: Splits the `comments` array into individual comment items.  
    - Inputs: From "Process commenst per ticket"  
    - Outputs: Individual comment items for filtering.  
    - Edge Cases: Empty comments array results in no items.

  - **New comment**  
    - Type: If  
    - Configuration: Compares each comment’s `created_at` timestamp to the last execution timestamp; filters only comments created after last run.  
    - Inputs: From "Split comments to separate items"  
    - Outputs: True branch processes new comments; false branch leads to NoOp2.  
    - Edge Cases: Timezone differences or malformed timestamps may cause missed or duplicated comments.

  - **Add comment as a note in Pipedrive**  
    - Type: Pipedrive (create note resource)  
    - Configuration: Creates a note with content formatted to include the Zendesk sender's name (or fallback), a separator, and the comment body. Associates the note with the Pipedrive person ID.  
    - Inputs: From "New comment" (true branch)  
    - Credentials: Pipedrive API credentials required.  
    - Outputs: Confirmation of note creation.  
    - Edge Cases: API failures, invalid person IDs, or missing comment body cause errors.

  - **NoOp2**  
    - Type: No Operation  
    - Purpose: Skips processing for comments not new enough (false branch of "New comment").  
    - Inputs: From "New comment" (false branch)  
    - Outputs: None.

  - **Done processing**  
    - Type: If  
    - Configuration: Checks if the `Process commenst per ticket` node has no items left to process (`context.noItemsLeft` = true).  
    - Inputs: From "Add comment as a note in Pipedrive"  
    - Outputs: True branch leads to setting new last execution timestamp; false branch loops back to continue processing.  
    - Edge Cases: If the flag isn't set correctly, workflow may hang or exit prematurely.

---

#### 2.7 Workflow Completion and Timestamp Update

- **Overview:**  
  Updates the static data with the current execution timestamp to ensure future runs only process new updates.

- **Nodes Involved:**  
  - Set new last execution timestamp (FunctionItem)

- **Node Details:**

  - **Set new last execution timestamp**  
    - Type: FunctionItem  
    - Configuration: Updates the global static data `lastExecution` with the current execution timestamp captured at the start of the workflow.  
    - Inputs: From "Done processing" (true branch)  
    - Outputs: Ends the workflow run.  
    - Edge Cases: Failure to update static data will cause future runs to reprocess old tickets.

---

### 3. Summary Table

| Node Name                         | Node Type            | Functional Role                                    | Input Node(s)                      | Output Node(s)                       | Sticky Note                                                                                     |
|----------------------------------|----------------------|--------------------------------------------------|----------------------------------|------------------------------------|------------------------------------------------------------------------------------------------|
| Every day at 09:00               | Cron                 | Workflow trigger daily at 09:00                   | None                             | Get last execution timestamp        |                                                                                                |
| Get last execution timestamp     | FunctionItem          | Retrieves last execution timestamp and sets current timestamp | Every day at 09:00              | Get tickets updated after last execution |                                                                                                |
| Get tickets updated after last execution | Zendesk              | Fetches tickets updated after last execution timestamp | Get last execution timestamp    | Channel is email                   |                                                                                                |
| Channel is email                 | If                   | Filters tickets where channel is email            | Get tickets updated after last execution | Set search email, NoOp           |                                                                                                |
| Set search email                | Set                  | Extracts email address for Pipedrive search       | Channel is email (true)          | Remove duplicates to make search efficient |                                                                                                |
| Remove duplicates to make search efficient | Item Lists           | Removes duplicate emails to optimize Pipedrive search | Set search email                | Search persons by email            |                                                                                                |
| Search persons by email          | Pipedrive             | Searches Pipedrive contacts by email              | Remove duplicates to make search efficient | Rename fields and keep only needed fields |                                                                                                |
| Rename fields and keep only needed fields | Set                  | Keeps only email and person id fields             | Search persons by email          | Add Pipedrive person Id to Zendesk tickets |                                                                                                |
| Add Pipedrive person Id to Zendesk tickets | Merge                 | Merges Pipedrive person IDs into Zendesk tickets  | Set search email, Rename fields  | Pipedrive person Id found          |                                                                                                |
| Pipedrive person Id found        | If                   | Checks if Pipedrive person id exists               | Add Pipedrive person Id to Zendesk tickets | Get Zendesk comments for tickets, NoOp1 |                                                                                                |
| Get Zendesk comments for tickets | HTTP Request          | Retrieves comments for each Zendesk ticket         | Pipedrive person Id found (true) | Add comments to tickets            |                                                                                                |
| Add comments to tickets          | Merge                 | Merges comments into ticket data                    | Get Zendesk comments for tickets, Pipedrive person Id found | Process commenst per ticket |                                                                                                |
| Process commenst per ticket      | SplitInBatches        | Processes tickets one by one to handle comments    | Add comments to tickets          | Split comments to separate items   |                                                                                                |
| Split comments to separate items | Item Lists           | Splits comment arrays into individual comment items | Process commenst per ticket      | New comment                      |                                                                                                |
| New comment                     | If                   | Filters comments created after last execution      | Split comments to separate items | Add comment as a note in Pipedrive, NoOp2 |                                                                                                |
| Add comment as a note in Pipedrive | Pipedrive             | Adds Zendesk comment as a note to Pipedrive person | New comment (true)               | Done processing                   |                                                                                                |
| NoOp                            | No Operation          | Skips processing for non-email channel tickets    | Channel is email (false)         | None                             |                                                                                                |
| NoOp1                           | No Operation          | Skips processing for tickets without Pipedrive person | Pipedrive person Id found (false) | None                             |                                                                                                |
| NoOp2                           | No Operation          | Skips comments not new enough to be added          | New comment (false)              | None                             |                                                                                                |
| Done processing                 | If                   | Checks if all comments processed                     | Add comment as a note in Pipedrive | Set new last execution timestamp, Process commenst per ticket (loop) |                                                                                                |
| Set new last execution timestamp | FunctionItem          | Updates last execution timestamp in static data    | Done processing (true)           | None                             |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node:**  
   - Type: Cron  
   - Name: "Every day at 09:00"  
   - Set to trigger daily at hour 9 (09:00).  
   - No credentials needed.

2. **Add FunctionItem Node to Retrieve Last Execution Timestamp:**  
   - Name: "Get last execution timestamp"  
   - Use `getWorkflowStaticData('global')` to get `lastExecution`.  
   - If not set, initialize with current ISO timestamp.  
   - Add current execution timestamp as `executionTimeStamp`.  
   - Connect output from Cron node.

3. **Add Zendesk Node to Fetch Tickets Updated Since Last Execution:**  
   - Name: "Get tickets updated after last execution"  
   - Operation: getAll tickets.  
   - Query parameter: `updated>{{ $json["lastExecution"] }}`  
   - Sort by `updated_at` descending.  
   - Use Zendesk credentials.  
   - Connect input from "Get last execution timestamp".

4. **Add If Node to Filter Tickets by Email Channel:**  
   - Name: "Channel is email"  
   - Condition: `$json["via"].channel == "email"`  
   - Connect input from Zendesk node.  
   - True branch continues; false branch connects to NoOp node.

5. **Add Set Node to Extract Email for Search:**  
   - Name: "Set search email"  
   - Create field `SearchEmail` from `$json["via"].source.from.address`.  
   - Connect true branch of "Channel is email".

6. **Add Item Lists Node to Remove Duplicate Emails:**  
   - Name: "Remove duplicates to make search efficient"  
   - Operation: removeDuplicates  
   - Compare on field: `SearchEmail`  
   - Connect from "Set search email".

7. **Add Pipedrive Node to Search Persons by Email:**  
   - Name: "Search persons by email"  
   - Resource: Person, Operation: Search  
   - Term: `{{$json["SearchEmail"]}}`  
   - Additional Fields: `fields` set to `email`  
   - Use Pipedrive credentials.  
   - Connect from "Remove duplicates to make search efficient".

8. **Add Set Node to Rename and Keep Only Needed Fields:**  
   - Name: "Rename fields and keep only needed fields"  
   - Keep only `id` as `PipeDrivePersonId` and `primary_email`.  
   - Connect from "Search persons by email".

9. **Add Merge Node to Add Pipedrive Person ID to Zendesk Tickets:**  
   - Name: "Add Pipedrive person Id to Zendesk tickets"  
   - Mode: mergeByKey  
   - PropertyName1: `via.source.from.address` (tickets)  
   - PropertyName2: `primary_email` (persons)  
   - Connect inputs: first from "Set search email" (tickets), second from "Rename fields and keep only needed fields" (persons).

10. **Add If Node to Check if Pipedrive Person ID Found:**  
    - Name: "Pipedrive person Id found"  
    - Condition: `$json["PipeDrivePersonId"]` is not empty  
    - Connect from "Add Pipedrive person Id to Zendesk tickets".  
    - True branch continues, false branch to NoOp1 node.

11. **Add HTTP Request Node to Get Zendesk Comments for Tickets:**  
    - Name: "Get Zendesk comments for tickets"  
    - Method: GET  
    - URL: `https://<your_zendesk_subdomain>.zendesk.com/api/v2/tickets/{{$json["id"]}}/comments`  
    - Authentication: Use Zendesk API credentials.  
    - Connect from "Pipedrive person Id found" (true branch).

12. **Add Merge Node to Add Comments to Tickets:**  
    - Name: "Add comments to tickets"  
    - Mode: mergeByIndex (inner join)  
    - Connect primary input from "Get Zendesk comments for tickets".  
    - Connect secondary input from "Pipedrive person Id found" (true branch).

13. **Add SplitInBatches Node to Process Tickets One by One:**  
    - Name: "Process commenst per ticket"  
    - Batch size: 1  
    - Connect from "Add comments to tickets".

14. **Add Item Lists Node to Split Comments into Separate Items:**  
    - Name: "Split comments to separate items"  
    - Field to split out: `comments`  
    - Connect from "Process commenst per ticket".

15. **Add If Node to Filter New Comments:**  
    - Name: "New comment"  
    - Condition: `$json["created_at"] > $item(0).$node["Get last execution timestamp"].json["lastExecution"]`  
    - Connect from "Split comments to separate items".  
    - True branch continues to add notes, false branch to NoOp2.

16. **Add Pipedrive Node to Add Comment as Note:**  
    - Name: "Add comment as a note in Pipedrive"  
    - Resource: Note  
    - Content:  
      ```
      Message imported from Zendesk
      ------------------------------------------------
      From {{$json["via"]["source"]["from"]["name"] ?? 'Zendesk user'}}
      ------------------------------------------------
      {{$json["body"]}}
      ```  
    - Additional Fields: `person_id` set to `{{$item(0).$node["Process commenst per ticket"].json["PipeDrivePersonId"]}}`  
    - Use Pipedrive credentials.  
    - Connect from "New comment" (true branch).

17. **Add NoOp Nodes:**  
    - Name: "NoOp" (for false branch of "Channel is email")  
    - Name: "NoOp1" (for false branch of "Pipedrive person Id found")  
    - Name: "NoOp2" (for false branch of "New comment").

18. **Add If Node to Check if Processing is Done:**  
    - Name: "Done processing"  
    - Condition: Boolean check on `{{$node["Process commenst per ticket"].context["noItemsLeft"]}} == true`  
    - Connect from "Add comment as a note in Pipedrive".  
    - True branch to "Set new last execution timestamp", false branch loops back to "Process commenst per ticket".

19. **Add FunctionItem Node to Update Last Execution Timestamp:**  
    - Name: "Set new last execution timestamp"  
    - Code:  
      ```js
      const staticData = getWorkflowStaticData('global');
      staticData.lastExecution = $item(0).$node["Get last execution timestamp"].executionTimeStamp;
      return item;
      ```  
    - Set to execute once.  
    - Connect from "Done processing" (true branch).

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Pipedrive and Zendesk accounts must be created or owned by the same person/email for credential compatibility. | Prerequisites section of workflow description                                                           |
| Pipedrive credentials documentation: https://docs.n8n.io/integrations/builtin/credentials/pipedrive/       | Credential setup in Pipedrive nodes                                                                     |
| Zendesk credentials documentation: https://docs.n8n.io/integrations/builtin/credentials/zendesk/           | Credential setup in Zendesk nodes                                                                        |
| The workflow respects API rate limits and filters tickets/comments by last execution timestamp to optimize calls. | Best practice and efficiency notes                                                                       |
| The "NoOp" nodes serve as safe termination points for branches that should not continue processing.         | Workflow control flow explanation                                                                         |

---

This structured documentation provides a complete understanding of the workflow logic, node configurations, and reproduction steps for both advanced users and automation tools.
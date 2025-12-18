Discord Server Anti-Impersonation / Scammer Tracker with Data Tables

https://n8nworkflows.xyz/workflows/discord-server-anti-impersonation---scammer-tracker-with-data-tables-11812


# Discord Server Anti-Impersonation / Scammer Tracker with Data Tables

### 1. Workflow Overview

This n8n workflow titled **"Discord Server Anti-Impersonation / Scammer Tracker with Data Tables"** is designed to monitor Discord server members for impersonation or scammer activity by tracking username and nickname changes over time. It leverages Discord's API to fetch server member data, compares current data with stored historical records in data tables, and updates these records accordingly. This enables early detection of suspicious account changes that may indicate impersonation or scam-related risks.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger and Initial Data Collection**  
  An hourly schedule triggers the workflow, which then fetches all current server members and prepares the configuration.

- **1.2 Data Preprocessing and User Existence Check**  
  Processes fetched members in batches, checks if users exist in the main database, and either adds new users or fetches their records for comparison.

- **1.3 Comparison and Decision Logic**  
  Compares current user data (username, nickname) with stored data, branching into multiple cases such as no changes, username-only change, nickname-only change, or both changed.

- **1.4 Database Update and Recording Changes**  
  Depending on the comparison outcome, updates data tables with new user data or logs changes, ensuring no duplicate change records.

- **1.5 Real-Time Nickname Change Event Handling**  
  Listens for live Discord nickname change events and records them similarly to scheduled checks.

- **1.6 Error Handling and No-Operation Paths**  
  Gracefully handles errors and no-op conditions to maintain workflow stability.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Initial Data Collection

- **Overview:**  
  This block initiates the workflow hourly, sets configuration parameters, and fetches all server members from Discord.

- **Nodes Involved:**  
  - Hourly Check Trigger  
  - Configuration Settings  
  - Get All Server Members  
  - Edit Fields

- **Node Details:**

  - **Hourly Check Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts workflow every hour to perform checks  
    - Configuration: Default hourly schedule, no additional parameters  
    - Inputs: None  
    - Outputs: Triggers "Configuration Settings"  
    - Edge Cases: Missed triggers due to downtime or network issues  

  - **Configuration Settings**  
    - Type: Set  
    - Role: Prepares or resets configuration parameters for the workflow run  
    - Configuration: Sets any static variables or configurations needed downstream (details not specified)  
    - Inputs: From Trigger  
    - Outputs: To "Get All Server Members"  
    - Edge Cases: Missing config parameters could break later steps  

  - **Get All Server Members**  
    - Type: Discord Node  
    - Role: Retrieves the full list of members of the Discord server  
    - Configuration: Uses Discord API credentials and "Get Members" operation  
    - Inputs: From Config  
    - Outputs: User member list to "Edit Fields"  
    - Edge Cases: Discord API rate limits, authentication failures  

  - **Edit Fields**  
    - Type: Set  
    - Role: Adjusts or normalizes the data fields retrieved from Discord before processing  
    - Configuration: May set or rename fields to standardize data structure  
    - Inputs: List of members  
    - Outputs: To "Loop Over Items"  
    - Edge Cases: Incorrect field mapping could cause errors downstream  

---

#### 2.2 Data Preprocessing and User Existence Check

- **Overview:**  
  Processes members in batches to avoid overload, checks if each user exists in the main database, and routes accordingly.

- **Nodes Involved:**  
  - Loop Over Items  
  - If userID does not exist (DataTable node)  
  - If userid does not exist (If node)  
  - Add new user to database (DataTable)  
  - Get row(s) from main DB (DataTable)

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes members one batch at a time (batch size not specified)  
    - Inputs: From "Edit Fields"  
    - Outputs: To "If userID does not exist"  
    - Edge Cases: Batch size too large may cause timeouts; too small increases total runtime  

  - **If userID does not exist (DataTable)**  
    - Type: DataTable (Lookup)  
    - Role: Queries main database to check if a userID is already stored  
    - Configuration: Query by userID field  
    - Inputs: Individual user item  
    - Outputs: Boolean result to If node  
    - Edge Cases: DB connectivity issues, empty results  

  - **If userid does not exist (If)**  
    - Type: If  
    - Role: Branches logic based on whether user is new or existing  
    - Inputs: From DataTable lookup  
    - Outputs: To either "Add new user to database" or "Get row(s) from main DB"  
    - Edge Cases: Expression evaluation errors  

  - **Add new user to database**  
    - Type: DataTable (Insert)  
    - Role: Inserts new user records into the main data table  
    - Inputs: From If node (true branch)  
    - Outputs: Loops back to "Loop Over Items"  
    - Edge Cases: Insert failures, duplicate keys  

  - **Get row(s) from main DB**  
    - Type: DataTable (Select)  
    - Role: Retrieves existing user data for comparison  
    - Inputs: From If node (false branch)  
    - Outputs: To "Switch" node for detailed comparison  
    - Edge Cases: Missing data, DB read errors  

---

#### 2.3 Comparison and Decision Logic

- **Overview:**  
  Compares current Discord data with database records to detect changes in username and nickname, directing flow through complex conditional branches.

- **Nodes Involved:**  
  - Switch  
  - Get row(s) from new DB (DataTable)  
  - Switch1  
  - Is the new change recorded already (If)  
  - If username and nickname hasn't been changed (If)  
  - If both username and nickname was changed but user ID is the same (If)  
  - If username is the same but nickname is diff (If)  
  - If username is the same but nickname is diff1 (If)

- **Node Details:**

  - **Switch**  
    - Type: Switch  
    - Role: Routes flow based on data comparison results (e.g., user exists, change type)  
    - Inputs: From "Get row(s) from main DB"  
    - Outputs: To "Get row(s) from new DB" or "Loop Over Items"  
    - Edge Cases: Incorrect condition setup may misroute data  

  - **Get row(s) from new DB**  
    - Type: DataTable (Select)  
    - Role: Fetches the latest change records for the user to check if new changes are already logged  
    - Inputs: From Switch  
    - Outputs: To Switch1  
    - Edge Cases: Empty results or DB errors  

  - **Switch1**  
    - Type: Switch  
    - Role: Determines if the user has a nickname change event or other change types  
    - Inputs: From "Get row(s) from new DB"  
    - Outputs: Either "nickname changed" or "is the new change recorded already"  
    - Edge Cases: Ambiguous conditions  

  - **Is the new change recorded already**  
    - Type: If  
    - Role: Checks if the current username/nickname changes are already recorded to avoid duplication  
    - Inputs: From Switch1  
    - Outputs: Either "new name and un is recorded, do nothing" or triggers further nickname change processing  
    - Edge Cases: False negatives or positives in duplication logic  

  - **If username and nickname hasn't been changed**  
    - Type: If  
    - Role: Detects if no change occurred; branches to no-op or further checks  
    - Inputs: From "aggregate all necessary data"  
    - Outputs: No operation or further analysis  
    - Edge Cases: Misidentification of changes  

  - **If both username and nickname was changed but user ID is the same**  
    - Type: If  
    - Role: Handles the scenario where both username and nickname changed for the same user ID  
    - Inputs: From "If username and nickname hasn't been changed" false branch  
    - Outputs: Updates both fields in records or further condition checks  
    - Edge Cases: Race conditions if multiple changes occur simultaneously  

  - **If username is the same but nickname is diff**  
    - Type: If  
    - Role: Determines if only nickname changed, leading to selective update actions  
    - Inputs: From previous If node  
    - Outputs: Either update only username or nickname or handle errors  
    - Edge Cases: Partial updates failing  

  - **If username is the same but nickname is diff1**  
    - Type: If  
    - Role: Secondary check in nickname difference path to route to update or error handling  
    - Inputs: From previous If  
    - Outputs: Update nickname or error handler  
    - Edge Cases: Processing failures  

---

#### 2.4 Database Update and Recording Changes

- **Overview:**  
  Updates records in the data tables based on detected changes, inserts new change logs, or skips operations if no changes detected.

- **Nodes Involved:**  
  - Add new user to database (DataTable)  
  - Insert user changes to new database (DataTable)  
  - Update both column in records (DataTable)  
  - Update only nickname column (DataTable)  
  - Update only username on record (DataTable)  
  - New name and un is recorded, do nothing (NoOp)  
  - No Operation, do nothing1 (NoOp)

- **Node Details:**

  - **Add new user to database**  
    - Role: Inserts new user entries as they are discovered (see above)  

  - **Insert user changes to new database**  
    - Type: DataTable (Insert)  
    - Role: Adds new change records (username/nickname changes) for users  
    - Inputs: From "nickname changed" node or change detection logic  
    - Outputs: Loops back to batch processing  
    - Edge Cases: Duplicate entries or insert failures  

  - **Update both column in records**  
    - Type: DataTable (Update)  
    - Role: Updates both username and nickname fields in the main database for a user  
    - Inputs: From "If both username and nickname was changed but user ID is the same"  
    - Outputs: To no-op node  
    - Edge Cases: Update conflicts or DB errors  

  - **Update only nickname column**  
    - Type: DataTable (Update)  
    - Role: Updates just the nickname column when only nickname changes detected  
    - Inputs: From relevant If nodes  
    - Outputs: To no-op or error handler  
    - Edge Cases: Partial update failures  

  - **Update only username on record**  
    - Type: DataTable (Update)  
    - Role: Updates just the username column when only username changes detected  
    - Inputs: From If nodes  
    - Outputs: To no-op  
    - Edge Cases: Update failures  

  - **New name and un is recorded, do nothing**  
    - Type: No Operation  
    - Role: Placeholder node to skip further processing if changes already recorded  
    - Inputs: From "is the new change recorded already" (true branch)  
    - Outputs: Loops to batch processing  

  - **No Operation, do nothing1**  
    - Type: No Operation  
    - Role: Catch-all for cases where no action is needed  
    - Inputs: From multiple conditional branches  
    - Outputs: Loops to batch processing  

---

#### 2.5 Real-Time Nickname Change Event Handling

- **Overview:**  
  Listens to real-time Discord webhook events for nickname changes and records these changes in the new database.

- **Nodes Involved:**  
  - Nickname changed (Discord node)  
  - Insert user changes to new database (DataTable)  
  - Nickname changed again (Discord node)  
  - Aggregate all necessary data (Set)  
  - Analysis Section nodes (Sticky Notes for grouping)  

- **Node Details:**

  - **Nickname changed**  
    - Type: Discord webhook node  
    - Role: Listens to Discord events specifically for nickname changes  
    - Configuration: Linked to Discord webhook "webhook-get-members-001"  
    - Inputs: External event trigger  
    - Outputs: To "Insert user changes to new database"  
    - Edge Cases: Missed webhook events, auth issues  

  - **Insert user changes to new database**  
    - Role: Inserts the real-time nickname change event to the new database (see above)  

  - **Nickname changed again**  
    - Type: Discord webhook node  
    - Role: Another webhook listener for nickname change events in a different context or for reprocessing  
    - Outputs: To "Aggregate all necessary data"  
    - Edge Cases: Duplicate events  

  - **Aggregate all necessary data**  
    - Type: Set  
    - Role: Prepares data structure for further analysis and decision making  
    - Inputs: From "Nickname changed again"  
    - Outputs: To subsequent conditional checks  

  - **Analysis Section Nodes**  
    - Sticky Notes grouping logical analysis stages for clarity and documentation only  

---

#### 2.6 Error Handling and No-Operation Paths

- **Overview:**  
  Handles any errors gracefully and provides no-operation fallbacks to ensure workflow continuity.

- **Nodes Involved:**  
  - Error handling (Discord node)  
  - No Operation, do nothing1 (NoOp node)

- **Node Details:**

  - **Error handling**  
    - Type: Discord node (likely message or logging)  
    - Role: Handles errors by retrying or logging, connected to no-op on success  
    - Configuration: Retry enabled; uses Discord webhook credentials  
    - Inputs: From error branches in update flows  
    - Outputs: To "No Operation, do nothing1"  
    - Edge Cases: Persistent failures, webhook issues  

  - **No Operation, do nothing1**  
    - See above  

---

### 3. Summary Table

| Node Name                          | Node Type          | Functional Role                                  | Input Node(s)                      | Output Node(s)                        | Sticky Note                                  |
|-----------------------------------|--------------------|------------------------------------------------|----------------------------------|-------------------------------------|----------------------------------------------|
| Hourly Check Trigger              | Schedule Trigger   | Triggers workflow hourly                        | None                             | Configuration Settings              |                                              |
| Configuration Settings           | Set                | Sets config params                              | Hourly Check Trigger             | Get All Server Members              |                                              |
| Get All Server Members           | Discord            | Fetches all server members                      | Configuration Settings           | Edit Fields                       |                                              |
| Edit Fields                     | Set                | Normalizes member data                           | Get All Server Members           | Loop Over Items                   |                                              |
| Loop Over Items                 | SplitInBatches     | Processes members in batches                     | Edit Fields                     | If userID does not exist           |                                              |
| If userID does not exist         | DataTable          | Checks if user exists in main DB                 | Loop Over Items                 | If userid does not exist            |                                              |
| If userid does not exist          | If                 | Branches for new or existing user                | If userID does not exist          | Add new user to database / Get row(s) from main DB |                                              |
| Add new user to database         | DataTable          | Inserts new user record                           | If userid does not exist (true) | Loop Over Items                   |                                              |
| Get row(s) from main DB          | DataTable          | Retrieves user record for comparison             | If userid does not exist (false) | Switch                           |                                              |
| Switch                         | Switch             | Routes based on user data                         | Get row(s) from main DB          | Get row(s) from new DB / Loop Over Items |                                              |
| Get row(s) from new DB           | DataTable          | Fetches latest change logs                        | Switch                         | Switch1                         |                                              |
| Switch1                        | Switch             | Determines type of change                          | Get row(s) from new DB           | nickname changed / is the new change recorded already |                                              |
| nickname changed                | Discord            | Listens to nickname change event                  | Switch1                        | Insert user changes to new database |                                              |
| Insert user changes to new database | DataTable          | Inserts change records                            | nickname changed / nickname changed again | Loop Over Items                   |                                              |
| is the new change recorded already | If                 | Checks for duplicate change records               | Switch1                        | new name and un is recorded, do nothing / nickname changed again |                                              |
| new name and un is recorded, do nothing | NoOp               | Skips processing if duplicate                       | is the new change recorded already | Loop Over Items                   |                                              |
| nickname changed again          | Discord            | Webhook for nickname changes                       | is the new change recorded already (false) | Aggregate all necessary data      |                                              |
| Aggregate all necessary data     | Set                | Prepares data for analysis                         | nickname changed again          | If username and nickname hasn't been changed |                                              |
| If username and nickname hasn't been changed | If                 | Detects no changes                                | Aggregate all necessary data     | No Operation, do nothing1 / If both username and nickname was changed but user ID is the same |                                              |
| No Operation, do nothing1        | NoOp               | No operation placeholder                          | Multiple If nodes               | Loop Over Items                   |                                              |
| If both username and nickname was changed but user ID is the same | If                 | Handles both fields changed                       | If username and nickname hasn't been changed (false) | Update both column in records / If username is the same but nickname is diff |                                              |
| Update both column in records    | DataTable          | Updates username and nickname                      | If both username and nickname was changed but user ID is the same | No Operation, do nothing1         |                                              |
| If username is the same but nickname is diff | If                 | Checks nickname-only changes                      | Previous If                    | Update only username on record / if username is the same but nickname is diff1 |                                              |
| Update only username on record   | DataTable          | Updates username only                              | If username is the same but nickname is diff | No Operation, do nothing1         |                                              |
| if username is the same but nickname is diff1 | If                 | Further nickname change handling                  | If username is the same but nickname is diff | Update only nickname column / error handling |                                              |
| Update only nickname column      | DataTable          | Updates nickname only                              | if username is the same but nickname is diff1 | No Operation, do nothing1         |                                              |
| error handling                  | Discord            | Handles errors and retries                         | if username is the same but nickname is diff1 (error path) | No Operation, do nothing1         |                                              |
| Get All Server Members           | Discord            | Fetches members                                   | Configuration Settings          | Edit Fields                       |                                              |
| Workflow Overview              | Sticky Note        | Documentation                                     | None                           | None                            |                                              |
| Collection Section              | Sticky Note        | Documentation                                     | None                           | None                            |                                              |
| Comparison Logic Section        | Sticky Note        | Documentation                                     | None                           | None                            |                                              |
| Analysis Section (1-5)          | Sticky Note        | Documentation sections grouping analysis          | None                           | None                            |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger** node named **"Hourly Check Trigger"**:  
   - Set to trigger every hour (default parameters).

2. **Add a Set node named "Configuration Settings"**:  
   - Configure any static settings or parameters required for the workflow (e.g., Discord server ID, API keys if needed).

3. **Add a Discord node named "Get All Server Members"**:  
   - Operation: "Get Members" of the target Discord server.  
   - Credentials: Provide Discord OAuth2 or bot token with permissions to read server members.  
   - Connect "Configuration Settings" output to this node.

4. **Add a Set node named "Edit Fields"**:  
   - Configure to normalize or prepare member data fields (e.g., extract userID, username, nickname).  
   - Connect "Get All Server Members" output to this node.

5. **Add a SplitInBatches node named "Loop Over Items"**:  
   - Set batch size as appropriate (e.g., 1 or 10) to process members without timeouts.  
   - Connect "Edit Fields" output to this node.

6. **Add a DataTable node named "If userID does not exist"**:  
   - Operation: Query (read) main database table filtering by userID.  
   - Connect "Loop Over Items" output to this node.

7. **Add an If node named "If userid does not exist"**:  
   - Condition: Check if query result is empty (user not found).  
   - Connect "If userID does not exist" output to this node.

8. **Add a DataTable node named "Add new user to database"**:  
   - Operation: Insert new user data into the main database.  
   - Connect "If userid does not exist" true branch to this node.  
   - Connect this node's output back to "Loop Over Items" for next batch.

9. **Add a DataTable node named "Get row(s) from main DB"**:  
   - Operation: Retrieve existing user record for comparison.  
   - Connect "If userid does not exist" false branch here.

10. **Add a Switch node named "Switch"**:  
    - Configure to route based on user record presence or status.  
    - Connect "Get row(s) from main DB" output here.

11. **Add a DataTable node named "Get row(s) from new DB"**:  
    - Operation: Fetch recent change logs for the user from a secondary database table.  
    - Connect appropriate "Switch" outputs here.

12. **Add a Switch node named "Switch1"**:  
    - Route based on whether a nickname changed event or other changes were detected.  
    - Connect "Get row(s) from new DB" output here.

13. **Add a Discord webhook node named "nickname changed"**:  
    - Configure to listen for Discord nickname change events using a webhook.  
    - Connect "Switch1" output for nickname change here.

14. **Add a DataTable node named "Insert user changes to new database"**:  
    - Operation: Insert nickname or username change events into the new database.  
    - Connect "nickname changed" output here and also from webhook event nodes.

15. **Add an If node named "is the new change recorded already"**:  
    - Checks for duplicate change records to avoid redundant inserts.  
    - Connect "Switch1" output to this node.

16. **Add a NoOp node named "new name and un is recorded, do nothing"**:  
    - For skipping processing if changes already recorded.  
    - Connect "is the new change recorded already" true branch here.

17. **Add a Discord webhook node named "nickname changed again"**:  
    - Another webhook listener for nickname changes, possibly for real-time reprocessing.  
    - Connect "is the new change recorded already" false branch here.

18. **Add a Set node named "aggregate all necessary data"**:  
    - Prepare data for detailed comparison and analysis.  
    - Connect "nickname changed again" output here.

19. **Add an If node named "If username and nickname hasn't been changed"**:  
    - Checks if both username and nickname remained unchanged.  
    - Connect "aggregate all necessary data" output here.

20. **Add a NoOp node named "No Operation, do nothing1"**:  
    - For skipping processing where no changes detected.  
    - Connect relevant If node true branches here.

21. **Add an If node named "If both username and nickname was changed but user ID is the same"**:  
    - Handles cases where both fields changed.  
    - Connect false branch of previous If here.

22. **Add DataTable update nodes:**  
    - "Update both column in records"  
    - "Update only nickname column"  
    - "Update only username on record"  
    - Configure each to update appropriate fields in the main database.  
    - Connect If node branches ("If both username and nickname was changed but user ID is the same", "If username is the same but nickname is diff", etc.) accordingly.

23. **Add an If node named "If username is the same but nickname is diff"** and a follow-up If "if username is the same but nickname is diff1"**:  
    - Manage nickname-only changes and error handling branches.

24. **Add a Discord node named "error handling"**:  
    - Configure to log or notify on errors, enable retry.  
    - Connect error branches from update flows here.

25. **Connect all NoOp nodes back to "Loop Over Items"** to continue batch processing.

26. **Add Sticky Note nodes** at logical points for documentation:  
    - "Workflow Overview", "Collection Section", "Comparison Logic Section", "Analysis Section 1-5" to group and label parts of the workflow for clarity.

27. **Set up Discord credentials** with required permissions (read members, receive webhooks).  
28. **Set up DataTable credentials** for main and secondary databases, ensuring proper access rights.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow uses Discord webhook "webhook-get-members-001" to listen for member and nickname events.    | Webhook setup in Discord developer portal required. |
| DataTables nodes interface with two tables: a main user database and a secondary change log database. | Ensure separate tables with appropriate schema exist. |
| Sticky notes are used extensively to document logical sections and improve workflow readability.     | Useful for maintenance and onboarding.           |
| Error handling node has retry enabled to improve resiliency against transient API or network issues. | Important for production stability.              |
| The workflow is designed for hourly scheduled checks plus real-time webhook triggers for nickname changes. | Balances periodic full audits and live updates.  |

---

This documentation fully describes the **Discord Server Anti-Impersonation / Scammer Tracker with Data Tables** workflow, enabling users or AI agents to understand, reproduce, and modify the workflow confidently.
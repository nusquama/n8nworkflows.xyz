Verify Telegram Channel Subscriptions with Access Control using Postgres

https://n8nworkflows.xyz/workflows/verify-telegram-channel-subscriptions-with-access-control-using-postgres-3615


# Verify Telegram Channel Subscriptions with Access Control using Postgres

### 1. Workflow Overview

This workflow is designed for Telegram bot developers and marketers to automate the verification of user subscriptions to specified Telegram channels. It ensures that users have subscribed to required channels before granting them access to resources or rewards, such as downloadable files (Google Drive integration is present but currently disabled). The workflow uses a Postgres database to store channel information and bot status, and Telegram nodes to interact with users and channels.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Receives Telegram messages via webhook, initializes variables, and determines user roles (admin or regular user).
- **1.2 Bot Status and Command Routing:** Retrieves bot status from Postgres, routes user commands, and defines the flow based on user input.
- **1.3 Channel Management:** Handles adding and removing Telegram channels to the verification list, including checking channel existence and updating Postgres.
- **1.4 Subscription Verification:** Checks if a user is subscribed to required channels, aggregates subscription statuses, and sends success or failure messages.
- **1.5 Referral and Messaging:** Manages referral logic and sends appropriate Telegram messages to users and managers.
- **1.6 (Disabled) File Delivery:** Intended to deliver Google Drive files upon successful subscription verification (currently disabled).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:**  
  This block captures incoming Telegram messages via webhook, initializes workflow variables, and determines if the user is an admin to control access to management commands.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Variables TG (Set)  
  - Initialization (Set)  
  - Is Admin? (If)  
  - Define Type (Switch)

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Entry point for all Telegram messages to the bot.  
    - Configuration: Listens to all incoming Telegram updates.  
    - Inputs: Telegram webhook  
    - Outputs: Passes Telegram message data downstream.  
    - Edge cases: Webhook misconfiguration, Telegram API downtime.

  - **Variables TG**  
    - Type: Set  
    - Role: Initializes or sets variables needed later (likely extracts user info).  
    - Configuration: Sets variables from Telegram message data.  
    - Inputs: Telegram Trigger output  
    - Outputs: Variables for next nodes.

  - **Initialization**  
    - Type: Set  
    - Role: Further variable initialization or preparation for role checking.  
    - Configuration: Sets or resets variables.  
    - Inputs: Variables TG output  
    - Outputs: Prepared data for role check.

  - **Is Admin?**  
    - Type: If  
    - Role: Checks if the Telegram user is an admin (likely by comparing user ID to a list or DB).  
    - Configuration: Conditional check on user role.  
    - Inputs: Initialization output  
    - Outputs: Two branches — admin or non-admin.  
    - Edge cases: Missing or malformed user data, false negatives.

  - **Define Type**  
    - Type: Switch  
    - Role: Routes flow based on user type or command type after admin check.  
    - Configuration: Switch on user role or message type.  
    - Inputs: Is Admin? output (admin branch)  
    - Outputs: Routes to buttons or bot status retrieval.

---

#### 2.2 Bot Status and Command Routing

- **Overview:**  
  Retrieves the current bot status from Postgres and routes the user commands accordingly to handle different bot functionalities.

- **Nodes Involved:**  
  - Get Bot Status (Postgres)  
  - Define flow (Switch)  
  - Buttons (Switch)  
  - Update bot status on START (Postgres)  
  - Success (Telegram)

- **Node Details:**

  - **Get Bot Status**  
    - Type: Postgres  
    - Role: Queries the current status of the bot or user session from the database.  
    - Configuration: SQL query to fetch status.  
    - Inputs: Define Type output  
    - Outputs: Bot status data.

  - **Define flow**  
    - Type: Switch  
    - Role: Routes the workflow based on the retrieved bot status or command type.  
    - Configuration: Switch on status or command.  
    - Inputs: Get Bot Status output  
    - Outputs: Routes to commands, add channel, get channels, or empty.

  - **Buttons**  
    - Type: Switch  
    - Role: Handles button presses or UI interaction commands from Telegram users.  
    - Configuration: Switch on button payload.  
    - Inputs: Define Type output (non-admin branch)  
    - Outputs: Updates bot status on start.

  - **Update bot status on START**  
    - Type: Postgres  
    - Role: Updates the bot status in the database when a start command or button is pressed.  
    - Configuration: SQL update or insert.  
    - Inputs: Buttons output  
    - Outputs: Success message.

  - **Success**  
    - Type: Telegram  
    - Role: Sends a success message to the user after updating status.  
    - Configuration: Telegram message with confirmation text.  
    - Inputs: Update bot status on START output.

---

#### 2.3 Channel Management

- **Overview:**  
  Manages the addition and removal of Telegram channels for subscription verification, including checking channel existence and updating the Postgres DB accordingly.

- **Nodes Involved:**  
  - Add Channel (Postgres)  
  - Request New Add Channel (Telegram)  
  - Channel Exists (Telegram)  
  - Delete Channel (Postgres)  
  - Request New Delete Channel (Telegram)  
  - Request Add Channel (Telegram)  
  - Request Delete Channel (Telegram)  
  - Update bot status on CHECK SUBSCRIPTION REQUEST CHANNEL (Postgres)  
  - Update bot status on CHECK SUBSCRIPTION REQUEST CHANNEL1 (Postgres)  
  - Channel Not Exists (Telegram)

- **Node Details:**

  - **Add Channel**  
    - Type: Postgres  
    - Role: Inserts a new channel ID into the database for subscription checks.  
    - Configuration: SQL insert with error continuation on failure.  
    - Inputs: Define flow output (Add Channel branch)  
    - Outputs: Telegram message requesting new channel or checking channel existence.

  - **Request New Add Channel**  
    - Type: Telegram  
    - Role: Sends a message to the user to provide a new channel ID to add.  
    - Inputs: Add Channel output (first branch)  
    - Outputs: Next steps for channel addition.

  - **Channel Exists**  
    - Type: Telegram  
    - Role: Checks if the channel exists or confirms addition.  
    - Inputs: Add Channel output (second branch)  
    - Outputs: Updates bot status on add request.

  - **Delete Channel**  
    - Type: Postgres  
    - Role: Removes a channel ID from the database.  
    - Configuration: SQL delete with error continuation.  
    - Inputs: If node output (true branch)  
    - Outputs: Telegram message requesting new delete channel or channel not exists.

  - **Request New Delete Channel**  
    - Type: Telegram  
    - Role: Requests the user to provide a channel ID to delete.  
    - Inputs: Delete Channel output (first branch)  
    - Outputs: Next steps for deletion.

  - **Request Add Channel**  
    - Type: Telegram  
    - Role: Handles user requests to add a channel.  
    - Inputs: Commands output (Add Channel command)  
    - Outputs: Updates bot status on add request.

  - **Request Delete Channel**  
    - Type: Telegram  
    - Role: Handles user requests to delete a channel.  
    - Inputs: Commands output (Delete Channel command)  
    - Outputs: Updates bot status on delete request.

  - **Update bot status on CHECK SUBSCRIPTION REQUEST CHANNEL**  
    - Type: Postgres  
    - Role: Updates the bot status after an add channel request.  
    - Inputs: Request Add Channel output  
    - Outputs: Further processing.

  - **Update bot status on CHECK SUBSCRIPTION REQUEST CHANNEL1**  
    - Type: Postgres  
    - Role: Updates the bot status after a delete channel request.  
    - Inputs: Request Delete Channel output  
    - Outputs: Further processing.

  - **Channel Not Exists**  
    - Type: Telegram  
    - Role: Sends a message to the user indicating the channel does not exist.  
    - Inputs: Delete Channel output (false branch)  
    - Outputs: End or retry.

---

#### 2.4 Subscription Verification

- **Overview:**  
  Verifies if a user is subscribed to all required Telegram channels by fetching channel IDs from Postgres and checking subscription statuses via Telegram API. Aggregates results and sends success or failure messages.

- **Nodes Involved:**  
  - Get Channels (Postgres)  
  - Get Subscription statuses (Telegram)  
  - Union statuses (Aggregate)  
  - Check (If)  
  - Check success (Telegram)  
  - Check failed (Telegram)

- **Node Details:**

  - **Get Channels**  
    - Type: Postgres  
    - Role: Retrieves the list of Telegram channel IDs to check subscriptions against.  
    - Inputs: Get Referal output or Commands output (Get Channels command)  
    - Outputs: Channel IDs for subscription check.

  - **Get Subscription statuses**  
    - Type: Telegram  
    - Role: Calls Telegram API to check if the user is subscribed to each channel.  
    - Inputs: Get Channels output  
    - Outputs: Subscription status per channel.

  - **Union statuses**  
    - Type: Aggregate  
    - Role: Aggregates individual subscription statuses into a single summary.  
    - Inputs: Get Subscription statuses output  
    - Outputs: Aggregated subscription result.

  - **Check**  
    - Type: If  
    - Role: Determines if the user meets subscription requirements (all subscribed).  
    - Inputs: Union statuses output  
    - Outputs: Branches to success or failure messages.

  - **Check success**  
    - Type: Telegram  
    - Role: Sends a success message to the user confirming subscription verification.  
    - Inputs: Check output (true branch)  
    - Outputs: End or next steps.

  - **Check failed**  
    - Type: Telegram  
    - Role: Sends a failure message to the user indicating subscription verification failed.  
    - Inputs: Check output (false branch)  
    - Outputs: End or retry.

---

#### 2.5 Referral and Messaging

- **Overview:**  
  Handles referral logic by checking if a user is referred, sending welcome messages, and updating referral status in Postgres.

- **Nodes Involved:**  
  - Referal? (If)  
  - Welcome message Referal (Telegram)  
  - Welcome message Manager (Telegram)  
  - Update bot status and referal (Postgres)  
  - Get Referal (Postgres)

- **Node Details:**

  - **Referal?**  
    - Type: If  
    - Role: Checks if the current user is a referral or manager.  
    - Inputs: Commands output (Referral command)  
    - Outputs: Branches to referral or manager welcome messages.

  - **Welcome message Referal**  
    - Type: Telegram  
    - Role: Sends a welcome message to referred users.  
    - Inputs: Referal? output (true branch)  
    - Outputs: Updates referral status.

  - **Welcome message Manager**  
    - Type: Telegram  
    - Role: Sends a welcome message to managers or non-referral users.  
    - Inputs: Referal? output (false branch)  
    - Outputs: Upserts bot status.

  - **Update bot status and referal**  
    - Type: Postgres  
    - Role: Updates the database with referral information and bot status.  
    - Inputs: Welcome message Referal output  
    - Outputs: Gets referral data.

  - **Get Referal**  
    - Type: Postgres  
    - Role: Retrieves referral information from the database.  
    - Inputs: Update bot status and referal output  
    - Outputs: Get Channels node for subscription check.

---

#### 2.6 (Disabled) File Delivery

- **Overview:**  
  Intended to deliver Google Drive files to users upon successful subscription verification, currently disabled.

- **Nodes Involved:**  
  - Get File (Google Drive) [disabled]  
  - Download File (Google Drive) [disabled]  
  - Check success + File (Telegram) [disabled]

- **Node Details:**

  - **Get File**  
    - Type: Google Drive  
    - Role: Retrieves file metadata or file from Google Drive.  
    - Disabled: true  
    - Inputs: Check output (success branch)  
    - Outputs: Download File.

  - **Download File**  
    - Type: Google Drive  
    - Role: Downloads the file from Google Drive.  
    - Disabled: true  
    - Inputs: Get File output  
    - Outputs: Check success + File.

  - **Check success + File**  
    - Type: Telegram  
    - Role: Sends the downloaded file to the user.  
    - Disabled: true  
    - Inputs: Download File output.

---

### 3. Summary Table

| Node Name                               | Node Type           | Functional Role                                      | Input Node(s)                      | Output Node(s)                               | Sticky Note                                   |
|----------------------------------------|---------------------|-----------------------------------------------------|----------------------------------|----------------------------------------------|-----------------------------------------------|
| Telegram Trigger                       | Telegram Trigger    | Entry point for Telegram messages                    | -                                | Variables TG                                 |                                               |
| Variables TG                          | Set                 | Initialize variables from Telegram message           | Telegram Trigger                 | Initialization                               |                                               |
| Initialization                       | Set                 | Prepare variables for role check                      | Variables TG                    | Is Admin?                                    |                                               |
| Is Admin?                           | If                  | Check if user is admin                                | Initialization                 | Define Type                                  |                                               |
| Define Type                         | Switch              | Route flow based on user type                          | Is Admin?                      | Buttons, Get Bot Status                       |                                               |
| Buttons                            | Switch              | Handle button commands                                | Define Type                    | Update bot status on START                    |                                               |
| Get Bot Status                     | Postgres            | Retrieve bot status from DB                            | Define Type                    | Define flow                                  |                                               |
| Define flow                       | Switch              | Route commands and flow                               | Get Bot Status                 | Commands, Add Channel, Get Channels           |                                               |
| Commands                          | Switch              | Handle user commands                                  | Define flow                   | Referal?, Get Channels, Request Add/Delete   |                                               |
| Referal?                         | If                  | Check referral status                                 | Commands                      | Welcome message Referal, Welcome message Manager |                                               |
| Welcome message Referal           | Telegram            | Send referral welcome message                         | Referal?                      | Update bot status and referal                 |                                               |
| Welcome message Manager           | Telegram            | Send manager welcome message                          | Referal?                      | Upsert bot status on START                     |                                               |
| Update bot status and referal    | Postgres            | Update referral and bot status                        | Welcome message Referal        | Get Referal                                   |                                               |
| Get Referal                      | Postgres            | Retrieve referral info                                | Update bot status and referal  | Get Channels                                  |                                               |
| Get Channels                    | Postgres            | Retrieve channels for subscription check             | Get Referal                   | Get Subscription statuses                      |                                               |
| Get Subscription statuses       | Telegram            | Check user subscription status on channels           | Get Channels                  | Union statuses                                |                                               |
| Union statuses                 | Aggregate           | Aggregate subscription statuses                       | Get Subscription statuses     | Check                                         |                                               |
| Check                          | If                  | Verify if user subscribed to all channels             | Union statuses                | Check success, Check failed                    |                                               |
| Check success                  | Telegram            | Send success message                                 | Check                        | -                                             |                                               |
| Check failed                  | Telegram            | Send failure message                                 | Check                        | -                                             |                                               |
| Add Channel                   | Postgres            | Add new channel to DB                                 | Define flow                  | Request New Add Channel, Channel Exists       |                                               |
| Request New Add Channel       | Telegram            | Ask user for new channel ID                           | Add Channel                  | -                                             |                                               |
| Channel Exists               | Telegram            | Confirm channel existence                             | Add Channel                  | Update bot status on CHECK SUBSCRIPTION REQUEST CHANNEL |                                               |
| Update bot status on CHECK SUBSCRIPTION REQUEST CHANNEL | Postgres | Update status after add channel request              | Request Add Channel           | -                                             |                                               |
| Delete Channel               | Postgres            | Delete channel from DB                                | If                          | Request New Delete Channel, Channel Not Exists |                                               |
| Request New Delete Channel   | Telegram            | Ask user for channel ID to delete                     | Delete Channel               | -                                             |                                               |
| Channel Not Exists           | Telegram            | Notify user channel does not exist                    | Delete Channel               | -                                             |                                               |
| Request Add Channel          | Telegram            | Handle add channel command                            | Commands                    | Update bot status on CHECK SUBSCRIPTION REQUEST CHANNEL |                                               |
| Request Delete Channel       | Telegram            | Handle delete channel command                         | Commands                    | Update bot status on CHECK SUBSCRIPTION REQUEST CHANNEL1 |                                               |
| Update bot status on CHECK SUBSCRIPTION REQUEST CHANNEL1 | Postgres | Update status after delete channel request           | Request Delete Channel        | -                                             |                                               |
| Upsert bot status on START   | Postgres            | Insert or update bot status on start                  | Welcome message Manager      | -                                             |                                               |
| Get Channels                | Postgres            | Retrieve channels for subscription check             | Commands                    | Get Subscription statuses                      |                                               |
| If                          | If                  | Conditional branching for channel existence           | Get Channels                 | Delete Channel, Channel Not Exists             |                                               |
| Union statuses             | Aggregate           | Aggregate subscription statuses                       | Get Subscription statuses    | Check                                         |                                               |
| Get Channels               | Postgres            | Retrieve channels for subscription check             | Get Referal                  | Get Subscription statuses                      |                                               |
| Channels                   | Telegram            | Retrieve Telegram channel info                        | Add Divide Channels          | -                                             |                                               |
| Add Divide Channels         | Summarize           | Summarize channel data                                | Get Channels                 | Channels                                      |                                               |
| Download File              | Google Drive        | Download file from Google Drive (disabled)            | Get File                    | Check success + File                           | Disabled nodes for file delivery               |
| Get File                  | Google Drive        | Get file metadata from Google Drive (disabled)        | Check                       | Download File                                 | Disabled nodes for file delivery               |
| Check success + File      | Telegram            | Send file to user after success (disabled)            | Download File               | -                                             | Disabled nodes for file delivery               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure webhook to receive all Telegram updates for the bot.

2. **Create Set node "Variables TG"**  
   - Initialize variables from Telegram message data (e.g., user ID, message text).

3. **Create Set node "Initialization"**  
   - Prepare variables for role checking or further processing.

4. **Create If node "Is Admin?"**  
   - Condition: Check if user ID is in admin list or matches admin criteria.

5. **Create Switch node "Define Type"**  
   - Route flow based on admin check result: admin branch leads to "Buttons" node, non-admin branch to "Get Bot Status".

6. **Create Switch node "Buttons"**  
   - Handle Telegram button presses or commands from users.

7. **Create Postgres node "Get Bot Status"**  
   - SQL: Select current bot status or session state for the user.

8. **Create Switch node "Define flow"**  
   - Route commands such as Add Channel, Get Channels, or default.

9. **Create Switch node "Commands"**  
   - Handle specific commands from users (e.g., referral, add/delete channel).

10. **Create If node "Referal?"**  
    - Check if the user is a referral or manager.

11. **Create Telegram node "Welcome message Referal"**  
    - Send welcome message to referred users.

12. **Create Telegram node "Welcome message Manager"**  
    - Send welcome message to managers.

13. **Create Postgres node "Update bot status and referal"**  
    - Update referral and bot status in DB.

14. **Create Postgres node "Get Referal"**  
    - Retrieve referral info from DB.

15. **Create Postgres node "Get Channels"**  
    - Retrieve list of channel IDs for subscription verification.

16. **Create Telegram node "Get Subscription statuses"**  
    - Check subscription status of user for each channel.

17. **Create Aggregate node "Union statuses"**  
    - Aggregate subscription statuses into a summary.

18. **Create If node "Check"**  
    - Condition: Are all subscriptions valid?

19. **Create Telegram node "Check success"**  
    - Send success message if user subscribed to all channels.

20. **Create Telegram node "Check failed"**  
    - Send failure message if subscription check fails.

21. **Create Postgres node "Add Channel"**  
    - Insert new channel ID into DB.

22. **Create Telegram node "Request New Add Channel"**  
    - Ask user to provide new channel ID.

23. **Create Telegram node "Channel Exists"**  
    - Confirm channel existence.

24. **Create Postgres node "Update bot status on CHECK SUBSCRIPTION REQUEST CHANNEL"**  
    - Update status after add channel request.

25. **Create Postgres node "Delete Channel"**  
    - Delete channel ID from DB.

26. **Create Telegram node "Request New Delete Channel"**  
    - Ask user for channel ID to delete.

27. **Create Telegram node "Channel Not Exists"**  
    - Notify user if channel does not exist.

28. **Create Telegram node "Request Add Channel"**  
    - Handle add channel command.

29. **Create Telegram node "Request Delete Channel"**  
    - Handle delete channel command.

30. **Create Postgres node "Update bot status on CHECK SUBSCRIPTION REQUEST CHANNEL1"**  
    - Update status after delete channel request.

31. **Create Postgres node "Upsert bot status on START"**  
    - Insert or update bot status on start command.

32. **Create Postgres node "Get Channels " (multiple instances)**  
    - Retrieve channels for various flows.

33. **Create If node "If"**  
    - Conditional branching for channel existence check.

34. **Create Summarize node "Add Divide Channels"**  
    - Summarize channel data for processing.

35. **Create Telegram node "Channels"**  
    - Retrieve Telegram channel info (execute once).

36. **(Optional, Disabled) Google Drive nodes**  
    - "Get File", "Download File", and "Check success + File" to deliver files after subscription check.

37. **Connect nodes as per described flow**  
    - Ensure all nodes are connected according to the workflow logic, respecting branching and error handling.

38. **Configure credentials**  
    - Telegram Bot API credentials for Telegram nodes.  
    - Postgres credentials for database nodes.  
    - Google Drive credentials (optional, for file delivery).

39. **Create required Postgres tables**  
    - Use provided SQL script, replacing schema name as needed.

40. **Test workflow**  
    - Trigger Telegram messages and verify subscription checks and channel management.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                 |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Add the bot as admin to your Telegram channels to enable subscription status checks.            | Setup instruction in workflow description.                                                     |
| Replace `"n8n"` in the SQL script with your Postgres schema name before running.                | Setup instruction in workflow description.                                                     |
| Google Drive file delivery nodes are currently disabled; enable and configure if needed.        | Workflow nodes disabled for file delivery.                                                     |
| Telegram bot credentials and Postgres credentials must be set up in n8n credentials manager.    | Credential setup requirement.                                                                  |
| Workflow tags include: telegram, module, sell.                                                  | Metadata for categorization.                                                                   |
| The workflow uses error continuation on Postgres nodes for Add/Delete channel to handle errors gracefully. | Implementation detail to avoid workflow stops on DB errors.                                    |
| For detailed SQL scripts and channel management, refer to the original workflow JSON or DB schema. | External resource reference.                                                                    |

---

This documentation provides a comprehensive understanding of the Telegram subscription verification workflow, enabling reproduction, modification, and troubleshooting by advanced users and automation agents.
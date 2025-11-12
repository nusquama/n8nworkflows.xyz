Daily Trello Task Tracker with Google Sheets History Log

https://n8nworkflows.xyz/workflows/daily-trello-task-tracker-with-google-sheets-history-log-7673


# Daily Trello Task Tracker with Google Sheets History Log

### 1. Workflow Overview

This workflow automates a daily snapshot of task statuses from a specified Trello board, logging comprehensive task details into a Google Sheets document to maintain a historical progress log. It is designed for project managers and teams who want to track task progress, due dates, and descriptions over time, enabling trend analysis and better project oversight.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Initialization:** Starts the workflow on a daily schedule and generates the current date timestamp.
- **1.2 Trello Data Retrieval:** Sequentially fetches the Trello board, its lists, and the cards (tasks) associated with each list.
- **1.3 Data Transformation:** Normalizes and maps Trello data fields into a structured format suitable for logging.
- **1.4 Data Aggregation:** Merges the transformed Trello data with the current date information.
- **1.5 Data Logging:** Appends the aggregated data rows into a Google Sheets document, creating a daily historical record.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Initialization

- **Overview:**  
  This block initiates the workflow at a set daily interval and generates the current date in ISO format to timestamp the data entries.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Today's Date1  

- **Node Details:**  

  - **Schedule Trigger**  
    - *Type & Role:* Schedule Trigger node that starts the workflow automatically every day.  
    - *Configuration:* Uses default daily interval with no specific time constraints set (runs once every 24 hours).  
    - *Inputs/Outputs:* No inputs; outputs trigger event to "Get Board2" and "Today's Date1".  
    - *Failure Modes:* Network or system downtime may cause missed triggers.  
    - *Credentials:* None required.  
 
  - **Today's Date1**  
    - *Type & Role:* Code node generating today’s date to tag data entries with a daily timestamp.  
    - *Configuration:* JavaScript code returns an ISO date string (YYYY-MM-DD).  
    - *Key Expression:* `new Date().toISOString().split('T')[0]`  
    - *Inputs/Outputs:* Triggered by schedule; output feeds into the Merge node.  
    - *Failure Modes:* Unlikely to fail unless JavaScript runtime errors occur (rare).  
    - *Version:* Uses version 2 of code node (modern JS support).

#### 1.2 Trello Data Retrieval

- **Overview:**  
  This block fetches the entire hierarchy of the Trello board: the board itself, its lists, and the cards within each list.

- **Nodes Involved:**  
  - Get Board2  
  - Get Lists2  
  - Get Cards2  

- **Node Details:**  

  - **Get Board2**  
    - *Type & Role:* Trello node fetching the specified Trello board metadata.  
    - *Configuration:* Board URL provided in URL mode ("https://trello.com/b/DCpuJbnd/administrative-tasks").  
    - *Credentials:* Uses Trello API credentials with API key and token.  
    - *Inputs/Outputs:* Triggered by Schedule Trigger; outputs board JSON to "Get Lists2".  
    - *Failure Modes:* API authentication errors, invalid board URL, or permission issues.  
    - *Version:* Version 1 of Trello node.  

  - **Get Lists2**  
    - *Type & Role:* Trello node retrieving all lists for the board.  
    - *Configuration:* Takes board ID dynamically from "Get Board2" output.  
    - *Credentials:* Same Trello credentials as above.  
    - *Inputs/Outputs:* Input from "Get Board2"; outputs list JSON to "Get Cards2".  
    - *Failure Modes:* Invalid board ID, API limits, or permission errors.  

  - **Get Cards2**  
    - *Type & Role:* Trello node fetching all cards for each list.  
    - *Configuration:* Takes list ID dynamically from "Get Lists2" output.  
    - *Credentials:* Same Trello credentials.  
    - *Inputs/Outputs:* Input from "Get Lists2"; outputs card JSON to "Map Fields2".  
    - *Failure Modes:* Permission issues, large number of cards may cause timeout or throttling.

#### 1.3 Data Transformation

- **Overview:**  
  Maps and normalizes raw Trello data into a consistent schema, selecting key properties from board, list, and card data for logging.

- **Nodes Involved:**  
  - Map Fields2  

- **Node Details:**  

  - **Map Fields2**  
    - *Type & Role:* Set node that creates a structured data object with specific fields for logging.  
    - *Configuration:* Assigns fields: Board Name, List Name, Task Name, Task Description, Due Date (trimmed to date only), and Card URL. Uses expressions to access Trello node outputs.  
    - *Key Expressions:*  
      - Board Name: `={{ $('Get Board2').item.json.name }}`  
      - List Name: `={{ $('Get Lists2').item.json.name }}`  
      - Task Name: `={{ $json.name }}`  
      - Task Description: `={{ $json.desc }}`  
      - Due Date: `={{ $json.badges.due.trim().substring(0, 10) }}`  
      - URL: `={{ $json.url }}`  
    - *Inputs/Outputs:* Input from "Get Cards2"; output to "Merge1".  
    - *Failure Modes:* Expression errors if expected JSON properties are missing or null, e.g., missing due dates.  
    - *Version:* Uses Set node version 3.4.

#### 1.4 Data Aggregation

- **Overview:**  
  Combines the daily date from "Today's Date1" with the mapped Trello task data into a single data stream for logging.

- **Nodes Involved:**  
  - Merge1  

- **Node Details:**  

  - **Merge1**  
    - *Type & Role:* Merge node combining two streams: task data and current date.  
    - *Configuration:* Mode set to "combine" with "combineAll" option, meaning it combines every item from both inputs into one output stream.  
    - *Inputs/Outputs:*  
      - Input 1: Output from "Today's Date1" (date info)  
      - Input 2: Output from "Map Fields2" (task data)  
      - Output to "Daily Progress to Sheet" node.  
    - *Failure Modes:* Mismatched input arrays could cause unexpected merge results.  
    - *Version:* Version 3.2.

#### 1.5 Data Logging

- **Overview:**  
  Appends the combined Trello task data and date into a Google Sheet to create a persistent daily history log.

- **Nodes Involved:**  
  - Daily Progress to Sheet  

- **Node Details:**  

  - **Daily Progress to Sheet**  
    - *Type & Role:* Google Sheets node appending rows to a spreadsheet.  
    - *Configuration:*  
      - Operation: Append  
      - Document ID: Google Sheet ID `1yAdFAsq38OTtZ52jJV2m9LpzGQH-PQmCW-ZPPBF3AAg`  
      - Sheet Name: "Sheet1" (gid=0)  
      - Columns mapped explicitly with values from merged data: URL, Date (from Today's Date1), Due Date, List Name, Task Name, Task Description.  
    - *Credentials:* Uses Google Sheets OAuth2 credentials.  
    - *Inputs/Outputs:* Input from "Merge1"; no output (end node).  
    - *Failure Modes:* OAuth token expiration, API limits, spreadsheet sharing permissions, schema mismatches.  
    - *Version:* Version 4.7.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                   | Input Node(s)             | Output Node(s)           | Sticky Note                                                  |
|---------------------|--------------------|---------------------------------|---------------------------|--------------------------|--------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger   | Initiates workflow daily         | —                         | Get Board2, Today's Date1 |                                                              |
| Get Board2          | Trello             | Fetches Trello board info        | Schedule Trigger           | Get Lists2               |                                                              |
| Get Lists2          | Trello             | Retrieves lists from board       | Get Board2                 | Get Cards2               |                                                              |
| Get Cards2          | Trello             | Retrieves cards from lists       | Get Lists2                 | Map Fields2              |                                                              |
| Map Fields2         | Set                | Maps Trello data to structured format | Get Cards2             | Merge1                   |                                                              |
| Today's Date1       | Code               | Generates today’s ISO date       | Schedule Trigger           | Merge1                   |                                                              |
| Merge1              | Merge              | Combines date and task data      | Today's Date1, Map Fields2 | Daily Progress to Sheet  |                                                              |
| Daily Progress to Sheet | Google Sheets    | Appends data to Google Sheet     | Merge1                     | —                        |                                                              |
| Sticky Note47       | Sticky Note        | Notes workflow purpose           | —                         | —                        | # 📋 Trello → Google Sheets Daily Task Status ...             |
| Sticky Note49       | Sticky Note        | Instructions for Trello API setup | —                       | —                        | ### 1️⃣ Connect Trello (Developer API)...                    |
| Sticky Note3        | Sticky Note        | Setup instructions + workflow summary | —                    | —                        | ## ⚙️ Setup Instructions... (detailed multi-step guidance)   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Parameters: Default daily interval (every 24 hours)  
   - No credentials needed  
   - Connect this as the first node.

2. **Create Today's Date1 Code Node**  
   - Type: Code  
   - JavaScript code:  
     ```javascript
     return [{
       json: {
         badges: {
           today: new Date().toISOString().split('T')[0]
         }
       }
     }];
     ```  
   - Connect input from Schedule Trigger  
   - Output will feed into Merge node.

3. **Create Get Board2 Trello Node**  
   - Type: Trello  
   - Operation: Get Board  
   - Board ID input mode: URL  
   - Board URL: `https://trello.com/b/DCpuJbnd/administrative-tasks`  
   - Credentials: Setup Trello API credentials with API key and token (see Sticky Note instructions)  
   - Connect input from Schedule Trigger.

4. **Create Get Lists2 Trello Node**  
   - Type: Trello  
   - Operation: Get All Lists  
   - Board ID: Use expression to get from Get Board2: `={{ $json.id }}`  
   - Credentials: Same Trello credentials  
   - Connect input from Get Board2.

5. **Create Get Cards2 Trello Node**  
   - Type: Trello  
   - Operation: Get Cards from List  
   - List ID: Use expression from Get Lists2: `={{ $json.id }}`  
   - Credentials: Same Trello credentials  
   - Connect input from Get Lists2.

6. **Create Map Fields2 Set Node**  
   - Type: Set  
   - Enable "Keep Only Set" fields (optional for clean output)  
   - Define fields with following assignments and expressions:  
     - Board Name: `={{ $('Get Board2').item.json.name }}`  
     - List Name: `={{ $('Get Lists2').item.json.name }}`  
     - Task Name: `={{ $json.name }}`  
     - Task Description: `={{ $json.desc }}`  
     - Due Date: `={{ $json.badges.due.trim().substring(0, 10) }}` (handle cases when due date is missing)  
     - url: `={{ $json.url }}`  
   - Connect input from Get Cards2.

7. **Create Merge1 Merge Node**  
   - Type: Merge  
   - Mode: Combine  
   - Combine By: combineAll  
   - Connect first input from Today's Date1  
   - Connect second input from Map Fields2.

8. **Create Daily Progress to Sheet Google Sheets Node**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: `1yAdFAsq38OTtZ52jJV2m9LpzGQH-PQmCW-ZPPBF3AAg`  
   - Sheet Name: "Sheet1" (gid=0)  
   - Columns mapping:  
     - url: `={{ $json.url }}`  
     - Date: `={{ $('Today\'s Date1').item.json.badges.today }}`  
     - Due Date: `={{ $json['Due Date'] }}`  
     - List Name: `={{ $json['List Name'] }}`  
     - Task Name: `={{ $json['Task Name'] }}`  
     - Task Description: `={{ $json['Task Description'] }}`  
   - Credentials: Setup Google Sheets OAuth2 credentials with access to the target spreadsheet  
   - Connect input from Merge1.

9. **Create Sticky Notes (Optional)**  
   - Add explanatory notes as per original workflow content for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Trello API Setup Instructions: Obtain API Key and Token at https://trello.com/app-key, then configure credentials in n8n under Trello API.                                                                                                                        | Sticky Notes in workflow; Trello Developer API documentation                                      |
| Workflow Explanation: This workflow pulls all tasks from a Trello board daily and appends them to a Google Sheet, enabling daily snapshots of project progress and task details.                                                                                   | Workflow description and Sticky Notes47, Sticky Note3                                             |
| Contact for Customization Assistance: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/, Website: https://ynteractive.com                                                                                                       | Provided in Sticky Note3                                                                           |
| Google Sheets OAuth2 requires proper authentication and sharing permissions on the target spreadsheet to allow appending data.                                                                                                                                      | Google Sheets API documentation                                                                    |
| Potential improvements: Add filters to process specific lists or cards, send automated reports via email or Slack, handle empty or missing due dates more robustly.                                                                                                | Suggested based on workflow context                                                               |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.
Easy Redmine and Microsoft Teams Workflow Template

https://n8nworkflows.xyz/workflows/easy-redmine-and-microsoft-teams-workflow-template-7293


# Easy Redmine and Microsoft Teams Workflow Template

### 1. Workflow Overview

This workflow automates the daily collection and distribution of Easy Redmine issues to a Microsoft Teams channel. It is designed for teams using Easy Redmine to manage tasks and want to receive a daily digest of relevant issues directly in their Teams workspace, improving visibility and reducing manual reporting efforts.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger:** Starts the workflow every workday at 8:30 AM.
- **1.2 Issue Retrieval:** Uses the Easy Redmine node to fetch issues based on a saved query/filter.
- **1.3 Data Splitting:** Splits the bulk issues array into individual issue items.
- **1.4 Data Preparation:** Keeps only relevant fields from each issue and adds a direct URL link to the issue in Easy Redmine.
- **1.5 Issue Loop:** Iterates over each processed issue to send individual messages.
- **1.6 Microsoft Teams Messaging:** Sends each issue’s formatted information as an HTML message into a specified Microsoft Teams channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow automatically every weekday at 8:30 AM, ensuring that task digests are sent out on workdays without manual intervention.

- **Nodes Involved:**  
  - Daily Trigger (8:30 work-days)

- **Node Details:**

  - **Node:** Daily Trigger (8:30 work-days)  
    - Type: Schedule Trigger  
    - Configuration: Trigger set for every Monday to Friday at 08:30 (hour 8, minute 30) weekly basis.  
    - Key Expressions: None (static schedule).  
    - Input: None (trigger node).  
    - Output: Initiates flow to Get Issues by Query node.  
    - Edge Cases: Scheduler misconfiguration or timezone issues may cause missed or delayed runs.  
    - Requirements: n8n version supporting scheduleTrigger typeVersion 1.2 or above.

#### 2.2 Issue Retrieval

- **Overview:**  
  Fetches issues from Easy Redmine using a saved query/filter that targets relevant tasks, e.g., assigned to a specific team, opened recently.

- **Nodes Involved:**  
  - Get Issues by Query

- **Node Details:**

  - **Node:** Get Issues by Query  
    - Type: Easy Redmine integration node  
    - Configuration: Uses `issueQueryId` 4809 representing a pre-saved filter in Easy Redmine.  
    - Credentials: Connected via Easy Redmine API credentials.  
    - Input: Triggered from Schedule Trigger node.  
    - Output: Returns an object containing an array of issues under `issues` field.  
    - Edge Cases: API authentication failure, network timeout, invalid query ID, or no issues matching query.  
    - Requirements: Valid Easy Redmine API credentials with permission to access issues data.  

#### 2.3 Data Splitting

- **Overview:**  
  Converts the single data item containing a list of issues into multiple items, each representing one issue, enabling individual processing.

- **Nodes Involved:**  
  - Split Out Issues

- **Node Details:**

  - **Node:** Split Out Issues  
    - Type: Split Out (array splitter)  
    - Configuration: Splits on the `issues` field, producing one output item per issue.  
    - Input: Receives an object with an `issues` array from Get Issues by Query node.  
    - Output: Multiple items, each representing a single issue.  
    - Edge Cases: If the `issues` array is empty or missing, downstream nodes may receive no input or invalid data.  
    - Requirements: n8n version supporting Split Out node typeVersion 1.  

#### 2.4 Data Preparation

- **Overview:**  
  Filters each issue’s data to keep only relevant fields (ID, subject, author name, description) and constructs a direct URL link to the issue for easy access.

- **Nodes Involved:**  
  - Keep Relevant Fields & Add Link

- **Node Details:**

  - **Node:** Keep Relevant Fields & Add Link  
    - Type: Set node (field assignment)  
    - Configuration:  
      - Keeps only `id`, `subject`, `author.name`, `description`.  
      - Adds a new field `link` with a constructed URL combining a base URL and the issue ID (`https://easy-redmine-application.com/issues/{{ $json.id }}`).  
    - Key Expressions: Uses expressions like `={{ $json.id }}` to map data, and string concatenation to build links.  
    - Input: Individual issue items from Split Out Issues node.  
    - Output: Enhanced issue item with selected fields and link.  
    - Edge Cases: Missing fields in original data (e.g., author or description) may result in incomplete messages.  
    - Requirements: n8n version supporting Set node typeVersion 3.4 or above.

#### 2.5 Issue Loop

- **Overview:**  
  Iterates over the processed individual issues to send a message for each one separately to the Teams channel.

- **Nodes Involved:**  
  - Run for Each Task (Split In Batches)

- **Node Details:**

  - **Node:** Run for Each Task  
    - Type: Split In Batches  
    - Configuration: Default batch size (implied 1), processes one issue per iteration.  
    - Input: Receives enriched issues from Keep Relevant Fields & Add Link node.  
    - Output: Feeds one issue at a time into the Microsoft Teams message node.  
    - Edge Cases: Large number of issues may slow execution or hit platform limits.  
    - Requirements: n8n version supporting Split In Batches typeVersion 3.

#### 2.6 Microsoft Teams Messaging

- **Overview:**  
  Sends each issue’s details formatted in HTML as a message into a specific Microsoft Teams channel.

- **Nodes Involved:**  
  - Message into Team Channel

- **Node Details:**

  - **Node:** Message into Team Channel  
    - Type: Microsoft Teams node  
    - Configuration:  
      - Sends a chatMessage resource type with HTML content.  
      - Message constructed with issue subject, link (ID as clickable), author name, and description.  
      - Target chat selected by chatId corresponding to a specific Teams channel.  
    - Credentials: Uses OAuth2 credentials for Microsoft Teams API.  
    - Input: Single issue item from Run for Each Task node.  
    - Output: Passes output to Run for Each Task again (loop continuation).  
    - Edge Cases: Authentication failures, API rate limits, invalid chatId, or malformed HTML content.  
    - Requirements: Valid Microsoft Teams OAuth2 credential with permissions to post messages in the target channel.

---

### 3. Summary Table

| Node Name                   | Node Type                     | Functional Role                                | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                                                          |
|-----------------------------|-------------------------------|-----------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Daily Trigger (8:30 work-days) | Schedule Trigger              | Starts workflow every weekday at 8:30 AM      |                             | Get Issues by Query          |                                                                                                                                        |
| Get Issues by Query          | Easy Redmine node              | Retrieves issues using saved Easy Redmine query | Daily Trigger (8:30 work-days) | Split Out Issues             |                                                                                                                                        |
| Split Out Issues             | Split Out                     | Splits issues array into individual issue items | Get Issues by Query          | Keep Relevant Fields & Add Link |                                                                                                                                        |
| Keep Relevant Fields & Add Link | Set                         | Keeps relevant fields and adds URL link        | Split Out Issues             | Run for Each Task            |                                                                                                                                        |
| Run for Each Task            | Split In Batches              | Loops over each issue to process individually   | Keep Relevant Fields & Add Link | Message into Team Channel    |                                                                                                                                        |
| Message into Team Channel    | Microsoft Teams               | Sends formatted issue message to MS Teams channel | Run for Each Task            | Run for Each Task            |                                                                                                                                        |
| Sticky Note2                | Sticky Note                   | Workflow overview and step explanation          |                             |                             | ## Call for issues in Easy Redmine & send relevant information to MS teams channel **1) GET issues by query via "Easy Redmine node"** - receive all issues corresponding to saved filter in Easy Redmine [(Read more about Easy Redmine filters)](https://www.easyredmine.com/documentation-of-easy-redmine/article/filters-1-0) **2) "Split out" issues array** - to work with separate issues as single items **3) "Edit fields" - keep relevant data & add URL to issue** - keep only issue id, subject, author, description, and create URL link to issue **4) "Loop over items" - send one issue at the time** **5) Send the message via "MS Teams node"** - use technical user & select relevant teams channel or chat |
| Sticky Note3                | Sticky Note                   | Final output example in Teams chat               |                             |                             | ## Final Output ### Received in teams chat Name: **Client ABC - request consulting help** Link: [1354](https://easyredmineapp.com/issues/1354) Author: **Jane Doe** Description: Service Request for consulting help: The necessary help including initiation of service delivery until the end of week. Client requested support with data migration and configuration of project modules including roles and permissions set-up. |
| Sticky Note4                | Sticky Note                   | Detailed workflow description, use case, and requirements |                             |                             | ## Daily Team Task Digest to MS Teams **Try out using a native Easy Redmine node to get issues according saved filter and send them to your MS Teams Channel.** ### About Workflow This workflow automatically collects newly assigned tasks for a specific team and posts them into a Microsoft Teams channel each morning. ### Use Case Any team working with issues in Easy Redmine can benefit from automatic summary send to Teams - making sure all issues are addressed in time & saving time on meeting preparation without any copy pasting and no obsolete data. ### How it works - Scheduled trigger will ensure the automatic run => every work-day at 8:30 - Easy Redmine node will GET all the issues by saved filter => Issues | Get Many | Query Name or ID - The filter is saved in Easy Redmine and assures all relevant issues are received => Assignee is "Consulting Team" | Status is "Open" | Updated within "last 3 days" - Split out received data => Output response is **one item**, which contains all received issues => Split array "issues" to have 1 issue per item - Edit fields - keep only relevant data, adjust description, create URL link => Fields: ID | Author | Subject | Description | URL Link => URL link: Add expression https://easyredmineapp.com/issues/{{issue ID}} to create a link to the issue - Loop over items - send one message per issue - Send to MS Teams - by native MS Teams node => Chat Message | Create | Selected Channel | HTML content => HTML content mapped from edit fields node ### How to use - Adjust the set-up to your team's need - Use your preffered filter and teams chat - Fetch open tasks assigned to a selected group or role. - Use the workflow for multiple teams - Adjust trigger for different intervals ### Requirements - Easy Redmine application => ideally technical user for API calls with specific permissions - MS Teams => ideally technical user for API calls with specific permissions ### Need Help? - Reach out through n8n community => https://community.n8n.io/u/easy8.ai - Contant our team directly => Easy8.ai - Visit our youtube channel => https://www.youtube.com/@easy8ai |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Scheduled Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger weekly on Monday to Friday at 08:30 (hour 8, minute 30).  
   - No credentials required.

2. **Add Easy Redmine node to Get Issues by Query**  
   - Type: Easy Redmine (custom n8n node)  
   - Set parameter `issueQueryId` to `4809` (or your own saved query ID in Easy Redmine).  
   - Configure Easy Redmine API credentials (API key/token with permissions to read issues).  
   - Connect the output of Schedule Trigger to this node.

3. **Add Split Out node to split `issues` array**  
   - Type: Split Out  
   - Configure to split on `issues` field from the previous node’s output.  
   - Connect output of Easy Redmine node to this node.

4. **Add Set node to keep relevant fields and add a link**  
   - Type: Set  
   - Configure to keep only the following fields copied from input:  
     - `id` (number) = `{{$json.id}}`  
     - `subject` (string) = `{{$json.subject}}`  
     - `author.name` (string) = `{{$json.author.name}}`  
     - `description` (string) = `{{$json.description}}`  
   - Add new field `link` (string) with value:  
     `https://easy-redmine-application.com/issues/{{$json.id}}`  
   - Connect output of Split Out node to this node.

5. **Add Split In Batches node for looping over issues**  
   - Type: Split In Batches  
   - Default batch size (1 item per batch) is appropriate.  
   - Connect output of Set node to this node.

6. **Add Microsoft Teams node to send message**  
   - Type: Microsoft Teams  
   - Resource: chatMessage  
   - Operation: Create  
   - Configure `chatId` by selecting your target Microsoft Teams channel or chat.  
   - Message content type: HTML  
   - Message content:  
     ```
     Name: <b>{{ $json.subject }}</b><br>Link: <a href="{{ $json.link }}">{{ $json.id }}</a><br>Author: <b>{{ $json.author.name }}</b><br>Description:<br>{{ $json.description }}
     ```  
   - Configure Microsoft Teams OAuth2 credentials with permissions to post messages.  
   - Connect output of Split In Batches node to this node.

7. **Connect Microsoft Teams node output back to Split In Batches node**  
   - This creates the loop until all items processed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                        |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| ## Daily Team Task Digest to MS Teams <br> Try out using a native Easy Redmine node to get issues according saved filter and send them to your MS Teams Channel. <br><br> This workflow automatically collects newly assigned tasks for a specific team and posts them into a Microsoft Teams channel each morning. <br><br> Use case: Any team working with issues in Easy Redmine can benefit from automatic summary send to Teams - making sure all issues are addressed in time & saving time on meeting preparation without any copy pasting and no obsolete data. <br><br> Requirements: Easy Redmine application and Microsoft Teams with technical users for API calls. <br><br> Help: Reach out on n8n community https://community.n8n.io/u/easy8.ai, contact Easy8.ai, or visit https://www.youtube.com/@easy8ai | Workflow instructions and context, help & support links                |
| Read more about Easy Redmine filters in the official Easy Redmine documentation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | https://www.easyredmine.com/documentation-of-easy-redmine/article/filters-1-0 |
| Example of final message format received in Teams chat: <br> Name: **Client ABC - request consulting help** <br> Link: [1354](https://easyredmineapp.com/issues/1354) <br> Author: **Jane Doe** <br> Description: Service request details including initiation, support needs, and configuration.                                                                                                                                                                                                                                                                                                                                                                                      | Example of output message format                                       |

---

This documentation fully describes the workflow structure, logic, node configurations, and provides all necessary instructions and notes to understand, reproduce, and maintain the workflow effectively.
Role-Based Access Control (RBAC) for Telegram Automations

https://n8nworkflows.xyz/workflows/role-based-access-control--rbac--for-telegram-automations-10171


# Role-Based Access Control (RBAC) for Telegram Automations

### 1. Workflow Overview

This workflow implements a **Role-Based Access Control (RBAC) system for Telegram automations**. Its main purpose is to control and route Telegram user interactions based on their assigned roles and positions stored in an employee database. The workflow validates the Telegram user, retrieves their role and position, then directs the automation flow accordingly, allowing different automation branches per user role.

The workflow is structured in these logical blocks:

- **1.1 Input Reception:** Captures incoming Telegram messages and triggers the workflow.
- **1.2 User Lookup:** Queries the employee database by Telegram username to retrieve user-related data (Position and Type).
- **1.3 Position Evaluation:** Switch node directs flow based on the user’s Position (Marketing, Sales, Administration).
- **1.4 Role Evaluation:** Nested switch node inside the Marketing branch evaluates the user’s Type (Target, SEO, SMM) to further refine access.
- **1.5 Access Branches:** Different branches execute per role, currently represented by No Operation nodes as placeholders.
- **1.6 Documentation and Notes:** Sticky Notes provide instructions, context, and usage guidance.


---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Captures Telegram messages to trigger the workflow and start the RBAC evaluation.

**Nodes Involved:**  
- Telegram Trigger

**Node Details:**  
- **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Role: Entry point listens for incoming Telegram messages (update type: "message")  
  - Configuration: Uses the Telegram API credential "Test Automatization"  
  - Input Connections: None (trigger node)  
  - Output Connections: Connects to "Employee database" node  
  - Version: 1.2  
  - Edge Cases:  
    - Failure if Telegram API credentials expired or revoked  
    - Messages without usernames will fail later during lookup  
    - Potential webhook timeout if Telegram service is down  

---

#### 2.2 User Lookup

**Overview:**  
Queries the employee database to retrieve user information based on the Telegram username.

**Nodes Involved:**  
- Employee database

**Node Details:**  
- **Employee database**  
  - Type: Data Table node  
  - Role: Retrieves user record matching Telegram username from a custom employee table  
  - Configuration:  
    - Operation: Get  
    - Filter: Matches "UserName" field with `{{$json.message.from.username}}`  
    - Data Table ID: Preconfigured to an employee table with columns UserName, Position, Type  
  - Input Connections: From "Telegram Trigger"  
  - Output Connections: To "Choose Position" switch node  
  - Version: 1  
  - Edge Cases:  
    - No matching username: results in empty output, causing downstream switches to receive no data  
    - Username field missing or empty in Telegram message  
    - Data table access errors or misconfiguration  

---

#### 2.3 Position Evaluation

**Overview:**  
Routes the workflow based on the Position field extracted from the employee database.

**Nodes Involved:**  
- Choose Position (Switch node)

**Node Details:**  
- **Choose Position**  
  - Type: Switch node  
  - Role: Checks the user's Position and routes to Marketing, Sales, or Administration branches  
  - Configuration:  
    - Conditions check `{{$json.Position}}` equality against "Marketing", "Sales", "Administration"  
  - Input Connections: From "Employee database"  
  - Output Connections:  
    - Marketing → Switch node "Switch" (Role evaluation)  
    - Sales → No Operation placeholder  
    - Administration → No Operation placeholder  
  - Version: 3.3  
  - Edge Cases:  
    - Position field missing or not matching any branch leads to no path taken (silent fail)  
    - Case sensitivity enforced by strict comparison  

---

#### 2.4 Role Evaluation (Inside Marketing branch)

**Overview:**  
Further refines access control by checking the user's Type within the Marketing position.

**Nodes Involved:**  
- Switch (Role)

**Node Details:**  
- **Switch**  
  - Type: Switch node  
  - Role: Checks employee Type (role) and routes to Target, SEO, or SMM branches  
  - Configuration:  
    - Compares `{{$('Employee database').item.json.Type}}` with "Target", "SEO", or "SMM"  
  - Input Connections: From "Choose Position" (Marketing output)  
  - Output Connections:  
    - Target → No Operation placeholder  
    - SEO → No Operation placeholder  
    - SMM → No Operation placeholder  
  - Version: 3.3  
  - Edge Cases:  
    - Type field missing or unmatched leads to no route taken  
    - Expression failures if Employee database output missing  
    - Case sensitivity on role strings  

---

#### 2.5 Access Branches

**Overview:**  
Placeholders representing actions or workflows executed per role/position.

**Nodes Involved:**  
- No Operation, do nothing (multiple instances)

**Node Details:**  
- **No Operation nodes**  
  - Type: NoOp (No Operation) nodes  
  - Role: Serve as placeholders for future automation or branching logic per role  
  - Configuration: Empty, default  
  - Input Connections: From Switch or Choose Position nodes according to branch  
  - Output Connections: None (end nodes)  
  - Version: 1  
  - Edge Cases: None (no processing)  

---

#### 2.6 Documentation and Notes

**Overview:**  
Sticky Notes provide contextual information and instructions for users to understand and customize the workflow.

**Nodes Involved:**  
- Sticky Note (multiple instances)

**Node Details:**  
- **Sticky Notes**  
  - Type: Sticky Note nodes  
  - Role: Provide documentation and guidance on workflow setup and logic  
  - Examples:  
    - "Start Workflow" near Telegram Trigger  
    - "Access level verification" near Employee database  
    - "First level of access" near Choose Position  
    - "Second level of access" near Role Switch  
    - Instructions for preparing employee table and configuring logic  
  - Input/Output: Visual annotations only  
  - Version: 1  

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role               | Input Node(s)        | Output Node(s)         | Sticky Note                                                                                              |
|--------------------------|---------------------|------------------------------|----------------------|------------------------|--------------------------------------------------------------------------------------------------------|
| Telegram Trigger         | Telegram Trigger    | Entry point for Telegram msgs | None                 | Employee database      | ## Start Workflow                                                                                       |
| Employee database        | Data Table          | Retrieve user info by username| Telegram Trigger     | Choose Position        | ## Access level verification                                                                            |
| Choose Position          | Switch              | Route by Position             | Employee database     | Switch (Role), NoOp(1), NoOp(2) | ## First level of access                                                                                |
| Switch                  | Switch              | Route by Role (Type)          | Choose Position (Marketing) | NoOp(3), NoOp(4), NoOp(5) | ## Second level of access                                                                               |
| No Operation, do nothing2| NoOp                | Target branch placeholder     | Switch (Target)       | None                   | ## Different Access Branch                                                                              |
| No Operation, do nothing3| NoOp                | SEO branch placeholder        | Switch (SEO)          | None                   | ## Different Access Branch                                                                              |
| No Operation, do nothing4| NoOp                | SMM branch placeholder        | Switch (SMM)          | None                   | ## Different Access Branch                                                                              |
| No Operation, do nothing1| NoOp                | Sales branch placeholder      | Choose Position (Sales) | None                   | ## Different Access Branch                                                                              |
| No Operation, do nothing | NoOp                | Administration branch placeholder | Choose Position (Administration) | None                   | ## Different Access Branch                                                                              |
| Sticky Note              | Sticky Note         | Documentation                 | None                 | None                   | ## Start Workflow                                                                                       |
| Sticky Note1             | Sticky Note         | Documentation                 | None                 | None                   | ## Access level verification                                                                            |
| Sticky Note2             | Sticky Note         | Documentation                 | None                 | None                   | ## First level of access                                                                                |
| Sticky Note3             | Sticky Note         | Documentation                 | None                 | None                   | ## Second level of access                                                                               |
| Sticky Note4             | Sticky Note         | Documentation                 | None                 | None                   | ## Different Access Branch                                                                              |
| Sticky Note5             | Sticky Note         | Documentation                 | None                 | None                   | ## Different Access Branch                                                                              |
| Sticky Note6             | Sticky Note         | Documentation                 | None                 | None                   | ## Different Access Branch                                                                              |
| Sticky Note7             | Sticky Note         | Documentation                 | None                 | None                   | ## Different Access Branch                                                                              |
| Sticky Note8             | Sticky Note         | Documentation                 | None                 | None                   | ## Different Access Branch                                                                              |
| Sticky Note9             | Sticky Note         | Documentation                 | None                 | None                   | ## Can be used                                                                                        |
| Sticky Note12            | Sticky Note         | Documentation & instructions | None                 | None                   | ## Instruction\n\n**1. Prepare Employee Table**\nCreate a Data Table with columns:\nUserName===Position===Type\n\nUser_1===Marketing===SEO  \nUser_2===Sales===Manager  \nUser_3===Administration===HR\n\nSelect this table in the Employee database node (Data Table ID).\n\n**2. Configure Access Logic**\n\nSwitch (Position): {{$json.Position}}\n→ branches: Marketing, Sales, Administration\n\nInside Marketing: Switch (Role) → {{ $('Employee database').item.json.Type }}\n→ roles: Target, SEO, SMM\n\n**3. Customize Role Actions**\n\nTarget → performance tracking\n\nSEO → Telegram confirmation / analytics sync\n\nSMM → content posting\n\nSales / Administration → custom workflows\n\n**4. Test the Flow**\nActivate workflow → send Telegram message →\nsystem checks username → finds Position & Type → routes to correct branch. |


---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters:  
     - Updates: Select "message"  
   - Credential: Connect with Telegram API credentials (e.g., "Test Automatization")  
   - Position: Start node (left side)

2. **Create Employee database node**  
   - Type: Data Table  
   - Operation: Get  
   - Filter: Add condition where "UserName" equals expression `{{$json.message.from.username}}`  
   - Data Table ID: Select or create a Data Table with columns: UserName, Position, Type  
   - Connect input from Telegram Trigger node

3. **Create Choose Position switch node**  
   - Type: Switch  
   - Add three rules for output:  
     - Marketing: `{{$json.Position}}` equals "Marketing"  
     - Sales: `{{$json.Position}}` equals "Sales"  
     - Administration: `{{$json.Position}}` equals "Administration"  
   - Connect input from Employee database node

4. **Create Role switch node** (inside Marketing branch)  
   - Type: Switch  
   - Add three rules for output:  
     - Target: `{{$('Employee database').item.json.Type}}` equals "Target"  
     - SEO: `{{$('Employee database').item.json.Type}}` equals "SEO"  
     - SMM: `{{$('Employee database').item.json.Type}}` equals "SMM"  
   - Connect input from Choose Position node’s Marketing output

5. **Create No Operation nodes for all branches**  
   - Create one NoOp node for each branch: Target, SEO, SMM, Sales, Administration  
   - Connect Target, SEO, SMM outputs of Role switch node to corresponding NoOp nodes  
   - Connect Sales and Administration outputs of Choose Position switch node to their respective NoOp nodes

6. **Add Sticky Notes for documentation** (optional but recommended)  
   - Add Sticky Notes near key nodes with explanations such as "Start Workflow", "Access level verification", "First level of access", "Second level of access", and branch descriptions

7. **Test the workflow**  
   - Activate the workflow  
   - Send a Telegram message from a user whose username exists in the employee table  
   - Verify the routing matches Position and Type fields correctly

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Workflow requires a pre-configured Data Table with columns: UserName, Position, Type. Example entries: User_1: Marketing/SEO, User_2: Sales/Manager, User_3: Administration/HR.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | See Sticky Note12 content                      |
| Role names and positions are case sensitive in switch conditions and must match exactly. Adjust switch rules to fit your organization's naming conventions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note12 instructions                      |
| No Operation nodes act as placeholders for custom workflows or actions per role. Replace them with actual automation nodes as needed (e.g., sending Telegram messages, syncing analytics, posting content).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note4/5/6/7/8 mentions "Different Access Branch" |
| Telegram API credentials must be valid and authorized for the bot to receive messages. Ensure the Telegram bot has required permissions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Telegram Trigger node credential                |
| This workflow assumes Telegram messages contain a valid username in `message.from.username`. Messages missing this field will not find a matching record and be dropped silently. Consider adding error handling for unmatched users if needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Edge cases in User Lookup block                 |

---

**Disclaimer**: The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects existing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.
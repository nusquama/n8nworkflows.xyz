Deduplicate Data Records Using JavaScript Array Methods

https://n8nworkflows.xyz/workflows/deduplicate-data-records-using-javascript-array-methods-5730


# Deduplicate Data Records Using JavaScript Array Methods

---

### 1. Workflow Overview

This n8n workflow, titled **"Deduplicate Data Records Using JavaScript Array Methods"**, demonstrates how to identify and remove duplicate entries from a dataset using JavaScript code within n8n. Its primary goal is to clean messy, real-world data—specifically user records with duplicate email addresses—before further processing such as CRM import or database insertion.

The workflow is structured into these logical blocks:

- **1.1 Manual Trigger and Initialization:** Starts the workflow execution.
- **1.2 Sample Data Creation:** Generates a set of sample user records with intentional duplicates.
- **1.3 Deduplication Logic:** Applies JavaScript array methods to remove duplicate users based on their email addresses.
- **1.4 Results Compilation and Display:** Prepares and outputs a summary of deduplication results including statistics.
- **1.5 Documentation and Context (Sticky Notes):** Several sticky notes provide explanations, learning objectives, and use case contexts throughout the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger and Initialization

**Overview:**  
This block initiates the workflow when manually triggered by the user.

**Nodes Involved:**  
- When clicking 'Test workflow'

**Node Details:**

- **When clicking 'Test workflow'**  
  - Type: Manual Trigger node  
  - Role: Entry point; starts workflow execution manually  
  - Configuration: Default, no parameters  
  - Inputs: None  
  - Outputs: Connects to "Create Sample Data" node  
  - Version: 1  
  - Edge Cases: None as trigger is manual; user must start manually  
  - Sub-workflow: None

---

#### 1.2 Sample Data Creation

**Overview:**  
This block creates a mock dataset simulating real-world user data containing duplicate records based on email addresses.

**Nodes Involved:**  
- Create Sample Data  
- Sticky Note1 (documentation)

**Node Details:**

- **Create Sample Data**  
  - Type: Code node (JavaScript)  
  - Role: Generates sample user data with duplicates  
  - Configuration: Inline JavaScript creates an array of six user objects, two of which are duplicates (same email)  
  - Key expressions:  
    - Creates `usersWithDuplicates` array with sample user objects  
    - Serializes array to JSON string under property `usersJson`  
    - Outputs a single item containing JSON string, total count, and a message  
  - Inputs: Receives trigger from manual start  
  - Outputs: Passes data to "Deduplicate Users" node  
  - Version: 2  
  - Edge Cases: None expected; static data generation  
  - Sub-workflow: None

- **Sticky Note1**  
  - Describes sample data purpose, highlights that data includes duplicates simulating messy real-world data  
  - Provides context about next step being deduplication

---

#### 1.3 Deduplication Logic

**Overview:**  
This block implements the core deduplication algorithm using JavaScript array methods to filter out duplicate user entries based on email addresses.

**Nodes Involved:**  
- Deduplicate Users  
- Sticky Note2 (documentation)

**Node Details:**

- **Deduplicate Users**  
  - Type: Code node (JavaScript)  
  - Role: Deduplicates the user list by email  
  - Configuration:  
    - Parses JSON string from input (`usersJson`) back to array  
    - Uses `filter()` combined with `findIndex()` to keep only the first occurrence of each unique email  
    - Logs counts of original, deduplicated, and removed duplicates to console  
    - Returns deduplicated users as separate items in n8n format  
  - Key expressions:  
    ```js
    const uniqueUsers = users.filter(
      (user, index, self) =>
        index === self.findIndex(u => u.email === user.email)
    );
    ```
  - Inputs: Receives JSON stringified users from "Create Sample Data"  
  - Outputs: Passes deduplicated user items to "Display Results"  
  - Version: 2  
  - Edge Cases:  
    - If input JSON is malformed, parsing will fail causing node error  
    - If dataset is very large, performance may degrade (filter/findIndex are O(n²))  
    - Duplicate detection is case-sensitive on email; variations in case may cause misses  
  - Sub-workflow: None

- **Sticky Note2**  
  - Explains deduplication concepts and algorithm steps  
  - Emphasizes use of `filter()` and `findIndex()` methods to retain first occurrences

---

#### 1.4 Results Compilation and Display

**Overview:**  
This block prepares a final summary of the deduplicated data and outputs statistics such as counts and process completion timestamp.

**Nodes Involved:**  
- Display Results  
- Sticky Note3 (documentation)

**Node Details:**

- **Display Results**  
  - Type: Code node (JavaScript)  
  - Role: Aggregates deduplicated user data and compiles a summary object  
  - Configuration:  
    - Counts the number of unique users  
    - Creates a summary JSON object including the deduplicated user list, statistics (count, timestamp), and a message string  
    - Returns a single item containing this summary  
  - Key expressions:  
    ```js
    const summary = {
      deduplicated_users: currentItems.map(item => item.json),
      statistics: {
        unique_users_count: uniqueCount,
        process_completed: true,
        timestamp: new Date().toISOString()
      },
      message: `Successfully deduplicated data - ${uniqueCount} unique users remaining`
    };
    ```
  - Inputs: Receives deduplicated user items from "Deduplicate Users"  
  - Outputs: Final output of workflow with deduplication summary  
  - Version: 2  
  - Edge Cases: None expected unless input is empty array  
  - Sub-workflow: None

- **Sticky Note3**  
  - Describes expected results: counts before and after deduplication, duplicates removed  
  - Suggests use cases such as cleaning data for CRM import or ETL pipeline

---

#### 1.5 Documentation and Context (Sticky Notes)

**Overview:**  
This block contains multiple sticky notes that provide documentation, learning objectives, author info, and coaching offers to enhance user understanding.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note4

**Node Details:**

- **Sticky Note**  
  - Content: Title, author info (David Olusola), and offers for personalized n8n coaching and consulting with email links  
  - Position: Top-left of workflow

- **Sticky Note4**  
  - Content: Learning objectives covering JavaScript array methods, data deduplication, JSON handling, data transformation, best practices (validation, performance, error handling)  
  - Positioned near manual trigger

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                        | Input Node(s)                | Output Node(s)          | Sticky Note                                                                                          |
|-------------------------|---------------------|-------------------------------------|-----------------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| When clicking 'Test workflow' | Manual Trigger      | Starts the workflow manually          | None                        | Create Sample Data       | See Sticky Note4 for learning objectives                                                          |
| Create Sample Data       | Code                | Generates sample user data with duplicates | When clicking 'Test workflow' | Deduplicate Users         | Sticky Note1 describes sample data creation and duplicates                                        |
| Deduplicate Users        | Code                | Filters out duplicates based on email | Create Sample Data           | Display Results          | Sticky Note2 explains deduplication logic and JavaScript array methods                            |
| Display Results          | Code                | Compiles summary and statistics of deduplicated data | Deduplicate Users            | None                    | Sticky Note3 provides results overview and use case recommendations                               |
| Sticky Note             | Sticky Note         | Documentation, author info, coaching offers | None                        | None                    | Contains author info and coaching/consulting offers with email links                               |
| Sticky Note1            | Sticky Note         | Documentation on sample data creation | None                        | None                    | Explains creation of sample data with intentional duplicates                                     |
| Sticky Note2            | Sticky Note         | Documentation on deduplication logic  | None                        | None                    | Explains use of filter() and findIndex() for deduplication                                       |
| Sticky Note3            | Sticky Note         | Documentation on final results         | None                        | None                    | Describes expected results and use cases                                                        |
| Sticky Note4            | Sticky Note         | Learning objectives and best practices | None                        | None                    | Lists learning goals and advises on validation, performance, and error handling                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking 'Test workflow'`  
   - Type: Manual Trigger  
   - No parameters needed  
   - This node will start the workflow manually.

2. **Create a Code Node for Sample Data**  
   - Name: `Create Sample Data`  
   - Type: Code (JavaScript)  
   - Set language to JavaScript  
   - Paste the following code to create sample users with duplicates:
     ```js
     const usersWithDuplicates = [
       { id: 1, name: "John Doe", email: "john@example.com", department: "Engineering" },
       { id: 2, name: "Jane Smith", email: "jane@example.com", department: "Marketing" },
       { id: 3, name: "John Doe", email: "john@example.com", department: "Engineering" }, // Duplicate
       { id: 4, name: "Bob Johnson", email: "bob@example.com", department: "Sales" },
       { id: 5, name: "Alice Brown", email: "alice@example.com", department: "HR" },
       { id: 6, name: "Jane Smith Updated", email: "jane@example.com", department: "Marketing" } // Duplicate
     ];
     return [{
       json: {
         usersJson: JSON.stringify(usersWithDuplicates),
         totalCount: usersWithDuplicates.length,
         message: "Sample data with duplicates created"
       }
     }];
     ```
   - Connect input from `When clicking 'Test workflow'`.

3. **Create a Code Node for Deduplication**  
   - Name: `Deduplicate Users`  
   - Type: Code (JavaScript)  
   - Paste the following code to perform deduplication by email:
     ```js
     const users = JSON.parse(items[0].json.usersJson);
     const uniqueUsers = users.filter(
       (user, index, self) =>
         index === self.findIndex(u => u.email === user.email)
     );
     console.log(`Original count: ${users.length}`);
     console.log(`Deduplicated count: ${uniqueUsers.length}`);
     console.log(`Duplicates removed: ${users.length - uniqueUsers.length}`);
     return uniqueUsers.map(user => ({ json: user }));
     ```
   - Connect input from `Create Sample Data`.

4. **Create a Code Node to Display Results**  
   - Name: `Display Results`  
   - Type: Code (JavaScript)  
   - Paste the following code to aggregate and summarize the results:
     ```js
     const uniqueCount = items.length;
     const summary = {
       deduplicated_users: items.map(item => item.json),
       statistics: {
         unique_users_count: uniqueCount,
         process_completed: true,
         timestamp: new Date().toISOString()
       },
       message: `Successfully deduplicated data - ${uniqueCount} unique users remaining`
     };
     return [{ json: summary }];
     ```
   - Connect input from `Deduplicate Users`.

5. **Link Nodes in Sequence:**  
   - Connect `When clicking 'Test workflow'` → `Create Sample Data` → `Deduplicate Users` → `Display Results`.

6. **Add Sticky Notes (Optional, for Documentation):**  
   - Insert sticky notes next to each functional block to document purpose, logic, and results as per the original content.

7. **Credentials:**  
   - No credentials are needed for this workflow as it uses only manual trigger and code nodes.

8. **Save and Activate Workflow:**  
   - Save workflow, ensure nodes have no errors, then run manually via the trigger node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                              |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Author: David Olusola offers personalized n8n coaching and consulting. Email for coaching: david@daexai.com with subject "n8n Coaching Request".               | Coaching Offer Link (mailto:david@daexai.com?subject=n8n%20Coaching%20Request) |
| For consulting services inquiries, email: david@daexai.com with subject "n8n Consultation Request".                                                             | Consulting Offer Link (mailto:david@daexai.com?subject=n8n%20Consultation%20Request) |
| Learning objectives include understanding JavaScript array methods, data deduplication techniques, JSON handling in n8n, and best practices for production use. | Sticky Note4                                                  |

---

**Disclaimer:**  
The provided text and workflow are generated exclusively from an automated n8n workflow environment, adhering strictly to current content policies. The data processed is legal and public with no illegal or offensive content.

---
Personal Budget & Expense Tracker with Google Sheets and Alerts MCP

https://n8nworkflows.xyz/workflows/personal-budget---expense-tracker-with-google-sheets-and-alerts-mcp-4612


# Personal Budget & Expense Tracker with Google Sheets and Alerts MCP

### 1. Workflow Overview

This workflow, titled **"Personal Budget & Expense Tracker with Google Sheets and Alerts MCP"**, provides a comprehensive system for managing personal finances by tracking expenses, incomes, and budgets via Google Sheets. It is designed for users who want to automate the recording, updating, and querying of their financial transactions and budgets. The workflow integrates with Google Sheets to store data and uses an AI-driven interface (via MCP Server Trigger) to dynamically process user requests.

The workflow logic is grouped into these main functional blocks:

- **1.1 Input Reception and Routing**: Receives input requests via the MCP Server Trigger node, parses commands, and routes them through a Switch node to appropriate handling subflows.
  
- **1.2 Transaction Management**: Handles listing, adding, updating, and removing transactions in Google Sheets, including generating unique IDs and validation.

- **1.3 Budget Management**: Manages adding, updating, retrieving, and removing monthly budgets for categories, with checks for budget existence and warnings if expenses exceed the budget.

- **1.4 Budget and Expense Validation**: After transaction changes, calculates monthly expenses versus budgets to generate actionable tasks and alerts.

- **1.5 Utilities and Helpers**: Contains nodes for filtering data by date, mapping AI inputs, and preparing execution data for routing.

- **1.6 Documentation and Instructions**: Sticky Notes provide guidance for setup and usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Routing

**Overview:**  
This block receives external requests via the MCP Server Trigger node, extracts the function and payload, and routes requests to the respective processing flows using a Switch node.

**Nodes Involved:**  
- MCP Server Trigger  
- Google Sheets Filter Workflow  
- Execution Data  
- Switch

**Node Details:**

- **MCP Server Trigger**  
  - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
  - Role: Entry point for AI-driven requests, exposes a webhook at path `/personal-expense`  
  - Configuration: Webhook path set to `"personal-expense"`  
  - Input: HTTP requests with JSON payload specifying function and parameters  
  - Output: Parsed JSON with `function` and `payload` fields  
  - Edge Cases: Missing or malformed payloads may cause routing failures  

- **Google Sheets Filter Workflow**  
  - Type: `n8n-nodes-base.executeWorkflowTrigger`  
  - Role: Passes on workflow inputs for filtering and further processing  
  - Configuration: Accepts two inputs: `function` (string) and `payload` (object)  
  - Input: Outputs from MCP Server Trigger  
  - Output: Prepares data for the Execution Data node  

- **Execution Data**  
  - Type: `n8n-nodes-base.executionData`  
  - Role: Stores function and payload from the previous node for routing  
  - Configuration: Saves `function` and stringified `payload`  
  - Input: Output from Google Sheets Filter Workflow  
  - Output: Feeds into the Switch node  

- **Switch**  
  - Type: `n8n-nodes-base.switch`  
  - Role: Routes requests based on the `function` field to appropriate subflows  
  - Configuration: Multiple cases for functions such as `list_transaction`, `add_transaction`, `update_transaction`, `remove_transaction`, `add_budget`, `update_budget`, `remove_budget`, `get_budget`  
  - Input: Execution Data node output  
  - Output: Triggers respective subflows for each function  
  - Edge Cases: Unrecognized function strings cause no output and workflow termination  

---

#### 2.2 Transaction Management

**Overview:**  
This block manages all transaction-related operations including listing transactions within date ranges, adding new transactions with unique IDs, updating existing transactions, and removing transactions by ID.

**Nodes Involved:**  
- Get list transactions (ToolWorkflow)  
- get all transactions (Google Sheets)  
- get all transactions1 (Google Sheets)  
- Filter  
- get by id (Google Sheets)  
- remove transaction (Google Sheets)  
- Crypto (Crypto node for ID generation)  
- add transaction (Google Sheets)  
- Update transaction (Google Sheets)  
- Add Transaction (ToolWorkflow)  
- Update Transaction (ToolWorkflow)  
- Remove Transaction (ToolWorkflow)  
- If  
- If1  
- If2  
- Various Code nodes for business logic  

**Node Details:**

- **Get list transactions**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Tool node that executes the same workflow internally to retrieve transactions filtered by dateStart and dateEnd  
  - Configuration: Calls with function `list_transaction` and payload containing date range from AI inputs  
  - Input: Triggered by MCP Server Trigger via Switch  
  - Output: Returns transaction list JSON  

- **get all transactions & get all transactions1**  
  - Type: `n8n-nodes-base.googleSheets`  
  - Role: Fetches all transactions from the "transactions" sheet filtered by month (due to Google Sheets limitations in filtering by date range)  
  - Configuration: Filters by `month` column using substring of dateStart or dateEnd  
  - Input: Conditional on date range spanning one or multiple months  
  - Output: Raw transaction data for further filtering  

- **Filter**  
  - Type: `n8n-nodes-base.filter`  
  - Role: Applies date range filtering on transactions fetched from Sheets, ensuring dates fall between dateStart and dateEnd  
  - Configuration: DateTime comparison filters for >= dateStart and <= dateEnd  
  - Input: Transactions data from get all transactions nodes  
  - Output: Filtered transaction list  

- **get by id**  
  - Type: `n8n-nodes-base.googleSheets`  
  - Role: Retrieves a transaction row by unique `id` column, necessary for update or delete operations  
  - Configuration: Filters by `id` column with lookup value from AI input  
  - Input: From Switch on remove_transaction or update_transaction commands  
  - Output: Transaction row data including row number for deletion or update  

- **remove transaction**  
  - Type: `n8n-nodes-base.googleSheets`  
  - Role: Deletes a transaction row from the sheet by row index  
  - Configuration: Uses `startIndex` from the row number of the transaction to remove  
  - Input: Output of get by id node  
  - Output: Confirmation of deletion  

- **Crypto**  
  - Type: `n8n-nodes-base.crypto`  
  - Role: Generates a unique ID string (`id`) for new transactions  
  - Configuration: Generate action with data property name set to `id`  
  - Input: From Switch on add_transaction command  
  - Output: Generated ID used for new transaction append  

- **add transaction**  
  - Type: `n8n-nodes-base.googleSheets`  
  - Role: Appends a new transaction row to the "transactions" sheet with all transaction fields  
  - Configuration: Maps fields from payload including `id`, `date`, `type`, `month` (derived), `amount`, `purpose`, and `category`  
  - Input: From Crypto node output (with generated id)  
  - Output: Added transaction confirmation  

- **Update transaction**  
  - Type: `n8n-nodes-base.googleSheets`  
  - Role: Updates an existing transaction row identified by `id` with new fields  
  - Configuration: Matches by `id` and updates columns accordingly  
  - Input: From Switch on update_transaction command  
  - Output: Updated transaction confirmation  

- **Add Transaction, Update Transaction, Remove Transaction (ToolWorkflow nodes)**  
  - Type: `@n8n/n8n-nodes-langchain.toolWorkflow`  
  - Role: Subworkflow invocations that wrap complex logic and interact with AI inputs  
  - Configuration: Accept AI inputs for transaction data; invoke main workflow for execution  
  - Input: Triggered by MCP Server Trigger and routed via Switch  
  - Output: Results and status for the AI interface  

- **If, If1, If2**  
  - Type: Conditional nodes  
  - Role: Branch logic based on transaction type or date range comparisons  
  - Configuration: E.g., checking if transaction type is "expense" or if dateStart and dateEnd belong to the same month  
  - Input/Output: Control flow routing  

- **Code nodes**  
  - Role: Calculate totals, prepare response objects, generate tasks or warnings related to budget checks after transaction changes  
  - Configuration: JavaScript snippets computing total expenses, comparing budgets, and preparing AI tasks  

**Edge Cases & Potential Failures:**  
- Invalid or missing transaction IDs for update or delete cause failure.  
- Google Sheets API limits or auth errors.  
- Date filtering complexity due to Sheets limitations may cause incomplete results if not configured properly.  
- Unique ID collisions are unlikely but possible if Crypto node fails.  
- Expression errors if AI inputs are malformed or missing fields.  

---

#### 2.3 Budget Management

**Overview:**  
This block manages monthly budgets per category including adding new budgets, updating existing ones, removing budgets, and retrieving budgets for validation or reporting.

**Nodes Involved:**  
- Add Budget (ToolWorkflow)  
- Add budget (Google Sheets)  
- Update Budget Tool (ToolWorkflow)  
- Update Budget (Google Sheets)  
- Remove Budget (ToolWorkflow)  
- Remove budget (Google Sheets)  
- Get Budget (ToolWorkflow)  
- Get Existing Budget, Get Existing Budget1, Get Existing Budget2, Get Existing Budget3 (Google Sheets)  
- budget not found? / budget not found?1 / budget not found?2 / budget not found?3 / budget not found?4 (If nodes)  
- Code6, Code7, Code8 (Code nodes for logic)  

**Node Details:**

- **Add Budget (ToolWorkflow)**  
  - Invokes the workflow to add a new monthly budget with parameters: month (YYYY-MM), category, amount  

- **Add budget (Google Sheets)**  
  - Appends a new row to the "budgets" sheet with month, category, and budget amount  
  - No matching columns since it's an append operation  

- **Update Budget Tool (ToolWorkflow)**  
  - Invokes the workflow to update an existing monthly budget  

- **Update Budget (Google Sheets)**  
  - Updates the budget row in the "budgets" sheet matching by `row_number` (row index)  
  - Updates month, category, and budget amount  

- **Remove Budget (ToolWorkflow)**  
  - Invokes the workflow to remove a monthly budget entry  

- **Remove budget (Google Sheets)**  
  - Deletes a row in the "budgets" sheet by `row_number`  

- **Get Budget (ToolWorkflow)**  
  - Retrieves budget for a given month and category  

- **Get Existing Budget nodes (Google Sheets)**  
  - Filter budgets sheet by month and category to find existing budgets  
  - Always output data to allow conditional checks  

- **budget not found? If nodes**  
  - Check if the filtered budget data is empty, indicating budget does not exist  
  - Branch to add new budget or update existing one accordingly  

- **Code6, Code7, Code8**  
  - Prepare tasks for AI client depending on whether budget exists or not  
  - E.g., "Budget already exists. Ask user to update it instead.", "Budget not found. Ask user to add budget instead.", or "Budget not found, nothing to remove."  

**Edge Cases & Potential Failures:**  
- Attempting to add a budget that already exists leads to suggestion to update instead.  
- Trying to update or remove a budget that does not exist triggers prompts to add or informs no action is possible.  
- Google Sheets API permissions and limits can cause failures.  
- Missing or malformed AI input fields cause errors in budget identification.  

---

#### 2.4 Budget and Expense Validation

**Overview:**  
This block calculates total monthly expenses per category after adding or updating transactions and compares them with budgets to generate warnings or next steps for the user.

**Nodes Involved:**  
- Get existing budget (Google Sheets)  
- Get Monthly Expenses, Get Monthly Expenses1 (Google Sheets)  
- Code, Code1, Code4 (Code nodes)  
- budget not found?, budget not found?1 (If nodes)  

**Node Details:**

- **Get existing budget**  
  - Retrieves budget for the month and category related to the transaction  

- **Get Monthly Expenses / Get Monthly Expenses1**  
  - Retrieves all transactions filtered by month and category to sum expenses  

- **Code nodes**  
  - Calculate total expenses from retrieved transactions  
  - Compute remaining budget by subtracting expenses from the budget limit  
  - Set task for AI client, e.g., warn user if budget exceeded, or prompt to add budget if missing  

- **If nodes (budget not found?)**  
  - Branch flow depending on whether budget data was found  

**Edge Cases & Potential Failures:**  
- Accurate summing depends on correct filtering by month and category.  
- If budgets are missing, system prompts user to add them.  
- Edge cases where expenses exactly match budget are handled gracefully.  
- Potential issues if transaction amounts are negative or malformed.  

---

#### 2.5 Utilities and Helpers

**Overview:**  
Nodes that assist with data processing, including filtering transactions by date, extracting execution data, and generating unique IDs.

**Nodes Involved:**  
- Filter  
- Execution Data  
- Crypto  
- Various Code nodes  

**Node Details:**

- **Filter**  
  - Filters transaction records by date range since Google Sheets lacks native date range filtering on columns  

- **Execution Data**  
  - Prepares key fields for routing in the main workflow  

- **Crypto**  
  - Generates unique transaction IDs to ensure data integrity  

- **Code nodes**  
  - Contains reusable snippets for data calculations and response formatting  

---

#### 2.6 Documentation and Instructions (Sticky Notes)

**Overview:**  
Sticky Notes nodes provide setup instructions, usage tips, and descriptions of various workflow parts.

**Sticky Notes Content Highlights:**

- Instructions for Google Sheets OAuth2 setup  
- Link to Google Sheets template: [Google Sheets Template](https://docs.google.com/spreadsheets/d/1XzoYEZflj1Rdo2MVKosRqXHSjkazE5vuwYWHLybX4b0/edit?usp=sharing)  
- Guidance on sharing the sheet and setting correct sheet names (`transactions` and `budgets`)  
- Explanation of main tools and how to use them with AI client  
- Descriptions for add/update/remove transaction and budget flows  
- Tips for adding budgets across multiple months via AI prompts  

---

### 3. Summary Table

| Node Name               | Node Type                                 | Functional Role                                 | Input Node(s)                     | Output Node(s)                              | Sticky Note                                                                                      |
|-------------------------|------------------------------------------|------------------------------------------------|----------------------------------|---------------------------------------------|-------------------------------------------------------------------------------------------------|
| MCP Server Trigger      | @n8n/n8n-nodes-langchain.mcpTrigger      | Entry point webhook for AI requests             | -                                | Google Sheets Filter Workflow                | # MCP Personal Expense template description                                                     |
| Google Sheets Filter Workflow | n8n-nodes-base.executeWorkflowTrigger | Pass workflow inputs for filtering              | MCP Server Trigger               | Execution Data                              |                                                                                                 |
| Execution Data          | n8n-nodes-base.executionData              | Extract function and payload for routing        | Google Sheets Filter Workflow   | Switch                                      |                                                                                                 |
| Switch                  | n8n-nodes-base.switch                      | Route requests based on function field           | Execution Data                  | Multiple subflows (e.g., Add Transaction)  |                                                                                                 |
| Get list transactions   | @n8n/n8n-nodes-langchain.toolWorkflow     | Retrieve transactions list                        | MCP Server Trigger (via Switch) | -                                           |                                                                                                 |
| get all transactions    | n8n-nodes-base.googleSheets                | Get all transactions for month                    | If2                            | Filter                                      | ### Get all transaction, date range filtering explanation                                       |
| get all transactions1   | n8n-nodes-base.googleSheets                | Get all transactions for month                    | If2                            | Filter                                      | ### Get all transaction, date range filtering explanation                                       |
| Filter                  | n8n-nodes-base.filter                      | Filter transactions by date range                 | get all transactions, get all transactions1 | -                                      |                                                                                                 |
| get by id               | n8n-nodes-base.googleSheets                | Retrieve transaction row by ID                     | Switch                        | remove transaction                          | ### Remove transaction explanation: row number needed for deletion                             |
| remove transaction      | n8n-nodes-base.googleSheets                | Delete transaction row by index                    | get by id                     | -                                           | ### Remove transaction explanation: row number needed for deletion                             |
| Crypto                  | n8n-nodes-base.crypto                      | Generate unique transaction ID                    | Switch (add_transaction)       | add transaction                            |                                                                                                 |
| add transaction         | n8n-nodes-base.googleSheets                | Append new transaction to sheet                    | Crypto                        | If                                         | ### Add transaction explanation: checks budgets and suggests next steps                         |
| Update transaction      | n8n-nodes-base.googleSheets                | Update transaction by ID                           | Switch (update_transaction)    | If1                                        | ### Update transaction explanation: budget checks duplicated due to n8n limitations            |
| Add Transaction         | @n8n/n8n-nodes-langchain.toolWorkflow     | Tool wrapper for adding transaction                | MCP Server Trigger (via Switch)| -                                           |                                                                                                 |
| Update Transaction      | @n8n/n8n-nodes-langchain.toolWorkflow     | Tool wrapper for updating transaction              | MCP Server Trigger (via Switch)| -                                           |                                                                                                 |
| Remove Transaction      | @n8n/n8n-nodes-langchain.toolWorkflow     | Tool wrapper for removing transaction              | MCP Server Trigger (via Switch)| -                                           |                                                                                                 |
| If                      | n8n-nodes-base.if                          | Check if transaction type is "expense"            | add transaction               | Get existing budget, Code2                   |                                                                                                 |
| If1                     | n8n-nodes-base.if                          | Check if updated transaction type is "expense"    | Update transaction            | Get existing budget2, Code5                  |                                                                                                 |
| If2                     | n8n-nodes-base.if                          | Check if dateStart and dateEnd are same month      | Switch                       | get all transactions1 or get all transactions|                                                                                                 |
| Code                    | n8n-nodes-base.code                        | Calculate expenses vs. budget, generate tasks     | Get Monthly Expenses           | -                                           |                                                                                                 |
| Code1                   | n8n-nodes-base.code                        | Calculate total expenses and budget remaining     | Get Monthly Expenses           | -                                           |                                                                                                 |
| Code2                   | n8n-nodes-base.code                        | Generate task after adding transaction            | budget not found?              | -                                           |                                                                                                 |
| Code3                   | n8n-nodes-base.code                        | Task response when budget not found after update  | budget not found?1             | -                                           |                                                                                                 |
| Code4                   | n8n-nodes-base.code                        | Calculate budget and expenses after update        | Get Monthly Expenses1          | -                                           |                                                                                                 |
| Code5                   | n8n-nodes-base.code                        | Congratulate user for new income                    | If1                          | -                                           |                                                                                                 |
| Code6                   | n8n-nodes-base.code                        | Task when budget already exists                      | budget not found?2             | -                                           |                                                                                                 |
| Code7                   | n8n-nodes-base.code                        | Task when budget not found, prompting for add      | budget not found?3             | -                                           |                                                                                                 |
| Code8                   | n8n-nodes-base.code                        | Task when budget not found for removal              | budget not found?4             | -                                           |                                                                                                 |
| Get existing budget     | n8n-nodes-base.googleSheets                | Retrieve budget for month and category              | If                          | budget not found?                            |                                                                                                 |
| Get existing budget2    | n8n-nodes-base.googleSheets                | Retrieve budget for month and category              | If1                         | budget not found?1                           |                                                                                                 |
| Get Existing Budget     | n8n-nodes-base.googleSheets                | Retrieve budget for various budget operations       | Switch                      | budget not found?2                           |                                                                                                 |
| Get Existing Budget1    | n8n-nodes-base.googleSheets                | Retrieve budget for update operation                 | Switch                      | budget not found?3                           |                                                                                                 |
| Get Existing Budget2    | n8n-nodes-base.googleSheets                | Retrieve budget for remove operation                 | Switch                      | budget not found?4                           |                                                                                                 |
| Get Existing Budget3    | n8n-nodes-base.googleSheets                | Retrieve budget for remove operation                 | Switch                      | budget not found?4                           |                                                                                                 |
| Get Monthly Expenses    | n8n-nodes-base.googleSheets                | Sum expenses by month and category                    | Code                         | Code1                                       |                                                                                                 |
| Get Monthly Expenses1   | n8n-nodes-base.googleSheets                | Sum expenses by month and category (update flow)     | Code3                        | Code4                                       |                                                                                                 |
| Add budget              | n8n-nodes-base.googleSheets                | Append new budget entry                               | budget not found?2            | Code6                                       | ### Add monthly budget pro tip: multi-month budgets possible                                   |
| Update Budget           | n8n-nodes-base.googleSheets                | Update existing budget row                            | budget not found?3            | Code7                                       | ### Update monthly budget                                                                        |
| Remove budget           | n8n-nodes-base.googleSheets                | Delete budget row by index                            | budget not found?4            | Code8                                       | ### Remove budget                                                                              |
| Add Budget              | @n8n/n8n-nodes-langchain.toolWorkflow     | Tool wrapper for adding budget                        | MCP Server Trigger (via Switch)| -                                           |                                                                                                 |
| Update Budget Tool      | @n8n/n8n-nodes-langchain.toolWorkflow     | Tool wrapper for updating budget                      | MCP Server Trigger (via Switch)| -                                           |                                                                                                 |
| Remove Budget           | @n8n/n8n-nodes-langchain.toolWorkflow     | Tool wrapper for removing budget                      | MCP Server Trigger (via Switch)| -                                           |                                                                                                 |
| Get Budget              | @n8n/n8n-nodes-langchain.toolWorkflow     | Tool wrapper for getting budget                        | MCP Server Trigger (via Switch)| -                                           |                                                                                                 |
| Sticky Note             | n8n-nodes-base.stickyNote                  | Documentation and setup instructions                  | -                            | -                                           | MCP Main tools instructions, Google Sheets OAuth and template setup instructions                |
| Sticky Note1            | n8n-nodes-base.stickyNote                  | Explanation of transaction list retrieval             | -                            | -                                           | Get all transaction explanation                                                                |
| Sticky Note2            | n8n-nodes-base.stickyNote                  | Explanation of remove transaction process             | -                            | -                                           | Remove transaction explanation                                                                 |
| Sticky Note3            | n8n-nodes-base.stickyNote                  | Explanation of add transaction process                | -                            | -                                           | Add transaction explanation                                                                    |
| Sticky Note4            | n8n-nodes-base.stickyNote                  | Explanation of update transaction process             | -                            | -                                           | Update transaction explanation                                                                 |
| Sticky Note5            | n8n-nodes-base.stickyNote                  | Explanation of add monthly budget                      | -                            | -                                           | Add monthly budget pro tip with multi-month example                                            |
| Sticky Note6            | n8n-nodes-base.stickyNote                  | Explanation of update monthly budget                   | -                            | -                                           | Update monthly budget explanation                                                              |
| Sticky Note7            | n8n-nodes-base.stickyNote                  | Explanation of remove budget                            | -                            | -                                           | Remove budget explanation                                                                      |
| Sticky Note8            | n8n-nodes-base.stickyNote                  | Explanation of get existing budget                      | -                            | -                                           | Get existing budget explanation                                                                |
| Sticky Note9            | n8n-nodes-base.stickyNote                  | Workflow title and summary                              | -                            | -                                           | MCP Personal Expense title and template description                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Server Trigger node**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Configure webhook path as `/personal-expense`  
   - This node will receive all AI client requests  

2. **Create the Google Sheets Filter Workflow node**  
   - Type: `n8n-nodes-base.executeWorkflowTrigger`  
   - Configure inputs: `function` (string), `payload` (object)  
   - Connect MCP Server Trigger output to this node  

3. **Create Execution Data node**  
   - Type: `n8n-nodes-base.executionData`  
   - Set key-value pairs: `function` and `payload` extracted from Google Sheets Filter Workflow node  
   - Connect Google Sheets Filter Workflow output to this node  

4. **Create Switch node for routing**  
   - Type: `n8n-nodes-base.switch`  
   - Setup rules for each function string (e.g., `list_transaction`, `add_transaction`, etc.)  
   - Connect Execution Data output to Switch node  

5. **Transaction Listing**  
   - Create `Get list transactions` toolWorkflow node  
   - Configure to call current workflow with function `list_transaction` and dateStart/dateEnd from AI inputs  
   - Connect Switch `list_transaction` output to this node  

6. **Get all transactions nodes**  
   - Add two Google Sheets nodes: `get all transactions` and `get all transactions1`  
   - Configure each to filter sheet `transactions` by `month` column using the dateStart and dateEnd substring (YYYY-MM)  
   - Connect conditional `If2` node to select which node to execute based on whether start and end dates are in the same month  
   - Connect both Google Sheets nodes to a Filter node that filters by exact date range  

7. **Add transaction flow**  
   - Create Crypto node to generate unique `id`  
   - Create Google Sheets node `add transaction` to append to `transactions` sheet; map fields from payload, compute `month` from date substring  
   - Create an If node to check if transaction `type` is `expense`  
   - Create Google Sheets node `Get existing budget` to fetch budget for month/category  
   - Create Code node to calculate expenses and evaluate budget status  
   - Connect nodes accordingly  
   - Wrap the above in an `Add Transaction` toolWorkflow node for AI integration  

8. **Update transaction flow**  
   - Create Google Sheets node `Update transaction` to update by `id` in `transactions` sheet  
   - Create If node to check if updated transaction is `expense`  
   - Create Google Sheets node `Get existing budget2` and `Get Monthly Expenses1` for expense calculation  
   - Create Code nodes for budget evaluation and task generation  
   - Wrap in `Update Transaction` toolWorkflow node  

9. **Remove transaction flow**  
   - Create Google Sheets node `get by id` to find transaction row by ID  
   - Create Google Sheets node `remove transaction` to delete row by row number  
   - Wrap in `Remove Transaction` toolWorkflow node  

10. **Budget management flows**  
    - Create Google Sheets nodes `Add budget`, `Update Budget`, `Remove budget`, and multiple `Get Existing Budget` nodes filtered by month/category  
    - Create If nodes to check if budget exists or not  
    - Create Code nodes to generate appropriate AI client tasks (e.g., prompt to add or update budget)  
    - Wrap each budget operation in corresponding toolWorkflow nodes (`Add Budget`, `Update Budget Tool`, `Remove Budget`, `Get Budget`)  

11. **Connect all toolWorkflow nodes outputs back to MCP Server Trigger as entry point**  

12. **Add Sticky Notes**  
    - Create Sticky Notes with instructions on Google Sheets OAuth setup, sheet template link, usage tips, and explanations for each main operation  

13. **Configure credentials**  
    - Google Sheets OAuth2 credentials must be created and assigned to all Google Sheets nodes  
    - Ensure access to the shared Google Sheets template is granted and sheets named `transactions` and `budgets` exist  
    - No external credentials needed for the Crypto or MCP nodes beyond standard n8n setup  

14. **Testing**  
    - Test each function with example AI inputs for adding, listing, updating, and removing transactions and budgets  
    - Validate that date filters work properly and budget warnings appear as expected  

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Google Sheets OAuth2 setup required, see documentation: https://docs.n8n.io/integrations/builtin/credentials/google/ | Credential setup for Google Sheets integration                                                      |
| Google Sheets template for transactions and budgets: https://docs.google.com/spreadsheets/d/1XzoYEZflj1Rdo2MVKosRqXHSjkazE5vuwYWHLybX4b0/edit?usp=sharing | Base spreadsheet template to copy and use                                                           |
| Share the spreadsheet with "anyone with the link" for read/write access                                            | Required for n8n Google Sheets nodes to access data                                                 |
| AI client can add budgets for multiple months by prompts like: "Add budget for fashion for this month until the end of the year, $500 per month" | Example of enhanced AI usage                                                                          |
| Due to Google Sheets filtering limitations, date range filtering is done by month substring and post-filtering      | Important for understanding transaction list retrieval complexity                                   |
| Budget checking logic duplicated for add and update transactions due to n8n limitations on node execution order    | Workflow implementation note and potential for future improvements                                  |

---

This documentation serves as a comprehensive reference to understand, reproduce, and maintain the "Personal Budget & Expense Tracker with Google Sheets and Alerts MCP" workflow. It ensures that users or automated systems can work with the workflow effectively including anticipating common errors and integration points.
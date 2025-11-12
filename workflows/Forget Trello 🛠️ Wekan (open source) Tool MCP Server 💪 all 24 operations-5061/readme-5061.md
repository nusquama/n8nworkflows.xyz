Forget Trello 🛠️ Wekan (open source) Tool MCP Server 💪 all 24 operations

https://n8nworkflows.xyz/workflows/forget-trello-----wekan--open-source--tool-mcp-server----all-24-operations-5061


# Forget Trello 🛠️ Wekan (open source) Tool MCP Server 💪 all 24 operations

### 1. Workflow Overview

This workflow, titled **"Forget Trello 🛠️ Wekan (open source) Tool MCP Server 💪 all 24 operations"**, is designed to provide comprehensive integration and automation capabilities for the open-source Kanban tool **Wekan** via n8n. It aims to expose all 24 core operations related to Wekan’s management of boards, cards, comments, checklists, checklist items, and lists. 

The workflow is structured as an API-triggered automation, where a central **MCP Server trigger node** accepts incoming requests and dispatches them to the appropriate Wekan operation node, effectively functioning as a complete API wrapper / management layer for Wekan.

**Logical blocks** are organized by functional entity types with clear node-to-node dependencies flowing from the central MCP Server trigger:

- **1.1 Trigger and Entry Point**
  - Receives all requests via the MCP Server node, initiating the appropriate Wekan operation.

- **1.2 Board Operations**
  - Create, Delete, Get single, and Get many boards.

- **1.3 Card Operations**
  - Create, Delete, Get single, Get many, Update cards.

- **1.4 Card Comment Operations**
  - Create, Delete, Get single, Get many comments on cards.

- **1.5 Checklist Operations**
  - Create, Delete, Get single, Get many checklists.

- **1.6 Checklist Item Operations**
  - Delete, Get single, Update checklist items.

- **1.7 List Operations**
  - Create, Delete, Get single, Get many lists.

Each operation is implemented as a dedicated **Wekan Tool** node, connected directly to the MCP Server trigger node. Sticky Notes are used as visual separators and placeholders for documentation or comments.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Trigger and Entry Point

- **Overview:**  
  This block provides the entry point for all incoming requests. The **MCP Server trigger** node listens for API calls, identifies the requested operation, and routes execution to the corresponding Wekan operation node.

- **Nodes Involved:**  
  - Wekan Tool MCP Server

- **Node Details:**  
  - **Wekan Tool MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: API trigger node designed to receive and dispatch commands for Wekan operations.  
    - Configuration: Uses a webhook with ID `604db406-ce1c-43ce-a519-c50f2d577185` to listen for HTTP requests.  
    - Inputs: External API calls.  
    - Outputs: Routes to all Wekan Tool nodes representing operations.  
    - Edge Cases:  
      - Webhook authentication failures or unauthorized access.  
      - Invalid or unsupported operation requests could cause routing failures.  
      - Timeout if downstream nodes take too long.  
    - Notes: Central dispatch node is crucial for workflow orchestration.  

---

#### 1.2 Board Operations

- **Overview:**  
  This block manages Wekan boards, supporting creation, deletion, retrieval of single boards, and retrieval of multiple boards.

- **Nodes Involved:**  
  - Create a board  
  - Delete a board  
  - Get a board  
  - Get many boards  
  - Sticky Note 1 (empty content, likely a placeholder)

- **Node Details:**  
  Each node is of type `n8n-nodes-base.wekanTool` with operation configured for specific board tasks.

  - **Create a board**  
    - Action: Creates a new Wekan board.  
    - Inputs: From MCP Server trigger.  
    - Outputs: API response for board creation.  
    - Possible failures: Validation errors if board parameters are missing or invalid.  

  - **Delete a board**  
    - Action: Deletes an existing Wekan board by ID.  
    - Inputs: Board identification from trigger.  
    - Failures: Errors if board ID is not found or permission denied.  

  - **Get a board**  
    - Action: Retrieves a specific board’s details.  
    - Inputs: Board ID from trigger.  
    - Failures: Board not found or access issues.  

  - **Get many boards**  
    - Action: Retrieves a list of boards, optionally filtered or paginated.  
    - Inputs: Parameters from trigger.  
    - Failures: API errors or empty results.  

---

#### 1.3 Card Operations

- **Overview:**  
  Handles card-level operations in Wekan including creation, deletion, retrieval (single and multiple), and updates.

- **Nodes Involved:**  
  - Create a card  
  - Delete a card  
  - Get a card  
  - Get many cards  
  - Update a card  
  - Sticky Note 2 (empty content)

- **Node Details:**  
  All nodes are `wekanTool` type with operation configured for card management.

  - **Create a card**  
    - Creates a new card on a specified board/list.  
    - Edge cases: Missing list or board IDs, invalid card data.  

  - **Delete a card**  
    - Deletes a card by card ID.  
    - Failures: Card ID not found or permission issues.  

  - **Get a card**  
    - Retrieves details of a single card.  
    - Failures: Card not found.  

  - **Get many cards**  
    - Retrieves multiple cards, possibly filtered by board or list.  
    - Failures: Empty results, API errors.  

  - **Update a card**  
    - Updates card attributes such as title, description, due dates, etc.  
    - Failures: Invalid update data, concurrency conflicts.  

---

#### 1.4 Card Comment Operations

- **Overview:**  
  Supports managing comments on cards: create, delete, get single comment, get many comments.

- **Nodes Involved:**  
  - Create a comment on a card  
  - Delete a comment from a card  
  - Get a card comment  
  - Get many card comments  
  - Sticky Note 3 (empty content)

- **Node Details:**  
  - **Create a comment on a card**  
    - Adds a comment to a specified card.  
    - Failures: Invalid card ID, empty comment text.  

  - **Delete a comment from a card**  
    - Removes a comment by ID.  
    - Failures: Comment not found or permission issues.  

  - **Get a card comment**  
    - Retrieves a single comment’s details.  
    - Failures: Comment not found.  

  - **Get many card comments**  
    - Lists comments on a card.  
    - Failures: Empty result or access errors.  

---

#### 1.5 Checklist Operations

- **Overview:**  
  Manages checklists on cards: create, delete, retrieve single and multiple checklists.

- **Nodes Involved:**  
  - Create a checklist  
  - Delete a checklist  
  - Get a checklist  
  - Get many checklists  
  - Sticky Note 4 (empty content)

- **Node Details:**  
  Standard Wekan checklist operations with typical API parameters.

---

#### 1.6 Checklist Item Operations

- **Overview:**  
  Operations for individual checklist items: delete, get, update.

- **Nodes Involved:**  
  - Delete a checklist item  
  - Get a checklist item  
  - Update a checklist item  
  - Sticky Note 5 (empty content)

- **Node Details:**  
  Allows fine-grained control over checklist items within checklists.

---

#### 1.7 List Operations

- **Overview:**  
  Covers creation, deletion, and retrieval of lists on boards.

- **Nodes Involved:**  
  - Create a list  
  - Delete a list  
  - Get a list  
  - Get many lists  
  - Sticky Note 6 (empty content)

- **Node Details:**  
  Manages Wekan lists, essential for organizing cards within boards.

---

### 3. Summary Table

| Node Name                    | Node Type                         | Functional Role                       | Input Node(s)             | Output Node(s)                | Sticky Note                      |
|------------------------------|----------------------------------|-------------------------------------|---------------------------|------------------------------|---------------------------------|
| Workflow Overview 0           | Sticky Note                      | Documentation placeholder           |                           |                              |                                 |
| Wekan Tool MCP Server         | MCP Trigger                     | Entry point, dispatch to operations | External HTTP request      | All Wekan Tool operation nodes|                                 |
| Create a board                | Wekan Tool                      | Create a new board                  | Wekan Tool MCP Server      |                              |                                 |
| Delete a board                | Wekan Tool                      | Delete an existing board            | Wekan Tool MCP Server      |                              |                                 |
| Get a board                  | Wekan Tool                      | Retrieve a single board             | Wekan Tool MCP Server      |                              |                                 |
| Get many boards              | Wekan Tool                      | Retrieve multiple boards            | Wekan Tool MCP Server      |                              |                                 |
| Sticky Note 1                | Sticky Note                     | Visual/documentation placeholder    |                           |                              |                                 |
| Create a card                | Wekan Tool                      | Create a new card                   | Wekan Tool MCP Server      |                              |                                 |
| Delete a card                | Wekan Tool                      | Delete a card                      | Wekan Tool MCP Server      |                              |                                 |
| Get a card                  | Wekan Tool                      | Get a single card                   | Wekan Tool MCP Server      |                              |                                 |
| Get many cards              | Wekan Tool                      | Get multiple cards                  | Wekan Tool MCP Server      |                              |                                 |
| Update a card               | Wekan Tool                      | Update card details                 | Wekan Tool MCP Server      |                              |                                 |
| Sticky Note 2               | Sticky Note                     | Visual/documentation placeholder    |                           |                              |                                 |
| Create a comment on a card  | Wekan Tool                      | Add comment to a card               | Wekan Tool MCP Server      |                              |                                 |
| Delete a comment from a card| Wekan Tool                      | Delete a comment                   | Wekan Tool MCP Server      |                              |                                 |
| Get a card comment          | Wekan Tool                      | Get single comment                  | Wekan Tool MCP Server      |                              |                                 |
| Get many card comments      | Wekan Tool                      | Get multiple comments               | Wekan Tool MCP Server      |                              |                                 |
| Sticky Note 3               | Sticky Note                     | Visual/documentation placeholder    |                           |                              |                                 |
| Create a checklist          | Wekan Tool                      | Create a checklist on a card        | Wekan Tool MCP Server      |                              |                                 |
| Delete a checklist          | Wekan Tool                      | Delete a checklist                  | Wekan Tool MCP Server      |                              |                                 |
| Get a checklist             | Wekan Tool                      | Get a single checklist              | Wekan Tool MCP Server      |                              |                                 |
| Get many checklists         | Wekan Tool                      | Get multiple checklists             | Wekan Tool MCP Server      |                              |                                 |
| Sticky Note 4               | Sticky Note                     | Visual/documentation placeholder    |                           |                              |                                 |
| Delete a checklist item     | Wekan Tool                      | Delete a checklist item             | Wekan Tool MCP Server      |                              |                                 |
| Get a checklist item        | Wekan Tool                      | Get a single checklist item         | Wekan Tool MCP Server      |                              |                                 |
| Update a checklist item     | Wekan Tool                      | Update checklist item details       | Wekan Tool MCP Server      |                              |                                 |
| Sticky Note 5               | Sticky Note                     | Visual/documentation placeholder    |                           |                              |                                 |
| Create a list               | Wekan Tool                      | Create a list on a board            | Wekan Tool MCP Server      |                              |                                 |
| Delete a list               | Wekan Tool                      | Delete a list                      | Wekan Tool MCP Server      |                              |                                 |
| Get a list                  | Wekan Tool                      | Get a single list                   | Wekan Tool MCP Server      |                              |                                 |
| Get many lists              | Wekan Tool                      | Get multiple lists                  | Wekan Tool MCP Server      |                              |                                 |
| Sticky Note 6               | Sticky Note                     | Visual/documentation placeholder    |                           |                              |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Server Trigger Node**
   - Add a node of type `MCP Trigger` (from Langchain MCP nodes).  
   - Configure the webhook ID (use a new webhook in your instance).  
   - This node listens for incoming API requests and dispatches them.

2. **Add Wekan Tool Nodes for Board Operations**
   - Create nodes of type `Wekan Tool` for:  
     - Create a board  
     - Delete a board  
     - Get a board  
     - Get many boards  
   - For each node, set the operation parameter accordingly (e.g., "createBoard", "deleteBoard", etc.).  
   - Connect the MCP Server node’s output to each of these nodes on the `ai_tool` output stream.

3. **Add Wekan Tool Nodes for Card Operations**
   - Add nodes:  
     - Create a card  
     - Delete a card  
     - Get a card  
     - Get many cards  
     - Update a card  
   - Configure each operation accordingly.  
   - Connect MCP Server node’s output to each card operation node.

4. **Add Wekan Tool Nodes for Card Comment Operations**
   - Add nodes:  
     - Create a comment on a card  
     - Delete a comment from a card  
     - Get a card comment  
     - Get many card comments  
   - Configure operations.  
   - Connect MCP Server node output.

5. **Add Wekan Tool Nodes for Checklist Operations**
   - Add nodes:  
     - Create a checklist  
     - Delete a checklist  
     - Get a checklist  
     - Get many checklists  
   - Set operations and connect them.

6. **Add Wekan Tool Nodes for Checklist Item Operations**
   - Add nodes:  
     - Delete a checklist item  
     - Get a checklist item  
     - Update a checklist item  
   - Configure and connect to MCP Server node.

7. **Add Wekan Tool Nodes for List Operations**
   - Add nodes:  
     - Create a list  
     - Delete a list  
     - Get a list  
     - Get many lists  
   - Configure and connect accordingly.

8. **Add Sticky Notes**
   - Add Sticky Note nodes to visually group or document each block as desired.  
   - Place them near corresponding nodes.

9. **Credential Setup**
   - Configure Wekan credentials for all Wekan Tool nodes (API URL, authentication tokens).  
   - Ensure MCP Server and webhook are accessible externally or internally as required.

10. **Testing**
    - Test the webhook by sending API requests for all 24 operations.  
    - Verify responses and error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                                     |
|--------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| This workflow represents an exhaustive operational interface for Wekan’s API within n8n.          | Workflow description and node organization                                                                           |
| MCP Server trigger node is key for routing requests to 24 operations                             | Documentation on MCP Server nodes in n8n standard node library                                                       |
| Wekan is an open-source Trello alternative, enabling self-hosted Kanban boards                   | https://wekan.github.io/                                                                                              |
| Sticky Notes currently have empty content; recommended to add operational instructions or comments | Useful for team collaboration and future maintenance                                                                |
| Ensure API credentials and webhook URLs are securely stored and managed                           | n8n credential management best practices                                                                             |
| Potential error scenarios include authentication failures, invalid parameters, and network timeouts| Implement retry and error nodes around critical Wekan Tool nodes for robustness                                      |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. It strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.
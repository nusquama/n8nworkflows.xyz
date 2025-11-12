Make AI Agents Create, Get, Update Custom Fields 🛠️ ConvertKit Tool MCP Server

https://n8nworkflows.xyz/workflows/make-ai-agents-create--get--update-custom-fields-----convertkit-tool-mcp-server-5316


# Make AI Agents Create, Get, Update Custom Fields 🛠️ ConvertKit Tool MCP Server

### 1. Workflow Overview

This workflow, titled **ConvertKit Tool MCP Server**, serves as a backend automation hub designed to facilitate Create, Read, Update, and Delete (CRUD) operations on ConvertKit custom fields and related subscriber data. It is intended for use cases involving automated management of ConvertKit marketing data via AI agents, enabling seamless integration and manipulation of tags, forms, sequences, subscribers, and custom fields.

The workflow is logically divided into the following blocks based on functional roles and node dependencies:

- **1.1 MCP Trigger (Input Reception)**  
  Entry point for AI-driven commands via the MCP (Multi-Channel Platform) trigger node.

- **1.2 Custom Fields Management**  
  Nodes handling creation, retrieval, update, and deletion of ConvertKit custom fields.

- **1.3 Subscriber and Subscription Management**  
  Nodes managing subscriber addition, retrieval of subscriptions, sequences, and forms.

- **1.4 Tag Management**  
  Nodes dedicated to creating tags, retrieving tags, adding/removing tags on subscribers, and managing tag subscriptions.

Each block is activated through the single MCP trigger node which routes AI-driven requests to the appropriate ConvertKit node.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Trigger (Input Reception)

- **Overview:**  
  This block serves as the workflow’s sole entry point, receiving AI agent commands via the MCP Trigger node. It acts as the interface between the AI environment and the ConvertKit operations.

- **Nodes Involved:**  
  - `ConvertKit Tool MCP Server`

- **Node Details:**

  - **ConvertKit Tool MCP Server**  
    - **Type & Role:** MCP Trigger node, specialized for receiving AI-driven workflow triggers.  
    - **Configuration:** No additional parameters configured; uses a webhook with ID `59b954b1-9605-4270-90de-e3a32d449742` to listen for incoming requests.  
    - **Key Expressions/Variables:** None explicitly configured; acts as an event listener.  
    - **Connections:** Outputs to all ConvertKit operation nodes via `ai_tool` connections.  
    - **Version Requirements:** Requires n8n version supporting `@n8n/n8n-nodes-langchain.mcpTrigger` node type.  
    - **Potential Failures:** Webhook trigger failures, permission or authentication errors if webhook is not properly secured.  
    - **Sub-workflow:** None.

#### 1.2 Custom Fields Management

- **Overview:**  
  This block manages ConvertKit custom fields, allowing creation, retrieval (many), updating, and deletion of custom fields based on AI commands.

- **Nodes Involved:**  
  - `Create a custom field`  
  - `Get many custom fields`  
  - `Update a custom field`  
  - `Delete a custom field`  
  - `Sticky Note 1` (empty content)

- **Node Details:**

  - **Create a custom field**  
    - Type: ConvertKit Tool node  
    - Role: Creates a new custom field in ConvertKit.  
    - Configuration: Default; expects parameters for field name, type, etc. passed via input.  
    - Inputs: Connected from MCP Trigger node (`ai_tool` connection).  
    - Outputs: None further connected.  
    - Edge Cases: API rate limits, invalid field parameters, authentication errors.  

  - **Get many custom fields**  
    - Type: ConvertKit Tool node  
    - Role: Retrieves a list of existing custom fields.  
    - Configuration: Default; may include pagination parameters if necessary.  
    - Inputs: From MCP Trigger node.  
    - Outputs: None further connected.  
    - Edge Cases: Empty result sets, API pagination handling.  

  - **Update a custom field**  
    - Type: ConvertKit Tool node  
    - Role: Updates properties of a specified custom field.  
    - Configuration: Requires field identifier and update data.  
    - Inputs: From MCP Trigger node.  
    - Outputs: None further connected.  
    - Edge Cases: Invalid field ID, permission issues, concurrent update conflicts.  

  - **Delete a custom field**  
    - Type: ConvertKit Tool node  
    - Role: Deletes a specified custom field from ConvertKit.  
    - Configuration: Requires field ID.  
    - Inputs: From MCP Trigger node.  
    - Outputs: None further connected.  
    - Edge Cases: Deletion of non-existent fields, permission errors, cascading effects on subscribers.  

  - **Sticky Note 1**  
    - No content; likely placeholder or visual separator.

#### 1.3 Subscriber and Subscription Management

- **Overview:**  
  This block manages subscriber data and their subscriptions to forms and sequences within ConvertKit.

- **Nodes Involved:**  
  - `Add a subscriber` (two nodes with same name but different positions)  
  - `Get many forms`  
  - `Get all subscriptions`  
  - `Get many sequences`  
  - `Get all subscriptions to a sequence`  
  - `Sticky Note 2` (empty content)  
  - `Sticky Note 3` (empty content)

- **Node Details:**

  - **Add a subscriber** (two instances)  
    - Type: ConvertKit Tool node  
    - Role: Adds a new subscriber to a form or sequence.  
    - Configuration: Parameters like subscriber email, form ID, sequence ID expected.  
    - Inputs: From MCP Trigger node.  
    - Outputs: None further connected.  
    - Edge Cases: Duplicate subscriber handling, invalid email format, API failures.  

  - **Get many forms**  
    - Type: ConvertKit Tool node  
    - Role: Retrieves a list of forms available in the account.  
    - Configuration: Supports pagination if needed.  
    - Inputs: From MCP Trigger node.  
    - Outputs: None further connected.  
    - Edge Cases: Empty form lists, API limits.  

  - **Get all subscriptions**  
    - Type: ConvertKit Tool node  
    - Role: Retrieves all subscriber subscriptions across forms or sequences.  
    - Configuration: May support filters or pagination.  
    - Inputs: From MCP Trigger node.  
    - Outputs: None further connected.  
    - Edge Cases: Large data volume, pagination handling.  

  - **Get many sequences**  
    - Type: ConvertKit Tool node  
    - Role: Retrieves available sequences for automation.  
    - Inputs: From MCP Trigger node.  
    - Outputs: None further connected.  

  - **Get all subscriptions to a sequence**  
    - Type: ConvertKit Tool node  
    - Role: Gets subscribers subscribed to a particular sequence.  
    - Inputs: From MCP Trigger node.  
    - Outputs: None further connected.  

  - **Sticky Note 2 & Sticky Note 3**  
    - Both empty content; likely placeholders or visual separators.

#### 1.4 Tag Management

- **Overview:**  
  This block facilitates tag operations including creation, retrieval, applying tags to subscribers, retrieving tag subscriptions, and removing tags from subscribers.

- **Nodes Involved:**  
  - `Create a tag`  
  - `Get many tags`  
  - `Add a tag to a subscriber`  
  - `Get many tag subscriptions`  
  - `Delete a tag from a subscriber`  
  - `Sticky Note 4` (empty content)  
  - `Sticky Note 5` (empty content)

- **Node Details:**

  - **Create a tag**  
    - Type: ConvertKit Tool node  
    - Role: Creates a new tag for subscribers.  
    - Inputs: From MCP Trigger node.  
    - Outputs: None further connected.  
    - Edge Cases: Duplicate tag names, invalid parameters.  

  - **Get many tags**  
    - Type: ConvertKit Tool node  
    - Role: Retrieves all existing tags.  
    - Inputs: From MCP Trigger node.  
    - Outputs: None further connected.  

  - **Add a tag to a subscriber**  
    - Type: ConvertKit Tool node  
    - Role: Assigns a tag to a subscriber.  
    - Inputs: From MCP Trigger node.  
    - Outputs: None further connected.  
    - Edge Cases: Subscriber or tag not found, API failures.  

  - **Get many tag subscriptions**  
    - Type: ConvertKit Tool node  
    - Role: Retrieves subscribers associated with tags.  
    - Inputs: From MCP Trigger node.  
    - Outputs: None further connected.  

  - **Delete a tag from a subscriber**  
    - Type: ConvertKit Tool node  
    - Role: Removes a tag from a subscriber.  
    - Inputs: From MCP Trigger node.  
    - Outputs: None further connected.  
    - Edge Cases: Subscriber or tag not found, permission errors.  

  - **Sticky Note 4 & Sticky Note 5**  
    - Both empty content; likely placeholders or visual separators.

---

### 3. Summary Table

| Node Name                    | Node Type                    | Functional Role                          | Input Node(s)               | Output Node(s)              | Sticky Note                        |
|------------------------------|------------------------------|----------------------------------------|-----------------------------|-----------------------------|----------------------------------|
| Workflow Overview 0           | Sticky Note                  | General workflow note (empty)           |                             |                             |                                  |
| ConvertKit Tool MCP Server    | MCP Trigger                 | Entry point receiving AI commands       |                             | Create a tag, Get many tags, etc. (all ConvertKit nodes) |                                  |
| Create a custom field         | ConvertKit Tool             | Create ConvertKit custom field           | ConvertKit Tool MCP Server   |                             |                                  |
| Delete a custom field         | ConvertKit Tool             | Delete ConvertKit custom field           | ConvertKit Tool MCP Server   |                             |                                  |
| Get many custom fields        | ConvertKit Tool             | Retrieve multiple custom fields          | ConvertKit Tool MCP Server   |                             |                                  |
| Update a custom field         | ConvertKit Tool             | Update ConvertKit custom field           | ConvertKit Tool MCP Server   |                             |                                  |
| Sticky Note 1                | Sticky Note                  | Visual separator or placeholder          |                             |                             |                                  |
| Add a subscriber (top)        | ConvertKit Tool             | Add subscriber to ConvertKit             | ConvertKit Tool MCP Server   |                             |                                  |
| Get many forms               | ConvertKit Tool             | List ConvertKit forms                     | ConvertKit Tool MCP Server   |                             |                                  |
| Get all subscriptions         | ConvertKit Tool             | Retrieve all subscriptions                | ConvertKit Tool MCP Server   |                             |                                  |
| Sticky Note 2                | Sticky Note                  | Visual separator or placeholder          |                             |                             |                                  |
| Add a subscriber (middle)     | ConvertKit Tool             | Add subscriber to ConvertKit             | ConvertKit Tool MCP Server   |                             |                                  |
| Get many sequences           | ConvertKit Tool             | List ConvertKit sequences                  | ConvertKit Tool MCP Server   |                             |                                  |
| Get all subscriptions to a sequence | ConvertKit Tool      | Retrieve subscribers to a sequence       | ConvertKit Tool MCP Server   |                             |                                  |
| Sticky Note 3                | Sticky Note                  | Visual separator or placeholder          |                             |                             |                                  |
| Create a tag                 | ConvertKit Tool             | Create ConvertKit tag                     | ConvertKit Tool MCP Server   |                             |                                  |
| Get many tags                | ConvertKit Tool             | List ConvertKit tags                      | ConvertKit Tool MCP Server   |                             |                                  |
| Sticky Note 4                | Sticky Note                  | Visual separator or placeholder          |                             |                             |                                  |
| Add a tag to a subscriber    | ConvertKit Tool             | Add tag to subscriber                     | ConvertKit Tool MCP Server   |                             |                                  |
| Get many tag subscriptions   | ConvertKit Tool             | Retrieve tag subscriptions                | ConvertKit Tool MCP Server   |                             |                                  |
| Delete a tag from a subscriber | ConvertKit Tool           | Remove tag from subscriber                | ConvertKit Tool MCP Server   |                             |                                  |
| Sticky Note 5                | Sticky Note                  | Visual separator or placeholder          |                             |                             |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node**
   - Add a node of type `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Name it `ConvertKit Tool MCP Server`.  
   - Configure the webhook ID (or let n8n generate one) to receive AI commands.  
   - No additional parameters are required.

2. **Create ConvertKit Tool Nodes for Custom Fields**
   - Add four `ConvertKit Tool` nodes named:  
     - `Create a custom field`  
     - `Get many custom fields`  
     - `Update a custom field`  
     - `Delete a custom field`
   - Configure each node to perform the respective API action on ConvertKit custom fields.  
   - Ensure that parameters such as field ID, field data, and pagination are configured to accept input dynamically from the MCP Trigger.  
   - Connect the output of `ConvertKit Tool MCP Server` to each node via the `ai_tool` connection.

3. **Add Subscriber and Subscription Management Nodes**
   - Add the following `ConvertKit Tool` nodes:  
     - Two instances of `Add a subscriber` (different positions for clarity)  
     - `Get many forms`  
     - `Get all subscriptions`  
     - `Get many sequences`  
     - `Get all subscriptions to a sequence`
   - Configure each node to perform their respective API calls.  
   - Ensure input parameters (subscriber email, form ID, sequence ID) are mapped dynamically.  
   - Connect the MCP Trigger node output to each.

4. **Add Tag Management Nodes**
   - Add the following `ConvertKit Tool` nodes:  
     - `Create a tag`  
     - `Get many tags`  
     - `Add a tag to a subscriber`  
     - `Get many tag subscriptions`  
     - `Delete a tag from a subscriber`
   - Configure each node properly with required parameters for tag and subscriber IDs.  
   - Connect the MCP Trigger node output to each.

5. **Add Sticky Notes (Optional)**
   - Add sticky note nodes near logical groupings for visual clarity. Use empty content or add descriptive comments as needed.

6. **Credentials Setup**
   - For all `ConvertKit Tool` nodes, configure the ConvertKit API credentials (API key or OAuth2) in the node credentials section.  
   - Ensure the MCP Trigger node is properly secured with authentication if required.

7. **Testing and Validation**
   - Test the webhook by sending sample AI commands to trigger different ConvertKit operations.  
   - Validate input parameter mapping and output responses.  
   - Handle any errors such as authentication failure, invalid inputs, or API limits.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| This workflow requires n8n version supporting `@n8n/n8n-nodes-langchain.mcpTrigger` node.           | n8n Documentation on MCP Trigger node                  |
| ConvertKit API credentials must be configured with appropriate permissions for custom fields and subscribers management. | ConvertKit API docs: https://developers.convertkit.com/ |
| No sticky notes contain descriptive content; consider adding notes for future maintainers.          | Visual organization within n8n editor                   |

---

**Disclaimer:** The provided text originates solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to prevailing content policies and does not contain any illegal, offensive, or protected elements. All data handled is legal and public.
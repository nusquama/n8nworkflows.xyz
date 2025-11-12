🛠️ Google Drive Tool MCP Server 💪 all 17 operations

https://n8nworkflows.xyz/workflows/----google-drive-tool-mcp-server----all-17-operations-5254


# 🛠️ Google Drive Tool MCP Server 💪 all 17 operations

### 1. Workflow Overview

This workflow, titled **"Google Drive Tool MCP Server"**, serves as a centralized automation server to handle **all 17 Google Drive operations** supported by the n8n Google Drive Tool node. It is designed to receive MCP (Multi-Channel Processing) trigger requests that specify which Google Drive operation to perform, then executes the corresponding action on Google Drive.

The workflow logically divides into the following blocks:

- **1.1 Input Reception:** The workflow starts with an MCP trigger node that listens for incoming requests specifying the desired Google Drive operation.

- **1.2 Google Drive Operations:** Seventeen different Google Drive Tool nodes implement the full range of supported operations, such as creating, copying, deleting, moving files and folders, sharing items, and managing shared drives.

- **1.3 Organizational Sticky Notes:** Several sticky notes are scattered throughout the workflow for visual grouping and organization but do not affect execution.

This structure facilitates easy extension and maintenance, enabling users or external systems to call a single endpoint and perform any Google Drive operation via this workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception: MCP Trigger Node

- **Overview:**  
  This block receives incoming MCP trigger requests that specify which Google Drive operation to execute.

- **Nodes Involved:**  
  - Google Drive Tool MCP Server (MCP Trigger node)

- **Node Details:**  

  - **Google Drive Tool MCP Server**  
    - Type: MCP Trigger (from `@n8n/n8n-nodes-langchain`)  
    - Role: Entry point that listens for HTTP webhook requests containing parameters for invoking Google Drive operations.  
    - Configuration: Uses a webhook ID (`9c8eeda6-d8d6-4ea8-8a5a-eab165ba8107`) to identify the trigger URL. No additional parameters configured, implying default MCP trigger behavior.  
    - Key variables: Receives input data that determines which Google Drive operation to execute and parameters for that operation.  
    - Input: External HTTP request via webhook.  
    - Output: Passes data to all Google Drive Tool nodes as "ai_tool" connections.  
    - Version-specific requirements: Requires n8n version supporting MCP Trigger (likely >= v1.95).  
    - Potential failures: Webhook not reachable, malformed requests, missing parameters, authentication failures downstream.  
    - Sub-workflow: None.

#### 2.2 Google Drive Operations

- **Overview:**  
  This block contains all Google Drive Tool nodes that perform specific Google Drive operations as requested by the MCP trigger node. Each node corresponds to one of the 17 supported operations.

- **Nodes Involved:**  
  - Copy file  
  - Create file from text  
  - Delete a file  
  - Download file  
  - Move file  
  - Share file  
  - Update file  
  - Upload file  
  - Search files and folders  
  - Create folder  
  - Delete folder  
  - Share folder  
  - Create shared drive  
  - Delete shared drive  
  - Get shared drive  
  - Get many shared drives  
  - Update shared drive

- **Node Details:**  

  Each node is a **Google Drive Tool** node (type: `n8n-nodes-base.googleDriveTool`, version 3). They share a common technical role: to execute a specific Google Drive API operation. Each node is connected directly from the MCP trigger node via an `ai_tool` connection, meaning all receive the input payload and decide internally whether to execute based on input parameters.

  Below are essential details for each:

  - **Copy file**  
    - Role: Copies a file to a specified location or with a new name.  
    - Configuration: Expects file ID to copy, destination folder, and optional new file name.  
    - Inputs/Outputs: Input from MCP trigger; outputs result of copy operation.  
    - Edge cases: File not found, insufficient permissions, quota exceeded.

  - **Create file from text**  
    - Role: Creates a new file with textual content.  
    - Configuration: Requires folder ID, file name, and text content.  
    - Edge cases: Invalid folder ID, file name conflicts, content too large.

  - **Delete a file**  
    - Role: Deletes a specified file by ID.  
    - Edge cases: File does not exist, insufficient permissions.

  - **Download file**  
    - Role: Downloads file content by ID.  
    - Edge cases: File not found, large file size causing timeouts.

  - **Move file**  
    - Role: Moves a file to a different folder.  
    - Edge cases: File or folder not found, permission errors.

  - **Share file**  
    - Role: Sets sharing permissions on a file.  
    - Configuration: Email or domain to share with, role (viewer/editor), type (user/group).  
    - Edge cases: Invalid email, permission denied.

  - **Update file**  
    - Role: Updates metadata or content of a file.  
    - Edge cases: File locked, invalid parameters.

  - **Upload file**  
    - Role: Uploads a file from binary data or local source.  
    - Edge cases: File size limits, network failures.

  - **Search files and folders**  
    - Role: Searches Drive using query syntax.  
    - Edge cases: Invalid query syntax, large result sets.

  - **Create folder**  
    - Role: Creates a new folder in Drive.  
    - Edge cases: Folder name conflicts.

  - **Delete folder**  
    - Role: Deletes a folder by ID.  
    - Edge cases: Folder not empty, permission denied.

  - **Share folder**  
    - Role: Shares a folder with specified users or groups.  
    - Edge cases: Invalid sharing parameters.

  - **Create shared drive**  
    - Role: Creates a new shared drive.  
    - Edge cases: Quota limits.

  - **Delete shared drive**  
    - Role: Deletes a shared drive by ID.  
    - Edge cases: Drive contains files, insufficient permissions.

  - **Get shared drive**  
    - Role: Retrieves metadata of a shared drive.  
    - Edge cases: Drive not found.

  - **Get many shared drives**  
    - Role: Lists multiple shared drives accessible to the user.  
    - Edge cases: Pagination handling.

  - **Update shared drive**  
    - Role: Updates metadata of a shared drive.  
    - Edge cases: Invalid parameters.

  Common to all these nodes:

  - Input: Data from MCP trigger node specifying operation and parameters.  
  - Output: Operation result sent back to MCP trigger response or further downstream processing (not shown).  
  - Credentials: Each node requires Google Drive OAuth2 credentials configured in n8n for authentication.  
  - Potential failures: Authentication errors, API rate limits, network errors, invalid parameters.

#### 2.3 Organizational Sticky Notes

- **Overview:**  
  Sticky notes are used purely for visual organization in the workflow canvas. They label or group sets of nodes but have no runtime function.

- **Nodes Involved:**  
  - Workflow Overview 0  
  - Sticky Note 1  
  - Sticky Note 2  
  - Sticky Note 3  
  - Sticky Note 4

- **Node Details:**  

  Each sticky note contains empty content or whitespace, indicating placeholders or visual guides only.

---

### 3. Summary Table

| Node Name               | Node Type                   | Functional Role                  | Input Node(s)               | Output Node(s)            | Sticky Note |
|-------------------------|-----------------------------|---------------------------------|-----------------------------|---------------------------|-------------|
| Workflow Overview 0     | Sticky Note                 | Visual organization             |                             |                           |             |
| Google Drive Tool MCP Server | MCP Trigger                 | Entry point receiving requests | (Webhook external)          | Copy file, Move file, Share file, Update file, Upload file, Share folder, Create folder, Delete a file, Delete folder, Download file, Get shared drive, Create shared drive, Delete shared drive, Get many shared drives, Update shared drive, Create file from text, Search files and folders |             |
| Copy file               | Google Drive Tool           | Copy a file                    | Google Drive Tool MCP Server |                           |             |
| Create file from text   | Google Drive Tool           | Create file with text content | Google Drive Tool MCP Server |                           |             |
| Delete a file           | Google Drive Tool           | Delete a file                  | Google Drive Tool MCP Server |                           |             |
| Download file           | Google Drive Tool           | Download a file                | Google Drive Tool MCP Server |                           |             |
| Move file               | Google Drive Tool           | Move a file                   | Google Drive Tool MCP Server |                           |             |
| Share file              | Google Drive Tool           | Share a file                  | Google Drive Tool MCP Server |                           |             |
| Update file             | Google Drive Tool           | Update file metadata/content  | Google Drive Tool MCP Server |                           |             |
| Upload file             | Google Drive Tool           | Upload a file                 | Google Drive Tool MCP Server |                           |             |
| Sticky Note 1           | Sticky Note                 | Visual organization            |                             |                           |             |
| Search files and folders | Google Drive Tool           | Search Drive contents         | Google Drive Tool MCP Server |                           |             |
| Sticky Note 2           | Sticky Note                 | Visual organization            |                             |                           |             |
| Create folder           | Google Drive Tool           | Create a folder               | Google Drive Tool MCP Server |                           |             |
| Delete folder           | Google Drive Tool           | Delete a folder               | Google Drive Tool MCP Server |                           |             |
| Share folder            | Google Drive Tool           | Share a folder                | Google Drive Tool MCP Server |                           |             |
| Sticky Note 3           | Sticky Note                 | Visual organization            |                             |                           |             |
| Create shared drive     | Google Drive Tool           | Create a shared drive         | Google Drive Tool MCP Server |                           |             |
| Delete shared drive     | Google Drive Tool           | Delete a shared drive         | Google Drive Tool MCP Server |                           |             |
| Get shared drive        | Google Drive Tool           | Get shared drive details      | Google Drive Tool MCP Server |                           |             |
| Get many shared drives  | Google Drive Tool           | List multiple shared drives   | Google Drive Tool MCP Server |                           |             |
| Update shared drive     | Google Drive Tool           | Update shared drive metadata  | Google Drive Tool MCP Server |                           |             |
| Sticky Note 4           | Sticky Note                 | Visual organization            |                             |                           |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n named "Google Drive Tool MCP Server".

2. **Add an MCP Trigger node**:
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`
   - Name: `Google Drive Tool MCP Server`
   - Configure webhook ID or accept default.
   - Leave parameters empty for default behavior.
   - This node will serve as the entry point for all incoming Google Drive operation requests.

3. **Add Google Drive Tool nodes** for each of the 17 operations:

   For each node below:

   - Node Type: `Google Drive Tool` (Type: `n8n-nodes-base.googleDriveTool`, Version 3)
   - Connect each node's input from the MCP Trigger node’s output.
   - Name nodes exactly as below for clarity.

   Operations to create:

   1. Copy file  
   2. Create file from text  
   3. Delete a file  
   4. Download file  
   5. Move file  
   6. Share file  
   7. Update file  
   8. Upload file  
   9. Search files and folders  
   10. Create folder  
   11. Delete folder  
   12. Share folder  
   13. Create shared drive  
   14. Delete shared drive  
   15. Get shared drive  
   16. Get many shared drives  
   17. Update shared drive

4. **Configure each Google Drive Tool node:**

   - Set the **Operation** field corresponding to the node’s function (e.g., "Copy file" node sets Operation to "Copy file").
   - Use expressions or incoming data from the MCP Trigger node to dynamically fill required parameters (e.g., file IDs, folder IDs, filenames, sharing emails).
   - Ensure all nodes use the same **Google Drive OAuth2 credentials** configured in n8n for authentication.
   - Handle optional parameters as needed.

5. **Add sticky notes for organization** (optional):

   - Add sticky notes near groups of nodes to visually separate logical blocks such as "Files Operations", "Folder Operations", "Shared Drive Operations".

6. **Connect output as needed**:

   - Typically, the Google Drive Tool nodes will output results back to the MCP Trigger node for response.
   - If further processing is needed, add nodes accordingly.

7. **Save and activate the workflow**.

8. **Test the workflow** by sending HTTP requests to the MCP Trigger webhook URL specifying the operation and parameters to verify each Google Drive operation works as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow covers **all 17 Google Drive Tool operations** in one MCP server endpoint, enabling centralized control. | n8n Google Drive Tool documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrivetool/ |
| MCP trigger allows multiple operations handled by one webhook, simplifying API access and integration.           | n8n MCP Trigger documentation: https://docs.n8n.io/nodes/n8n-nodes-langchain.mcpTrigger/           |
| Google Drive OAuth2 credentials must be configured with appropriate scopes to support all operations.            | Google Drive API scopes: https://developers.google.com/drive/api/v3/about-auth                      |
| Potential errors include Google API quota limits, permission errors, and malformed requests—handle these gracefully in production. |                                                                                                    |

---

**Disclaimer:** The text provided is extracted exclusively from an n8n automated workflow. It complies strictly with all applicable content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.
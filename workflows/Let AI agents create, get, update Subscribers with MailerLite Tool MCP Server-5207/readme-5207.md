Let AI agents create, get, update Subscribers with MailerLite Tool MCP Server

https://n8nworkflows.xyz/workflows/let-ai-agents-create--get--update-subscribers-with-mailerlite-tool-mcp-server-5207


# Let AI agents create, get, update Subscribers with MailerLite Tool MCP Server

### 1. Workflow Overview

This workflow functions as a MailerLite Tool MCP (Multi-Channel Platform) Server designed to allow AI agents to manage MailerLite subscribers. It supports four core subscriber operations—create, get, get all, and update—exposed via a webhook interface. The workflow is intended for use cases where AI agents dynamically interact with MailerLite subscriber data without manual parameter entry.

The workflow logic is organized into the following blocks:

- **1.1 MCP Server Initialization**: Receives incoming AI requests via a webhook and routes them to the appropriate subscriber operation.
- **1.2 Subscriber Operations**: Four distinct MailerLite Tool nodes implement subscriber management functions:
  - Create a subscriber
  - Get a subscriber by ID
  - Get multiple subscribers with optional limits and filters
  - Update a subscriber by ID

Each subscriber operation node extracts parameters dynamically from AI input expressions (`$fromAI()`), facilitating flexible, zero-configuration AI interactions.

---

### 2. Block-by-Block Analysis

#### 1.1 MCP Server Initialization

- **Overview:**  
  This block contains the MCP trigger node which acts as the entry point for AI-driven requests. It listens on a configured webhook path and forwards requests to the appropriate subscriber operation node based on AI agent input.

- **Nodes Involved:**  
  - MailerLite Tool MCP Server (MCP Trigger)

- **Node Details:**

  | Node Name                 | Description                                                                                     |
  |---------------------------|-------------------------------------------------------------------------------------------------|
  | MailerLite Tool MCP Server| Type: `mcpTrigger` (Langchain MCP Trigger)                                                    |
  |                           | Configuration: Listens on webhook path `/mailerlite-tool-mcp`                                  |
  |                           | Role: Entry point for AI agents sending commands to perform subscriber operations              |
  |                           | Key Expressions: None (receives all AI inputs via webhook automatically)                       |
  |                           | Input: HTTP webhook requests                                                                   |
  |                           | Output: Routes to subscriber operation nodes via `ai_tool` output                             |
  |                           | Version: Uses n8n Langchain MCP Trigger node, compatible with n8n 2.0+                        |
  |                           | Failures: Potential webhook connectivity issues, authentication errors if credentials missing  |
  |                           | Sub-workflow: None                                                                             |

#### 1.2 Subscriber Operations

- **Overview:**  
  This block implements the four core subscriber management operations using MailerLite Tool nodes. Each node extracts required parameters dynamically from AI input expressions using `$fromAI()`, allowing AI agents to specify parameters like `Email`, `Subscriber_Id`, `Limit`, and `Return_All`.

- **Nodes Involved:**  
  - Create a subscriber  
  - Get a subscriber  
  - Get many subscribers  
  - Update a subscriber  

- **Node Details:**

  1. **Create a subscriber**

     | Attribute               | Detail                                                             |
     |-------------------------|-------------------------------------------------------------------|
     | Type                    | `mailerLiteTool` (MailerLite API integration node)                |
     | Operation               | Create (default operation)                                        |
     | Configuration           | Email parameter set dynamically via `$fromAI('Email', '', 'string')` expression |
     | Additional Fields       | Empty object (no additional subscriber fields configured)          |
     | Input                   | Triggered from MCP Server node via `ai_tool` connection          |
     | Output                  | Subscriber creation response from MailerLite API                  |
     | Failures                | Input validation failure if email missing or invalid              |
     |                         | API auth errors if credentials incorrect or expired               |

  2. **Get a subscriber**

     | Attribute               | Detail                                                             |
     |-------------------------|-------------------------------------------------------------------|
     | Type                    | `mailerLiteTool`                                                  |
     | Operation               | Get                                                              |
     | Configuration           | Subscriber ID dynamically populated via `$fromAI('Subscriber_Id', '', 'string')` |
     | Input                   | MCP Server node                                                   |
     | Output                  | Subscriber details from MailerLite API                           |
     | Failures                | Missing or invalid subscriber ID causes API errors               |
     |                         | Authentication failures                                           |

  3. **Get many subscribers**

     | Attribute               | Detail                                                             |
     |-------------------------|-------------------------------------------------------------------|
     | Type                    | `mailerLiteTool`                                                  |
     | Operation               | Get All                                                          |
     | Configuration           | Limit: `$fromAI('Limit', '', 'number')`                          |
     |                         | Return All flag: `$fromAI('Return_All', '', 'boolean')`           |
     |                         | Filters: Empty (no filters applied)                              |
     | Input                   | MCP Server node                                                   |
     | Output                  | List of subscribers retrieved from MailerLite                    |
     | Failures                | Invalid parameter types or exceeding API rate limits             |

  4. **Update a subscriber**

     | Attribute               | Detail                                                             |
     |-------------------------|-------------------------------------------------------------------|
     | Type                    | `mailerLiteTool`                                                  |
     | Operation               | Update                                                          |
     | Configuration           | Subscriber ID: `$fromAI('Subscriber_Id', '', 'string')`          |
     |                         | Additional Fields: Empty (no fields configured for update)       |
     | Input                   | MCP Server node                                                   |
     | Output                  | API response from subscriber update                              |
     | Failures                | Invalid subscriber ID or missing update data                     |

- **Sticky Note:**  
  A sticky note labeled "Subscriber" groups these nodes conceptually.

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                  | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                                                                                                                                                      |
|---------------------------|----------------------------------|---------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview 0       | Sticky Note                      | Documentation / Info            | -                           | -                           | ## 🛠️ MailerLite Tool MCP Server  \n\n### 📋 Available Operations (4 total)\n\n**Subscriber**: create, get, get all, update\n\n### ⚙️ Setup Instructions\n\n1. Import Workflow\n2. Add Credentials\n3. Activate\n4. Get URL\n5. Connect AI agents\n\n### ✨ Features\n• Zero configuration\n• AI-driven parameters\n• Error handling\n• Modifiable defaults\n\n### 💬 Need Help?\nCheck [n8n docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/)\nor ping [discord](https://discord.me/cfomodz). |
| MailerLite Tool MCP Server| MCP Trigger (Langchain)          | Entry point - AI request intake | -                           | Create a subscriber, Get a subscriber, Get many subscribers, Update a subscriber |                                                                                                                                                                                                                                                  |
| Create a subscriber       | MailerLite Tool                  | Create subscriber               | MailerLite Tool MCP Server  | -                           |                                                                                                                                                                                                                                                  |
| Get a subscriber          | MailerLite Tool                  | Retrieve subscriber by ID       | MailerLite Tool MCP Server  | -                           |                                                                                                                                                                                                                                                  |
| Get many subscribers      | MailerLite Tool                  | List subscribers                | MailerLite Tool MCP Server  | -                           |                                                                                                                                                                                                                                                  |
| Update a subscriber       | MailerLite Tool                  | Update subscriber by ID         | MailerLite Tool MCP Server  | -                           |                                                                                                                                                                                                                                                  |
| Sticky Note 1             | Sticky Note                     | Node grouping label             | -                           | -                           | ## Subscriber                                                                                                                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note (Workflow Overview 0)**  
   - Type: Sticky Note  
   - Content: Include the workflow overview and setup instructions as detailed in section 3.  
   - Size: Width 420, Height 780  
   - Position: Roughly top-left area for documentation.

2. **Add MCP Trigger Node (MailerLite Tool MCP Server)**  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Path: `mailerlite-tool-mcp` (this is the webhook path)  
   - Position: Near center-left  
   - No credentials required on this node.  
   - This node acts as the webhook entry point.

3. **Create MailerLite Tool Node for Creating Subscriber**  
   - Node Type: `mailerLiteTool`  
   - Operation: Default (Create)  
   - Parameters:  
     - Email: Set expression `={{ $fromAI('Email', '', 'string') }}`  
     - Additional Fields: Leave empty  
   - Credentials: Configure MailerLite Tool credentials  
   - Connect input from MCP Trigger node's `ai_tool` output.  
   - Position: Left-middle area.

4. **Create MailerLite Tool Node for Getting a Subscriber**  
   - Node Type: `mailerLiteTool`  
   - Operation: `get`  
   - Parameters:  
     - Subscriber ID: Expression `={{ $fromAI('Subscriber_Id', '', 'string') }}`  
   - Credentials: Use MailerLite Tool credentials (can be same as above)  
   - Connect input from MCP Trigger node's `ai_tool` output.  
   - Position: Middle-left area.

5. **Create MailerLite Tool Node for Getting Many Subscribers**  
   - Node Type: `mailerLiteTool`  
   - Operation: `getAll`  
   - Parameters:  
     - Limit: Expression `={{ $fromAI('Limit', '', 'number') }}`  
     - Return All: Expression `={{ $fromAI('Return_All', '', 'boolean') }}`  
     - Filters: Leave empty  
   - Credentials: MailerLite Tool credentials  
   - Connect input from MCP Trigger node's `ai_tool` output.  
   - Position: Center area.

6. **Create MailerLite Tool Node for Updating a Subscriber**  
   - Node Type: `mailerLiteTool`  
   - Operation: `update`  
   - Parameters:  
     - Subscriber ID: Expression `={{ $fromAI('Subscriber_Id', '', 'string') }}`  
     - Additional Fields: Leave empty (can be configured to update fields if desired)  
   - Credentials: MailerLite Tool credentials  
   - Connect input from MCP Trigger node's `ai_tool` output.  
   - Position: Right-middle area.

7. **Create Sticky Note (Subscriber block)**  
   - Type: Sticky Note  
   - Content: "## Subscriber"  
   - Position: Overlapping the four subscriber operation nodes for grouping.

8. **Credential Setup**  
   - Configure MailerLite Tool credentials with valid API key or OAuth2 as per MailerLite integration documentation.  
   - Add credentials to each MailerLite Tool node.

9. **Activate Workflow**  
   - Enable the workflow.  
   - Copy the webhook URL from the MCP Trigger node for AI agent integration.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| The MCP Server allows zero-configuration AI agents to manage MailerLite subscribers dynamically via `$fromAI()` expressions.                                           | Workflow Description                                                                                                 |
| For detailed setup and troubleshooting, refer to the official n8n MCP integration documentation.                                                                        | https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/                        |
| Community support and customization help available on the MCP Discord server.                                                                                           | https://discord.me/cfomodz                                                                                            |
| All API calls depend on valid MailerLite credentials; ensure API keys are current and permissions are sufficient to manage subscribers.                                | MailerLite API and n8n credentials management                                                                        |
| The workflow uses dynamic parameter population via AI expressions, enabling flexible and extensible automation scenarios without manual intervention.                 | `$fromAI()` expression usage                                                                                         |
| Error handling is native to n8n nodes; consider adding custom error workflows or notifications if needed.                                                               | n8n error handling best practices                                                                                    |

---

**Disclaimer:** The content provided is exclusively derived from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal or protected material. All data handled is legal and public.
Add product ideas to Notion via a Slack command

https://n8nworkflows.xyz/workflows/add-product-ideas-to-notion-via-a-slack-command-2140


# Add product ideas to Notion via a Slack command

### 1. Workflow Overview

This n8n workflow enables Slack users to submit product ideas directly into a Notion database via a custom Slack slash command `/idea`. It is designed to streamline idea collection by allowing users to add structured ideas without leaving Slack, integrating Slack commands with Notion’s database capabilities.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception:** Receives Slack slash command invocations via a webhook.
- **1.2 Command Routing and Validation:** Determines which Slack command was issued and routes requests accordingly.
- **1.3 Notion Integration:** Adds the submitted idea to a Notion database, associating it with the user who submitted it.
- **1.4 Slack Notification:** Sends a confirmation message back to the user in Slack with a link to add more details to the idea.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block handles incoming HTTP POST requests from Slack when the `/idea` command is invoked. It captures the data payload sent by Slack, including the idea text, user information, and command metadata.

- **Nodes Involved:**  
  - Webhook  
  - Set me up (partially, as it sets configuration for later nodes)

- **Node Details:**  

  - **Webhook**  
    - *Type:* Webhook Trigger  
    - *Technical Role:* Listens for POST requests on the path `slack-trigger` to capture Slack slash command payloads.  
    - *Configuration:* HTTP method POST, no authentication specified (public endpoint).  
    - *Inputs/Outputs:* No input; outputs request JSON payload.  
    - *Expressions:* Not used here beyond default payload capture.  
    - *Failure Modes:* Possible failures include invalid HTTP method calls, missing or malformed payload from Slack, or Slack verification issues if request signing is not verified (not covered in this workflow).  
    - *Notes:* Webhook URL must be set as the Request URL for the Slack slash command.

  - **Set me up**  
    - *Type:* Set Node  
    - *Technical Role:* Stores configuration data, specifically the Notion Database URL needed for integration.  
    - *Configuration:* Assigns a string value to “Notion URL” representing the Notion database page URL.  
    - *Inputs/Outputs:* Input from Webhook; outputs this configuration for downstream nodes.  
    - *Failure Modes:* Misconfiguration of the Notion URL will cause Notion node failures.  

---

#### 2.2 Command Routing and Validation

- **Overview:**  
  This block checks which slash command was issued (currently supports `/idea`) and routes the workflow accordingly. It is extensible to support additional commands.

- **Nodes Involved:**  
  - Switch  
  - Sticky Note2 (documentation/support note)

- **Node Details:**  

  - **Switch**  
    - *Type:* Switch Node  
    - *Technical Role:* Routes workflow execution based on the command string received from Slack.  
    - *Configuration:*  
      - Condition 1: If `body.command` equals `/idea`, route to "New idea" output.  
      - Condition 2: Placeholder for other commands (e.g., `/some-other-command`).  
    - *Inputs/Outputs:* Input from “Set me up”; outputs to Notion node on `/idea`.  
    - *Expressions:* Uses expression `={{ $json.body.command }}` to dynamically evaluate the slash command.  
    - *Failure Modes:* If an unknown command is received, no matching output is triggered; workflow ends silently.  

  - **Sticky Note2**  
    - *Type:* Sticky Note  
    - *Role:* Provides information about extensibility for additional commands.  
    - *Content:* "You can easily support more commands, like `/bug` or `/pain` here."  

---

#### 2.3 Notion Integration

- **Overview:**  
  This block creates a new page in a Notion database to store the submitted idea, associating the idea’s title with the Slack user's name as the creator.

- **Nodes Involved:**  
  - Notion

- **Node Details:**  

  - **Notion**  
    - *Type:* Notion Database Page Node (resource: databasePage)  
    - *Technical Role:* Inserts a new page into a specified Notion database.  
    - *Configuration:*  
      - `databaseId` dynamically set from the "Notion URL" stored in “Set me up” node.  
      - Title property assigned from `body.text` (the idea text from Slack).  
      - Creator property assigned from `body.user_name` (Slack username).  
    - *Inputs/Outputs:* Input from Switch node; outputs Notion response data.  
    - *Credentials:* Uses pre-configured Notion API credentials with integration access.  
    - *Expressions:*  
      - Title: `={{ $json.body.text }}`  
      - Creator: `={{ $json.body.user_name }}`  
      - Database ID extracted from URL parameter.  
    - *Failure Modes:*  
      - Invalid or expired Notion credentials.  
      - Incorrect database URL or permissions missing on the Notion side.  
      - Network errors or API rate limits.  

---

#### 2.4 Slack Notification

- **Overview:**  
  Sends a confirmation message back to the Slack user who submitted the idea, including a prompt to add more detailed information via a link.

- **Nodes Involved:**  
  - Hidden message to add feature details

- **Node Details:**  

  - **Hidden message to add feature details**  
    - *Type:* HTTP Request Node  
    - *Technical Role:* Sends a POST request to Slack’s `response_url` to display an ephemeral message to the user.  
    - *Configuration:*  
      - URL dynamically set from the Slack `response_url` field of the webhook payload.  
      - Method: POST  
      - Body JSON includes a formatted message referencing the idea text and user, along with a URL for adding details (provided in `$json["url"]`).  
    - *Inputs/Outputs:* Input from Notion node; no output required.  
    - *Expressions:*  
      - URL: `={{ $('Webhook').item.json.body.response_url }}`  
      - Message text interpolates Slack user and idea text from the webhook payload.  
    - *Failure Modes:*  
      - Slack API errors (invalid response_url, expired URL).  
      - Network or timeout issues.  
      - Missing or malformed response_url.  

---

### 3. Summary Table

| Node Name                      | Node Type              | Functional Role                      | Input Node(s)               | Output Node(s)                      | Sticky Note                                                                                                          |
|-------------------------------|------------------------|------------------------------------|-----------------------------|------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Webhook                       | Webhook Trigger        | Receive Slack slash command payload| —                           | Set me up                         | See "Needed pre-work: Add a Slack App" sticky note for Slack app setup instructions                                  |
| Set me up                     | Set                    | Store Notion database URL config   | Webhook                     | Switch                           |                                                                                                                      |
| Switch                       | Switch                 | Route based on Slack command       | Set me up                   | Notion                          | "You can easily support more commands, like `/bug` or `/pain` here."                                                 |
| Notion                        | Notion Database Page   | Add idea to Notion database        | Switch                      | Hidden message to add feature details |                                                                                                                      |
| Hidden message to add feature details | HTTP Request         | Notify Slack user of submission    | Notion                      | —                              |                                                                                                                      |
| Sticky Note                   | Sticky Note            | Slack app setup instructions       | —                           | —                                | "## Needed pre-work: Add a Slack App\n1. Visit https://api.slack.com/apps, click on `New App` and choose a name ..." |
| Sticky Note1                  | Sticky Note            | Setup instructions for Notion and workflow | —                           | —                                | "## Setup\n1. Add a Database in Notion with the columns `Name` and `Creator`..."                                     |
| Sticky Note2                  | Sticky Note            | Suggestion to extend commands       | —                           | —                                | "You can easily support more commands, like `/bug` or `/pain` here"                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `slack-trigger`  
   - Purpose: Receive Slack slash command payloads.  
   - Save and copy the generated webhook URL.

2. **Create Set Node (Named "Set me up")**  
   - Type: Set  
   - Add a string field named `Notion URL`.  
   - Set its value to your Notion database page URL (e.g., `https://www.notion.so/yourworkspace/yourdatabaseid`).  
   - Connect Webhook node output to this node.

3. **Create Switch Node**  
   - Type: Switch  
   - Add rule:  
     - Condition: If `{{$json.body.command}}` equals `/idea`  
     - Output label: "New idea"  
   - Optional: Add additional rules for other commands if needed.  
   - Connect "Set me up" node output to Switch node input.

4. **Create Notion Node**  
   - Type: Notion (resource: databasePage)  
   - Credentials: Setup Notion API credentials and connect to your integration with access to your target database.  
   - Database ID: Use the expression to extract from `Notion URL` in "Set me up": `={{ $('Set me up').first().json['Notion URL'] }}`  
   - Properties:  
     - Set `Name` or Title property to `{{$json.body.text}}` (the idea text from Slack).  
     - Set `Creator` property (rich text) to `{{$json.body.user_name}}`.  
   - Connect Switch node “New idea” output to this node.

5. **Create HTTP Request Node (Named "Hidden message to add feature details")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: Set with expression `={{ $('Webhook').item.json.body.response_url }}` (Slack response_url)  
   - Body Parameters (JSON):  
     ```json
     {
       "text": "Thanks for adding the idea `{{$json.body.text}}` <@{{$json.body.user_id}}> :rocket: Please make sure to add some details and a hypothesis to it to make it easier for us to work with it.\n\n:point_right: <{{$json.url}}|Add your details here>"
     }
     ```  
   - Connect Notion node output to this node.

6. **Connect Nodes**  
   - Webhook → Set me up → Switch → Notion → Hidden message to add feature details.

7. **Credential Setup**  
   - Set up Notion API credentials with access to the target database.  
   - No Slack OAuth credential needed here, as Slack uses the response_url from the slash command payload to send messages.

8. **Slack App Setup**  
   - Create a Slack app at https://api.slack.com/apps.  
   - Add `chat:write` scope under Bot token scopes.  
   - Create a slash command `/idea` with Request URL set to your Webhook URL.  
   - Install the app to your workspace.

9. **Database Setup in Notion**  
   - Create a database with at least `Name` (title) and `Creator` (rich text) columns.  
   - Share the database with your integration.

10. **Testing and Activation**  
    - Test the workflow by triggering the slash command in Slack.  
    - Once verified, activate the workflow in n8n and update Slack slash command Request URL to the production webhook URL.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow designed to facilitate team idea collection via Slack and Notion integration, enabling easy submission and tracking.                                                                                                                                                                              | General workflow purpose                                                                                                                                    |
| Slack app creation steps and scopes needed: https://api.slack.com/apps.                                                                                                                                                                                                                                       | Slack app setup instructions                                                                                                                               |
| You can expand this workflow by adding more slash commands like `/bug` or `/pain` that integrate with other tools (e.g., Linear) and use AI for classification.                                                                                                                                            | Workflow extensibility suggestions                                                                                                                         |
| Weekly reporting workflows can be combined to highlight popular ideas and active contributors.                                                                                                                                                                                                               | Workflow enhancement note                                                                                                                                  |
| Notion integration requires setting up an integration and sharing the database with it: https://developers.notion.com/docs/integrations                                                                                                                                                                     | Notion API integration setup                                                                                                                               |
| The Slack confirmation message includes a link for the user to add more details to their idea, improving collaboration and data quality.                                                                                                                                                                    | Slack user notification detail                                                                                                                             |
| For security, consider verifying Slack request signatures (not implemented here) to prevent unauthorized webhook calls.                                                                                                                                                                                    | Security recommendation                                                                                                                                   |

---

This document provides a complete, detailed structure and guidance for understanding, reproducing, and extending the "Add product ideas to Notion via a Slack command" workflow in n8n.
Create a channel, add a member, and post a message to the channel on Mattermost

https://n8nworkflows.xyz/workflows/create-a-channel--add-a-member--and-post-a-message-to-the-channel-on-mattermost-832


# Create a channel, add a member, and post a message to the channel on Mattermost

### 1. Workflow Overview

This workflow automates the process of creating a channel in Mattermost, adding a specific user as a member to that channel, and then posting a welcome message within it. It is ideal for onboarding processes, team collaboration setups, or automating communication channels programmatically.

The workflow is logically divided into the following blocks:

- **1.1 Input Trigger:** Manual initiation of the workflow.
- **1.2 Channel Creation:** Creating a new channel in a specified Mattermost team.
- **1.3 Add Member:** Adding a specified user to the newly created channel.
- **1.4 Post Message:** Sending a welcome message to the channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  This block initiates the workflow execution manually by the user.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Node Name:** On clicking 'execute'  
  - **Type:** Manual Trigger  
  - **Technical Role:** Starts the workflow upon manual user action in the n8n editor or UI.  
  - **Configuration:** No parameters configured; default manual trigger.  
  - **Key Expressions/Variables:** None.  
  - **Input Connections:** None (starting node).  
  - **Output Connections:** Output connected to the "Mattermost" node (channel creation).  
  - **Version Requirements:** n8n version supporting `manualTrigger` node (standard in most versions).  
  - **Edge Cases / Failures:** None expected; manual trigger depends on user action.

#### 1.2 Channel Creation

- **Overview:**  
  This block creates a new channel within a specified Mattermost team.

- **Nodes Involved:**  
  - Mattermost

- **Node Details:**  
  - **Node Name:** Mattermost  
  - **Type:** Mattermost Node  
  - **Technical Role:** Creates a channel using Mattermost API.  
  - **Configuration Choices:**  
    - Resource: `channel`  
    - Operation: (default create implied)  
    - Team ID: `"4zhpirmh97fn7jgp7qhyue5a6e"` (fixed team identifier)  
    - Channel Name: `"docs"` (channel unique identifier)  
    - Display Name: `"Docs"` (human-readable channel name)  
  - **Credentials:** Uses stored Mattermost API credentials named "Mattermost Credentials".  
  - **Key Expressions/Variables:** None; static values used.  
  - **Input Connections:** From "On clicking 'execute'".  
  - **Output Connections:** To "Mattermost1" node (adding user).  
  - **Version Requirements:** Mattermost node version 1 or higher.  
  - **Edge Cases / Failures:**  
    - API authentication failure (invalid or expired credentials).  
    - Team ID or channel parameters invalid or not authorized.  
    - Channel name already exists (possible conflict error).  
    - Network or API timeout.  
  - **Sub-workflow:** None.

#### 1.3 Add Member

- **Overview:**  
  This block adds a specific user as a member to the channel created in the previous step.

- **Nodes Involved:**  
  - Mattermost1

- **Node Details:**  
  - **Node Name:** Mattermost1  
  - **Type:** Mattermost Node  
  - **Technical Role:** Adds a user to an existing Mattermost channel.  
  - **Configuration Choices:**  
    - Resource: `channel`  
    - Operation: `addUser`  
    - User ID: `"5oiy71hukjgd9eprj1o4a3poio"` (fixed user identifier)  
    - Channel ID: Dynamically set to the ID of the channel created in the previous step using expression: `={{$node["Mattermost"].json["id"]}}`  
  - **Credentials:** Uses "Mattermost Credentials".  
  - **Key Expressions/Variables:** Expression for channelId.  
  - **Input Connections:** From "Mattermost" node.  
  - **Output Connections:** To "Mattermost2" node (posting message).  
  - **Version Requirements:** Mattermost node version 1 or higher.  
  - **Edge Cases / Failures:**  
    - User ID invalid or user not existing.  
    - User already member of the channel (may cause duplicate or error).  
    - Authorization failure.  
    - API or network errors.  
  - **Sub-workflow:** None.

#### 1.4 Post Message

- **Overview:**  
  This block posts a welcome message to the newly created channel.

- **Nodes Involved:**  
  - Mattermost2

- **Node Details:**  
  - **Node Name:** Mattermost2  
  - **Type:** Mattermost Node  
  - **Technical Role:** Sends a message to a Mattermost channel.  
  - **Configuration Choices:**  
    - Message: `"Hey! Welcome to the channel!"` (static welcome text)  
    - Channel ID: Dynamic reference to created channel ID via expression: `={{$node["Mattermost"].json["id"]}}`  
    - Attachments: Empty array (no attachments).  
    - Other Options: Empty (default behavior).  
  - **Credentials:** Uses "Mattermost Credentials".  
  - **Key Expressions/Variables:** Expression for channelId.  
  - **Input Connections:** From "Mattermost1" node.  
  - **Output Connections:** None (end of workflow).  
  - **Version Requirements:** Mattermost node version 1 or higher.  
  - **Edge Cases / Failures:**  
    - Message content empty or invalid format (unlikely here).  
    - Channel ID invalid (if previous step failed silently).  
    - API or network errors.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role             | Input Node(s)         | Output Node(s)      | Sticky Note                                              |
|---------------------|--------------------|----------------------------|-----------------------|---------------------|----------------------------------------------------------|
| On clicking 'execute'| Manual Trigger     | Start workflow execution   | None                  | Mattermost          |                                                          |
| Mattermost          | Mattermost Node     | Create a channel           | On clicking 'execute' | Mattermost1         | Refer to [documentation](https://docs.n8n.io/nodes/n8n-nodes-base.mattermost/#mattermost) for building this workflow from scratch. |
| Mattermost1         | Mattermost Node     | Add a user to channel      | Mattermost            | Mattermost2         |                                                          |
| Mattermost2         | Mattermost Node     | Post welcome message       | Mattermost1           | None                |                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**
   - Name: `On clicking 'execute'`
   - Leave default settings.
   - This node will serve as the manual start trigger.

3. **Add a Mattermost node to create a channel:**
   - Name: `Mattermost`
   - Set **Resource** to `channel`.
   - Set **Team ID** to `"4zhpirmh97fn7jgp7qhyue5a6e"` (replace with your actual team ID).
   - Set **Channel** to `"docs"` (this is the unique channel name).
   - Set **Display Name** to `"Docs"` (this is the human-friendly channel name).
   - Select or create credentials for Mattermost API under "Mattermost Credentials".
   - Connect the output of `On clicking 'execute'` to this node.

4. **Add another Mattermost node to add a user:**
   - Name: `Mattermost1`
   - Set **Resource** to `channel`.
   - Set **Operation** to `addUser`.
   - Set **User ID** to `"5oiy71hukjgd9eprj1o4a3poio"` (replace with actual user ID).
   - Set **Channel ID** to an expression referencing the previously created channel's ID:  
     `={{$node["Mattermost"].json["id"]}}`
   - Use the same Mattermost credentials.
   - Connect the output of `Mattermost` node to this node.

5. **Add a final Mattermost node to post a message:**
   - Name: `Mattermost2`
   - Set **Message** to `"Hey! Welcome to the channel!"`.
   - Set **Channel ID** to the same dynamic expression:  
     `={{$node["Mattermost"].json["id"]}}`
   - Leave **Attachments** empty.
   - Leave **Other Options** empty.
   - Use the same Mattermost credentials.
   - Connect the output of `Mattermost1` node to this node.

6. **Save and activate the workflow (optional).**

7. **Run the workflow manually by clicking ‘Execute’ in n8n UI.**

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| For detailed Mattermost node capabilities and configuration options, see the official n8n documentation. | https://docs.n8n.io/nodes/n8n-nodes-base.mattermost/#mattermost                                                |
| This workflow requires valid Mattermost API credentials with permissions to create channels, add users, and post messages. | Ensure your Mattermost API token and user permissions are correctly configured in n8n credentials.              |
| Channel names must be unique per team in Mattermost; creating a channel with an existing name may cause errors. | Mattermost channel naming rules and uniqueness constraints apply.                                               |

---

This document should enable both human users and AI systems to fully understand, replicate, or modify the workflow without referring back to the original JSON.
Automate Telegram Message Processing - Separate Text and Files

https://n8nworkflows.xyz/workflows/automate-telegram-message-processing---separate-text-and-files-9254


# Automate Telegram Message Processing - Separate Text and Files

### 1. Workflow Overview

This workflow automates the processing of incoming Telegram messages to intelligently separate textual content and file attachments for downstream use. It is designed to handle three types of messages: text-only, file-only, and file with caption (text). The workflow routes these messages through dedicated branches that extract and prepare the data accordingly, including downloading attachments to binary format, readying them for further automation steps like storage, AI processing, or logging.

The main logical blocks are:

- **1.1 Input Reception:** Listens for new messages sent to a Telegram bot and captures raw message data.
- **1.2 Message Type Routing:** Determines the message type (text, file, or file+caption) using conditional logic and routes accordingly.
- **1.3 Content Extraction & Preparation:** Extracts and formats the text or file metadata; downloads files to binary data where needed.
- **1.4 Finalization:** Sends the processed content to placeholder nodes (NoOp) where further custom logic can be attached.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures all new messages sent to the Telegram bot and outputs raw message data including text, captions, and files.

- **Nodes Involved:**  
  - Waiting For Message

- **Node Details:**

  - **Waiting For Message**  
    - Type: Telegram Trigger  
    - Role: Listens for new Telegram messages in real-time.  
    - Configuration: Trigger on `message` update type; does not auto-download files at this stage (`download: false`).  
    - Key Expressions: None (listens passively).  
    - Inputs: None (trigger node).  
    - Outputs: Raw Telegram message JSON including text, captions, document metadata.  
    - Credentials: Uses configured Telegram API credential named "Template".  
    - Edge Cases: Failure if credential is invalid or bot token is revoked; will not trigger if workflow is inactive or incorrect bot is messaged.

#### 2.2 Message Type Routing

- **Overview:**  
  Inspects incoming message content to classify into one of three categories: file with caption, file only, or text only, then routes execution accordingly.

- **Nodes Involved:**  
  - Switch

- **Node Details:**

  - **Switch**  
    - Type: Switch (conditional routing)  
    - Role: Routes messages based on the presence of `message.caption`, `message.document`, or `message.text`.  
    - Configuration:  
      - Output "Attachment+Message(Caption)" if `message.caption` exists and is non-empty string.  
      - Output "Attachment Only" if `message.document` object exists.  
      - Output "Message Only" if `message.text` exists and is non-empty string.  
    - Key Expressions: Uses expression-based exists checks on JSON fields.  
    - Inputs: From "Waiting For Message".  
    - Outputs: Three outputs mapped to respective processing branches.  
    - Edge Cases: Messages with no recognized content may not match any condition (not handled explicitly); complex media types like photos or videos are not handled here.

#### 2.3 Content Extraction & Preparation

This block contains three parallel branches aligned with the three message types identified.

##### 2.3.1 Branch: Attachment + Caption

- **Overview:**  
  Separates file data and caption text into distinct items to process them independently.

- **Nodes Involved:**  
  - Split Out  
  - Get Chat Message Content  
  - Get Attachment  
  - Get & Download Attachment  
  - Next Step ! (NoOp)

- **Node Details:**

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the incoming item into multiple items, extracting `message.document.file_id`, `message.document.file_name`, and `message.caption` into separate items.  
    - Configuration: Splits out on fields: `message.document.file_id`, `message.document.file_name`, `message.caption`.  
    - Inputs: From Switch output "Attachment+Message(Caption)".  
    - Outputs: Two items downstream, one containing file data, one containing caption text.

  - **Get Chat Message Content**  
    - Type: Set  
    - Role: Assigns the caption text to a new field `chat_message_content` for clarity.  
    - Configuration: Sets `chat_message_content` = `message.caption`.  
    - Inputs: From Split Out (caption item).  
    - Outputs: To next NoOp node.

  - **Get Attachment**  
    - Type: Set  
    - Role: Extracts file metadata `file_id` and `file_name` from the split item.  
    - Configuration: Sets `file_id` and `file_name` from `message.document.file_id` and `message.document.file_name`.  
    - Inputs: From Split Out (file item).  
    - Outputs: To "Get & Download Attachment".

  - **Get & Download Attachment**  
    - Type: Telegram (file download)  
    - Role: Downloads the file binary data using the file ID.  
    - Configuration: Uses file ID from input JSON (`file_id` or fallback to `message.document.file_id`).  
    - Credentials: Uses the same Telegram API credential.  
    - Inputs: From "Get Attachment".  
    - Outputs: To NoOp "Next Step !  ".

  - **Next Step !** (NoOp nodes)  
    - Type: No Operation (placeholder)  
    - Role: Marks the end of this processing branch, ready for custom logic.  
    - Inputs: From "Get Chat Message Content" (text) and from "Get & Download Attachment" (file).  
    - Outputs: None.

- **Edge Cases:**  
  - Missing or malformed file metadata could cause download failure.  
  - Empty caption field may produce empty text output.  
  - Telegram API rate limits or invalid file IDs may cause download errors.

##### 2.3.2 Branch: Attachment Only

- **Overview:**  
  Extracts file metadata and downloads the file for messages containing only attachments.

- **Nodes Involved:**  
  - Get Attachment Only  
  - Get & Download Attachment  
  - Next Step !  (NoOp)

- **Node Details:**

  - **Get Attachment Only**  
    - Type: Set  
    - Role: Extracts `file_id` and `file_name` from the incoming message document.  
    - Configuration: Sets `file_id` = `message.document.file_id`, `file_name` = `message.document.file_name`.  
    - Inputs: From Switch output "Attachment Only".  
    - Outputs: To "Get & Download Attachment".

  - **Get & Download Attachment**  
    - As above (shared node with previous branch).

  - **Next Step !  ** (NoOp)  
    - Role: Placeholder marking end of this branch.  
    - Inputs: From "Get & Download Attachment".  
    - Outputs: None.

- **Edge Cases:**  
  - Same as 2.3.1 for file download errors and metadata issues.

##### 2.3.3 Branch: Message Only

- **Overview:**  
  Extracts plain text messages and prepares them for downstream processing.

- **Nodes Involved:**  
  - Get Chat Message Only  
  - Next Step !  (NoOp)

- **Node Details:**

  - **Get Chat Message Only**  
    - Type: Set  
    - Role: Initializes a new field `text` with the message text or empty string if none.  
    - Configuration: Sets `text` = "" (empty string) by default; actual text comes from raw JSON at runtime.  
    - Inputs: From Switch output "Message Only".  
    - Outputs: To "Next Step ! ".

  - **Next Step !** (NoOp)  
    - Role: Placeholder marking end of processing for text-only messages.  
    - Inputs: From "Get Chat Message Only".  
    - Outputs: None.

- **Edge Cases:**  
  - Empty messages or messages with unsupported content types may pass through with empty text.

---

### 3. Summary Table

| Node Name                | Node Type               | Functional Role                         | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                      |
|--------------------------|-------------------------|---------------------------------------|------------------------|--------------------------|-------------------------------------------------------------------------------------------------|
| Waiting For Message       | Telegram Trigger        | Input reception of new Telegram messages | None                   | Switch                   |                                                                                                 |
| Switch                   | Switch                  | Routes messages by type                | Waiting For Message     | Split Out, Get Attachment Only, Get Chat Message Only |                                                                                                 |
| Split Out                | Split Out               | Separates file and caption items      | Switch                  | Get Chat Message Content, Get Attachment |                                                                                                 |
| Get Chat Message Content  | Set                     | Extracts caption text into field      | Split Out               | Next Step !              |                                                                                                 |
| Get Attachment           | Set                     | Extracts file metadata (file_id, name) | Split Out               | Get & Download Attachment |                                                                                                 |
| Get & Download Attachment | Telegram                | Downloads file binary data             | Get Attachment, Get Attachment Only | Next Step !              |                                                                                                 |
| Get Attachment Only      | Set                     | Extracts file metadata for file-only messages | Switch                  | Get & Download Attachment |                                                                                                 |
| Get Chat Message Only    | Set                     | Extracts text-only message content     | Switch                  | Next Step !              |                                                                                                 |
| Next Step !              | NoOp                    | Placeholder for caption text path      | Get Chat Message Content | None                     |                                                                                                 |
| Next Step !              | NoOp                    | Placeholder for downloaded file path   | Get & Download Attachment | None                    |                                                                                                 |
| Next Step !              | NoOp                    | Placeholder for text-only message path | Get Chat Message Only    | None                     |                                                                                                 |
| Sticky Note3             | Sticky Note             | Workflow purpose and benefits overview | None                   | None                     | ## 💬 **Telegram Message Parser - Separate Text and Files** ᯓ➤ ... (full content in workflow)  |
| Sticky Note              | Sticky Note             | Workflow flow explanation               | None                   | None                     | ## 🔄 **WORKFLOW FLOW EXPLAINED** ... (full content in workflow)                                 |
| Sticky Note1             | Sticky Note             | Thank you and feedback request          | None                   | None                     | ## 🙏 **Thank You for Trying This Workflow** ... (full content in workflow)                      |
| Sticky Note2             | Sticky Note             | Customization options and extension tips | None                   | None                     | ## 🛠️ **CUSTOMIZATION OPTIONS** ... (full content in workflow)                                  |
| Sticky Note4             | Sticky Note             | Troubleshooting common issues           | None                   | None                     | ## 🩺 TROUBLESHOOTING ... (full content in workflow)                                            |
| Sticky Note5             | Sticky Note             | Step-by-step setup guide for Telegram bot and n8n | None                   | None                     | ## 🔧 **STEP-BY-STEP SETUP GUIDE** ... (full content in workflow)                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials:**  
   - Use Telegram `@BotFather` to create a new bot and obtain the API Token.  
   - In n8n, add new Telegram API credentials with this token. Name appropriately.

2. **Create Telegram Trigger Node:**  
   - Add node: Telegram Trigger  
   - Set update types to listen for `message`.  
   - Disable file auto-download.  
   - Assign Telegram credentials created in step 1.  
   - Name: "Waiting For Message".

3. **Add Switch Node:**  
   - Add node: Switch  
   - Add three rules:  
     - Output "Attachment+Message(Caption)": condition `message.caption` exists and is non-empty string.  
     - Output "Attachment Only": condition `message.document` object exists.  
     - Output "Message Only": condition `message.text` exists and is non-empty string.  
   - Connect "Waiting For Message" output to this Switch node input.

4. **Branch 1: Attachment + Caption**  
   - Add node: Split Out  
     - Configure to split out fields:  
       - `message.document.file_id`  
       - `message.document.file_name`  
       - `message.caption`  
   - Connect Switch output "Attachment+Message(Caption)" to Split Out input.

   - Add node: Set (name "Get Chat Message Content")  
     - Assign field `chat_message_content` = `{{$json["message.caption"]}}`.  
   - Connect Split Out output corresponding to caption to this node.

   - Add node: Set (name "Get Attachment")  
     - Assign fields:  
       - `file_id` = `{{$json["message.document.file_id"]}}`  
       - `file_name` = `{{$json["message.document.file_name"]}}`  
   - Connect Split Out output corresponding to file to this node.

   - Add node: Telegram (name "Get & Download Attachment")  
     - Resource: file  
     - File ID: `{{$json.file_id || $json["message.document.file_id"]}}`  
     - Assign Telegram credentials.  
   - Connect "Get Attachment" output to this node.

   - Add node: NoOp (name "Next Step !")  
     - Connect "Get Chat Message Content" output to this NoOp.  
     - Add another NoOp (name "Next Step !  ") and connect "Get & Download Attachment" output here.

5. **Branch 2: Attachment Only**  
   - Add node: Set (name "Get Attachment Only")  
     - Assign fields:  
       - `file_id` = `{{$json["message.document.file_id"]}}`  
       - `file_name` = `{{$json["message.document.file_name"]}}`  
   - Connect Switch output "Attachment Only" to this node.

   - Reuse "Get & Download Attachment" Telegram node from above (or create a new one).  
   - Connect "Get Attachment Only" output to the Telegram node.

   - Add NoOp node (name "Next Step !  ")  
   - Connect Telegram node output to this NoOp.

6. **Branch 3: Message Only**  
   - Add node: Set (name "Get Chat Message Only")  
     - Assign field `text` = `""` (empty string), but at runtime this will hold the message text.  
   - Connect Switch output "Message Only" to this node.

   - Add NoOp node (name "Next Step ! ")  
   - Connect "Get Chat Message Only" output to this NoOp.

7. **Activate Workflow:**  
   - Review all connections and node configurations.  
   - Activate the workflow to start listening for Telegram messages.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| 💬 **Telegram Message Parser - Separate Text and Files:** Quick demo and benefits summary including smart content routing and ready-to-use binary files.                                                                                                                                                     | Sticky Note3 in workflow                                                                              |
| 🔄 **Workflow Flow Explained:** Highlights input reception, routing via Switch node, content extraction steps, and final output placeholders.                                                                                                                                                                | Sticky Note                                                                                           |
| 🙏 **Thank You for Trying This Workflow:** Invites feedback for improvement and new ideas to enhance the template.                                                                                                                                                                                           | Sticky Note1                                                                                          |
| 🛠️ **Customization Options:** Suggestions to extend functionality with cloud storage nodes, AI processing, database logging; also how to handle more Telegram media types by updating Switch rules and Set nodes accordingly.                                                                                 | Sticky Note2                                                                                          |
| 🩺 **Troubleshooting:** Common issues include workflow not triggering and file download failures; checklist includes verifying credentials and testing all message types.                                                                                                                                   | Sticky Note4                                                                                          |
| 🔧 **Step-by-Step Setup Guide:** Detailed instructions to create Telegram bot, configure credentials, link nodes, and test with all message types.                                                                                                                                                            | Sticky Note5                                                                                          |

---

This document provides a clear and complete understanding of the Telegram message processing workflow, enabling users and AI agents to confidently replicate, maintain, and extend it.
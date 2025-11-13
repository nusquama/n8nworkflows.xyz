WhatsApp to Chatwoot Message Forwarder with Media Support

https://n8nworkflows.xyz/workflows/whatsapp-to-chatwoot-message-forwarder-with-media-support-6988


# WhatsApp to Chatwoot Message Forwarder with Media Support

### 1. Workflow Overview

This workflow automates the forwarding of WhatsApp messages, including media files, received via the Evolution API into Chatwoot as conversation messages. It supports text, audio, images, videos, documents, contacts, and stickers, ensuring that media files are properly handled and validated before uploading. The workflow also checks for duplicate messages to avoid resending, manages contact creation and lookup on Chatwoot, and ensures conversations are properly initiated or reused.

Logical blocks and their roles:

- **1.1 Input Reception & Initial Processing**: Receives webhook triggers or execution from other workflows containing incoming WhatsApp messages, extracts and structures message details.

- **1.2 Message Validation & Deduplication**: Validates message type and phone number, checks Redis cache for duplicate messages to prevent resending.

- **1.3 Contact Management**: Searches for existing contacts in Chatwoot based on the phone number or creates new contacts if none are found.

- **1.4 Conversation Management**: Checks for existing open conversations in Chatwoot for the contact or creates a new conversation.

- **1.5 Message Forwarding**: Sends the outgoing or incoming message to Chatwoot, handling text and media files, including file type validation and base64 conversion.

- **1.6 Media Handling**: Downloads media files from Evolution API in base64 format, converts them to binary files, and uploads them to Chatwoot if allowed.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initial Processing

- **Overview**: Receives incoming WhatsApp message data (via webhook or other workflow execution), extracts key message fields, and structures them into a normalized message object.

- **Nodes Involved**:  
  - When Executed by Another Workflow  
  - Wait  
  - PARAMETERS  
  - CODE - MESSAGE OBJECT

- **Node Details**:

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point to receive parameters (`cd_account`, `cd_inbox_chatwoot`, `ds_msg`) from another workflow.  
    - Configuration: Accepts workflow inputs for account, inbox, and message object.  
    - Connections: Outputs to `Wait`.  
    - Failure Modes: Missing inputs or malformed parameters.

  - **Wait**  
    - Type: Wait (Webhook Trigger node)  
    - Role: Pauses execution until webhook is called, ensuring ordered processing.  
    - Configuration: Default wait with webhook ID assigned.  
    - Connections: Outputs to `PARAMETERS`.

  - **PARAMETERS**  
    - Type: Set  
    - Role: Extracts and assigns key parameters from input JSON for downstream usage.  
    - Configuration: Sets variables such as `cd_account`, `cd_inbox_chatwoot`, `ds_msg`, `cd_mensagem_evolution` (message ID), and `nm_instancia` (instance name).  
    - Connections: Outputs to `CODE - MESSAGE OBJECT`.

  - **CODE - MESSAGE OBJECT**  
    - Type: Code (JavaScript)  
    - Role: Parses the raw message data to build a structured message object with fields like contact name, contact number, message content, message validity, and attachments.  
    - Key Logic:  
      - Extracts contact key from `remoteJid`.  
      - Detects if message is sent from own number.  
      - Handles different message types (text, audio, image, video, document, reaction, sticker, contact).  
      - Parses quoted messages for context.  
      - Processes vCard data for shared contacts.  
    - Connections: Outputs to `TIPO MENSAGEM VÁLIDO?`.  
    - Edge Cases: Unsupported message types, missing fields, malformed vCard data.

---

#### 1.2 Message Validation & Deduplication

- **Overview**: Validates if the message type is supported and if the phone number is valid (Brazilian prefix "55"). Checks Redis if the message ID was already processed to avoid duplicates.

- **Nodes Involved**:  
  - TIPO MENSAGEM VÁLIDO?  
  - IS THE PHONE NUMBER VALID?  
  - REDIS - EVOLUTION MSG - GET  
  - MESSAGE SENT?  
  - REDIS - EVOLUTION MSG - SET

- **Node Details**:

  - **TIPO MENSAGEM VÁLIDO?**  
    - Type: If  
    - Role: Checks if `sn_Mensagem_Valida` flag is true (valid message types include text, audio, image, video, document, reaction, sticker, contact).  
    - Connections: If true, proceeds to phone validation.

  - **IS THE PHONE NUMBER VALID?**  
    - Type: If  
    - Role: Validates that the phone number prefix (first two digits of contact key) equals "55" (Brazil country code).  
    - Connections: If true, proceeds to Redis get.

  - **REDIS - EVOLUTION MSG - GET**  
    - Type: Redis  
    - Role: Checks if the message ID already exists in Redis cache to prevent duplicate processing.  
    - Key: Composed dynamically from account, inbox, and message ID.  
    - Connections: Outputs to `MESSAGE SENT?`.

  - **MESSAGE SENT?**  
    - Type: If  
    - Role: Checks if Redis returned an empty value, meaning the message was not processed yet.  
    - Connections: If true, proceeds to set Redis and continue processing.

  - **REDIS - EVOLUTION MSG - SET**  
    - Type: Redis  
    - Role: Marks the message ID as processed in Redis with expiration to avoid duplicate sending.  
    - Connections: Outputs to `LOAD CONTACT`.  
    - Failure Modes: Redis connection errors, key conflicts.

---

#### 1.3 Contact Management

- **Overview**: Searches for the contact in Chatwoot by phone number. If found, retrieves contact ID; if missing, creates a new contact.

- **Nodes Involved**:  
  - LOAD CONTACT  
  - IS THERE CONTACT ON CHATWOOT?  
  - EXISTING CONTACT CODE  
  - CREATE CONTACT  
  - NEW CONTACT CODE  
  - CONTACT - ID

- **Node Details**:

  - **LOAD CONTACT**  
    - Type: Chatwoot API Node (Contact Search)  
    - Role: Searches Chatwoot contacts by phone number (prefixed with “+”).  
    - Parameters: `q` set to contact number, sorted by phone number descending, scoped to account.  
    - Connections: Outputs to `IS THERE CONTACT ON CHATWOOT?`.

  - **IS THERE CONTACT ON CHATWOOT?**  
    - Type: If  
    - Role: Checks if search returned any contacts (`meta.count > 0`).  
    - Connections: True path to `EXISTING CONTACT CODE`, false path to `CREATE CONTACT`.

  - **EXISTING CONTACT CODE**  
    - Type: Set  
    - Role: Extracts the existing contact’s ID from search results and sets `cd_contact`.  
    - Connections: Outputs to `CONTACT - ID`.

  - **CREATE CONTACT**  
    - Type: Chatwoot API Node (Contact Create)  
    - Role: Creates a new contact in Chatwoot with name, phone number, inbox ID, and custom attributes (e.g., Spark name and CPF).  
    - Connections: Outputs to `NEW CONTACT CODE`.

  - **NEW CONTACT CODE**  
    - Type: Set  
    - Role: Extracts the newly created contact’s ID and sets `cd_contact`.  
    - Connections: Outputs to `CONTACT - ID`.

  - **CONTACT - ID**  
    - Type: Set  
    - Role: Standardizes the contact ID for downstream use.  
    - Connections: Outputs to `Get Conversations`.

---

#### 1.4 Conversation Management

- **Overview**: Checks if there is an open conversation for the contact in the specified Chatwoot inbox and account. If none exists, creates a new conversation.

- **Nodes Involved**:  
  - Get Conversations  
  - IS THERE CONVERSATION ON CHATWOOT?  
  - EXISTING CONVERSATION CODE  
  - CREATE CONVERSATION  
  - NEW CONVERSATION CODE  
  - CONVERSATION - ID

- **Node Details**:

  - **Get Conversations**  
    - Type: Postgres (Chatwoot database)  
    - Role: Queries the `conversations` table filtering by inbox, contact, status (open), and account, sorted by last updated date.  
    - Connections: Outputs to `IS THERE CONVERSATION ON CHATWOOT?`.

  - **IS THERE CONVERSATION ON CHATWOOT?**  
    - Type: If  
    - Role: Checks if any conversations were found (checks if `id` exists).  
    - Connections: True to `EXISTING CONVERSATION CODE`, false to `CREATE CONVERSATION`.

  - **EXISTING CONVERSATION CODE**  
    - Type: Set  
    - Role: Sets `cd_conversation` to the existing conversation’s display ID.  
    - Connections: Outputs to `CONVERSATION - ID`.

  - **CREATE CONVERSATION**  
    - Type: Chatwoot API Node (New Conversation)  
    - Role: Creates new conversation with inbox, contact, and account IDs.  
    - Connections: Outputs to `NEW CONVERSATION CODE`.

  - **NEW CONVERSATION CODE**  
    - Type: Set  
    - Role: Sets `cd_conversation` to the newly created conversation's ID.  
    - Connections: Outputs to `CONVERSATION - ID`.

  - **CONVERSATION - ID**  
    - Type: Set  
    - Role: Finalizes conversation ID for message sending nodes.  
    - Connections: Outputs to `WAS IT THE NUMBER ITSELF THAT IS CONNECTED TO EVOLUTION THAT SENT THE MESSAGE?`.

---

#### 1.5 Message Forwarding

- **Overview**: Determines if the message originated from the connected number (outgoing) or from a contact (incoming) and sends the message text or media to Chatwoot accordingly.

- **Nodes Involved**:  
  - WAS IT THE NUMBER ITSELF THAT IS CONNECTED TO EVOLUTION THAT SENT THE MESSAGE?  
  - SEND OUTGOING MESSAGE  
  - SEND INCOMING MESSAGE  
  - SENT FILE?  
  - EVOLUTION - GET FILE  
  - CHECK FILE TYPE  
  - BASE64  
  - Convert to File  
  - SEND CHATWOOT FILE  
  - SEND OUTGOING MESSAGE EXTENSION NOT ALLOWED

- **Node Details**:

  - **WAS IT THE NUMBER ITSELF THAT IS CONNECTED TO EVOLUTION THAT SENT THE MESSAGE?**  
    - Type: If  
    - Role: Checks if the message is outgoing (`sn_Proprio_Numero == true`).  
    - Connections: True to `SEND OUTGOING MESSAGE`, False to `SEND INCOMING MESSAGE`.

  - **SEND OUTGOING MESSAGE**  
    - Type: Chatwoot API Node (Create Message)  
    - Role: Sends outgoing text message to conversation with `private=true` and appropriate message type.  
    - Connections: Outputs to `SENT FILE?`.

  - **SEND INCOMING MESSAGE**  
    - Type: Chatwoot API Node (Create Message)  
    - Role: Sends incoming text message with `private=false`, includes custom attribute with Evolution message ID.  
    - Connections: Outputs to `SENT FILE?`.

  - **SENT FILE?**  
    - Type: If  
    - Role: Checks if message has attachments (`arquivos.length > 0`).  
    - Connections: True to `EVOLUTION - GET FILE`, False ends flow here for text-only.

  - **EVOLUTION - GET FILE**  
    - Type: Evolution API Node  
    - Role: Downloads media file associated with the message ID in Base64 format.  
    - Connections: Outputs to `CHECK FILE TYPE`.

  - **CHECK FILE TYPE**  
    - Type: Switch  
    - Role: Validates the media MIME type. If allowed, proceeds; if `.msi` extension or disallowed, routes to error message.  
    - Connections: Allowed path to `BASE64`, disallowed to `SEND OUTGOING MESSAGE EXTENSION NOT ALLOWED`.

  - **BASE64**  
    - Type: Code  
    - Role: Extracts Base64 data from Evolution file download for conversion.  
    - Connections: Outputs to `Convert to File`.

  - **Convert to File**  
    - Type: Convert To File  
    - Role: Converts Base64 string into binary file with correct filename and MIME type for upload.  
    - Connections: Outputs to `SEND CHATWOOT FILE`.

  - **SEND CHATWOOT FILE**  
    - Type: HTTP Request  
    - Role: Uploads the media file as multipart/form-data to Chatwoot conversation messages endpoint, specifying message type and privacy.  
    - Connections: Terminal node for media upload.

  - **SEND OUTGOING MESSAGE EXTENSION NOT ALLOWED**  
    - Type: Chatwoot API Node (Create Message)  
    - Role: Sends a warning message to Chatwoot if the uploaded file’s extension is not allowed.  
    - Connections: Terminal node for error reporting.

---

#### 1.6 Media Handling

- **Overview**: Handles the retrieval, validation, conversion, and upload of media files attached to WhatsApp messages.

- **Nodes involved**:  
  - EVOLUTION - GET FILE  
  - CHECK FILE TYPE  
  - BASE64  
  - Convert to File  
  - SEND CHATWOOT FILE

- **Node Details**:

  - **EVOLUTION - GET FILE**  
    - Downloads media in base64 from the Evolution API using message ID.

  - **CHECK FILE TYPE**  
    - Filters allowed MIME types, specifically blocking `application/x-msdownload` (.msi files).

  - **BASE64**  
    - Extracts base64 encoded file content from API response.

  - **Convert to File**  
    - Converts base64 string to a binary file with proper filename and MIME type.

  - **SEND CHATWOOT FILE**  
    - Uploads the file as a multipart form data attachment to Chatwoot conversation message.

---

### 3. Summary Table

| Node Name                                | Node Type                        | Functional Role                                         | Input Node(s)                       | Output Node(s)                         | Sticky Note                                                                                                 |
|-----------------------------------------|---------------------------------|---------------------------------------------------------|-----------------------------------|---------------------------------------|-------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow       | Execute Workflow Trigger         | Entry point to receive incoming message parameters       | Wait                              | Wait                                  |                                                                                                             |
| Wait                                    | Wait                            | Waits for webhook trigger or execution                   | When Executed by Another Workflow | PARAMETERS                           |                                                                                                             |
| PARAMETERS                              | Set                             | Sets key parameters from input JSON                       | Wait                              | CODE - MESSAGE OBJECT                |                                                                                                             |
| CODE - MESSAGE OBJECT                   | Code                            | Parses and normalizes incoming message data               | PARAMETERS                       | TIPO MENSAGEM VÁLIDO?               |                                                                                                             |
| TIPO MENSAGEM VÁLIDO?                   | If                              | Checks if message type is valid                           | CODE - MESSAGE OBJECT              | IS THE PHONE NUMBER VALID?           | ## MESSAGE TYPE Checks if it is a valid message type, text, audio, image and video                           |
| IS THE PHONE NUMBER VALID?               | If                              | Validates phone number prefix for Brazil ("55")          | TIPO MENSAGEM VÁLIDO?             | REDIS - EVOLUTION MSG - GET          | ## MESSAGE TYPE Checks if it is a valid mobile phone number by checking the IDD prefix                       |
| REDIS - EVOLUTION MSG - GET              | Redis                           | Checks if message ID was already processed                | IS THE PHONE NUMBER VALID?         | MESSAGE SENT?                      | ## CHECK IF WHATSAPP MESSAGE ID HAS ALREADY BEEN SENT Node to check if evolution API is sending same msg ID twice |
| MESSAGE SENT?                          | If                              | Determines if message is new (not previously sent)       | REDIS - EVOLUTION MSG - GET        | REDIS - EVOLUTION MSG - SET          |                                                                                                             |
| REDIS - EVOLUTION MSG - SET              | Redis                           | Marks message ID as processed in Redis                    | MESSAGE SENT?                    | LOAD CONTACT                      |                                                                                                             |
| LOAD CONTACT                          | Chatwoot Contact Search         | Searches contact by phone number in Chatwoot             | REDIS - EVOLUTION MSG - SET        | IS THERE CONTACT ON CHATWOOT?       | ## CONTATO Checks if there is already a contact registered with the cell phone number, returns or creates contact |
| IS THERE CONTACT ON CHATWOOT?            | If                              | Checks contact search results                             | LOAD CONTACT                    | EXISTING CONTACT CODE, CREATE CONTACT |                                                                                                             |
| EXISTING CONTACT CODE                  | Set                             | Sets existing contact ID                                  | IS THERE CONTACT ON CHATWOOT?      | CONTACT - ID                      |                                                                                                             |
| CREATE CONTACT                        | Chatwoot Contact Create         | Creates new contact in Chatwoot                          | IS THERE CONTACT ON CHATWOOT?      | NEW CONTACT CODE                 |                                                                                                             |
| NEW CONTACT CODE                      | Set                             | Sets new contact ID                                      | CREATE CONTACT                  | CONTACT - ID                      |                                                                                                             |
| CONTACT - ID                         | Set                             | Standardizes contact ID for next steps                   | EXISTING CONTACT CODE, NEW CONTACT CODE | Get Conversations               |                                                                                                             |
| Get Conversations                   | Postgres Query                  | Checks for open conversation for contact                 | CONTACT - ID                   | IS THERE CONVERSATION ON CHATWOOT? | ## CONVERSATION Checks if there is already an open conversation for the contact or creates a new one          |
| IS THERE CONVERSATION ON CHATWOOT?      | If                              | Determines if an open conversation exists                | Get Conversations             | EXISTING CONVERSATION CODE, CREATE CONVERSATION |                                                                                                             |
| EXISTING CONVERSATION CODE           | Set                             | Sets existing conversation ID                            | IS THERE CONVERSATION ON CHATWOOT? | CONVERSATION - ID                |                                                                                                             |
| CREATE CONVERSATION                 | Chatwoot Conversation Create   | Creates new conversation                                 | IS THERE CONVERSATION ON CHATWOOT? | NEW CONVERSATION CODE          |                                                                                                             |
| NEW CONVERSATION CODE               | Set                             | Sets new conversation ID                                | CREATE CONVERSATION           | CONVERSATION - ID                |                                                                                                             |
| CONVERSATION - ID                  | Set                             | Standardizes conversation ID for message sending         | EXISTING CONVERSATION CODE, NEW CONVERSATION CODE | WAS IT THE NUMBER ITSELF THAT IS CONNECTED TO EVOLUTION THAT SENT THE MESSAGE? |                                                                                                             |
| WAS IT THE NUMBER ITSELF THAT IS CONNECTED TO EVOLUTION THAT SENT THE MESSAGE? | If                              | Checks if message is outgoing (sent by own number)      | CONVERSATION - ID              | SEND OUTGOING MESSAGE, SEND INCOMING MESSAGE |                                                                                                             |
| SEND OUTGOING MESSAGE              | Chatwoot Message Create         | Sends outgoing text message                               | WAS IT THE NUMBER ITSELF THAT IS CONNECTED TO EVOLUTION THAT SENT THE MESSAGE? | SENT FILE?                  |                                                                                                             |
| SEND INCOMING MESSAGE              | Chatwoot Message Create         | Sends incoming text message                               | WAS IT THE NUMBER ITSELF THAT IS CONNECTED TO EVOLUTION THAT SENT THE MESSAGE? | SENT FILE?                  |                                                                                                             |
| SENT FILE?                         | If                              | Checks if message has attachments                        | SEND OUTGOING MESSAGE, SEND INCOMING MESSAGE | EVOLUTION - GET FILE, (end if no file) |                                                                                                             |
| EVOLUTION - GET FILE               | Evolution API                   | Downloads media file in base64 from Evolution API        | SENT FILE?                   | CHECK FILE TYPE                |                                                                                                             |
| CHECK FILE TYPE                   | Switch                         | Validates media MIME type, blocks disallowed extensions | EVOLUTION - GET FILE          | BASE64, SEND OUTGOING MESSAGE EXTENSION NOT ALLOWED |                                                                                                             |
| BASE64                           | Code                           | Extracts Base64 content from API response                 | CHECK FILE TYPE (ALLOWED)     | Convert to File               |                                                                                                             |
| Convert to File                  | Convert To File                | Converts Base64 string to binary file                     | BASE64                       | SEND CHATWOOT FILE            |                                                                                                             |
| SEND CHATWOOT FILE              | HTTP Request                   | Uploads media file to Chatwoot conversation              | Convert to File              | (end)                         |                                                                                                             |
| SEND OUTGOING MESSAGE EXTENSION NOT ALLOWED | Chatwoot Message Create         | Sends warning message for unsupported file extension    | CHECK FILE TYPE (.MSI)        | (end)                         |                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
     - Define inputs: `cd_account` (number), `cd_inbox_chatwoot` (number), `ds_msg` (object).  

   - Add a **Wait** node named `Wait` connected from `When Executed by Another Workflow`.  

2. **Parameter Extraction**  
   - Add a **Set** node named `PARAMETERS` connected from `Wait`.  
   - Configure to assign:  
     - `cd_account` from input JSON  
     - `cd_inbox_chatwoot`  
     - `ds_msg`  
     - `cd_mensagem_evolution` from `ds_msg.data.key.id`  
     - `nm_instancia` from `ds_msg.instance`

3. **Message Object Construction**  
   - Add a **Code** node named `CODE - MESSAGE OBJECT` connected from `PARAMETERS`.  
   - Implement JavaScript to parse `ds_msg` for contact info, message text, type, validity, attachments array, and quoted messages.  
   - Include vCard parsing helper function.  

4. **Message Type Validation**  
   - Add an **If** node `TIPO MENSAGEM VÁLIDO?` connected from `CODE - MESSAGE OBJECT`.  
   - Check if `sn_Mensagem_Valida` equals true.  

5. **Phone Number Validation**  
   - Add an **If** node `IS THE PHONE NUMBER VALID?` connected from `TIPO MENSAGEM VÁLIDO?`.  
   - Check if phone number prefix equals `"55"`.

6. **Duplicate Message Check**  
   - Add a **Redis** node `REDIS - EVOLUTION MSG - GET` connected from `IS THE PHONE NUMBER VALID?`.  
   - Configure Redis credentials.  
   - Key: `wa_{{cd_account}}_{{cd_inbox_chatwoot}}_{{cd_mensagem_evolution}}`.  

   - Add an **If** node `MESSAGE SENT?` connected from Redis GET.  
   - Condition: If Redis value is empty (message not sent).  

   - Add a **Redis** node `REDIS - EVOLUTION MSG - SET` connected from `MESSAGE SENT?`.  
   - Same key as GET, set value to message ID with expiration.  

7. **Contact Lookup & Creation**  
   - Add a **Chatwoot Contact Search** node `LOAD CONTACT` connected from Redis SET.  
   - Search by phone number with `q = +{{nr_Chave_Contato}}`.  

   - Add an **If** node `IS THERE CONTACT ON CHATWOOT?` connected from `LOAD CONTACT`.  
   - Check if `meta.count > 0`.  

   - On true, add a **Set** node `EXISTING CONTACT CODE` to extract contact ID from search results.  
   - On false, add a **Chatwoot Contact Create** node `CREATE CONTACT`.  
   - Set contact details including name, phone number, inbox ID, and custom attributes.  

   - Add a **Set** node `NEW CONTACT CODE` to extract new contact ID.  

   - Merge outputs to a **Set** node `CONTACT - ID` that standardizes contact ID.  

8. **Conversation Lookup & Creation**  
   - Add a **Postgres** node `Get Conversations` connected from `CONTACT - ID`.  
   - Query `conversations` table filtering by inbox, contact ID, status=0 (open), and account ID.  
   - Sort by `updated_at` descending, limit 1.  

   - Add an **If** node `IS THERE CONVERSATION ON CHATWOOT?` connected from Postgres node.  
   - Check if conversation `id` exists.  

   - On true: add a **Set** node `EXISTING CONVERSATION CODE` to set conversation display ID.  
   - On false: add a **Chatwoot Conversation Create** node `CREATE CONVERSATION`.  
   - Set inbox, account, and contact IDs.  

   - Add a **Set** node `NEW CONVERSATION CODE` to set new conversation ID.  

   - Merge to a **Set** node `CONVERSATION - ID` for standardizing conversation ID.  

9. **Message Sending Decision**  
   - Add an **If** node `WAS IT THE NUMBER ITSELF THAT IS CONNECTED TO EVOLUTION THAT SENT THE MESSAGE?` connected from `CONVERSATION - ID`.  
   - Condition on `sn_Proprio_Numero` (true if own number).  

   - On true: add **Chatwoot Message Create** node `SEND OUTGOING MESSAGE`.  
   - Parameters:  
     - content: message text  
     - conversation ID  
     - message type: outgoing  
     - private: true  

   - On false: add **Chatwoot Message Create** node `SEND INCOMING MESSAGE`.  
   - Parameters:  
     - content: message text  
     - conversation ID  
     - message type: incoming  
     - private: false  
     - content_attributes: include Evolution message ID  

10. **File Attachment Handling**  
    - Add an **If** node `SENT FILE?` connected from both message send nodes.  
    - Check if `arquivos.length > 0`.  

    - On true:  
      - Add **Evolution API** node `EVOLUTION - GET FILE` to download media base64 using message ID and instance name.  

      - Add **Switch** node `CHECK FILE TYPE` to check MIME type and allow or block `.msi` extension.  

      - For allowed types:  
        - Add **Code** node `BASE64` to extract base64 string from API response.  
        - Add **Convert To File** node to convert base64 to binary file with proper filename and MIME type.  
        - Add **HTTP Request** node `SEND CHATWOOT FILE` to upload file to Chatwoot conversation using multipart/form-data.  

      - For blocked types (`.msi`):  
        - Add **Chatwoot Message Create** node `SEND OUTGOING MESSAGE EXTENSION NOT ALLOWED` to send warning message.  

    - On false: end.

11. **Set Up Credentials**  
    - Configure Redis credentials for Redis nodes.  
    - Configure Evolution API credentials for Evolution API node.  
    - Configure Chatwoot API credentials for Chatwoot nodes.  
    - Configure Postgres credentials for Postgres node.  

12. **Add Sticky Notes**  
    - Add sticky notes near logical blocks with contents from the original workflow for documentation and clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow supports forwarding WhatsApp messages (text and media) from Evolution API to Chatwoot with media validation.        | Workflow description                                                                                |
| Media files with `.msi` extension are blocked and a warning message is sent to Chatwoot.                                      | Media handling block                                                                                |
| Contact creation includes custom attributes such as `nm_spark` and `nr_cpf` to enrich contact data.                           | Contact creation node                                                                               |
| Redis is used to prevent duplicate message forwarding by caching message IDs with expiration.                                | Deduplication block                                                                                |
| For vCard contacts, message is parsed to extract name and phone number and formatted with emojis for Chatwoot message.       | CODE - MESSAGE OBJECT node JavaScript code                                                        |
| PostgreSQL database query on Chatwoot conversations table uses `status=0` to find open conversations for routing messages.   | Conversation lookup node                                                                            |
| Chatwoot API nodes require API key credentials and properly scoped permissions to create contacts, conversations, and messages.| Credential configuration                                                                            |
| Workflow is not active by default; ensure activation after importing or rebuilding in n8n.                                    | Workflow metadata                                                                                   |

---

**Disclaimer**: The provided text stems exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.
Sync two-way support conversations between Telegram and Zoho Desk

https://n8nworkflows.xyz/workflows/sync-two-way-support-conversations-between-telegram-and-zoho-desk-20991


# Sync two-way support conversations between Telegram and Zoho Desk

### 1. Workflow Overview

This workflow provides a two-way synchronization mechanism between Telegram bot conversations and Zoho Desk. Its primary purpose is to allow customer support agents to handle Telegram support inquiries directly within Zoho Desk. When an end-user sends or edits a Telegram message, the workflow matches or provisions a Zoho Desk contact and routes the message into an active or newly created support ticket. Conversely, when an agent publishes a comment on a Zoho Desk ticket, the system triggers a webhook, looks up the associated Telegram chat, and delivers the agent's response back to the user on Telegram.

The workflow logic is divided into two distinct functional paths:
- **Telegram-to-Zoho Desk Ingestion Block:** Handles incoming Telegram messages, normalizes metadata, maps user identifiers via an internal n8n Data Table, provisions missing Zoho Desk contacts, evaluates open tickets, and appends messages as internal ticket comments.
- **Zoho Desk-to-Telegram Outbound Block:** Receives comment webhooks from Zoho Desk, validates event relevance (filtering out internal comments or unmapped contacts), fetches the comment content, and pushes the reply to the correct Telegram chat.
- **Initial Setup & Provisioning Block:** Automatically creates the user lookup table and registers the Zoho Desk webhook endpoint.

---

### 2. Block-by-Block Analysis

#### 2.1 Initial Setup & Provisioning
- **Overview:** Initializes the underlying n8n Data Table required for user mapping and establishes the Zoho Desk webhook subscription to capture agent responses.
- **Nodes Involved:** `Initial Setup`, `Create User Table`, `ZDesk: List Webhooks`, `ZDesk: Create Webhook`
- **Node Details:**
  - `Initial Setup`
    - Type: `n8n-nodes-base.manualTrigger`
    - Role: Manual entry point to trigger one-time configuration tasks.
    - Input: None
    - Output: `Create User Table`
    - Failure Types: N/A (Manual trigger)
  - `Create User Table`
    - Type: `n8n-nodes-base.dataTable`
    - Role: Creates the user lookup table (`oz_zohoMessengerIntegration_userLookup`) with columns for `telegram_user_id`, `zdesk_contact_id`, and `telegram_chat_id` if it does not already exist.
    - Configuration: Operation set to `create`, table name configured, `createIfNotExists` enabled.
    - Input: `Initial Setup`
    - Output: `ZDesk: List Webhooks`
    - Failure Types: Table name conflicts or database access permissions.
  - `ZDesk: List Webhooks`
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Queries existing webhooks in Zoho Desk to inspect current registrations.
    - Configuration: GET request to `https://desk.zoho.com/api/v1/webhooks`, uses Generic OAuth2 API credentials.
    - Input: `Create User Table`
    - Output: `ZDesk: Create Webhook`
    - Failure Types: Invalid or expired OAuth2 credentials, incorrect Zoho data center URL.
  - `ZDesk: Create Webhook`
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Registers a webhook in Zoho Desk for `Ticket_Comment_Add` and `Ticket_Comment_Update` events, pointing to the n8n execution resume/webhook URL.
    - Configuration: POST request to `https://desk.zoho.com/api/v1/webhooks`, JSON body configured with event subscriptions, name, description, ignore source ID, and target URL derived from `$execution.resumeUrl`.
    - Key Expressions: `={{ $execution.resumeUrl.split('/webhook-waiting/')[0] + '/webhook/send_messenger_reply' }}`
    - Input: `ZDesk: List Webhooks`
    - Output: None (Terminal node for this branch)
    - Failure Types: Webhook URL unreachable by Zoho servers, duplicate webhook registrations.

---

#### 2.2 Telegram Message Reception & Normalization
- **Overview:** Listens for incoming messages or edits from a Telegram bot and standardizes the payload into a clean format containing user details, chat identifiers, and message content.
- **Nodes Involved:** `Telegram Trigger`, `Normalize Input`
- **Node Details:**
  - `Telegram Trigger`
    - Type: `n8n-nodes-base.telegramTrigger`
    - Role: Listens for webhook events from Telegram when messages are sent or edited.
    - Configuration: Updates monitored: `message`, `edited_message`. Credentials: `telegramApi`.
    - Input: None
    - Output: `Normalize Input`
    - Failure Types: Telegram API token revocation, webhook connection drops.
  - `Normalize Input`
    - Type: `n8n-nodes-base.set`
    - Role: Extracts and normalizes message text, user IDs, names, and chat IDs from either standard messages or edited messages.
    - Configuration: Sets explicit assignments for department ID, message text, message ID, update flag, user ID, username, first name, last name, and chat ID.
    - Key Expressions: 
      - `message`: `={{ $json.message?.text ?? $json.edited_message?.text ?? '' }}`
      - `message_id`: `={{ $json.message?.message_id ?? $json.edited_message.message_id }}`
      - `is_update`: `={{ !!$json.edited_message }}`
      - `user_id`: `={{ $json.message?.from?.id ?? $json.edited_message.from.id }}`
      - `username`: `={{ $json.message?.from?.username ?? $json.edited_message.from.username }}`
      - `first_name`: `={{ $json.message?.from?.first_name ?? $json.edited_message.from.first_name }}`
      - `last_name`: `={{ $json.message?.from?.last_name ?? $json.edited_message.last_name }}`
      - `chat_id`: `={{ $json.message?.chat?.id ?? $json.edited_message.chat.id }}`
    - Input: `Telegram Trigger`
    - Output: `Find or Create Telegram User Mapping`
    - Failure Types: Missing payload fields causing undefined property evaluation errors.

---

#### 2.3 User Mapping & Contact Provisioning
- **Overview:** Manages the association between Telegram users and Zoho Desk contacts by checking an internal data table, creating a Zoho Desk contact if necessary, and persisting the resulting identifiers.
- **Nodes Involved:** `Find or Create Telegram User Mapping`, `Zoho Desk Contact Linked?`, `ZDesk: Create Contact`, `Save Zoho Desk Contact Mapping`
- **Node Details:**
  - `Find or Create Telegram User Mapping`
    - Type: `n8n-nodes-base.dataTable`
    - Role: Upserts user records in the lookup data table using `telegram_user_id` and `telegram_chat_id` as matching criteria.
    - Configuration: Operation set to `upsert`, filtered on `telegram_user_id` and `telegram_chat_id`.
    - Key Expressions: 
      - `telegram_chat_id`: `={{ $json.chat_id }}`
      - `telegram_user_id`: `={{ $json.user_id }}`
    - Input: `Normalize Input`
    - Output: `Zoho Desk Contact Linked?`
    - Failure Types: Data table unavailability or locked database states.
  - `Zoho Desk Contact Linked?`
    - Type: `n8n-nodes-base.if`
    - Role: Verifies whether the lookup record already contains a linked Zoho Desk contact ID.
    - Configuration: Checks if `{{ $json.zdesk_contact_id }}` is not empty.
    - Input: `Find or Create Telegram User Mapping`
    - Output: 
      - True (Contact exists): Connects to `ZDesk: Get Tickets`
      - False (Contact missing): Connects to `ZDesk: Create Contact`
    - Failure Types: Evaluation errors due to unexpected data types.
  - `ZDesk: Create Contact`
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Creates a new contact in Zoho Desk when no existing mapping is found.
    - Configuration: POST request to `https://desk.zoho.com/api/v1/contacts`. Body includes `lastName`, `firstName`, and `description`.
    - Key Expressions:
      - `lastName`: `={{ $('Normalize Input').item.json.last_name ?? $('Normalize Input').item.json.username }}`
      - `firstName`: `={{ $('Normalize Input').item.json.first_name }}`
      - `description`: `=Created by message from {{ $json.source }}`
    - Input: `Zoho Desk Contact Linked?` (False branch)
    - Output: `Save Zoho Desk Contact Mapping`
    - Failure Types: API validation errors (e.g., duplicate email/phone constraints), invalid OAuth scopes (`Desk.contacts.CREATE`).
  - `Save Zoho Desk Contact Mapping`
    - Type: `n8n-nodes-base.dataTable`
    - Role: Updates the internal data table with the newly generated Zoho Desk contact ID.
    - Configuration: Operation set to `upsert`, updates `zdesk_contact_id` filtered by user ID and chat ID.
    - Key Expressions:
      - `zdesk_contact_id`: `={{ $json.id }}`
    - Input: `ZDesk: Create Contact`
    - Output: `ZDesk: Get Tickets`
    - Failure Types: Mismatched filter keys preventing record updates.

---

#### 2.4 Ticket Matching & Comment Appending
- **Overview:** Retrieves existing tickets for the contact, selects an active open ticket or creates a new one, and appends the Telegram message as an internal comment.
- **Nodes Involved:** `ZDesk: Get Tickets`, `Select Existing Open Ticket`, `Open Ticket Exists?`, `ZDesk: Create Ticket`, `ZDesk: Append User Comment`
- **Node Details:**
  - `ZDesk: Get Tickets`
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Fetches all tickets associated with the resolved Zoho Desk contact ID.
    - Configuration: GET request to `https://desk.zoho.com/api/v1/contacts/{contactId}/tickets`.
    - Key Expressions: `={{ $('Find or Create Telegram User Mapping').item.json.zdesk_contact_id ?? $('ZDesk: Create Contact').item.json.id }}`
    - Input: `Zoho Desk Contact Linked?` (True branch) or `Save Zoho Desk Contact Mapping`
    - Output: `Select Existing Open Ticket`
    - Failure Types: Non-existent contact ID in API path, timeout errors.
  - `Select Existing Open Ticket`
    - Type: `n8n-nodes-base.set`
    - Role: Parses the ticket list response to locate the most recent open ticket (excluding closed statuses).
    - Configuration: Assigns a parsed ticket object or `null` to the `ticket` variable.
    - Key Expressions: `={{ JSON.parse($json.data).data.find((t) => t.status != "Closed" && t.status != "Closed bad") || null }}`
    - Input: `ZDesk: Get Tickets`
    - Output: `Open Ticket Exists?`
    - Failure Types: Malformed JSON responses from Zoho API.
  - `Open Ticket Exists?`
    - Type: `n8n-nodes-base.if`
    - Role: Determines whether an open ticket was found or if a new ticket needs to be created.
    - Configuration: Checks if `{{ $json.ticket }}` is not empty.
    - Input: `Select Existing Open Ticket`
    - Output:
      - True (Open ticket found): Connects to `ZDesk: Append User Comment`
      - False (No open ticket): Connects to `ZDesk: Create Ticket`
    - Failure Types: Conditional evaluation failures.
  - `ZDesk: Create Ticket`
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Creates a new support ticket in Zoho Desk when no open ticket is available.
    - Configuration: POST request to `https://desk.zoho.com/api/v1/tickets`. Payload includes `subject`, `contactId`, and `departmentId`.
    - Key Expressions:
      - `subject`: `=New Message from Telegram by {{ $('Normalize Input').item.json.first_name ? `${$('Normalize Input').item.json.first_name}${$('Normalize Input').item.json.last_name ? ' '+$('Normalize Input').item.json.last_name : ''}` : $('Normalize Input').item.json.username }}`
      - `contactId`: `={{ $('Find or Create Telegram User Mapping').item.json.zdesk_contact_id ?? $('ZDesk: Create Contact').item.json.id }}`
      - `departmentId`: `={{ $('Normalize Input').item.json.zohoDesk_departmenId }}`
    - Input: `Open Ticket Exists?` (False branch)
    - Output: `ZDesk: Append User Comment`
    - Failure Types: Invalid department ID, missing contact association, missing `Desk.tickets.CREATE` scope.
  - `ZDesk: Append User Comment`
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Appends the incoming Telegram message (or message edit notice) as a private plain-text comment to the target ticket.
    - Configuration: POST request to `https://desk.zoho.com/api/v1/tickets/{ticketId}/comments`. Payload defines `content`, `isPublic` (false), and `contentType` (plainText).
    - Key Expressions:
      - URL: `=https://desk.zoho.com/api/v1/tickets/{{ $json.ticket?.id ?? $json.id }}/comments`
      - Content: `={{ $('Normalize Input').item.json.is_update ? `[User Updated Message]: ${ $('Normalize Input').item.json.message }` : `[New User Message]: ${ $('Normalize Input').item.json.message }` }}`
    - Input: `Open Ticket Exists?` (True branch) or `ZDesk: Create Ticket`
    - Output: None (Terminal node for ingestion path)
    - Failure Types: Ticket closure state preventing comment addition, API rate limits.

---

#### 2.5 Outbound Agent Replies (Zoho Desk to Telegram)
- **Overview:** Receives webhook notifications when agents add comments to tickets in Zoho Desk, filters out internal or unmapped interactions, and routes agent replies back to the correct Telegram chat.
- **Nodes Involved:** `Receive Zoho Desk Comment Webhook`, `ZDesk: Get Ticket`, `Get user by zdesk ticket`, `Stop if unrelated comment`, `ZDesk: Get Comment`, `Send Agent Reply to Telegram`
- **Node Details:**
  - `Receive Zoho Desk Comment Webhook`
    - Type: `n8n-nodes-base.webhook`
    - Role: Entry point capturing incoming webhook events (`Ticket_Comment_Add`, `Ticket_Comment_Update`) from Zoho Desk.
    - Configuration: Webhook path set to `send_messenger_reply`, HTTP method POST, allowed origins `*`.
    - Input: None
    - Output: `ZDesk: Get Ticket`
    - Failure Types: Network routing blocks, invalid webhook payload structure.
  - `ZDesk: Get Ticket`
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Retrieves detailed ticket and contact information from Zoho Desk using the ticket ID provided in the webhook payload.
    - Configuration: GET request to `https://desk.zoho.com/api/v1/tickets/{ticketId}?include=contacts`.
    - Key Expressions: `=https://desk.zoho.com/api/v1/tickets/{{ $json.body[0].payload.ticketId }}?include=contacts`
    - Input: `Receive Zoho Desk Comment Webhook`
    - Output: `Get user by zdesk ticket`
    - Failure Types: Deleted tickets or permission errors.
  - `Get user by zdesk ticket`
    - Type: `n8n-nodes-base.dataTable`
    - Role: Queries the user lookup data table to find the Telegram chat ID linked to the Zoho Desk contact ID.
    - Configuration: Operation set to `get`, matching `zdesk_contact_id` against `$json.contact.id`, returns 1 match with `alwaysOutputData` enabled.
    - Key Expressions: `zdesk_contact_id` filter set to `={{ $json.contact.id }}`
    - Input: `ZDesk: Get Ticket`
    - Output: `Stop if unrelated comment`
    - Failure Types: Unmapped contacts yielding empty data table rows.
  - `Stop if unrelated comment`
    - Type: `n8n-nodes-base.filter`
    - Role: Drops execution flow if the comment is internal/private (`isPublic` is false) or if no user mapping was found.
    - Configuration: Conditions require that the data table item is not empty AND `isPublic` is strictly false (note: Zoho's `isPublic` payload flag indicates public agent visibility; evaluated against `false` to ensure replies are sent).
    - Key Expressions:
      - Condition 1: `={{ !$json.isEmpty() }}`
      - Condition 2: `={{ $('Receive Zoho Desk Comment Webhook').item.json.body[0].payload.isPublic }}` (evaluated as boolean check)
    - Input: `Get user by zdesk ticket`
    - Output: `ZDesk: Get Comment`
    - Failure Types: Unexpected boolean inversions in webhook payloads.
  - `ZDesk: Get Comment`
    - Type: `n8n-nodes-base.httpRequest`
    - Role: Retrieves the plain-text content of the specific comment added or updated in Zoho Desk.
    - Configuration: GET request to `https://desk.zoho.com/api/v1/tickets/{ticketId}/comments/{commentId}` with query parameter `include=plainText`.
    - Key Expressions: URL uses `{{ $('Receive Zoho Desk Comment Webhook').item.json.body[0].payload.ticketId }}` and `{{ $('Receive Zoho Desk Comment Webhook').item.json.body[0].payload.id }}`.
    - Input: `Stop if unrelated comment`
    - Output: `Send Agent Reply to Telegram`
    - Failure Types: API errors fetching specific comment payloads.
  - `Send Agent Reply to Telegram`
    - Type: `n8n-nodes-base.telegram`
    - Role: Sends the agent's comment text back to the customer's Telegram chat.
    - Configuration: Action set to send message, HTML parse mode enabled, credentials set to `telegramApi`.
    - Key Expressions:
      - Text: `={{ $json.plainText }}`
      - Chat ID: `={{ $('Get user by zdesk ticket').item.json.telegram_chat_id }}`
    - Input: `ZDesk: Get Comment`
    - Output: None (Terminal node for outbound path)
    - Failure Types: Blocked bot interactions (e.g., user stopped the bot), invalid HTML formatting in agent replies.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Initial Setup | `n8n-nodes-base.manualTrigger` | Manual entry point for provisioning | None | Create User Table | Initial Setup<br>Just click here once to create the database table required for this workflow and register the Zoho Desk webhook to this n8n's endpoint |
| Create User Table | `n8n-nodes-base.dataTable` | Creates the user lookup data table | Initial Setup | ZDesk: List Webhooks | Initial Setup<br>Just click here once to create the database table required for this workflow and register the Zoho Desk webhook to this n8n's endpoint |
| ZDesk: List Webhooks | `n8n-nodes-base.httpRequest` | Lists existing Zoho Desk webhooks | Create User Table | ZDesk: Create Webhook | Initial Setup<br>Just click here once to create the database table required for this workflow and register the Zoho Desk webhook to this n8n's endpoint |
| ZDesk: Create Webhook | `n8n-nodes-base.httpRequest` | Registers the Zoho Desk comment webhook | ZDesk: List Webhooks | None | Initial Setup<br>Just click here once to create the database table required for this workflow and register the Zoho Desk webhook to this n8n's endpoint |
| Telegram Trigger | `n8n-nodes-base.telegramTrigger` | Listens for incoming Telegram messages | None | Normalize Input | Telegram Message received |
| Normalize Input | `n8n-nodes-base.set` | Normalizes Telegram message properties | Telegram Trigger | Find or Create Telegram User Mapping | Telegram Message received |
| Find or Create Telegram User Mapping | `n8n-nodes-base.dataTable` | Upserts Telegram user in lookup table | Normalize Input | Zoho Desk Contact Linked? | Find Zoho Desk Contact<br>We check if the n8n table already contains zoho desk contact id, and if not - we create a new one |
| Zoho Desk Contact Linked? | `n8n-nodes-base.if` | Checks if Zoho Desk contact ID exists | Find or Create Telegram User Mapping | ZDesk: Get Tickets, ZDesk: Create Contact | Find Zoho Desk Contact<br>We check if the n8n table already contains zoho desk contact id, and if not - we create a new one |
| ZDesk: Create Contact | `n8n-nodes-base.httpRequest` | Creates a new Zoho Desk contact | Zoho Desk Contact Linked? | Save Zoho Desk Contact Mapping | Find Zoho Desk Contact<br>We check if the n8n table already contains zoho desk contact id, and if not - we create a new one |
| Save Zoho Desk Contact Mapping | `n8n-nodes-base.dataTable` | Saves new contact ID to lookup table | ZDesk: Create Contact | ZDesk: Get Tickets | Find Zoho Desk Contact<br>We check if the n8n table already contains zoho desk contact id, and if not - we create a new one |
| ZDesk: Get Tickets | `n8n-nodes-base.httpRequest` | Fetches tickets for the contact | Zoho Desk Contact Linked?, Save Zoho Desk Contact Mapping | Select Existing Open Ticket | Find Ticket<br>We fetch all tickets, and find the latest open one |
| Select Existing Open Ticket | `n8n-nodes-base.set` | Identifies the active open ticket | ZDesk: Get Tickets | Open Ticket Exists? | Find Ticket<br>We fetch all tickets, and find the latest open one |
| Open Ticket Exists? | `n8n-nodes-base.if` | Checks if an open ticket was found | Select Existing Open Ticket | ZDesk: Append User Comment, ZDesk: Create Ticket | Sending comment<br>We either select an existing, or create a new ticket, and then append a new user's comment to this ticket |
| ZDesk: Create Ticket | `n8n-nodes-base.httpRequest` | Creates a new support ticket | Open Ticket Exists? | ZDesk: Append User Comment | Sending comment<br>We either select an existing, or create a new ticket, and then append a new user's comment to this ticket |
| ZDesk: Append User Comment | `n8n-nodes-base.httpRequest` | Appends user message as ticket comment | Open Ticket Exists?, ZDesk: Create Ticket | None | Sending comment<br>We either select an existing, or create a new ticket, and then append a new user's comment to this ticket |
| Receive Zoho Desk Comment Webhook | `n8n-nodes-base.webhook` | Receives comment webhooks from Zoho | None | ZDesk: Get Ticket | New Zoho Desk comment<br>We receive a new comment webhook by Zoho Desk and get the related ticket |
| ZDesk: Get Ticket | `n8n-nodes-base.httpRequest` | Fetches ticket and contact details | Receive Zoho Desk Comment Webhook | Get user by zdesk ticket | New Zoho Desk comment<br>We receive a new comment webhook by Zoho Desk and get the related ticket |
| Get user by zdesk ticket | `n8n-nodes-base.dataTable` | Looks up Telegram chat by contact ID | ZDesk: Get Ticket | Stop if unrelated comment | Check if the comment is relevant<br>We discard the event in case the ticket points to unknown user, or the comment is private |
| Stop if unrelated comment | `n8n-nodes-base.filter` | Filters out private or unmapped comments | Get user by zdesk ticket | ZDesk: Get Comment | Check if the comment is relevant<br>We discard the event in case the ticket points to unknown user, or the comment is private |
| ZDesk: Get Comment | `n8n-nodes-base.httpRequest` | Fetches plain text of the agent comment | Stop if unrelated comment | Send Agent Reply to Telegram | Sending agent response<br>We fetch the plain text of the comment and send it back to the user |
| Send Agent Reply to Telegram | `n8n-nodes-base.telegram` | Sends agent reply to Telegram chat | ZDesk: Get Comment | None | Sending agent response<br>We fetch the plain text of the comment and send it back to the user |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Create the Data Table
1. Create a new n8n Data Table named `oz_zohoMessengerIntegration_userLookup`.
2. Define the following columns:
   - `telegram_user_id` (String)
   - `telegram_chat_id` (String)
   - `zdesk_contact_id` (String)

#### Step 2: Set Up Credentials
1. **Telegram API:** Configure a Telegram Bot via BotFather, copy the Bot Token, and create a `telegramApi` credential in n8n named `crmoz_publicWorkflow2bot`.
2. **Zoho Desk OAuth2:** 
   - Register a server-based application in the [Zoho API Console](https://api-console.zoho.com/#web).
   - Set Homepage and Authorized Redirect URI to `https://<your-n8n-domain>/rest/oauth2-credential/callback`.
   - Configure OAuth2 credentials in n8n (`PublicWorkflow2`):
     - **Grant Type:** Authorization Code
     - **Authorization URL:** `https://accounts.zoho.com/oauth/v2/auth` (adjust TLD for your data center, e.g., `.eu`)
     - **Access Token URL:** `https://accounts.zoho.com/oauth/v2/token`
     - **Client ID & Client Secret:** Insert your Zoho App credentials.
     - **Scope:** `Desk.contacts.CREATE,Desk.contacts.READ,Desk.contacts.UPDATE,Desk.tickets.CREATE,Desk.tickets.READ,Desk.tickets.UPDATE,Desk.webhooks.CREATE,Desk.webhooks.READ,Desk.webhooks.UPDATE`
     - **Auth URI Query Parameters:** `access_type=offline`
     - **Authentication:** Header

#### Step 3: Build the Ingestion Pipeline (Telegram to Zoho Desk)
1. **Telegram Trigger:** Add a node of type `n8n-nodes-base.telegramTrigger`, select updates `message` and `edited_message`, and assign your Telegram credential.
2. **Normalize Input:** Add an `n8n-nodes-base.set` node connected to the trigger. Add assignments for:
   - `zohoDesk_departmenId` = `1145788+1234567890` (replace with your actual Zoho department ID)
   - `message` = `={{ $json.message?.text ?? $json.edited_message?.text ?? '' }}`
   - `message_id` = `={{ $json.message?.message_id ?? $json.edited_message.message_id }}`
   - `is_update` = `={{ !!$json.edited_message }}`
   - `user_id` = `={{ $json.message?.from?.id ?? $json.edited_message.from.id }}`
   - `username` = `={{ $json.message?.from?.username ?? $json.edited_message.from.username }}`
   - `first_name` = `={{ $json.message?.from?.first_name ?? $json.edited_message.from.first_name }}`
   - `last_name` = `={{ $json.message?.from?.last_name ?? $json.edited_message.last_name }}`
   - `chat_id` = `={{ $json.message?.chat?.id ?? $json.edited_message.chat.id }}`
3. **Find or Create Telegram User Mapping:** Add a Data Table node, select table `oz_zohoMessengerIntegration_userLookup`, set operation to `upsert`, matching on `telegram_user_id` and `telegram_chat_id`.
4. **Zoho Desk Contact Linked?:** Add an `n8n-nodes-base.if` node checking if `{{ $json.zdesk_contact_id }}` is not empty.
5. **ZDesk: Create Contact (False branch):** Add an HTTP Request node (POST to `https://desk.zoho.com/api/v1/contacts`) using Zoho OAuth2 credentials. Map body fields `lastName`, `firstName`, and `description`.
6. **Save Zoho Desk Contact Mapping:** Add a Data Table node (upsert operation) to update `zdesk_contact_id` (`={{ $json.id }}`) filtered by user ID and chat ID. Connect `ZDesk: Create Contact` here.
7. **ZDesk: Get Tickets:** Add an HTTP Request node (GET to `https://desk.zoho.com/api/v1/contacts/{{ $('Find or Create Telegram User Mapping').item.json.zdesk_contact_id ?? $('ZDesk: Create Contact').item.json.id }}/tickets`). Connect both the `True` branch of `Zoho Desk Contact Linked?` and `Save Zoho Desk Contact Mapping` to this node.
8. **Select Existing Open Ticket:** Add a Set node using JavaScript expression: `={{ JSON.parse($json.data).data.find((t) => t.status != "Closed" && t.status != "Closed bad") || null }}`.
9. **Open Ticket Exists?:** Add an IF node checking if `{{ $json.ticket }}` is not empty.
10. **ZDesk: Create Ticket (False branch):** Add an HTTP Request node (POST to `https://desk.zoho.com/api/v1/tickets`) with `subject`, `contactId`, and `departmentId`.
11. **ZDesk: Append User Comment:** Add an HTTP Request node (POST to `https://desk.zoho.com/api/v1/tickets/{{ $json.ticket?.id ?? $json.id }}/comments`). Set body parameters: `content` (with update/new message prefix), `isPublic` = `false`, `contentType` = `plainText`. Connect both the `True` branch of `Open Ticket Exists?` and `ZDesk: Create Ticket` here.

#### Step 4: Build the Outbound Pipeline (Zoho Desk to Telegram)
1. **Receive Zoho Desk Comment Webhook:** Add a Webhook node with path `send_messenger_reply`, HTTP method `POST`.
2. **ZDesk: Get Ticket:** Add an HTTP Request node (GET to `https://desk.zoho.com/api/v1/tickets/{{ $json.body[0].payload.ticketId }}?include=contacts`).
3. **Get user by zdesk ticket:** Add a Data Table node (get operation) querying `oz_zohoMessengerIntegration_userLookup` where `zdesk_contact_id` equals `={{ $json.contact.id }}`.
4. **Stop if unrelated comment:** Add a Filter node with conditions ensuring the data table output is not empty and `isPublic` evaluates correctly.
5. **ZDesk: Get Comment:** Add an HTTP Request node (GET to `https://desk.zoho.com/api/v1/tickets/{{ $('Receive Zoho Desk Comment Webhook').item.json.body[0].payload.ticketId }}/comments/{{ $('Receive Zoho Desk Comment Webhook').item.json.body[0].payload.id }}` with query parameter `include=plainText`).
6. **Send Agent Reply to Telegram:** Add a Telegram node set to send a message. Set text to `={{ $json.plainText }}` and Chat ID to `={{ $('Get user by zdesk ticket').item.json.telegram_chat_id }}`. Enable HTML parse mode.

#### Step 5: Build the Initial Setup Pipeline
1. **Initial Setup:** Add a Manual Trigger node.
2. **Create User Table:** Add a Data Table node to create `oz_zohoMessengerIntegration_userLookup`.
3. **ZDesk: List Webhooks:** Add an HTTP Request node (GET to `https://desk.zoho.com/api/v1/webhooks`).
4. **ZDesk: Create Webhook:** Add an HTTP Request node (POST to `https://desk.zoho.com/api/v1/webhooks`) configuring subscriptions for `Ticket_Comment_Add` and `Ticket_Comment_Update`, pointing to the n8n webhook URL.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Zoho API Console | [Zoho API Console](https://api-console.zoho.com/#web) |
| Configuring Zoho Credentials | Ensure your data center domain matches your Zoho account (e.g., `.com` vs `.eu`) for all API URLs. |
| Telegram Bot Setup | Create and manage bots via [@BotFather](https://t.me/BotFather). |
| Webhook Registration Prerequisite | Publish the workflow before running the manual "Initial Setup" trigger so that the webhook endpoint is active and verifiable by Zoho Desk. |
| Optional Zoho Desk Notification Adjustment | To prevent contacts from receiving duplicate email notifications for public comments, navigate to Zoho Desk Settings > Customization > Notifications > Adding a public comment and set to false. |
LinkedIn Auto Message Router and Responder with Request Detection

https://n8nworkflows.xyz/workflows/linkedin-auto-message-router-and-responder-with-request-detection-4889


# LinkedIn Auto Message Router and Responder with Request Detection

### 1. Workflow Overview

This workflow, titled **"LinkedIn Auto Message Router and Responder with Request Detection"**, is designed to automate the processing and response to inbound LinkedIn messages received via the Unipile platform. It enables intelligent routing, context enrichment, AI-generated replies, and Slack-based approval workflows for managing LinkedIn communications efficiently.

**Target Use Cases:**  
- Automate first-level handling of LinkedIn messages for individuals or organizations.  
- Enrich incoming message data with LinkedIn user or organization profile information.  
- Generate AI-powered suggested replies based on historical requests and responses stored in an external Notion database.  
- Route messages appropriately in Slack for manual review, approval, or archival.  
- Support interaction with Slack buttons to approve or archive replies, triggering message sending back to LinkedIn via Unipile APIs.

**Logical Blocks:**  
- **1.1 Input Reception:** Receive LinkedIn messages via webhook and filter messages sent by the account owner.  
- **1.2 Data Enrichment:** Retrieve LinkedIn user or organization data from Unipile API.  
- **1.3 AI Processing:** Aggregate request routing data from Notion, generate AI responses with LangChain/OpenAI integration.  
- **1.4 Message Routing & Slack Notification:** Decide message type (user vs organization), send notifications to Slack with interactive buttons for approval.  
- **1.5 Slack Button Handling:** Process Slack button presses, approve or archive messages, send responses to LinkedIn, and manage Slack message deletion.  
- **1.6 Auxiliary & Global Variable Management:** Maintain global variables and handle payload parsing for consistent data flow.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives incoming LinkedIn messages via a webhook, sets global variables for message context, and filters out messages sent by the account owner to avoid self-processing.

**Nodes Involved:**  
- Webhook  
- Global Variables  
- Check if from me  
- New Message  
- Response, No action  
- Sticky Note (LinkedIn Message Processing)

**Node Details:**  

- **Webhook**  
  - Type: Webhook (HTTP POST)  
  - Role: Entry point receiving LinkedIn message payloads from Unipile webhook.  
  - Config: POST method, unique webhook path.  
  - Inputs: External HTTP POST requests.  
  - Outputs: JSON message payload forwarded downstream.  
  - Edge Cases: Invalid or malformed webhook calls, rate limits.

- **Global Variables**  
  - Type: Set node  
  - Role: Extracts and stores key fields from incoming JSON (message text, sender info, message ID, chat ID).  
  - Config: Maps fields like `type`, `message`, `message_id`, `chat_id`, `sender`, `sender_profile`, `sender_id` from webhook payload.  
  - Inputs: Webhook output.  
  - Outputs: Structured JSON with global variables.  
  - Edge Cases: Missing or unexpected fields in payload.

- **Check if from me**  
  - Type: If node  
  - Role: Checks if the sender ID matches the account owner IDs to avoid responding to self-sent messages.  
  - Config: Two conditions combined with OR: sender_id not equal to one owner ID or equal to another (specific IDs used).  
  - Inputs: Global Variables output.  
  - Outputs: Routes messages to "New Message" if from others, or "Response, No action" if from self.  
  - Edge Cases: Incorrect sender_id formatting, IDs needing update if account changes.

- **New Message**  
  - Type: NoOp (no operation placeholder)  
  - Role: Placeholder for messages to be processed further.  
  - Inputs: Check if from me [true path].  
  - Outputs: Connects to data enrichment.  

- **Response, No action**  
  - Type: NoOp  
  - Role: Placeholder for ignoring messages from self.  
  - Inputs: Check if from me [false path].  
  - Outputs: Ends workflow path.

- **Sticky Note (LinkedIn Message Processing)**  
  - Provides guidance on setting up the webhook in Unipile dashboard.

---

#### 1.2 Data Enrichment

**Overview:**  
Enriches LinkedIn message data by querying Unipile APIs for user or organization profile information. Data is shaped into a consistent format for downstream AI processing.

**Nodes Involved:**  
- New Message (entry)  
- Get Linkedin User Data from Unipile  
- Set User Data from Unipile  
- Group in one object - User  
- Get Linkedin Org Data from Unipile  
- Set Linkedin Org Data from Unipile  
- Group in one object - Org  
- Set unable to find data object  
- Set all potential Inputs  
- Sticky Notes (LinkedIn logos, credential notes)

**Node Details:**  

- **Get Linkedin User Data from Unipile**  
  - Type: HTTP Request  
  - Role: Retrieves LinkedIn user profile info via Unipile API using sender provider ID.  
  - Config: GET request with header auth (X-API-KEY), query param account_id fixed.  
  - Inputs: From "New Message" node.  
  - Outputs: User profile JSON or error (onError set to continue).  
  - Edge Cases: User not found, API errors, timeout.

- **Set User Data from Unipile**  
  - Type: Set node  
  - Role: Maps Unipile user JSON fields into standardized variables (first_name, last_name, headline, follower count, etc.) including a 'type' field as "user".  
  - Inputs: User data from HTTP request.  
  - Outputs: Structured user profile object.  

- **Group in one object - User**  
  - Type: Aggregate  
  - Role: Combines user data into a single aggregated object field named `linkedinprofile`.  
  - Inputs: Set User Data output.  
  - Outputs: Aggregated user profile.  

- **Get Linkedin Org Data from Unipile**  
  - Type: HTTP Request  
  - Role: Queries Unipile API for organization data if user data not found or as fallback.  
  - Config: Similar to user data request but different API endpoint.  
  - Inputs: From "Get Linkedin User Data" node error output.  
  - Outputs: Organization JSON or error.  
  - Edge Cases: Org not found, API errors.

- **Set Linkedin Org Data from Unipile**  
  - Type: Set node  
  - Role: Maps organization JSON fields into standardized variables with `type` set to "organization".  
  - Inputs: HTTP request output.  
  - Outputs: Structured org profile object.

- **Group in one object - Org**  
  - Type: Aggregate  
  - Role: Aggregates organization data into `linkedinprofile` field.  
  - Inputs: Set Linkedin Org Data output.  
  - Outputs: Aggregated org profile.  

- **Set unable to find data object**  
  - Type: Set node  
  - Role: Creates fallback object with type "none" and message "Unable to find LinkedIn Data" if neither user nor org data found.  
  - Inputs: Org HTTP error output.  
  - Outputs: Fallback profile object.  

- **Set all potential Inputs**  
  - Type: Set node  
  - Role: Combines all enriched data and original message inputs into one object for AI processing.  
  - Inputs: Aggregated user/org profile or fallback, plus original global variables.  
  - Outputs: Unified data object.  

- **Sticky Notes**  
  - Provide instructions for credential updates and Unipile UserID configuration.

---

#### 1.3 AI Processing

**Overview:**  
Fetches a Notion database of past request-response mappings, formats it for AI context, aggregates into one item, and uses an AI agent with OpenAI GPT model to generate a tailored LinkedIn message reply.

**Nodes Involved:**  
- Isolate parent workflow data for AI  
- Get Request Router Directory Database (Notion)  
- Format DB data for AI Context  
- Aggregate DB objects into one item  
- AI Agent (LangChain agent with OpenAI)  
- OpenAI Chat Model  
- Structured Output Parser  
- Simple Memory  
- Sticky Notes (Notion, AI response generation)

**Node Details:**  

- **Isolate parent workflow data for AI**  
  - Type: Set node  
  - Role: Extracts sender name, message, and chat ID from enriched input for AI prompt.  
  - Inputs: "Set all potential Inputs" output.  
  - Outputs: Simplified object for AI input.  

- **Get Request Router Directory Database**  
  - Type: Notion node  
  - Role: Retrieves all pages from a Notion database containing example requests and preferred responses.  
  - Inputs: "Isolate parent workflow data for AI" output.  
  - Outputs: List of database pages.  
  - Edge Cases: Notion API limits, credential expiry.

- **Format DB data for AI Context**  
  - Type: Set node  
  - Role: Formats each Notion page into a string combining request example, description, and action.  
  - Inputs: Notion DB output.  
  - Outputs: Formatted strings per page.  

- **Aggregate DB objects into one item**  
  - Type: Aggregate  
  - Role: Combines all formatted strings into a single string to pass as context to AI.  
  - Inputs: Formatted DB data.  
  - Outputs: Single aggregated string.

- **AI Agent**  
  - Type: LangChain agent  
  - Role: Generates AI response to incoming LinkedIn message using provided context, including request database and LinkedIn profile info.  
  - Config: Custom prompt template with system message instructing tone and behavior, uses GPT-4o model.  
  - Inputs: Aggregated DB objects, isolated message data.  
  - Outputs: JSON with fields `output` (reply text) and `found` (boolean if matched example found).  
  - Edge Cases: API errors, prompt format issues, unexpected output format.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Underlying language model used by AI Agent.  
  - Config: GPT-4o model, linked with OpenAI API credentials.  

- **Structured Output Parser**  
  - Type: LangChain output parser  
  - Role: Parses AI output from raw text into structured JSON format for further use.  

- **Simple Memory**  
  - Type: Memory Buffer Window (LangChain)  
  - Role: Maintains session context based on chat ID for conversation continuity.  
  - Inputs: AI Agent output.  
  - Outputs: Memory object passed back to AI Agent.

- **Sticky Notes**  
  - Explain the dynamic, evolving nature of the AI bot referencing Notion DB, and the rationale for AI-generated responses.

---

#### 1.4 Message Routing & Slack Notification

**Overview:**  
Decides whether the LinkedIn message is from a user or organization, then sends a formatted Slack message to predefined users with interactive approval buttons and suggested AI replies.

**Nodes Involved:**  
- Decide on Message Type (Switch)  
- Send User Message (Slack)  
- Send Org Message (Slack)  
- Send Nothing Found Message (Slack)  
- Sticky Notes (Slack Bot Button Press, credential notes)

**Node Details:**  

- **Decide on Message Type**  
  - Type: Switch node  
  - Role: Routes messages based on the `type` field in LinkedIn profile object ("user" or "organization").  
  - Config: Two rules checking equality to "user" or "organization", with fallback output "extra".  
  - Inputs: AI Agent output via "Set all potential Inputs".  
  - Outputs: To respective Slack message nodes or default.

- **Send User Message**  
  - Type: Slack node  
  - Role: Sends a Slack message to a specific user (hardcoded user ID), including profile card, incoming message, and AI suggested reply with interactive buttons for approval or archiving.  
  - Config: Uses Slack blocks UI with dynamic content from LinkedIn profile and AI output.  
  - Inputs: Route from Decide on Message Type "User".  
  - Outputs: Message posted on Slack.  
  - Credentials: Slack API with OAuth2.  
  - Edge Cases: Slack API rate limits, invalid user ID.

- **Send Org Message**  
  - Type: Slack node  
  - Role: Similar to Send User Message but formats the message card for organization profiles.  
  - Inputs: Route from Decide on Message Type "Organization".  
  - Outputs: Slack message posted.  

- **Send Nothing Found Message**  
  - Type: Slack node  
  - Role: Sends a generic message when message type is unknown or no profile data found, also including AI suggested reply and buttons.  
  - Inputs: Route from Decide on Message Type fallback.  
  - Outputs: Slack message posted.

- **Sticky Notes**  
  - Instructions for Slack app interactivity setup, including setting webhook URL and app manifest link.

---

#### 1.5 Slack Button Handling

**Overview:**  
Handles button presses from Slack messages (approve or archive), processes the payload, sends approved replies back to LinkedIn via Unipile API, and deletes Slack messages accordingly.

**Nodes Involved:**  
- Slack Button Press (Webhook)  
- Verify Webhook (Respond to Slack challenge)  
- Parse Webhook (Set node to parse payload)  
- User Switcher (Switch node to route by Slack user)  
- Isolate Message and Payload  
- Isolate Payload  
- Send webhook to Coworker Workflow (HTTP request forwarding)  
- Check if approved (If node)  
- Send Message to LinkedIn via Unipile (HTTP Request)  
- Delete Approved Message from Slack (Slack delete message)  
- Delete Denied Message from Slack (Slack delete message)  
- Sticky Notes (Slack Bot Button Press)

**Node Details:**  

- **Slack Button Press**  
  - Type: Webhook (HTTP POST)  
  - Role: Entry point for Slack interactive button presses.  
  - Config: POST method, responds with challenge for Slack verification.  
  - Inputs: Slack button press payloads.  
  - Outputs: Forwards to Verify Webhook node.  

- **Verify Webhook**  
  - Type: RespondToWebhook  
  - Role: Responds to Slack's verification challenge with proper JSON response.  
  - Inputs: Slack Button Press node.  
  - Outputs: Forwards to Parse Webhook.  

- **Parse Webhook**  
  - Type: Set node  
  - Role: Extracts the payload object from Slack webhook body for easier processing.  
  - Inputs: Verify Webhook output.  
  - Outputs: Parsed payload forwarded.  

- **User Switcher**  
  - Type: Switch node  
  - Role: Routes the Slack button presses based on the user ID who clicked the button (supports multiple Slack users).  
  - Inputs: Parse Webhook output.  
  - Outputs: Two outputs for two different Slack users.  

- **Isolate Message and Payload**  
  - Type: Set node  
  - Role: Extracts the actual user input text from Slack interactive message and the button action payload.  
  - Inputs: User Switcher output.  
  - Outputs: Structured message and action value.  

- **Isolate Payload**  
  - Type: Set node  
  - Role: Extracts payload object for further HTTP forwarding.  
  - Inputs: User Switcher output.  
  - Outputs: Payload object.  

- **Send webhook to Coworker Workflow**  
  - Type: HTTP Request  
  - Role: Forwards Slack button press payload to an external coworker workflow via webhook for additional handling or logging.  
  - Inputs: Isolate Payload output.  
  - Outputs: Response from external workflow.  

- **Check if approved**  
  - Type: If node  
  - Role: Checks if the Slack button action is "approve".  
  - Inputs: Isolate Message and Payload output.  
  - Outputs: Routes to sending LinkedIn message or deleting denied message.

- **Send Message to LinkedIn via Unipile**  
  - Type: HTTP Request  
  - Role: Posts the approved reply message back to LinkedIn chat via Unipile API.  
  - Config: POST with text message, uses generic header auth with X-API-KEY.  
  - Inputs: Check if approved [true].  
  - Outputs: Triggers deletion of Slack approval message.  
  - Edge Cases: API errors, message send failure.

- **Delete Approved Message from Slack**  
  - Type: Slack node  
  - Role: Deletes the Slack message after approval to clean up interface.  
  - Inputs: Send Message to LinkedIn output.  

- **Delete Denied Message from Slack**  
  - Type: Slack node  
  - Role: Deletes Slack message when action is not approved (archived).  
  - Inputs: Check if approved [false].

- **Sticky Notes**  
  - Instructions for Slack app button press setup and credentials.

---

#### 1.6 Auxiliary & Global Variable Management

**Overview:**  
Supporting nodes managing data isolation, session memory, and global data assignment to ensure smooth data flow and state management throughout the workflow.

**Nodes Involved:**  
- Set all potential Inputs  
- Isolate parent workflow data for AI  
- Simple Memory  

**Node Details:**  

- **Set all potential Inputs**  
  - Type: Set node  
  - Role: Collates enriched LinkedIn profile data and original message context into one comprehensive JSON object passed to AI and routing logic.  
  - Inputs: Aggregated user/org data or fallback object.  
  - Outputs: Unified input for AI and routing decision nodes.

- **Isolate parent workflow data for AI**  
  - Type: Set node  
  - Role: Simplifies the input object to only key fields needed by AI agent (sender name, message, chat ID).  
  - Inputs: Set all potential Inputs output.  

- **Simple Memory**  
  - Type: LangChain memoryBufferWindow  
  - Role: Maintains conversation memory keyed by chat ID for context continuity in AI agent interactions.  

---

### 3. Summary Table

| Node Name                        | Node Type                       | Functional Role                           | Input Node(s)                         | Output Node(s)                         | Sticky Note                                                                                                           |
|---------------------------------|--------------------------------|-----------------------------------------|-------------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Webhook                         | Webhook                        | Receive LinkedIn incoming messages      | -                                   | Global Variables                      | Add me to Unipile Webhook Dashboard [Link](https://dashboard.unipile.com/webhooks)                                     |
| Global Variables                | Set                           | Extract key fields from incoming data   | Webhook                             | Check if from me                     |                                                                                                                       |
| Check if from me                | If                            | Filter out messages sent by self        | Global Variables                    | New Message / Response, No action    | Update with your Unipile UserID                                                                                        |
| New Message                    | NoOp                          | Placeholder for further processing      | Check if from me                    | Get Linkedin User Data from Unipile  |                                                                                                                       |
| Response, No action            | NoOp                          | Ignore own messages                      | Check if from me                    | -                                   |                                                                                                                       |
| Get Linkedin User Data from Unipile | HTTP Request                  | Fetch LinkedIn user profile              | New Message                        | Set User Data from Unipile           | Update Credentials: Use generic header auth "X-API-KEY" [Link](https://dashboard.unipile.com/access-tokens)           |
| Set User Data from Unipile     | Set                           | Map user profile fields                  | Get Linkedin User Data from Unipile | Group in one object - User           |                                                                                                                       |
| Group in one object - User     | Aggregate                     | Aggregate user profile fields            | Set User Data from Unipile          | Set all potential Inputs              |                                                                                                                       |
| Get Linkedin Org Data from Unipile | HTTP Request                  | Fetch LinkedIn organization profile      | Get Linkedin User Data from Unipile (error) | Set Linkedin Org Data from Unipile / Set unable to find data object | Update Credentials: Use generic header auth "X-API-KEY" [Link](https://dashboard.unipile.com/access-tokens)           |
| Set Linkedin Org Data from Unipile | Set                           | Map organization profile fields          | Get Linkedin Org Data from Unipile  | Group in one object - Org            |                                                                                                                       |
| Group in one object - Org      | Aggregate                     | Aggregate organization profile fields    | Set Linkedin Org Data from Unipile  | Set all potential Inputs             |                                                                                                                       |
| Set unable to find data object | Set                           | Fallback when no LinkedIn data found     | Get Linkedin Org Data from Unipile (error) | Set all potential Inputs          |                                                                                                                       |
| Set all potential Inputs       | Set                           | Consolidate enriched data and message    | Group in one object - User / Org / fallback | Isolate parent workflow data for AI |                                                                                                                       |
| Isolate parent workflow data for AI | Set                           | Prepare AI input fields                   | Set all potential Inputs            | Get Request Router Directory Database |                                                                                                                       |
| Get Request Router Directory Database | Notion                        | Retrieve request-response database        | Isolate parent workflow data for AI | Format DB data for AI Context        | AI bot references a shared internal database for dynamic routing and responses                                         |
| Format DB data for AI Context  | Set                           | Format Notion DB pages for AI context    | Get Request Router Directory Database | Aggregate DB objects into one item   |                                                                                                                       |
| Aggregate DB objects into one item | Aggregate                     | Combine DB entries into single string    | Format DB data for AI Context       | AI Agent                           |                                                                                                                       |
| AI Agent                      | LangChain agent               | Generate AI response to LinkedIn message | Aggregate DB objects into one item / Isolate parent workflow data for AI | Decide on Message Type              | AI generates personalized replies based on message and request database                                                |
| OpenAI Chat Model             | LangChain OpenAI Model        | Underlying GPT model for AI Agent         | -                                 | AI Agent                           | Update Credentials                                                                                                    |
| Structured Output Parser      | LangChain Output Parser       | Parse AI output into structured JSON     | OpenAI Chat Model                  | AI Agent                           |                                                                                                                       |
| Simple Memory                 | LangChain Memory Buffer       | Maintain chat session memory              | Isolate parent workflow data for AI | AI Agent                           |                                                                                                                       |
| Decide on Message Type        | Switch                       | Route message by LinkedIn profile type   | AI Agent                         | Send User Message / Send Org Message / Send Nothing Found Message |                                                                                                                       |
| Send User Message             | Slack                        | Send Slack notification for user messages | Decide on Message Type            | -                                   | Add me to your Slack App Interactivity and Shortcuts [Link](https://dashboard.unipile.com/webhooks)                    |
| Send Org Message              | Slack                        | Send Slack notification for organization messages | Decide on Message Type            | -                                   | Add me to your Slack App Interactivity and Shortcuts [Link](https://dashboard.unipile.com/webhooks)                    |
| Send Nothing Found Message    | Slack                        | Send Slack message when no profile found  | Decide on Message Type            | -                                   |                                                                                                                       |
| Slack Button Press            | Webhook                      | Receive Slack button press events          | -                                 | Verify Webhook                     | Add me to your Slack App Interactivity and Shortcuts [Link](https://dashboard.unipile.com/webhooks)                    |
| Verify Webhook               | RespondToWebhook             | Respond to Slack verification challenge   | Slack Button Press                | Parse Webhook                      |                                                                                                                       |
| Parse Webhook                | Set                         | Extract payload from Slack webhook         | Verify Webhook                   | User Switcher                     |                                                                                                                       |
| User Switcher               | Switch                      | Route Slack events by user ID               | Parse Webhook                   | Isolate Message and Payload / Isolate Payload |                                                                                                                       |
| Isolate Message and Payload | Set                         | Extract message text and button action     | User Switcher                   | Check if approved                 |                                                                                                                       |
| Isolate Payload             | Set                         | Extract payload object                      | User Switcher                   | Send webhook to Coworker Workflow |                                                                                                                       |
| Send webhook to Coworker Workflow | HTTP Request                  | Forward Slack interaction to external workflow | Isolate Payload               | -                                   |                                                                                                                       |
| Check if approved           | If                          | Check if action is "approve"                | Isolate Message and Payload     | Send Message to LinkedIn / Delete Denied Message from Slack |                                                                                                                       |
| Send Message to LinkedIn via Unipile | HTTP Request                  | Send approved reply back to LinkedIn via API | Check if approved [true]        | Delete Approved Message from Slack | Update Credentials: Use generic header auth "X-API-KEY" [Link](https://dashboard.unipile.com/access-tokens)           |
| Delete Approved Message from Slack | Slack                        | Delete Slack message after approval          | Send Message to LinkedIn via Unipile | -                                   |                                                                                                                       |
| Delete Denied Message from Slack | Slack                        | Delete Slack message after denial/archive    | Check if approved [false]        | -                                   |                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node (Webhook):**  
   - Method: POST  
   - Path: Unique identifier string  
   - Purpose: Receive LinkedIn messages from Unipile webhook.

2. **Add a Set node (Global Variables):**  
   - Extract and map from webhook JSON:  
     - `type` from `body.account_info.type`  
     - `message` from `body.message`  
     - `message_id` from `body.message_id`  
     - `chat_id` from `body.chat_id`  
     - `sender` from `body.sender.attendee_name`  
     - `sender_profile` from `body.sender.attendee_profile_url`  
     - `sender_id` from `body.sender.attendee_id`

3. **Add an If node (Check if from me):**  
   - Condition: Sender ID not equal to your own user IDs (replace with your actual IDs).  
   - True output: Continue processing (New Message), False output: No action.

4. **Add NoOp nodes (New Message and Response, No action):**  
   - For message routing.

5. **From New Message, add HTTP Request node (Get Linkedin User Data from Unipile):**  
   - Method: GET  
   - URL: `https://api9.unipile.com:13976/api/v1/users/{{ $json.body.sender.attendee_provider_id }}`  
   - Query param: `account_id` (your account ID)  
   - Headers: `accept: application/json`  
   - Authentication: Generic header auth using `X-API-KEY` with your Unipile API key.

6. **Add Set node (Set User Data from Unipile):**  
   - Map user profile fields from API response to variables (first_name, last_name, headline, follower_count, etc.)  
   - Add a field `type` with value `"user"`.

7. **Add Aggregate node (Group in one object - User):**  
   - Aggregate all mapped user profile fields into an array field named `linkedinprofile`.

8. **From Get Linkedin User Data node's error output, add HTTP Request node (Get Linkedin Org Data from Unipile):**  
   - Method: GET  
   - URL: `https://api9.unipile.com:13976/api/v1/linkedin/company/{{ $json.body.sender.attendee_provider_id }}`  
   - Same authentication and headers as user data request.

9. **Add Set node (Set Linkedin Org Data from Unipile):**  
   - Map organization profile fields similarly, including `type` = `"organization"`.

10. **Add Aggregate node (Group in one object - Org):**  
    - Aggregate mapped org fields into `linkedinprofile`.

11. **From Org data node error output, add Set node (Set unable to find data object):**  
    - Set `linkedinprofile` to `"Unable to find LinkedIn Data"` and `type` to `"none"`.

12. **Add Set node (Set all potential Inputs):**  
    - Combine the aggregated user/org/fallback profile data with original global variables.

13. **Add Set node (Isolate parent workflow data for AI):**  
    - Extract `sender`, `message`, `chatid` from combined inputs.

14. **Add Notion node (Get Request Router Directory Database):**  
    - Retrieve all pages from your Notion request-response database.  
    - Configure Notion API credentials.

15. **Add Set node (Format DB data for AI Context):**  
    - For each Notion page, create a formatted string combining request example, description, and action.

16. **Add Aggregate node (Aggregate DB objects into one item):**  
    - Combine all formatted DB strings into a single string for AI context.

17. **Add LangChain Agent node (AI Agent):**  
    - Configure prompt with system message instructing AI to generate personalized LinkedIn replies using context from Notion DB and LinkedIn profile.  
    - Use GPT-4o model via OpenAI Chat Model node.  
    - Provide credentials for OpenAI API.  
    - Link output parser node to parse JSON output from AI.  
    - Use Simple Memory node linked as AI memory buffer with session key per chat ID.

18. **Add Switch node (Decide on Message Type):**  
    - Route by `linkedinprofile.type` field: "user" → Send User Message, "organization" → Send Org Message, fallback → Send Nothing Found Message.

19. **Create Slack nodes (Send User Message, Send Org Message, Send Nothing Found Message):**  
    - Use Slack API credentials.  
    - Send blocks UI messages with profile info, incoming message, AI suggested reply input box, and buttons for Approve and Archive.  
    - Set user ID to your Slack user where notifications will appear.

20. **Create Webhook node (Slack Button Press):**  
    - POST method at unique path to receive Slack interactive button events.

21. **Add RespondToWebhook node (Verify Webhook):**  
    - Send back Slack challenge response.

22. **Add Set node (Parse Webhook):**  
    - Extract payload object from Slack webhook body.

23. **Add Switch node (User Switcher):**  
    - Route button presses by Slack user ID for multi-user support.

24. **Add Set nodes (Isolate Message and Payload, Isolate Payload):**  
    - Extract message input text and action value from Slack interaction payload.

25. **Add HTTP Request node (Send webhook to Coworker Workflow):**  
    - Forward Slack button actions to external workflow via webhook URL.

26. **Add If node (Check if approved):**  
    - Condition: Check if action value is "approve".

27. **Add HTTP Request node (Send Message to LinkedIn via Unipile):**  
    - POST reply message text to Unipile chat message endpoint with authentication.

28. **Add Slack nodes (Delete Approved Message from Slack, Delete Denied Message from Slack):**  
    - Delete Slack message to clean interface after approval or archive.

29. **Connect all nodes according to described logic and test end-to-end.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Add me to Unipile Webhook Dashboard — create webhook in Unipile dashboard and link to Webhook node.       | https://dashboard.unipile.com/webhooks                                                             |
| Add me to your Slack App Interactivity and Shortcuts — set Slack app’s interactivity URL to Slack Button Press webhook. Use provided App Manifest template for scopes. | https://dashboard.unipile.com/webhooks                                                             |
| Update Credentials — Use generic header auth with header name `X-API-KEY` for Unipile API requests.       | https://dashboard.unipile.com/access-tokens                                                        |
| AI Bot references a shared internal Notion database for dynamic, updatable request routing and responses. | Notion Notion API Setup                                                                             |
| Workflow supports multi-user Slack routing by user ID for button press handling.                           | Slack user IDs configured in User Switcher node                                                    |
| AI response tone guidelines: friendly, confident, professional, lightly personalized; avoid robotic or sales pitch tone. | Defined in AI Agent system message prompt                                                          |

---

**Disclaimer:**  
The provided workflow originates exclusively from an automated n8n workflow. It complies with all content policies and contains no illegal or protected data. All processed data is legal and public.
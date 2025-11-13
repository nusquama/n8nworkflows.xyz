Summarize & Reply to Slack Channel Messages with Claude AI Assistant

https://n8nworkflows.xyz/workflows/summarize---reply-to-slack-channel-messages-with-claude-ai-assistant-5846


# Summarize & Reply to Slack Channel Messages with Claude AI Assistant

### 1. Workflow Overview

This workflow automates summarizing recent unread messages from a specified Slack channel and posting a concise summary with suggested replies back to Slack as an ephemeral message. It is designed to be triggered via a Slack slash command, enabling users to quickly get AI-generated insights and response suggestions for ongoing conversations.

**Target Use Cases:**  
- Slack users wanting quick summaries of recent channel activity  
- Teams needing AI-assisted synthesis of Slack threads for decision-making  
- Automated conversational AI assistance inside Slack via Claude AI  

**Logical Blocks:**  
- **1.1 Input Reception:** Receives and parses the Slack slash command webhook payload.  
- **1.2 Slack Data Fetching:** Retrieves recent unread messages from the specified Slack channel.  
- **1.3 AI Prompt Preparation:** Formats the fetched messages into a textual prompt.  
- **1.4 AI Summarization:** Sends the prompt to Claude AI for summarization and suggested replies.  
- **1.5 Slack Message Construction:** Builds Slack block message payload with the summary.  
- **1.6 Slack Response Posting:** Posts the ephemeral summary message back to Slack for the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Accepts an incoming POST request triggered by a Slack slash command and extracts essential parameters for subsequent API calls.

- **Nodes Involved:**  
  - *Slash Command Webhook*  
  - *Parse Request*

- **Node Details:**  

  - **Slash Command Webhook**  
    - *Type & Role:* Webhook node to receive HTTP POST requests from Slack slash commands.  
    - *Configuration:* Path set to `/summarize`, HTTP method POST. No additional options enabled.  
    - *Key Expressions:* None (raw webhook input used).  
    - *Connections:* Output connected to *Parse Request*.  
    - *Edge Cases:* Unauthorized requests (Slack token verification not explicitly done here), malformed requests.  
    - *Version:* v1.

  - **Parse Request**  
    - *Type & Role:* Code node to extract relevant data fields from the webhook JSON payload.  
    - *Configuration:* JavaScript extracts `text, user_id, channel_id, token, trigger_id` from the incoming request body; returns `channel_id, token, user_id`.  
    - *Key Expressions:*  
      ```js
      const { text, user_id, channel_id, token, trigger_id } = $json.body;
      return { json: { channel_id, token, user_id } };
      ```  
    - *Connections:* Input from *Slash Command Webhook*, output to *Fetch Unread Messages*.  
    - *Edge Cases:* Missing fields in the request body, runtime code errors.  
    - *Version:* v2.

#### 1.2 Slack Data Fetching

- **Overview:**  
  Calls Slack's Conversations API to retrieve the latest messages from the specified channel.

- **Nodes Involved:**  
  - *Fetch Unread Messages*

- **Node Details:**  

  - **Fetch Unread Messages**  
    - *Type & Role:* HTTP Request node to call Slack API endpoint `conversations.history`.  
    - *Configuration:*  
      - URL: `https://slack.com/api/conversations.history`  
      - Query parameters: `channel` set dynamically to `{{$json.channel_id}}`, `limit` to 20 messages.  
      - Authorization header: Bearer token dynamically from `{{$json.token}}`.  
    - *Key Expressions:*  
      - Query: `={{ $json.channel_id }}`, `20`  
      - Header: `=Bearer {{$json.token}}`  
    - *Connections:* Input from *Parse Request*, output to *Prepare Prompt for Claude*.  
    - *Edge Cases:*  
      - API rate limits, invalid or expired tokens, channel not found, no messages returned.  
    - *Version:* v1.

#### 1.3 AI Prompt Preparation

- **Overview:**  
  Transforms the raw Slack messages into a formatted prompt string, suitable for AI summarization.

- **Nodes Involved:**  
  - *Prepare Prompt for Claude*

- **Node Details:**  

  - **Prepare Prompt for Claude**  
    - *Type & Role:* Code node that formats message objects into a readable prompt.  
    - *Configuration:*  
      - Iterates over `$json.messages` array, formats each message as `[timestamp] <@user>: text`, joined by newlines.  
      - Returns JSON with `prompt` text plus metadata for channel, token, and user.  
    - *Key Expressions:*  
      ```js
      const messages = $json.messages || [];
      const formatted = messages.map(m => `[${m.ts}] <@${m.user}>: ${m.text}`).join('\n');
      return { json: { prompt: formatted, channel_id: $json.channel_id, token: $json.token, user_id: $json.user_id } };
      ```  
    - *Connections:* Input from *Fetch Unread Messages*, output to *Claude AI: Summarize*.  
    - *Edge Cases:* Empty message list, malformed message objects, missing fields.  
    - *Version:* v2.

#### 1.4 AI Summarization

- **Overview:**  
  Sends the prepared prompt to an Anthropic Claude AI model to generate grouped thread summaries with suggested replies.

- **Nodes Involved:**  
  - *Claude AI: Summarize*  
  - *Anthropic Chat Model*

- **Node Details:**  

  - **Claude AI: Summarize**  
    - *Type & Role:* LangChain chain LLM node that orchestrates AI summarization using an attached chat model.  
    - *Configuration:*  
      - Passes a system message instructing summarization of Slack messages into grouped threads including brief summaries and 2–3 suggested replies per thread.  
    - *Key Expressions:* Fixed message:  
      `"Summarize these Slack messages into grouped threads with brief summaries and 2–3 suggested replies for each."`  
    - *Connections:* Input from *Prepare Prompt for Claude*, output to *Build Slack Blocks*.  
    - *Version:* v1.

  - **Anthropic Chat Model**  
    - *Type & Role:* AI language model node configured with Anthropic Claude Sonnet 4 model.  
    - *Configuration:*  
      - Model: `claude-sonnet-4-20250514` (Claude 4 Sonnet variant)  
      - Credentials: Anthropic API Key required.  
    - *Connections:* Connected as AI language model backend to *Claude AI: Summarize*.  
    - *Edge Cases:* API key issues, request timeouts, model unavailability.  
    - *Version:* v1.3.

#### 1.5 Slack Message Construction

- **Overview:**  
  Builds a JSON payload with Slack Block Kit formatting to present the summary as an ephemeral message.

- **Nodes Involved:**  
  - *Build Slack Blocks*

- **Node Details:**  

  - **Build Slack Blocks**  
    - *Type & Role:* Code node that constructs Slack message blocks using the AI summary text.  
    - *Configuration:*  
      - Uses the AI response text (`$json.text`) as the summary content.  
      - Builds a simple Slack block section with markdown text starting with "*Here's a summary:*".  
      - Includes channel, user, and token for posting.  
    - *Key Expressions:*  
      ```js
      const summary = $json.text;
      return { json: {
        channel: $json.channel_id,
        user: $json.user_id,
        token: $json.token,
        blocks: [
          { type: 'section', text: { type: 'mrkdwn', text: '*Here\'s a summary:*\n' + summary } }
        ]
      }};
      ```  
    - *Connections:* Input from *Claude AI: Summarize*, output to *Post to Slack (Ephemeral)*.  
    - *Edge Cases:* Missing or empty AI summary text, malformed Slack block JSON.  
    - *Version:* v2.

#### 1.6 Slack Response Posting

- **Overview:**  
  Posts the constructed ephemeral message back to Slack, visible only to the user who triggered the slash command.

- **Nodes Involved:**  
  - *Post to Slack (Ephemeral)*

- **Node Details:**  

  - **Post to Slack (Ephemeral)**  
    - *Type & Role:* HTTP Request node to Slack API `chat.postEphemeral`.  
    - *Configuration:*  
      - URL: `https://slack.com/api/chat.postEphemeral`  
      - Headers: Authorization Bearer token and Content-Type application/json, both dynamically assigned.  
      - Body: JSON payload from *Build Slack Blocks*.  
    - *Key Expressions:*  
      - Header Authorization: `=Bearer {{$json.token}}`  
    - *Connections:* Input from *Build Slack Blocks*.  
    - *Edge Cases:* Invalid tokens, permission errors, rate limiting, malformed request body, Slack API downtime.  
    - *Version:* v1.

---

### 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                      | Input Node(s)          | Output Node(s)          | Sticky Note                               |
|---------------------------|----------------------------------------|------------------------------------|------------------------|-------------------------|-------------------------------------------|
| Slash Command Webhook      | Webhook                                | Receive Slack slash command input  | —                      | Parse Request           |                                           |
| Parse Request             | Code                                   | Extract relevant fields from webhook | Slash Command Webhook   | Fetch Unread Messages    |                                           |
| Fetch Unread Messages      | HTTP Request                           | Retrieve last 20 messages from Slack channel | Parse Request          | Prepare Prompt for Claude |                                           |
| Prepare Prompt for Claude  | Code                                   | Format messages into AI prompt     | Fetch Unread Messages   | Claude AI: Summarize    |                                           |
| Claude AI: Summarize       | LangChain Chain LLM                    | Send prompt for AI summarization   | Prepare Prompt for Claude | Build Slack Blocks      |                                           |
| Anthropic Chat Model       | LangChain AI Language Model            | Claude AI model backend             | — (used by Claude AI: Summarize) | Claude AI: Summarize (ai_languageModel) | Requires Anthropic API credentials        |
| Build Slack Blocks         | Code                                   | Create Slack Block Kit message     | Claude AI: Summarize    | Post to Slack (Ephemeral) |                                           |
| Post to Slack (Ephemeral)  | HTTP Request                           | Post ephemeral Slack message       | Build Slack Blocks      | —                       |                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Slash Command Webhook"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `summarize`  
   - Purpose: Entry point for Slack slash command requests.

2. **Create Code Node: "Parse Request"**  
   - Connect input from "Slash Command Webhook".  
   - JavaScript to extract `channel_id`, `token`, and `user_id` from incoming request body:  
     ```js
     const { text, user_id, channel_id, token, trigger_id } = $json.body;
     return { json: { channel_id, token, user_id } };
     ```  
   - Output to next node.

3. **Create HTTP Request Node: "Fetch Unread Messages"**  
   - Connect input from "Parse Request".  
   - Method: GET  
   - URL: `https://slack.com/api/conversations.history`  
   - Query Parameters:  
     - `channel`: `={{ $json.channel_id }}`  
     - `limit`: `20`  
   - Headers:  
     - `Authorization`: `=Bearer {{$json.token}}`  
   - Purpose: Retrieve the last 20 messages from specified Slack channel.

4. **Create Code Node: "Prepare Prompt for Claude"**  
   - Connect input from "Fetch Unread Messages".  
   - JavaScript to format messages into prompt string:  
     ```js
     const messages = $json.messages || [];
     const formatted = messages.map(m => `[${m.ts}] <@${m.user}>: ${m.text}`).join('\n');
     return { json: { prompt: formatted, channel_id: $json.channel_id, token: $json.token, user_id: $json.user_id } };
     ```  
   - Output to AI summarization node.

5. **Set up Anthropic AI Credential**  
   - Add Anthropic API key credentials in n8n credentials manager.

6. **Create LangChain AI Language Model Node: "Anthropic Chat Model"**  
   - Configure model: `claude-sonnet-4-20250514` (Claude 4 Sonnet)  
   - Attach Anthropic API credentials.

7. **Create LangChain Chain LLM Node: "Claude AI: Summarize"**  
   - Connect input from "Prepare Prompt for Claude".  
   - Set message prompt:  
     `"Summarize these Slack messages into grouped threads with brief summaries and 2–3 suggested replies for each."`  
   - Link the AI model input to "Anthropic Chat Model".

8. **Create Code Node: "Build Slack Blocks"**  
   - Connect input from "Claude AI: Summarize".  
   - JavaScript to build Slack message blocks:  
     ```js
     const summary = $json.text;
     return { json: {
       channel: $json.channel_id,
       user: $json.user_id,
       token: $json.token,
       blocks: [
         { type: 'section', text: { type: 'mrkdwn', text: '*Here\'s a summary:*\n' + summary } }
       ]
     }};
     ```  
   - Output to final Slack post node.

9. **Create HTTP Request Node: "Post to Slack (Ephemeral)"**  
   - Connect input from "Build Slack Blocks".  
   - Method: POST  
   - URL: `https://slack.com/api/chat.postEphemeral`  
   - Headers:  
     - `Authorization`: `=Bearer {{$json.token}}`  
     - `Content-Type`: `application/json`  
   - Body: JSON (raw) with `channel`, `user`, and `blocks` from previous node output.  
   - Purpose: Post ephemeral message visible only to command user.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Workflow is designed for Slack slash commands, expecting a valid bearer token for Slack API calls. | Slack Apps & OAuth 2.0 setup required to generate tokens with `chat:write` and `channels:history` scopes. |
| Anthropic Claude API credentials must be configured in n8n to enable summarization.                  | https://www.anthropic.com/index/claude-api                                                                |
| Slack ephemeral messages require `user` parameter to specify the recipient of the message.         | https://api.slack.com/methods/chat.postEphemeral                                                        |
| The workflow uses LangChain nodes for AI orchestration, requiring n8n version supporting LangChain. | n8n v0.2023+ recommended                                                                                 |
| Slash command security best practice: Validate Slack tokens and request signatures for security.    | https://api.slack.com/authentication/verifying-requests-from-slack                                      |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.
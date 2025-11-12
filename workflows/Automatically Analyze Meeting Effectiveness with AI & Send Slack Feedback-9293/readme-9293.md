Automatically Analyze Meeting Effectiveness with AI & Send Slack Feedback

https://n8nworkflows.xyz/workflows/automatically-analyze-meeting-effectiveness-with-ai---send-slack-feedback-9293


# Automatically Analyze Meeting Effectiveness with AI & Send Slack Feedback

### 1. Workflow Overview

This n8n workflow is designed to automatically analyze the effectiveness of meetings using AI and send constructive feedback via Slack. It targets organizations or teams that want to improve their meeting quality by leveraging meeting transcripts or minutes stored in Google Docs and integrating calendar event data from Google Calendar.

The workflow is structured into the following logical blocks:

- **1.1 Trigger and Event Data Retrieval**: Listening for the end of a Google Calendar event or manual chat trigger to start the workflow and retrieve event details.
- **1.2 Meeting Minutes Location and Retrieval**: Determining the Google Docs document containing meeting minutes either by URL or by searching Google Drive based on meeting title.
- **1.3 AI Analysis**: Passing the meeting minutes to an AI Agent (powered by LangChain and OpenAI GPT-5) that analyzes meeting effectiveness, communication quality, and other criteria, producing detailed feedback.
- **1.4 Format Conversion and Slack Notification**: Converting the AI’s markdown feedback into Slack Block Kit format and sending it as a Slack message to a specified user.
- **1.5 Control and Conditional Logic**: Handling cases where relevant meeting minutes are missing, to avoid errors or unnecessary processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Event Data Retrieval

- **Overview**:  
  This block initiates the workflow upon the end of a Google Calendar event or via a chat trigger for testing. It then retrieves detailed information about the event.

- **Nodes Involved**:  
  - Google Calendar Trigger  
  - Wait  
  - Get an event  
  - If  

- **Node Details**:

  - **Google Calendar Trigger**  
    - *Type*: Trigger node for Google Calendar events  
    - *Configuration*: Triggers every minute, specifically on "eventEnded" for a configured calendar ID.  
    - *Input/Output*: No input; outputs event data JSON.  
    - *Potential Failures*: Auth errors with Google, rate limits, or calendar ID misconfiguration.  
    - *Sticky Note*: "Triggers: End of Google Calendar Event, Chat Trigger for testing"

  - **Wait**  
    - *Type*: Wait node  
    - *Configuration*: Waits a specified time (minutes) before continuing, likely to ensure meeting minutes are finalized before processing.  
    - *Input/Output*: Receives event trigger, outputs delayed data.  
    - *Failures*: Timeout misconfiguration, webhook issues.

  - **Get an event**  
    - *Type*: Google Calendar node (get event)  
    - *Configuration*: Retrieves full event details by event ID and calendar ID.  
    - *Input*: Event ID from trigger.  
    - *Output*: Detailed event JSON including attachments.  
    - *Failures*: Auth errors, invalid event ID.

  - **If**  
    - *Type*: Conditional node  
    - *Configuration*: Checks if the event JSON contains an attachment array (`$json.attachment` exists).  
    - *Input*: Event data.  
    - *Output*:  
      - True branch: attachments exist → proceed to extract meeting doc info  
      - False branch: no attachments → no operation (ends workflow).  
    - *Failures*: Expression evaluation errors if event JSON malformed.

---

#### 2.2 Meeting Minutes Location and Retrieval

- **Overview**:  
  This block prepares the input for AI analysis by formatting meeting information and locating the meeting minutes in Google Docs. If a Google Docs URL is provided, it directly retrieves the doc; otherwise, it searches Google Drive by meeting title.

- **Nodes Involved**:  
  - Edit Fields  
  - GetDoc  
  - SearchDoc  

- **Node Details**:

  - **Edit Fields**  
    - *Type*: Set node  
    - *Configuration*: Creates two new fields:  
      - `chatInput`: a formatted string summarizing the meeting title and Google Docs URL and fileId from event attachments.  
      - `sessionId`: a prefixed event ID string used for session tracking.  
    - *Input*: Event JSON with attachment info.  
    - *Output*: JSON with new keys for AI input.  
    - *Failures*: If attachment data missing or malformed, string construction may fail.

  - **GetDoc**  
    - *Type*: Google Docs node  
    - *Configuration*: Retrieves the content of the meeting minutes document using the URL or fileId extracted from attachments.  
    - *Input*: The `documentURL` parameter is dynamically set to the Google Docs URL.  
    - *Output*: Raw meeting minutes content.  
    - *Failures*: Auth errors, invalid or inaccessible doc URLs, document fetch errors.

  - **SearchDoc**  
    - *Type*: Google Drive node (search files)  
    - *Configuration*: Searches Google Drive files using a query based on meeting title if no direct URL is provided.  
    - *Input*: Search query string derived from meeting title.  
    - *Output*: List of matching files (possibly meeting minutes).  
    - *Failures*: Auth errors, rate limits, or no files found.

  - The **AI Agent** node is connected as an AI tool input from both GetDoc and SearchDoc, indicating either direct retrieval or searching results feed the AI analysis.

---

#### 2.3 AI Analysis

- **Overview**:  
  This core block uses an AI Agent configured with a detailed system prompt to analyze the meeting minutes for effectiveness, communication quality, speaking time, interruptions, disagreements, and more. It uses LangChain with an OpenAI GPT-5 language model.

- **Nodes Involved**:  
  - AI Agent  
  - OpenAI Chat Model  
  - Simple Memory  

- **Node Details**:

  - **AI Agent**  
    - *Type*: LangChain Agent node  
    - *Configuration*: Uses a comprehensive system prompt specifying how to analyze meeting minutes, extract specific metrics, score communication quality, and format output with constructive tone and actionable suggestions.  
    - *Input*: Meeting minutes content either from GetDoc or SearchDoc results.  
    - *Output*: Markdown formatted detailed feedback report.  
    - *Failures*: AI model errors, prompt interpretation issues, API limits.

  - **OpenAI Chat Model**  
    - *Type*: LangChain OpenAI chat model node  
    - *Configuration*: Uses GPT-5 model, linked as AI language model for AI Agent.  
    - *Credentials*: OpenAI API key configured.  
    - *Failures*: API quota or connectivity issues.

  - **Simple Memory**  
    - *Type*: LangChain Memory Buffer Window  
    - *Configuration*: Maintains conversational context for the AI Agent if multiple interactions occur, ensuring consistency.  
    - *Input*: Connects as AI memory to AI Agent.  
    - *Failures*: Memory overflow or context loss unlikely but possible.

---

#### 2.4 Format Conversion and Slack Notification

- **Overview**:  
  Converts the AI's markdown feedback into Slack Block Kit format using a custom code node, then sends the formatted message to a specified Slack user via Slack API.

- **Nodes Involved**:  
  - Code  
  - Send a message  

- **Node Details**:

  - **Code**  
    - *Type*: Code node (JavaScript)  
    - *Configuration*:  
      - Parses the AI Agent’s markdown output.  
      - Converts markdown elements (headers, lists, quotes, code blocks, inline styles like bold/italic/links) into Slack rich text blocks and block elements for Slack Block Kit.  
      - Extracts a text fallback for Slack notifications.  
    - *Input*: AI Agent markdown output.  
    - *Output*: JSON with `blocks` array and `slackPayload` object for Slack message body.  
    - *Failures*: Parsing errors if markdown format is unexpected, or code exceptions.

  - **Send a message**  
    - *Type*: Slack node  
    - *Configuration*:  
      - Sends a Slack message to a specific user (by user ID).  
      - Uses OAuth2 authentication.  
      - Sends message as Block Kit JSON from the code node output.  
      - Message supports markdown formatting and blocks.  
    - *Input*: Slack payload blocks from Code node.  
    - *Failures*: Slack API errors, auth token expiry, invalid user IDs.

- *Sticky Notes on this block:*
  - "Output node: Change format from regular markdown to Slack mrkdown by code node"
  - "Post a slack message"
  - "You can change the settings to send other channels"

---

#### 2.5 Control and Conditional Logic

- **Overview**:  
  Handles workflow branching for cases where meeting minutes are missing or attachments are absent.

- **Nodes Involved**:  
  - If  
  - No Operation, do nothing  

- **Node Details**:

  - **If** (described in 2.1)  
    - Routes workflow based on presence of attachment data.

  - **No Operation, do nothing**  
    - *Type*: No-op node  
    - *Configuration*: Ends the workflow branch silently if no meeting minutes are attached.  
    - *Failures*: None expected.

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                                   | Input Node(s)                  | Output Node(s)             | Sticky Note                                             |
|---------------------------|--------------------------------------|-------------------------------------------------|-------------------------------|----------------------------|---------------------------------------------------------|
| Google Calendar Trigger    | googleCalendarTrigger                 | Trigger workflow on event end                    | -                             | Wait                       | Triggers: End of Google Calendar Event, Chat Trigger for testing |
| Wait                      | wait                                 | Delay processing after event end                 | Google Calendar Trigger        | Get an event               |                                                         |
| Get an event              | googleCalendar                      | Retrieve detailed event information               | Wait                          | If                         |                                                         |
| If                        | if                                   | Branch if event has attachments                   | Get an event                  | Edit Fields, No Operation   |                                                         |
| Edit Fields               | set                                  | Format meeting info and prepare AI input         | If (true branch)              | AI Agent                   | Adjust input format for next node that is AI Agent      |
| GetDoc                    | googleDocsTool                       | Get meeting minutes from Google Docs URL          | From AI Agent (ai_tool)       | AI Agent                   |                                                         |
| SearchDoc                 | googleDriveTool                     | Search Google Drive for meeting minutes           | From AI Agent (ai_tool)       | AI Agent                   |                                                         |
| AI Agent                  | langchain.agent                     | Analyze meeting minutes with AI                    | Edit Fields, GetDoc, SearchDoc| Code                       | This AI Agent analyze your meeting transcription, and give a feedback |
| OpenAI Chat Model          | langchain.lmChatOpenAi              | OpenAI GPT-5 language model for AI Agent          | AI Agent (ai_languageModel)    | AI Agent                   |                                                         |
| Simple Memory             | langchain.memoryBufferWindow        | Maintain AI session memory                          | AI Agent (ai_memory)          | AI Agent                   |                                                         |
| Code                      | code                                 | Convert AI markdown output to Slack Block Kit     | AI Agent                      | Send a message             | Output node: Change format from regular markdown to Slack mrkdown by code node; Post a slack message; You can change the settings to send other channels |
| Send a message            | slack                                | Send formatted feedback message to Slack user     | Code                         | -                          |                                                         |
| No Operation, do nothing  | noOp                                 | End workflow silently if no meeting minutes       | If (false branch)             | -                          |                                                         |
| Test Trigger              | langchain.chatTrigger                | Manual trigger for testing AI analysis             | -                            | AI Agent                   |                                                         |
| Sticky Note               | stickyNote                          | Visual notes for workflow documentation            | -                            | -                          | Triggers: End of Google Calendar Event, Chat Trigger for testing |
| Sticky Note1              | stickyNote                          | Notes about input formatting                        | -                            | -                          | Adjust input format for next node that is AI Agent       |
| Sticky Note2              | stickyNote                          | Notes about AI Agent function                       | -                            | -                          | This AI Agent analyze your meeting transcription, and give a feedback |
| Sticky Note3              | stickyNote                          | Notes about output and Slack messaging              | -                            | -                          | Output node: Change format from regular markdown to Slack mrkdown by code node; Post a slack message; You can change the settings to send other channels |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Calendar Trigger node**  
   - Set trigger to "eventEnded" for a specific calendar.  
   - Poll every minute.  
   - Configure Google Calendar OAuth2 credentials.

2. **Add a Wait node**  
   - Configure to wait a specified number of minutes (e.g., to ensure meeting minutes are ready).  

3. **Add a Google Calendar node (Get an event)**  
   - Use event ID from trigger as input.  
   - Configure calendar ID and OAuth2 credentials.

4. **Add an If node**  
   - Condition: Check if `$json.attachment` exists in the event data.  
   - True branch proceeds to next steps; False branch ends workflow.

5. **Add a Set node (Edit Fields)**  
   - Create two custom fields:  
     - `chatInput`: formatted string using event summary and attachment file URL and ID.  
     - `sessionId`: string prefixed with event ID.  

6. **Add Google Docs node (GetDoc)**  
   - Operation: Get document content.  
   - Document URL: Dynamic value from attachment URL or fileId.  
   - Configure Google Docs OAuth2 credentials.

7. **Add Google Drive node (SearchDoc)**  
   - Operation: Search files.  
   - Query: Meeting title extracted or set dynamically.  
   - Configure Google Drive OAuth2 credentials.

8. **Add LangChain AI Agent node**  
   - Set system prompt as specified (Meeting Coach AI System Prompt).  
   - Connect AI Agent’s `ai_tool` input from both GetDoc and SearchDoc nodes.  
   - Configure AI Agent to use OpenAI Chat Model and Simple Memory nodes.

9. **Add LangChain OpenAI Chat Model node**  
   - Select GPT-5 model.  
   - Configure OpenAI API credentials.

10. **Add LangChain Simple Memory node**  
    - Use default buffer window to maintain AI session context.

11. **Add a Code node**  
    - Paste the JavaScript code that converts AI markdown output into Slack Block Kit JSON format.

12. **Add Slack node (Send a message)**  
    - Set message type to "block".  
    - Use blocks from Code node output (`$json.slackPayload`).  
    - Set user ID or channel to send the message.  
    - Configure Slack OAuth2 credentials.

13. **Add a No Operation node**  
    - Connect to the false branch of the If node to silently end workflow if no attachment.

14. **Optional: Add a Chat Trigger node for manual testing**  
    - Connect directly to AI Agent node for testing without calendar trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow triggers on the end of Google Calendar events or manually via chat trigger for testing.                      | Sticky Note near Google Calendar Trigger and Test Trigger nodes                                   |
| The AI Agent uses a detailed system prompt to analyze meeting effectiveness with specific scoring and constructive feedback. | System prompt in AI Agent node parameters                                                        |
| The output uses Slack Block Kit messages to present feedback in a rich, readable format.                                    | Sticky Note near Code and Send a message nodes                                                   |
| Slack messages are sent to a specific user but can be configured to send to other channels or users as needed.             | Sticky Note near Send a message node                                                             |
| The workflow assumes meeting minutes are saved in Google Docs linked from event attachments or searchable by meeting title. | Workflow logic in Edit Fields, GetDoc, SearchDoc nodes                                            |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly complies with current content policies and contains no illegal, offensive, or protected content. All manipulated data is legal and public.
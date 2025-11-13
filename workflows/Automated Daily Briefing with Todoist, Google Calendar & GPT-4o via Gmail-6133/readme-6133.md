Automated Daily Briefing with Todoist, Google Calendar & GPT-4o via Gmail

https://n8nworkflows.xyz/workflows/automated-daily-briefing-with-todoist--google-calendar---gpt-4o-via-gmail-6133


# Automated Daily Briefing with Todoist, Google Calendar & GPT-4o via Gmail

### 1. Workflow Overview

This workflow automates the generation and emailing of a personalized daily briefing. It targets users who want a consolidated morning summary of their tasks and calendar events, formatted and enhanced by GPT-4o, and delivered via Gmail.  

The workflow is logically divided into these blocks:  
- **1.1 Scheduled Trigger**: Initiates the workflow daily at 6:00 AM.  
- **1.2 Data Retrieval**: Fetches today's Google Calendar events and Todoist tasks from a specific project.  
- **1.3 Data Merging and Formatting**: Combines both data sets and formats them into readable strings.  
- **1.4 AI Processing**: Sends the formatted information to GPT-4o for a natural language summary, structured with markdown formatting.  
- **1.5 HTML Conversion**: Converts GPT markdown output into styled HTML for email readability.  
- **1.6 Email Sending**: Sends the final formatted briefing to the user’s email inbox using Gmail.  

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Triggers the entire workflow automatically every day at 6:00 AM to ensure timely morning briefings.  

- **Nodes Involved:**  
  - Schedule Trigger  

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: `Schedule Trigger` (n8n core node)  
    - Configuration: Set to trigger once daily at 6:00 AM (hour=6)  
    - Input: None (start node)  
    - Output: Triggers downstream nodes for data retrieval  
    - Edge Cases: Workflow won't run if n8n instance is offline or system time misconfigured  
    - Sticky Note: Emphasizes automation of morning routine prep  

#### 2.2 Data Retrieval

- **Overview:**  
  Retrieves all calendar events for the day from Google Calendar and all tasks from a specified Todoist project.  

- **Nodes Involved:**  
  - Get many events (Google Calendar)  
  - Get many tasks (Todoist)  

- **Node Details:**  
  - **Get many events**  
    - Type: `Google Calendar` node  
    - Configuration:  
      - Operation: Get all events  
      - Calendar: Uses connected Gmail calendar account (dynamic email placeholder)  
      - Credentials: Google Calendar OAuth2  
    - Input: Trigger from Schedule Trigger  
    - Output: All calendar events for the day  
    - Edge Cases: Authentication failure, empty calendar, API rate limiting  
    - Sticky Note: Explains calendar event fetching and calendar filtering  
  - **Get many tasks**  
    - Type: `Todoist` node  
    - Configuration:  
      - Operation: Get all tasks  
      - Filter: Specific Todoist project ID (must be updated to target correct project)  
      - Credentials: Todoist API  
    - Input: Trigger from Schedule Trigger  
    - Output: All tasks under specified project  
    - Edge Cases: Wrong project ID yields no tasks, authentication errors  
    - Sticky Note: Reminds to verify project ID  

#### 2.3 Data Merging and Formatting

- **Overview:**  
  Merges the two data streams (calendar events and tasks) and processes them to create human-readable formatted strings suitable for GPT input.  

- **Nodes Involved:**  
  - Merge  
  - Code (JavaScript)  

- **Node Details:**  
  - **Merge**  
    - Type: `Merge` node  
    - Configuration: Default merging mode (likely merge by index)  
    - Input: From Get many events (calendar) and Get many tasks (Todoist)  
    - Output: Combined dataset to Code node  
    - Edge Cases: Uneven inputs, empty inputs result in partial merges  
    - Sticky Note: Describes merging necessity for feeding single code node  
  - **Code**  
    - Type: `Code` (JavaScript) node  
    - Configuration:  
      - Separates input items into calendar events and tasks based on JSON properties  
      - Formats calendar events as numbered list with event name and time (localized to en-IN, 2-digit hour/minute)  
      - Formats tasks similarly, showing content and due time if set, or "No time set"  
      - Outputs JSON with two fields: `formattedEvents` and `formattedTasks` as newline-separated strings  
    - Input: Merged data  
    - Output: Formatted task and event strings for GPT input  
    - Edge Cases: Missing dates, empty lists handled by fallback text ("No calendar events today.", "No tasks today.")  
    - Sticky Note: Explains formatting logic and purpose  

#### 2.4 AI Processing

- **Overview:**  
  Sends the formatted tasks and calendar events to GPT-4o via the LangChain OpenAI node, prompting it to produce a structured, markdown-formatted daily briefing message.  

- **Nodes Involved:**  
  - Message a model (LangChain OpenAI)  

- **Node Details:**  
  - **Message a model**  
    - Type: `OpenAI` node via LangChain integration  
    - Configuration:  
      - Model: `chatgpt-4o-latest`  
      - Message prompt includes embedded `formattedTasks` and `formattedEvents` fields  
      - Instructions to GPT: provide a markdown summary with sections for daily summary, top priorities with checkboxes and times, and motivational quote focus mantra  
      - Emphasizes line breaks and email-friendly formatting  
      - Credentials: OpenAI API key  
    - Input: Formatted strings from Code node  
    - Output: GPT-generated markdown summary text  
    - Edge Cases: API key invalid, rate limits, prompt execution failures, no or malformed input strings  
    - Sticky Note: Explains AI usage and prompt structure  

#### 2.5 HTML Conversion

- **Overview:**  
  Converts GPT markdown output into styled HTML with proper line breaks, bold text, and preserved emojis for email display.  

- **Nodes Involved:**  
  - Convert OpenAI Text to HTML (Code node)  

- **Node Details:**  
  - **Convert OpenAI Text to HTML**  
    - Type: `Code` node  
    - Configuration:  
      - Processes GPT markdown output: converts markdown bold (`**text**`) to HTML `<strong>` tags  
      - Converts single and double newlines to `<br>` tags for spacing  
      - Retains emojis as-is for visual clarity  
      - Wraps entire content in a styled `<div>` with Arial font, size 14px, and line height 1.6 for readability  
    - Input: GPT message content from Message a model  
    - Output: JSON with `htmlBody` field containing ready-to-send HTML string  
    - Edge Cases: Missing GPT output, unexpected markdown formats  
    - Sticky Note: Notes on markdown to HTML conversion and styling  

#### 2.6 Email Sending

- **Overview:**  
  Sends the constructed HTML daily briefing via Gmail to the configured recipient with a fixed subject line.  

- **Nodes Involved:**  
  - Send a message (Gmail node)  

- **Node Details:**  
  - **Send a message**  
    - Type: `Gmail` node  
    - Configuration:  
      - Recipient: Dynamic email placeholder (user’s email)  
      - Message body: uses `htmlBody` from previous node  
      - Subject: "Morning Briefings"  
      - Credentials: Gmail OAuth2 account  
    - Input: HTML content from Convert OpenAI Text to HTML  
    - Output: Email sent confirmation  
    - Edge Cases: Invalid recipient email, Gmail auth errors, quota exceeded  
    - Sticky Note: Indicates email subject and destination  

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                    | Input Node(s)           | Output Node(s)               | Sticky Note                                                                                                         |
|---------------------------|--------------------------------|----------------------------------|------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger               | Starts workflow daily at 6 AM    | None                   | Get many events, Get many tasks | ⏰ Triggers the workflow at 6:00 AM daily. Use this to automate your morning routine prep.                            |
| Get many events           | Google Calendar                | Retrieves all calendar events    | Schedule Trigger       | Merge                       | 📅 Fetches all Google Calendar events for the day. Filters are configured using your linked Gmail calendar.          |
| Get many tasks            | Todoist                       | Retrieves all Todoist tasks      | Schedule Trigger       | Merge                       | 📝 Fetches all Todoist tasks under a specified project. Make sure `projectId` is up-to-date.                         |
| Merge                    | Merge                         | Combines event and task data     | Get many events, Get many tasks | Code                        | 🔀 Merges calendar events and tasks. Required to feed both datasets into a single Code node.                         |
| Code                     | Code (JavaScript)             | Separates & formats data         | Merge                  | Message a model             | 🧠 Separates & formats calendar events and tasks. Returns structured strings for GPT to summarize.                   |
| Message a model           | OpenAI (LangChain)            | Generates daily summary via GPT  | Code                   | Convert OpenAI Text to HTML | 🤖 Sends task and calendar summary to GPT-4o. Prompt includes layout guidance and markdown formatting.               |
| Convert OpenAI Text to HTML | Code (JavaScript)             | Converts markdown to HTML        | Message a model         | Send a message              | 🧱 Converts markdown-style GPT output into clean HTML. Ensures line breaks, bold headings, and emojis are email-friendly. |
| Send a message            | Gmail                         | Sends final email briefing       | Convert OpenAI Text to HTML | None                      | 📧 Sends the final HTML email to your inbox. Email subject: “Morning Briefings”                                     |
| Sticky Note               | Sticky Note                   | Documentation & notes            | None                   | None                       | ## 📌 **Daily Briefing Automation** This workflow fetches today's calendar events and tasks from Google Calendar and Todoist, formats them using GPT-4o, converts the summary to HTML, and emails it automatically at 6:00 AM daily. Use this to start your day focused and aligned with your top priorities. **Highlights:** - Auto-scheduled morning brief - Task & calendar merging - Natural language summary by GPT - HTML email output (readable & styled) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 6:00 AM (triggerAtHour: 6)  
   - No credentials required  

2. **Add Google Calendar node ("Get many events"):**  
   - Type: Google Calendar  
   - Operation: Get all events  
   - Calendar: Select your Gmail calendar account (OAuth2 credentials required)  
   - Connect input from Schedule Trigger  
   - Set output to pass all events  

3. **Add Todoist node ("Get many tasks"):**  
   - Type: Todoist  
   - Operation: Get all tasks  
   - Filter: Enter your specific Todoist project ID  
   - Use Todoist API credentials  
   - Connect input from Schedule Trigger  

4. **Add Merge node:**  
   - Type: Merge  
   - Connect one input from Google Calendar node (Get many events)  
   - Connect second input from Todoist node (Get many tasks)  
   - Default merge mode (concatenate inputs)  

5. **Add Code node ("Format Events & Tasks"):**  
   - Type: Code (JavaScript)  
   - Connect input from Merge node  
   - Paste the following code (adjust localization if needed):  
   ```javascript
   const calendarEvents = [];
   const todoistTasks = [];

   for (const item of items) {
     if (item.json?.start?.dateTime) {
       calendarEvents.push(item.json);
     } else if (item.json?.content) {
       todoistTasks.push(item.json);
     }
   }

   const formattedEvents = calendarEvents.map((event, i) => {
     const time = new Date(event.start.dateTime).toLocaleTimeString('en-IN', { hour: '2-digit', minute: '2-digit' });
     return `${i + 1}. ${event.summary} at ${time}`;
   }).join('\n');

   const formattedTasks = todoistTasks.map((task, i) => {
     const time = task.due?.datetime
       ? new Date(task.due.datetime).toLocaleTimeString('en-IN', { hour: '2-digit', minute: '2-digit' })
       : 'No time set';
     return `${i + 1}. ${task.content} (Due at ${time})`;
   }).join('\n');

   return [
     {
       json: {
         formattedEvents: formattedEvents || 'No calendar events today.',
         formattedTasks: formattedTasks || 'No tasks today.'
       }
     }
   ];
   ```  

6. **Add OpenAI LangChain node ("Message a model"):**  
   - Type: OpenAI (LangChain)  
   - Model: `chatgpt-4o-latest`  
   - Credentials: Set up OpenAI API key  
   - Configure message prompt with:  
     ```
     Here's my plan for today:

     🗂️ **Tasks**
     {{ $json.formattedTasks }}

     📅 **Calendar**
     {{ $json.formattedEvents }}

     ---

     ✅ Structure the reply in this exact markdown format:

     **Daily Summary**  
     [one-sentence summary of the day]

     🎯 **Top Priorities**  
     1. ✅ [Task 1] – ⏰ [Time]  
     2. ✅ [Task 2] – ⏰ [Time]  
     ...

     💡 **Focus Mantra**  
     "[motivational quote]"

     Add **line breaks** between each section and each task, so it's email-friendly and easy to scan.
     Avoid long paragraphs. Keep it clean and structured.
     ```  
   - Connect input from Code node  

7. **Add Code node ("Convert OpenAI Text to HTML"):**  
   - Type: Code (JavaScript)  
   - Connect input from OpenAI node  
   - Paste conversion code:  
   ```javascript
   const raw = $json.message?.content || 'No message';

   let html = raw
     .replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>')
     .replace(/\n{2,}/g, '<br><br>')
     .replace(/\n/g, '<br>')
     .replace(/✅/g, '✅')
     .replace(/🎯/g, '🎯')
     .replace(/💡/g, '💡')
     .replace(/📅/g, '📅')
     .replace(/🗂️/g, '🗂️');

   return [
     {
       json: {
         htmlBody: `<div style="font-family: Arial, sans-serif; font-size: 14px; line-height: 1.6;">${html}</div>`
       }
     }
   ];
   ```  

8. **Add Gmail node ("Send a message"):**  
   - Type: Gmail  
   - Credentials: Gmail OAuth2 account connected  
   - Set recipient email (can be static or dynamic placeholder)  
   - Subject: "Morning Briefings"  
   - Message: Use `htmlBody` field from previous Code node (expression: `={{ $json.htmlBody }}`)  
   - Connect input from HTML conversion Code node  

9. **Activate workflow:**  
   - Ensure all credentials are valid and tested  
   - Test run and verify the email is received with formatted daily briefing  

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                      |
|----------------------------------------------------------------------------------------------------------------------------------|------------------------------------|
| This workflow fetches today's calendar events and tasks from Google Calendar and Todoist, formats them using GPT-4o, converts the summary to HTML, and emails it automatically at 6:00 AM daily. | Workflow purpose and summary        |
| Use this to start your day focused and aligned with your top priorities.                                                         | User benefit                        |
| Highlights include auto-scheduling, task & calendar merging, natural language summary by GPT, and HTML email output (readable & styled). | Feature summary                     |
| Gmail OAuth2 and Google Calendar OAuth2 credentials require correct OAuth consent and API setup in Google Cloud Console.         | Credential setup note               |
| Todoist API token must have access to the specified project ID.                                                                  | Credential setup note               |
| OpenAI API key must have access to GPT-4o latest model and sufficient quota.                                                      | Credential setup note               |
| Email subject fixed as "Morning Briefings" but can be customized in the Gmail node if desired.                                    | Customization tip                   |

---

**Disclaimer:** The provided text exclusively originates from an automated n8n workflow. All data handled is legal and public, and the process complies with applicable content policies.
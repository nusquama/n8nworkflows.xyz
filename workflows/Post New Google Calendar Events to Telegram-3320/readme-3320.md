Post New Google Calendar Events to Telegram

https://n8nworkflows.xyz/workflows/post-new-google-calendar-events-to-telegram-3320


# Post New Google Calendar Events to Telegram

### 1. Workflow Overview

This workflow automates the process of notifying a Telegram chat whenever a new event is created in a specified Google Calendar. It listens for newly created events, extracts essential event details, and sends a formatted message to a Telegram chat using a bot.

The workflow is logically divided into two main blocks:

- **1.1 Input Reception:** Captures new event creation in Google Calendar using a trigger node.
- **1.2 Notification Dispatch:** Formats and sends the event details as a Telegram message.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block monitors a Google Calendar for any newly created events. It triggers the workflow execution immediately when a new event is detected.

**Nodes Involved:**  
- Google Calendar Trigger

**Node Details:**

- **Google Calendar Trigger**  
  - **Type:** Trigger node  
  - **Technical Role:** Listens for new events created in a specified Google Calendar.  
  - **Configuration:**  
    - Trigger Type: "Event Created" — the node activates when a new event is added.  
    - Polling Interval: Every minute (pollTimes set to everyMinute).  
    - Calendar Selection: User selects the specific Google Calendar to monitor via OAuth2 authentication.  
  - **Key Expressions/Variables:**  
    - Outputs event data JSON including fields like `summary` (event name), `description`, `location`, `start.dateTime`, `end.dateTime`, and `creator.email`.  
  - **Input Connections:** None (trigger node).  
  - **Output Connections:** Connected to the Telegram node to pass event data.  
  - **Version-Specific Requirements:** Requires n8n version supporting Google Calendar Trigger node (v1 used here).  
  - **Potential Failures:**  
    - Authentication errors if OAuth2 token expires or is invalid.  
    - API rate limits or quota exceeded errors from Google Calendar API.  
    - Network timeouts or connectivity issues.  
  - **Sub-workflow Reference:** None.

#### 2.2 Notification Dispatch

**Overview:**  
This block receives event data from the trigger node, formats a message with key event details, and sends it to a specified Telegram chat using a bot.

**Nodes Involved:**  
- Telegram

**Node Details:**

- **Telegram**  
  - **Type:** Action node  
  - **Technical Role:** Sends a message to a Telegram chat using a bot token.  
  - **Configuration:**  
    - Action: "Send Message"  
    - Authentication: Telegram Bot Token (Bot created via BotFather).  
    - Chat ID: User specifies the target chat ID (personal or group).  
    - Message Text: Uses expressions to format the message with event details:  
      ```
      Event Name:  {{ $json.summary }}
      Description: {{ $json.description }}
      Event Location: {{ $json.location }}
      Start Date: {{ $json.start.dateTime }}
      End Date: {{ $json.end.dateTime }}
      Creator: {{ $json.creator.email }}
      ```  
    - Additional Fields: `appendAttribution` set to false to avoid adding "Sent from n8n" attribution.  
  - **Key Expressions/Variables:** Uses mustache-style expressions to extract event data from the incoming JSON.  
  - **Input Connections:** Receives data from Google Calendar Trigger node.  
  - **Output Connections:** None (end node).  
  - **Version-Specific Requirements:** Requires n8n version supporting Telegram node v1.2 or higher for these features.  
  - **Potential Failures:**  
    - Invalid or expired Telegram Bot Token causing authentication failure.  
    - Incorrect or missing Chat ID leading to message delivery failure.  
    - Network issues or Telegram API downtime.  
    - Expression errors if event data fields are missing or null.  
  - **Sub-workflow Reference:** None.

---

### 3. Summary Table

| Node Name               | Node Type                    | Functional Role                     | Input Node(s)           | Output Node(s) | Sticky Note                                                                                  |
|-------------------------|------------------------------|-----------------------------------|------------------------|----------------|----------------------------------------------------------------------------------------------|
| Google Calendar Trigger  | Google Calendar Trigger       | Detect new Google Calendar events | None                   | Telegram       |                                                                                              |
| Telegram                | Telegram                      | Send event details as Telegram msg| Google Calendar Trigger | None           |                                                                                              |
| Sticky Note             | Sticky Note                  | Workflow title display             | None                   | None           | ## Post new Google Calendar events to Telegram                                              |
| Sticky Note1            | Sticky Note                  | Workflow description display       | None                   | None           | ## Description This n8n workflow automatically sends a Telegram message whenever a new event is added to Google Calendar. It extracts key event details such as event name, description, event creator, start date, end date, and location and forwards them to a specified Telegram chat. This ensures you stay updated on all newly scheduled events directly from Telegram. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Calendar Trigger Node**  
   - Add a new node and select **Google Calendar Trigger**.  
   - Authenticate with your Google Account using OAuth2 credentials with Google Calendar API enabled.  
   - Set **Trigger On** to **Event Created**.  
   - Select the specific calendar to monitor from your account.  
   - Set polling interval to every minute (default).  
   - Save and test the node by clicking **Execute Node** to ensure connectivity.

2. **Create Telegram Node**  
   - Add a new node and select **Telegram**.  
   - Choose the **Send Message** action.  
   - Authenticate using your Telegram Bot Token (create a bot via BotFather if you haven't).  
   - Set the **Chat ID** to the target Telegram chat (personal or group).  
   - In the **Text** field, enter the following message template using expressions:  
     ```
     Event Name:  {{ $json.summary }}
     Description: {{ $json.description }}
     Event Location: {{ $json.location }}
     Start Date: {{ $json.start.dateTime }}
     End Date: {{ $json.end.dateTime }}
     Creator: {{ $json.creator.email }}
     ```  
   - Under **Additional Fields**, disable **Append Attribution** to avoid extra text.  
   - Save and test the node by clicking **Execute Node**.

3. **Connect Nodes**  
   - Connect the output of the **Google Calendar Trigger** node to the input of the **Telegram** node.

4. **Test the Workflow**  
   - Activate the workflow (toggle active).  
   - Create a new event in the selected Google Calendar.  
   - Verify that the Telegram chat receives the formatted event notification message.

5. **Optional: Add Sticky Notes**  
   - Add sticky notes to document the workflow title and description for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow created by the AI development team at WeblineIndia. We help businesses automate processes and scale faster.              | https://www.weblineindia.com/ai-development.html                                               |
| Hire AI developers for custom workflow automation tailored to your needs.                                                          | https://www.weblineindia.com/hire-ai-developers.html                                           |
| Prerequisites: Enable Google Calendar API, create Telegram Bot via BotFather, obtain Telegram Chat ID, and configure OAuth2 tokens.| See workflow description for setup instructions                                                |

---

This documentation provides a complete understanding of the workflow, enabling reproduction, modification, and troubleshooting by both advanced users and automation agents.
Send Severe Weather Alerts from Visual Crossing to Telegram

https://n8nworkflows.xyz/workflows/send-severe-weather-alerts-from-visual-crossing-to-telegram-10285


# Send Severe Weather Alerts from Visual Crossing to Telegram

### 1. Workflow Overview

This workflow automates the monitoring of severe weather alerts for a specified home location by querying the Visual Crossing weather API and sending notifications via Telegram. It is designed for users who want to receive timely and automated severe weather alerts on their messaging platform to stay informed of hazardous conditions in their area.

**Logical Blocks:**

- **1.1 Hourly Trigger:** Initiates the workflow every hour to perform weather checks regularly.
- **1.2 Weather Data Retrieval:** Fetches current day weather data and alerts from Visual Crossing API for the home location.
- **1.3 Alert Filtering:** Processes the weather data to identify if any severe weather alerts are present.
- **1.4 Conditional Routing:** Determines whether to proceed with notification based on alert presence.
- **1.5 Notification Delivery:** Sends severe weather alert messages via Telegram if alerts exist.

---

### 2. Block-by-Block Analysis

#### 2.1 Hourly Trigger Block

- **Overview:**  
  This block starts the workflow execution every hour, ensuring periodic weather checks without manual intervention.

- **Nodes Involved:**  
  - Hourly Trigger

- **Node Details:**

  **Hourly Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow execution on a defined schedule.  
  - Configuration: Set to trigger the workflow every 1 hour.  
  - Expressions/Variables: None  
  - Input: None (trigger node)  
  - Output: Connects to "Get Home Weather" node  
  - Version: 1  
  - Potential Failures: Scheduler failure (rare), workflow execution delays if system overloaded  
  - Notes: Triggers every hour as intended.  

---

#### 2.2 Weather Data Retrieval Block

- **Overview:**  
  This block calls the Visual Crossing API to get today's weather forecast and any severe weather alerts for the specified home location.

- **Nodes Involved:**  
  - Get Home Weather

- **Node Details:**

  **Get Home Weather**  
  - Type: HTTP Request  
  - Role: Makes an external API call to Visual Crossing to retrieve weather data.  
  - Configuration:  
    - Method: GET (default)  
    - URL pattern:  
      `https://weather.visualcrossing.com/VisualCrossingWebServices/rest/services/timeline/YOUR_HOME_LOCATION?unitGroup=metric&key=YOUR_VISUAL_CROSSING_API_KEY&include=alerts,today`  
    - Query parameters include:  
      - `unitGroup=metric` for metric units  
      - `include=alerts,today` to fetch alerts and today's weather  
    - Authentication: API key passed as query parameter (`key`)  
  - Expressions/Variables: Replace `YOUR_HOME_LOCATION` and `YOUR_VISUAL_CROSSING_API_KEY` with actual values.  
  - Input: Triggered by "Hourly Trigger" node  
  - Output: JSON data containing weather and alerts, passed to "Check for Home Weather Alerts"  
  - Version: 1  
  - Potential Failures:  
    - HTTP errors (network issues, invalid API key)  
    - Malformed response or unexpected data format  
    - API rate limits exceeded  
  - Notes: Fetches today's weather and alerts for home location.  

---

#### 2.3 Alert Filtering Block

- **Overview:**  
  This block filters the API response to continue only if there are severe weather alerts present in the data.

- **Nodes Involved:**  
  - Check for Home Weather Alerts

- **Node Details:**

  **Check for Home Weather Alerts**  
  - Type: Function  
  - Role: Filters the items to only those where the `alerts` array exists and is non-empty.  
  - Configuration: Custom JavaScript code:  
    ```js
    return items.filter(item => item.json.alerts && item.json.alerts.length > 0);
    ```  
  - Expressions/Variables: Accesses `alerts` field in JSON response  
  - Input: JSON weather data from "Get Home Weather"  
  - Output: Filtered data with alerts passed to "If" node  
  - Version: 1  
  - Potential Failures:  
    - If `alerts` field missing or malformed, may return empty array (no alerts)  
    - Expression syntax errors (unlikely)  
  - Notes: Continues only if there are severe weather alerts at home location.  

---

#### 2.4 Conditional Routing Block

- **Overview:**  
  This block evaluates the presence of severe weather alerts and routes the workflow to the notification block only if alerts exist.

- **Nodes Involved:**  
  - If

- **Node Details:**

  **If**  
  - Type: If (Boolean condition)  
  - Role: Checks if the current item JSON contains `alerts` with length > 0.  
  - Configuration:  
    - Condition: Expression evaluates to boolean true if alerts exist:  
      ```  
      {{$json["alerts"] && $json["alerts"].length > 0}}  
      ```  
    - Uses version 2 condition syntax, strict type validation, case sensitive  
  - Input: Filtered items from "Check for Home Weather Alerts"  
  - Output:  
    - True branch: Connects to "Send Home Weather Alert" node  
    - False branch: No further action (workflow ends)  
  - Version: 2.2  
  - Potential Failures: Expression evaluation errors if data format changes  
  - Notes: Controls workflow continuation based on alert presence.  

---

#### 2.5 Notification Delivery Block

- **Overview:**  
  Sends the severe weather alert notifications to the user via Telegram messaging platform.

- **Nodes Involved:**  
  - Send Home Weather Alert

- **Node Details:**

  **Send Home Weather Alert**  
  - Type: Telegram  
  - Role: Sends a message containing the severe weather alert information to a Telegram chat.  
  - Configuration:  
    - Operation: Send message  
    - Telegram credentials: Must be configured with bot token and chat ID  
    - Message content: Typically uses data from previous nodes (not explicitly detailed, but assumed)  
  - Input: Triggered from the "If" node’s true branch  
  - Output: None (end node)  
  - Version: 1  
  - Potential Failures:  
    - Telegram authentication errors (invalid bot token or chat ID)  
    - Network issues preventing message delivery  
    - Message size or formatting errors  
  - Notes: Ensure Telegram credentials and chat ID are set in the node.  

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                               | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                  |
|--------------------------|--------------------|-----------------------------------------------|---------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Hourly Trigger           | Schedule Trigger   | Starts workflow every hour                     | None                      | Get Home Weather            | Starts the workflow every day at 7 AM to check calendar events and weather.                  |
| Get Home Weather         | HTTP Request       | Fetches today’s weather and alerts for home   | Hourly Trigger            | Check for Home Weather Alerts | Fetches today's weather and alerts for home location.                                        |
| Check for Home Weather Alerts | Function         | Filters items to only those with alerts       | Get Home Weather          | If                          | Continues only if there are severe weather alerts at home location.                          |
| If                       | If                 | Conditional routing based on alert presence   | Check for Home Weather Alerts | Send Home Weather Alert      |                                                                                              |
| Send Home Weather Alert  | Telegram           | Sends severe weather alert notification       | If (true branch)          | None                        | Sends severe weather alert notification for home location, if any alerts present.            |
| Sticky Note              | Sticky Note        | Documentation                                 | None                      | None                        | ## Hourly Trigger\nTriggers the flow every hour as intended.                                |
| Sticky Note1             | Sticky Note        | Documentation                                 | None                      | None                        | ## Get Weather \nThis fetches today’s weather and any severe alerts for the home location.  |
| Sticky Note2             | Sticky Note        | Documentation                                 | None                      | None                        | ## Check Weather Alerts \nFilters items to only those with alerts                            |
| Sticky Note3             | Sticky Note        | Documentation                                 | None                      | None                        | ## Send notification\nMake sure Telegram credentials and chat ID are set in the node.       |
| Sticky Note4             | Sticky Note        | Documentation                                 | None                      | None                        | ## How it works\nThis workflow automates fetching weather and sending alerts via Telegram. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Name: `Hourly Trigger`  
   - Type: `Schedule Trigger`  
   - Parameters: Set to trigger every 1 hour (e.g., under “Interval” select “hours” and set to 1)  
   - No credentials required.

3. **Add an HTTP Request node:**  
   - Name: `Get Home Weather`  
   - Type: `HTTP Request`  
   - Parameters:  
     - URL:  
       `https://weather.visualcrossing.com/VisualCrossingWebServices/rest/services/timeline/YOUR_HOME_LOCATION?unitGroup=metric&key=YOUR_VISUAL_CROSSING_API_KEY&include=alerts,today`  
       (Replace `YOUR_HOME_LOCATION` with your city or coordinates, and `YOUR_VISUAL_CROSSING_API_KEY` with your API key.)  
     - Method: GET  
     - Response Format: JSON (default)  
   - Connect the output of `Hourly Trigger` to this node.

4. **Add a Function node:**  
   - Name: `Check for Home Weather Alerts`  
   - Type: `Function`  
   - Parameters: Enter the following JavaScript code to filter items that have alerts:  
     ```js
     return items.filter(item => item.json.alerts && item.json.alerts.length > 0);
     ```  
   - Connect the output of `Get Home Weather` to this node.

5. **Add an If node:**  
   - Name: `If`  
   - Type: `If`  
   - Parameters:  
     - Condition Type: Boolean  
     - Expression:  
       ```
       {{$json["alerts"] && $json["alerts"].length > 0}}
       ```  
     - Use version 2 condition settings (strict type validation, case sensitive)  
   - Connect the output of `Check for Home Weather Alerts` to this node.

6. **Add a Telegram node:**  
   - Name: `Send Home Weather Alert`  
   - Type: `Telegram`  
   - Parameters:  
     - Operation: Send message  
     - Message Text: Compose the message using the alert data from previous nodes (e.g., include alert descriptions or titles from the JSON).  
   - Credentials: Configure Telegram Bot credentials with Bot Token and Chat ID.  
   - Connect the **true** output of the `If` node to this node.

7. **Verify all connections:**  
   - `Hourly Trigger` → `Get Home Weather` → `Check for Home Weather Alerts` → `If` → (true) → `Send Home Weather Alert`  

8. **Test the workflow:**  
   - Manually trigger or wait for scheduled trigger.  
   - Ensure the Visual Crossing API key is valid and your Telegram credentials are correctly set.  
   - Confirm that alerts are received correctly in your Telegram chat.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                              | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow automates fetching weather forecasts for your home location, including severe weather alerts, and sends timely notifications via Telegram. It uses the Visual Crossing API for detailed weather data and integrates with Telegram for messaging and alerts.                                    | Workflow description and overview                |
| You can replace the Telegram node with other notification channels like email, WhatsApp, or SMS to receive severe weather alerts across multiple platforms.                                                                                                                                              | Workflow customization note                       |
| Ensure your Visual Crossing API key is active and has sufficient quota to avoid request failures.                                                                                                                                                                                                         | API usage consideration                           |
| Telegram credentials require a bot token and a chat ID; verify both for successful message delivery.                                                                                                                                                                                                     | Telegram integration note                         |
| Visual Crossing API URL format documentation: https://www.visualcrossing.com/resources/documentation/weather-api/timeline-weather-api/                                                                                                                                                                  | External API documentation                        |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.
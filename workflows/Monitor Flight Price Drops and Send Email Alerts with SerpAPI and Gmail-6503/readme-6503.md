Monitor Flight Price Drops and Send Email Alerts with SerpAPI and Gmail

https://n8nworkflows.xyz/workflows/monitor-flight-price-drops-and-send-email-alerts-with-serpapi-and-gmail-6503


# Monitor Flight Price Drops and Send Email Alerts with SerpAPI and Gmail

### 1. Workflow Overview

This workflow automates monitoring flight prices for a specified route and date using SerpAPI's Google Flights data, and sends an email alert via Gmail if the price drops below a predefined threshold. It is ideal for users tracking flight deals to make timely booking decisions. The workflow is structured into four logical blocks:

- **1.1 Scheduled Trigger:** Periodically initiates the price check at a set time daily.
- **1.2 Price Fetching:** Requests flight price data from SerpAPI.
- **1.3 Price Evaluation:** Compares the fetched price against a threshold to detect a drop.
- **1.4 Notification:** Sends an email alert if a price drop is detected.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow automatically every day at 8:00 AM to start the price monitoring process.

- **Nodes Involved:**  
  - Cron

- **Node Details:**  

  - **Cron**  
    - Type and Technical Role: Time-based trigger node that schedules workflow runs.  
    - Configuration: Set to trigger once daily at 08:00 hours.  
    - Key Expressions/Variables: None.  
    - Input Connections: None (starting node).  
    - Output Connections: Connects to the "Fetch Price - SerpAPI" node.  
    - Version-Specific Requirements: Uses version 1 of the Cron node.  
    - Potential Failures: Misconfigured time zone may affect trigger timing; no authentication issues.  
    - Sub-workflow Reference: None.

#### 1.2 Price Fetching

- **Overview:**  
  This block fetches the current flight price data from SerpAPI’s Google Flights engine for a specified route and date.

- **Nodes Involved:**  
  - Fetch Price - SerpAPI

- **Node Details:**  

  - **Fetch Price - SerpAPI**  
    - Type and Technical Role: HTTP Request node configured to query SerpAPI for flight prices.  
    - Configuration:  
      - HTTP Method: GET (default for URL request).  
      - URL: `https://serpapi.com/search?engine=google_flights`  
      - Query Parameters:  
        - `departure_id`: "BLR" (Bangalore airport code).  
        - `arrival_id`: "DEL" (Delhi airport code).  
        - `outbound_date`: "2025-09-06" (flight date to monitor).  
        - `currency`: "INR" (Indian Rupee).  
        - `type`: "2" (likely indicating round-trip or a specific flight type).  
      - Authentication: Predefined credential using SerpAPI credentials.  
    - Key Expressions/Variables: None inside the node, all parameters fixed.  
    - Input Connections: Receives trigger from Cron node.  
    - Output Connections: Outputs JSON response to "Check Price Drop".  
    - Version-Specific Requirements: Uses version 2 of the HTTP Request node.  
    - Potential Failures:  
      - API key invalid or expired causing auth errors.  
      - Network timeout or SerpAPI service issues.  
      - Unexpected or empty API response structure.  
    - Sub-workflow Reference: None.

#### 1.3 Price Evaluation

- **Overview:**  
  Processes the API response to extract the best flight price and evaluates whether it is below a set threshold to determine if a price drop has occurred.

- **Nodes Involved:**  
  - Check Price Drop  
  - IF Price Dropped?

- **Node Details:**  

  - **Check Price Drop**  
    - Type and Technical Role: Function node that parses JSON input and computes a price drop boolean.  
    - Configuration:  
      - Extracts `best_flights[0].price` from the API response JSON.  
      - Uses a fallback price of 999,999 if extraction fails.  
      - Compares price with a threshold of 4000 INR.  
      - Outputs an object with:  
        - `priceDrop` (boolean)  
        - `currentPrice` (numeric)  
    - Key Expressions/Variables:  
      - `items[0].json` for input JSON.  
      - `price < threshold` for price drop logic.  
    - Input Connections: Receives data from "Fetch Price - SerpAPI".  
    - Output Connections: Sends result to "IF Price Dropped?" node.  
    - Version-Specific Requirements: Uses version 1 of Function node.  
    - Potential Failures:  
      - Malformed or missing `best_flights` array causing extraction error.  
      - Logic errors if API response structure changes.  
    - Sub-workflow Reference: None.

  - **IF Price Dropped?**  
    - Type and Technical Role: Conditional node that routes workflow based on price drop boolean.  
    - Configuration:  
      - Condition: Boolean check if `priceDrop` is true.  
    - Key Expressions/Variables:  
      - `={{ $json["priceDrop"] }}` for condition evaluation.  
    - Input Connections: Receives from "Check Price Drop".  
    - Output Connections:  
      - True branch: to "Send a message" node.  
      - False branch: ends workflow (no further nodes).  
    - Version-Specific Requirements: Uses version 1 of IF node.  
    - Potential Failures: Expression evaluation issues if input JSON malformed.  
    - Sub-workflow Reference: None.

#### 1.4 Notification

- **Overview:**  
  Sends an email notification via Gmail to alert the user about the flight price drop.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**  

  - **Send a message**  
    - Type and Technical Role: Gmail node that sends an email message.  
    - Configuration:  
      - Recipient: `x22yashc@iima.ac.in`  
      - Subject: "Hi Yashi! Flight price to Delhi Dropped"  
      - Message Body (text):  
        ```
        Hi Yash,
        
        The Flight price to Delhi dropped to {{ $json.currentPrice }}.
        
        You can book the flight now
        
        Regards
        Team Yash Choudhary
        ```  
      - Email Type: plain text  
      - Options: default (no attachments or advanced settings).  
      - Uses OAuth2 Gmail credentials.  
    - Key Expressions/Variables:  
      - `{{ $json.currentPrice }}` inserted dynamically in message body.  
    - Input Connections: Triggered from true branch of "IF Price Dropped?" node.  
    - Output Connections: None (workflow ends here).  
    - Version-Specific Requirements: Uses version 2.1 of Gmail node.  
    - Potential Failures:  
      - OAuth token expired or revoked causing auth failure.  
      - Gmail API limits or quota exceeded.  
      - Invalid recipient email format or delivery errors.  
    - Sub-workflow Reference: None.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                     | Input Node(s)           | Output Node(s)          | Sticky Note                                                |
|-----------------------|--------------------|-----------------------------------|------------------------|-------------------------|------------------------------------------------------------|
| Cron                  | Cron Trigger       | Scheduled daily trigger at 8:00AM | None                   | Fetch Price - SerpAPI   |                                                            |
| Fetch Price - SerpAPI | HTTP Request       | Fetch flight price from SerpAPI   | Cron                   | Check Price Drop        |                                                            |
| Check Price Drop      | Function           | Extract price and detect drop     | Fetch Price - SerpAPI   | IF Price Dropped?       |                                                            |
| IF Price Dropped?     | IF Condition       | Route workflow based on drop bool | Check Price Drop        | Send a message (true)   |                                                            |
| Send a message        | Gmail              | Send email alert about price drop | IF Price Dropped?       | None                    |                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron node:**  
   - Type: Cron Trigger  
   - Set trigger time to daily at 08:00 AM (local timezone).  
   - No credentials needed.

2. **Create an HTTP Request node (named "Fetch Price - SerpAPI"):**  
   - Connect input from Cron node.  
   - Set HTTP Method: GET (default).  
   - URL: `https://serpapi.com/search?engine=google_flights`  
   - Query Parameters:  
     - `departure_id`: "BLR"  
     - `arrival_id`: "DEL"  
     - `outbound_date`: "2025-09-06"  
     - `currency`: "INR"  
     - `type`: "2"  
   - Set Authentication to predefined credential type and select or create SerpAPI credentials (API key).  
   - Use node version 2.

3. **Create a Function node (named "Check Price Drop"):**  
   - Connect input from "Fetch Price - SerpAPI".  
   - Paste the following JavaScript code:
     ```javascript
     const result = items[0].json;
     let price = null;

     try {
       price = result.best_flights[0].price;
     } catch (e) {
       price = 999999; // fallback if no price found
     }

     const threshold = 4000; // update as needed

     return [{
       json: {
         priceDrop: price < threshold,
         currentPrice: price
       }
     }];
     ```
   - Use node version 1.

4. **Create an IF node (named "IF Price Dropped?"):**  
   - Connect input from "Check Price Drop".  
   - Set condition type: Boolean  
   - Condition: `{{$json["priceDrop"]}}` equals `true`.  
   - Use node version 1.

5. **Create a Gmail node (named "Send a message"):**  
   - Connect input from the "true" output branch of the "IF Price Dropped?" node.  
   - Credentials: Configure Gmail OAuth2 credentials with proper permissions to send emails.  
   - Recipient email: `x22yashc@iima.ac.in` (modify as needed).  
   - Subject: `Hi Yashi! Flight price to Delhi Dropped`  
   - Message type: Plain Text  
   - Message body:  
     ```
     Hi Yash,

     The Flight price to Delhi dropped to {{ $json.currentPrice }}.

     You can book the flight now

     Regards
     Team Yash Choudhary
     ```  
   - Use node version 2.1.

6. **Verify Connections:**  
   - Cron → Fetch Price - SerpAPI → Check Price Drop → IF Price Dropped? → (true) Send a message  
   - The false branch of IF node does not connect further (ends workflow if no drop).

7. **Activate the workflow:**  
   - Test by running manually or waiting for scheduled trigger.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                    |
|-----------------------------------------------------------------------------------------------|---------------------------------------------------|
| The threshold for price drop detection is currently set at 4000 INR, adjust this as needed.   | Parameter inside the Function node "Check Price Drop" |
| Gmail node requires OAuth2 credentials with send email scope properly configured.             | Gmail OAuth2 credential setup in n8n              |
| SerpAPI requires an active API key and might have usage limitations or quota restrictions.    | SerpAPI documentation: https://serpapi.com/docs/ |
| Flight search parameters (departure, arrival, date) are hardcoded; modify for other routes.   | Change query parameters in "Fetch Price - SerpAPI" node |
| Timezone of Cron node should be verified to match user’s local time for accurate scheduling.  | n8n Cron node settings                             |

---

Disclaimer: The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.
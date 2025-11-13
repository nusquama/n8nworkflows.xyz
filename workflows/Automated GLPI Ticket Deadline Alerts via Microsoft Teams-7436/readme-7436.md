Automated GLPI Ticket Deadline Alerts via Microsoft Teams

https://n8nworkflows.xyz/workflows/automated-glpi-ticket-deadline-alerts-via-microsoft-teams-7436


# Automated GLPI Ticket Deadline Alerts via Microsoft Teams

### 1. Workflow Overview

This workflow automates the notification process for tickets that are about to expire in GLPI (an IT asset and service management tool). It queries GLPI for tickets nearing their deadlines and sends alert messages to the appropriate support technicians via Microsoft Teams chats.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling & Configuration:** Triggers the workflow daily at a set time and initializes key configuration variables such as GLPI server URL and tokens.

- **1.2 Authentication & Data Retrieval:** Authenticates to the GLPI REST API to obtain a session token, then queries tickets that will expire within a defined time frame (default 2 days).

- **1.3 Data Processing & Looping:** Processes the list of expiring tickets, splitting and aggregating data for sequential handling.

- **1.4 Conditional Notification Routing:** Routes each ticket notification to the appropriate support technician based on ticket assignment.

- **1.5 Notification via Microsoft Teams:** Sends formatted alert messages to technician-specific Microsoft Teams chat channels.

- **1.6 Session Termination & Workflow End:** Properly closes the GLPI session and performs no-operation nodes as workflow placeholders.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling & Configuration

**Overview:**  
This block triggers the workflow every day at 9 AM and defines essential configuration variables including the GLPI API URL and the app token needed for authentication.

**Nodes Involved:**  
- Schedule Trigger  
- Configuration Variables

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Configuration: Fires daily at 09:00 (9 AM) server time.  
  - Inputs: None (trigger node)  
  - Outputs: Configuration Variables node  
  - Edge Cases: Misconfigured timezone may cause unexpected trigger times.

- **Configuration Variables**  
  - Type: Set  
  - Configuration: Sets two string variables:  
    - `glpi_url`: Base URL of the GLPI server API (e.g., "https://your_glpi_server.com")  
    - `app_token`: GLPI application token string for authentication  
  - Inputs: Schedule Trigger  
  - Outputs: Get session token node  
  - Edge Cases: Missing or incorrect URL or token will cause authentication failure downstream.

#### 2.2 Authentication & Data Retrieval

**Overview:**  
This block authenticates to GLPI using HTTP Basic Auth and retrieves a session token. It then performs a search query via GLPI API to fetch tickets that expire within two days.

**Nodes Involved:**  
- Get session token  
- Tickets about to expire

**Node Details:**

- **Get session token**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: `{{ $json.glpi_url }}/apirest.php/initSession`  
    - Method: GET (default)  
    - Headers: Content-Type: application/json, App-Token from configuration  
    - Authentication: HTTP Basic Auth via stored GLPI credentials  
  - Inputs: Configuration Variables  
  - Outputs: Tickets about to expire  
  - Edge Cases: Invalid credentials or app token result in HTTP 401; network issues cause request failure.

- **Tickets about to expire**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: `{{ $('Configuration Variables').item.json.glpi_url }}/apirest.php/search/Ticket`  
    - Query Parameters (criteria):  
      - Field 18 (likely due date or deadline) less than a calculated date two days ahead:  
        `{{new Date(Date.now() + (2 * 24 * 60 * 60 * 1000)).toISOString().split('T')[0]}}`  
    - Headers: Content-Type: application/json, Session-Token (from previous node), App-Token  
  - Inputs: Get session token  
  - Outputs: Split Out  
  - Edge Cases: Date calculation errors, expired session token, or API downtime can cause failures.

#### 2.3 Data Processing & Looping

**Overview:**  
This block splits the retrieved ticket data into individual items for processing and loops over them to aggregate data and conditionally route notifications.

**Nodes Involved:**  
- Split Out  
- Loop Over Items  
- Aggregate

**Node Details:**

- **Split Out**  
  - Type: Split Out  
  - Configuration: Splits the field `data` from the HTTP response into separate items.  
  - Inputs: Tickets about to expire  
  - Outputs: Loop Over Items  
  - Edge Cases: If the `data` field is missing or empty, no items will be processed.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Configuration: Default batch size (1 item per batch) to process tickets one by one.  
  - Inputs: Split Out; also connected downstream after notifications to re-aggregate.  
  - Outputs: Aggregate, Support Technician 1?  
  - Edge Cases: Empty input leads to no iterations; batch misconfiguration may cause performance issues.

- **Aggregate**  
  - Type: Aggregate  
  - Configuration: Aggregates all processed items back into a single data structure after loop completion.  
  - Inputs: Loop Over Items (second output path)  
  - Outputs: End session1  
  - Edge Cases: No input items cause empty aggregation.

#### 2.4 Conditional Notification Routing

**Overview:**  
This block decides which support technician should receive the notification message based on the ticket’s assigned user ID.

**Nodes Involved:**  
- Support Technician 1?  
- Support Technician 2?

**Node Details:**

- **Support Technician 1?**  
  - Type: If  
  - Configuration: Checks if the ticket attribute at JSON key `"5"` equals string `"7"` (user ID for Support Technician 1).  
  - Inputs: Loop Over Items  
  - Outputs: Send a message to Support Technician 1 (if true), Support Technician 2? (if false)  
  - Edge Cases: Missing or malformed `"5"` field can cause logical routing errors.

- **Support Technician 2?**  
  - Type: If  
  - Configuration: Checks if the ticket attribute at JSON key `"5"` equals string `"8"` (user ID for Support Technician 2).  
  - Inputs: Support Technician 1? (false branch)  
  - Outputs: Send a message to Support Technician 2 (if true), No Operation, do nothing3 (if false)  
  - Edge Cases: Same as above; tickets assigned to other users are effectively ignored.

#### 2.5 Notification via Microsoft Teams

**Overview:**  
This block sends alert messages to the respective technicians in Microsoft Teams, notifying them about tickets that are about to expire.

**Nodes Involved:**  
- Send a message to Support Technician 1  
- Send a message to Support Technician 2

**Node Details:**

- **Send a message to Support Technician 1**  
  - Type: Microsoft Teams  
  - Configuration:  
    - Chat ID: Hardcoded to a one-on-one chat for Support Technician 1  
    - Message: Markdown formatted, includes ticket title (`$json['1']`), ID (`$json['2']`), and expiry info (`$json['18']`)  
    - Credential: Microsoft Teams OAuth2  
  - Inputs: Support Technician 1? (true branch)  
  - Outputs: Loop Over Items (to continue looping)  
  - Edge Cases: Invalid chat ID or expired OAuth tokens can cause message failure.

- **Send a message to Support Technician 2**  
  - Type: Microsoft Teams  
  - Configuration:  
    - Chat ID: Hardcoded to a one-on-one chat for Support Technician 2  
    - Message: Same format as for Technician 1  
    - Credential: Microsoft Teams OAuth2  
  - Inputs: Support Technician 2? (true branch)  
  - Outputs: Loop Over Items (to continue looping)  
  - Edge Cases: Same as above.

#### 2.6 Session Termination & Workflow End

**Overview:**  
This block closes the GLPI session properly and includes no-operation nodes as placeholders or to maintain workflow structure.

**Nodes Involved:**  
- Aggregate  
- End session1  
- No Operation, do nothing2  
- No Operation, do nothing3

**Node Details:**

- **Aggregate**  
  - See above (2.3)

- **End session1**  
  - Type: HTTP Request  
  - Configuration:  
    - URL: `{{ $('Configuration Variables').item.json.glpi_url }}apirest.php/killSession`  
    - Headers: Session-Token (from Get session token), App-Token  
  - Inputs: Aggregate  
  - Outputs: No Operation, do nothing2  
  - Edge Cases: Session token already expired or invalid causes failure but no critical impact.

- **No Operation, do nothing2**  
  - Type: No Operation  
  - Purpose: Placeholder node, no processing.  
  - Inputs: End session1  
  - Outputs: None

- **No Operation, do nothing3**  
  - Type: No Operation  
  - Purpose: Placeholder node, no processing.  
  - Inputs: Support Technician 2? (false branch)  
  - Outputs: Loop Over Items (to continue processing)  

---

### 3. Summary Table

| Node Name                    | Node Type          | Functional Role                      | Input Node(s)               | Output Node(s)              | Sticky Note                                                    |
|------------------------------|--------------------|------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------|
| Schedule Trigger             | Schedule Trigger   | Triggers workflow daily at 9 AM    | None                        | Configuration Variables      |                                                               |
| Configuration Variables      | Set                | Holds GLPI URL and App Token       | Schedule Trigger            | Get session token           | Contains instructions on required GLPI URLs, tokens, and user ID mapping. |
| Get session token            | HTTP Request       | Authenticates and gets session token | Configuration Variables     | Tickets about to expire      |                                                               |
| Tickets about to expire      | HTTP Request       | Queries tickets expiring within 2 days | Get session token          | Split Out                   | Explains date calculation for ticket expiry filtering.        |
| Split Out                   | Split Out          | Splits ticket list into items      | Tickets about to expire     | Loop Over Items             |                                                               |
| Loop Over Items             | Split In Batches   | Processes tickets one-by-one        | Split Out, Send messages    | Aggregate, Support Technician 1? |                                                               |
| Aggregate                   | Aggregate          | Re-aggregates processed items      | Loop Over Items             | End session1                |                                                               |
| End session1                | HTTP Request       | Terminates GLPI session             | Aggregate                   | No Operation, do nothing2   |                                                               |
| No Operation, do nothing2   | No Operation       | Placeholder after session end       | End session1                | None                        |                                                               |
| Support Technician 1?       | If                 | Checks if ticket assigned to user ID 7 | Loop Over Items             | Send message to Technician 1, Support Technician 2? |                                                               |
| Send a message to Support Technician 1 | Microsoft Teams | Sends Teams message to Technician 1 | Support Technician 1?       | Loop Over Items             |                                                               |
| Support Technician 2?       | If                 | Checks if ticket assigned to user ID 8 | Support Technician 1?       | Send message to Technician 2, No Operation, do nothing3 |                                                               |
| Send a message to Support Technician 2 | Microsoft Teams | Sends Teams message to Technician 2 | Support Technician 2?       | Loop Over Items             |                                                               |
| No Operation, do nothing3   | No Operation       | Placeholder for tickets not assigned to IDs 7 or 8 | Support Technician 2?       | Loop Over Items             |                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Set to trigger once daily at 09:00 (server time).

2. **Create a Set Node named "Configuration Variables"**  
   - Define two string fields:  
     - `glpi_url` with your GLPI server URL (e.g., `https://your_glpi_server.com`)  
     - `app_token` with your GLPI application token.

3. **Connect Schedule Trigger to Configuration Variables.**

4. **Create an HTTP Request Node named "Get session token"**  
   - Method: GET  
   - URL: `{{ $json.glpi_url }}/apirest.php/initSession`  
   - Headers:  
     - `Content-Type`: `application/json`  
     - `App-Token`: `{{ $json.app_token }}`  
   - Authentication: HTTP Basic Auth with GLPI user credentials (create and select the credential in n8n).  
   - Connect Configuration Variables → Get session token.

5. **Create an HTTP Request Node named "Tickets about to expire"**  
   - Method: GET (default)  
   - URL: `{{ $('Configuration Variables').item.json.glpi_url }}/apirest.php/search/Ticket`  
   - Query Parameters:  
     - `criteria[0][field]`: `18` (field for deadline)  
     - `criteria[0][searchtype]`: `lessthan`  
     - `criteria[0][value]`: `{{new Date(Date.now() + (2 * 24 * 60 * 60 * 1000)).toISOString().split('T')[0]}}`  
   - Headers:  
     - `Content-Type`: `application/json`  
     - `Session-Token`: `{{ $json.session_token }}` (from Get session token)  
     - `App-Token`: `{{ $('Configuration Variables').item.json.app_token }}`  
   - Connect Get session token → Tickets about to expire.

6. **Create a Split Out Node named "Split Out"**  
   - Field to split out: `data`  
   - Connect Tickets about to expire → Split Out.

7. **Create a Split In Batches Node named "Loop Over Items"**  
   - Leave default batch size (1) for sequential processing.  
   - Connect Split Out → Loop Over Items.

8. **Create an Aggregate Node named "Aggregate"**  
   - Aggregate all item data after processing.  
   - Connect Loop Over Items (second output) → Aggregate.

9. **Create an HTTP Request Node named "End session1"**  
   - Method: GET (default)  
   - URL: `{{ $('Configuration Variables').item.json.glpi_url }}apirest.php/killSession`  
   - Headers:  
     - `Session-Token`: `{{ $('Get session token').item.json.session_token }}`  
     - `App-Token`: `{{ $('Configuration Variables').item.json.app_token }}`  
   - Connect Aggregate → End session1.

10. **Create a No Operation Node named "No Operation, do nothing2"**  
    - Connect End session1 → No Operation, do nothing2.

11. **Create an If Node named "Support Technician 1?"**  
    - Condition: Check if `{{ $json["5"] }}` equals string `"7"` (GLPI user ID for technician 1).  
    - Connect Loop Over Items → Support Technician 1?.

12. **Create an If Node named "Support Technician 2?"**  
    - Condition: Check if `{{ $json["5"] }}` equals string `"8"` (GLPI user ID for technician 2).  
    - Connect Support Technician 1? (false branch) → Support Technician 2?.

13. **Create Microsoft Teams Nodes:**  
    - "Send a message to Support Technician 1"  
      - Chat ID: Use the Teams chat ID for technician 1  
      - Message: Markdown with variables:  
        ```
        ## Action Required: Ticket about to expire

        **Title:** {{ $json['1'] }}

        **ID:** {{ $json['2'] }}

        **Expires in:** {{ $json['18'] }}
        ```  
      - Use Microsoft Teams OAuth2 credentials.  
      - Connect Support Technician 1? (true branch) → Send message node → Loop Over Items (to continue).

    - "Send a message to Support Technician 2"  
      - Chat ID: Use the Teams chat ID for technician 2  
      - Message: Same format as above.  
      - Use Microsoft Teams OAuth2 credentials.  
      - Connect Support Technician 2? (true branch) → Send message node → Loop Over Items.

14. **Create a No Operation Node named "No Operation, do nothing3"**  
    - Connect Support Technician 2? (false branch) → No Operation, do nothing3 → Loop Over Items (to continue processing next items).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                                |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Requires GLPI REST API access and a user with application administrator privileges.                                                                                                                                                                                                                                                                                                                                                  | See GLPI API documentation for authentication and token usage.                                                                |
| Update the "Configuration Variables" node with your GLPI server URL and your app token.                                                                                                                                                                                                                                                                                                                                              | Example: `glpi_url`: "https://your_glpi_server.com", `app_token`: "Your App Token is here".                                    |
| GLPI User IDs for technicians must be identified to properly route alerts. IDs can be found by navigating in GLPI Admin > Users and checking the ID in the browser URL.                                                                                                                                                                                                                                                         | For example, `id=7` for Support Technician 1 and `id=8` for Support Technician 2.                                               |
| The "Tickets about to expire" node uses dynamic date calculation to filter tickets with deadlines less than 2 days from current date. Modify the expression to adjust this window.                                                                                                                                                                                                                                                    | Example expression: `{{new Date(Date.now() + (2 * 24 * 60 * 60 * 1000)).toISOString().split('T')[0]}}`.                        |
| Microsoft Teams chat IDs must be obtained beforehand for correct message delivery.                                                                                                                                                                                                                                                                                                                                                  | Chat IDs are visible in Teams client links or can be obtained via Microsoft Graph API.                                         |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow created using n8n, an integration and automation platform. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.
Post images and text from Google Drive and Sheets to LinkedIn

https://n8nworkflows.xyz/workflows/post-images-and-text-from-google-drive-and-sheets-to-linkedin-4702


# Post images and text from Google Drive and Sheets to LinkedIn

---
### 1. Workflow Overview

This workflow automates the process of posting images and text content from Google Drive and Google Sheets to LinkedIn, with a manual approval step via Telegram. It is designed for social media managers or marketing teams who want to streamline content publishing while retaining control over post approvals.

The workflow logic is divided into the following main blocks:

- **1.1 Triggering and Data Retrieval:** Initiates the workflow on a schedule or Telegram command, retrieves the next post data from Google Sheets.
- **1.2 Post Availability Check:** Determines if there is a next post to process or sends a notification if none are available.
- **1.3 Telegram-Based Approval Workflow:** Sends the post content (image and text) to Telegram for manual approval and awaits response.
- **1.4 Posting to LinkedIn:** Upon approval, posts the content to LinkedIn via API and updates the Google Sheet accordingly.
- **1.5 Post-Processing Notifications:** Sends Telegram messages confirming success or decline, and updates the sheet on decline.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggering and Data Retrieval

- **Overview:**  
This block starts the workflow either on a scheduled timer or via a Telegram command. It then fetches post data from Google Sheets to find the next post ready for publication.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Telegram Trigger  
  - Google Sheets  
  - Find Next Post3 (Function)  

- **Node Details:**  

  - **Schedule Trigger**  
    - Type: Trigger (Schedule)  
    - Role: Automatically starts workflow on a predefined schedule (e.g., daily).  
    - Configuration: Uses cron or interval settings (default or user-defined).  
    - Inputs: None  
    - Outputs: Connected to Google Sheets node to retrieve data.  
    - Edge Cases: Scheduling errors, missed triggers due to downtime.

  - **Telegram Trigger**  
    - Type: Trigger (Telegram)  
    - Role: Starts workflow when receiving a specific Telegram command/message.  
    - Configuration: Requires Telegram credentials and webhook setup.  
    - Inputs: Telegram chat/message data.  
    - Outputs: Connected to Google Sheets node.  
    - Edge Cases: Telegram API outages, message parsing failures.

  - **Google Sheets**  
    - Type: Data Retrieval (Google Sheets)  
    - Role: Reads sheet data containing post metadata and content references.  
    - Configuration: Uses Google Sheets API credentials, accesses a specific spreadsheet and range.  
    - Inputs: Trigger from Schedule or Telegram.  
    - Outputs: Passes data to the function node to determine the next post.  
    - Edge Cases: Authentication errors, sheet access permission issues, empty or malformed data.

  - **Find Next Post3**  
    - Type: Function (JavaScript)  
    - Role: Processes sheet data to find the next post that has not been published or declined.  
    - Configuration: Contains logic to filter posts based on status columns.  
    - Inputs: Data array from Google Sheets.  
    - Outputs: Passes filtered post data to the next block.  
    - Edge Cases: Logic errors, missing fields, empty post list.

---

#### 2.2 Post Availability Check

- **Overview:**  
Checks if a valid post is available. If no post is found, sends a Telegram notification to inform no posts are pending; otherwise, proceeds to send content for approval.

- **Nodes Involved:**  
  - Check If Post Available (IF)  
  - Telegram No Posts  
  - HTTP Request1  

- **Node Details:**  

  - **Check If Post Available**  
    - Type: IF Condition  
    - Role: Evaluates if the function node returned any post data.  
    - Configuration: Checks if the filtered posts array length > 0 or if a key field is populated.  
    - Inputs: Output from Find Next Post3.  
    - Outputs: True branch to HTTP Request1 (send photo), False branch to Telegram No Posts.  
    - Edge Cases: Logic errors in condition, unexpected data format.

  - **Telegram No Posts**  
    - Type: Telegram Message  
    - Role: Sends a Telegram message indicating no posts are available to publish.  
    - Configuration: Uses Telegram bot credentials and chat ID.  
    - Inputs: Triggered if no post is available.  
    - Outputs: None (end of branch).  
    - Edge Cases: Telegram API downtime, message delivery failure.

  - **HTTP Request1**  
    - Type: HTTP Request  
    - Role: Downloads or retrieves the image asset from Google Drive or a URL to be sent to Telegram.  
    - Configuration: Configured for GET request to the image URL extracted from post data.  
    - Inputs: Post data including image URL.  
    - Outputs: Passes image binary data to Send Photo node.  
    - Edge Cases: HTTP errors (404, timeout), invalid URLs.

---

#### 2.3 Telegram-Based Approval Workflow

- **Overview:**  
Sends the image and text content to Telegram for manual approval, then waits for user response. Based on approval or decline, directs workflow accordingly.

- **Nodes Involved:**  
  - Send Photo (Telegram)  
  - Telegram Approval (Telegram)  
  - Check Approval (IF)  
  - HTTP Request  
  - Code1  
  - Code  

- **Node Details:**  

  - **Send Photo**  
    - Type: Telegram Message (photo)  
    - Role: Sends the post image to Telegram chat for review.  
    - Configuration: Uses Telegram credentials, photo binary from HTTP Request1, caption with post text.  
    - Inputs: Image binary and text data.  
    - Outputs: Triggers Telegram Approval node to wait for user reply.  
    - Edge Cases: Photo size limits, Telegram API limits.

  - **Telegram Approval**  
    - Type: Telegram Trigger (Webhook)  
    - Role: Listens for approval or decline commands from Telegram user.  
    - Configuration: Webhook set up with Telegram bot, expects specific commands for approval or decline.  
    - Inputs: User’s Telegram response.  
    - Outputs: To Check Approval IF node.  
    - Edge Cases: Missing or invalid commands, webhook connectivity.

  - **Check Approval**  
    - Type: IF Condition  
    - Role: Checks if the user approved or declined the post.  
    - Configuration: Evaluates Telegram response text or payload for approval keyword.  
    - Inputs: Telegram Approval node output.  
    - Outputs: True branch to HTTP Request (for LinkedIn post), False branch to Code1 (for decline handling).  
    - Edge Cases: Ambiguous responses, delayed responses.

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Sends the post data to LinkedIn API for publishing.  
    - Configuration: Configured with LinkedIn API URL, headers, and payload including text and media references. Uses OAuth2 credentials.  
    - Inputs: Approved post data.  
    - Outputs: On success, passes to LinkedIn node.  
    - Edge Cases: API rate limits, authentication failures, invalid payloads.

  - **Code1**  
    - Type: Code (JavaScript)  
    - Role: Handles post decline logic, preparing data to update the Google Sheet with declined status.  
    - Configuration: Updates post status to declined and prepares sheet update data.  
    - Inputs: Decline branch from Check Approval.  
    - Outputs: Update Sheet - Declined node.  
    - Edge Cases: Logic errors, malformed data.

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Processes LinkedIn response and prepares data to update Google Sheet with posted status.  
    - Configuration: Extracts post ID or status, formats update request.  
    - Inputs: Output from LinkedIn node.  
    - Outputs: Update Sheet - Posted node.  
    - Edge Cases: API response errors, missing data fields.

---

#### 2.4 Posting to LinkedIn and Sheet Update

- **Overview:**  
Posts content to LinkedIn and updates the Google Sheet to mark the post as published.

- **Nodes Involved:**  
  - LinkedIn  
  - Update Sheet - Posted  
  - Telegram Success  

- **Node Details:**  

  - **LinkedIn**  
    - Type: LinkedIn API Node  
    - Role: Publishes the post to LinkedIn using OAuth2 credentials.  
    - Configuration: Requires LinkedIn app credentials, scopes for posting, and payload including text and media.  
    - Inputs: Prepared post data from HTTP Request.  
    - Outputs: Passes LinkedIn response to Code node.  
    - Edge Cases: Token expiration, API limits, permission errors.

  - **Update Sheet - Posted**  
    - Type: Google Sheets  
    - Role: Updates the post record in Google Sheets marking it as posted.  
    - Configuration: Uses Google Sheets API to update a particular row and column status.  
    - Inputs: Data from Code node indicating post success.  
    - Outputs: Triggers Telegram Success notification.  
    - Edge Cases: Sheet write conflicts, API quota.

  - **Telegram Success**  
    - Type: Telegram Message  
    - Role: Sends confirmation message to Telegram chat indicating post was successfully published.  
    - Configuration: Uses Telegram bot credentials and chat ID.  
    - Inputs: Triggered after sheet update.  
    - Outputs: None (end of workflow).  
    - Edge Cases: Telegram API issues.

---

#### 2.5 Decline Handling and Notifications

- **Overview:**  
Handles post decline cases by updating Google Sheets and notifying via Telegram.

- **Nodes Involved:**  
  - Update Sheet - Declined  
  - Telegram Decline  

- **Node Details:**  

  - **Update Sheet - Declined**  
    - Type: Google Sheets  
    - Role: Updates the post status to declined in Google Sheets.  
    - Configuration: Writes decline status to the relevant row.  
    - Inputs: Output from Code1 node.  
    - Outputs: Triggers Telegram Decline notification.  
    - Edge Cases: API write errors, incorrect row identification.

  - **Telegram Decline**  
    - Type: Telegram Message  
    - Role: Sends a message to Telegram indicating the post was declined.  
    - Configuration: Uses Telegram credentials and chat ID.  
    - Inputs: Triggered after sheet update.  
    - Outputs: None (end of workflow).  
    - Edge Cases: Telegram delivery issues.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                              | Input Node(s)           | Output Node(s)           | Sticky Note                    |
|-------------------------|---------------------|----------------------------------------------|-------------------------|--------------------------|-------------------------------|
| Schedule Trigger        | Schedule Trigger    | Starts workflow on schedule                   | -                       | Google Sheets            |                               |
| Telegram Trigger        | Telegram Trigger    | Starts workflow on Telegram command           | -                       | Google Sheets            |                               |
| Google Sheets           | Google Sheets       | Reads posts data from sheet                    | Schedule Trigger, Telegram Trigger | Find Next Post3            | Sticky Note - Data             |
| Find Next Post3         | Function            | Finds next post to process                      | Google Sheets            | Check If Post Available  | Sticky Note - Selection        |
| Check If Post Available | IF Condition        | Checks post availability                        | Find Next Post3          | HTTP Request1, Telegram No Posts | Sticky Note - Check        |
| Telegram No Posts       | Telegram Message    | Notifies no posts available                     | Check If Post Available  | -                        |                               |
| HTTP Request1           | HTTP Request        | Retrieves post image                            | Check If Post Available  | Send Photo               |                               |
| Send Photo              | Telegram Message    | Sends image and text to Telegram for approval | HTTP Request1            | Telegram Approval        | Sticky Note - Approval         |
| Telegram Approval       | Telegram Trigger    | Waits for approval response                     | Send Photo               | Check Approval           |                               |
| Check Approval          | IF Condition        | Decides approval or decline                     | Telegram Approval        | HTTP Request, Code1      |                               |
| HTTP Request            | HTTP Request        | Posts content to LinkedIn API                   | Check Approval (approved) | LinkedIn                 | Sticky Note - Publish          |
| LinkedIn                | LinkedIn API        | Publishes post to LinkedIn                       | HTTP Request             | Code                     |                               |
| Code                    | Code                | Processes LinkedIn response for sheet update   | LinkedIn                 | Update Sheet - Posted    |                               |
| Update Sheet - Posted   | Google Sheets       | Marks post as posted in sheet                    | Code                     | Telegram Success         |                               |
| Telegram Success        | Telegram Message    | Confirms success via Telegram                    | Update Sheet - Posted    | -                        |                               |
| Code1                   | Code                | Processes decline action for sheet update       | Check Approval (declined) | Update Sheet - Declined  |                               |
| Update Sheet - Declined | Google Sheets       | Marks post as declined in sheet                  | Code1                    | Telegram Decline         |                               |
| Telegram Decline        | Telegram Message    | Notifies decline via Telegram                     | Update Sheet - Declined  | -                        |                               |
| Sticky Note - Trigger   | Sticky Note         | Visual grouping for trigger block                | -                       | -                        |                               |
| Sticky Note - Data      | Sticky Note         | Visual grouping for data retrieval block         | -                       | -                        |                               |
| Sticky Note - Selection | Sticky Note         | Visual grouping for post selection logic         | -                       | -                        |                               |
| Sticky Note - Check     | Sticky Note         | Visual grouping for post availability check      | -                       | -                        |                               |
| Sticky Note - Approval  | Sticky Note         | Visual grouping for Telegram approval process    | -                       | -                        |                               |
| Sticky Note - Publish   | Sticky Note         | Visual grouping for posting to LinkedIn          | -                       | -                        |                               |
| Sticky Note - Decline   | Sticky Note         | Visual grouping for decline handling              | -                       | -                        |                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set trigger frequency (e.g., daily at a specific time).  
   - No inputs required.

2. **Create a Telegram Trigger node**  
   - Configure with Telegram bot credentials.  
   - Setup webhook for receiving commands/messages.  
   - No inputs required.

3. **Create a Google Sheets node**  
   - Connect both Schedule Trigger and Telegram Trigger outputs to this node.  
   - Configure credentials for Google Sheets API.  
   - Set spreadsheet ID and range covering posts data (including image URLs, text, status columns).  
   - Set operation to “Read Rows”.

4. **Create a Function node "Find Next Post3"**  
   - Connect Google Sheets output to this node.  
   - Add JavaScript code to filter for the next post where status is neither “posted” nor “declined”.  
   - Output filtered post data.

5. **Create an IF node "Check If Post Available"**  
   - Connect from Function node.  
   - Condition: Check if filtered post array is empty or not.  
   - True branch: Proceed to download image.  
   - False branch: Send Telegram notification “No posts available”.

6. **Create a Telegram Message node "Telegram No Posts"**  
   - Connect from IF node’s false output.  
   - Configure to send message to Telegram chat informing no posts are pending.

7. **Create an HTTP Request node "HTTP Request1"**  
   - Connect from IF node’s true output.  
   - Configure as GET request to the image URL from post data.  
   - Set to download binary data.

8. **Create a Telegram Message node "Send Photo"**  
   - Connect from HTTP Request1.  
   - Configure to send photo with caption (post text) to Telegram chat.  
   - Use Telegram credentials.

9. **Create a Telegram Trigger node "Telegram Approval"**  
   - Configure to listen for approval/decline commands from Telegram chat.  
   - Connect output to next IF node.

10. **Create an IF node "Check Approval"**  
    - Connect from Telegram Approval.  
    - Condition: Check if message text equals “approve” or similar keyword.  
    - True: Proceed to post to LinkedIn.  
    - False: Proceed to decline handling.

11. **Create an HTTP Request node**  
    - Connect from IF node’s true output.  
    - Configure POST request to LinkedIn API endpoint for creating posts.  
    - Use OAuth2 LinkedIn credentials.  
    - Prepare payload with post text and media references.

12. **Create a LinkedIn node**  
    - Connect from HTTP Request node.  
    - Handles LinkedIn API interaction (if using n8n LinkedIn node).  
    - Passes response to Code node.

13. **Create a Code node "Code"**  
    - Connect from LinkedIn node.  
    - Extract relevant response data (e.g., post ID).  
    - Prepare update payload for Google Sheets.

14. **Create a Google Sheets node "Update Sheet - Posted"**  
    - Connect from Code node.  
    - Configure to update the spreadsheet row status to “posted”.  
    - Use the same spreadsheet and credentials as before.

15. **Create a Telegram Message node "Telegram Success"**  
    - Connect from Update Sheet - Posted.  
    - Send confirmation message to Telegram.

16. **Create a Code node "Code1"**  
    - Connect from IF node’s false output (decline).  
    - Prepare update data to mark post as “declined”.

17. **Create a Google Sheets node "Update Sheet - Declined"**  
    - Connect from Code1.  
    - Update spreadsheet row status to “declined”.

18. **Create a Telegram Message node "Telegram Decline"**  
    - Connect from Update Sheet - Declined.  
    - Notify in Telegram about the decline.

19. **Arrange all connections as per the logical flow above.**

20. **Create Sticky Notes for visual grouping (optional).**

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow integrates Google Sheets, Google Drive (image URLs), Telegram (for approval), and LinkedIn API for posting. | Multi-service integration requires proper credential setup for each service in n8n.             |
| Telegram commands must be clearly defined (e.g., “approve”, “decline”) for the approval step to function correctly.        | Telegram bot setup documentation: https://core.telegram.org/bots/api                            |
| LinkedIn API requires OAuth2 authentication with appropriate permissions to post content programmatically.               | LinkedIn Developer docs: https://learn.microsoft.com/en-us/linkedin/marketing/integrations      |
| Google Sheets API quotas and limits may affect high-frequency runs.                                                                      | Google Sheets API docs: https://developers.google.com/sheets/api                                |
| Consider implementing error handling for HTTP requests and API interactions to manage rate limits and transient failures.   |                                                                                                 |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.
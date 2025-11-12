Automated Instagram Comment Response with DMs & Google Sheets Tracking

https://n8nworkflows.xyz/workflows/automated-instagram-comment-response-with-dms---google-sheets-tracking-5941


# Automated Instagram Comment Response with DMs & Google Sheets Tracking

### 1. Workflow Overview

This workflow automates the process of monitoring Instagram post comments, replying automatically via direct messages (DMs), and tracking interactions in Google Sheets. It is designed to:

- Periodically check new comments on a specified Instagram post or reel.
- Filter out comments that have already been replied to.
- Send a predefined reply message to new commenters.
- Log successful and failed replies for auditing.
- Maintain a Google Sheets document with records of contacted users and their comment details.
- Provide a summary report after each monitoring cycle.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Configuration:** Initiates the workflow manually or on a schedule and sets up target post details and reply messages.
- **1.2 Data Retrieval:** Reads the list of already contacted users from Google Sheets and fetches comments from the Instagram post.
- **1.3 Comment Filtering:** Filters out comments that have already been replied to, ensuring no duplicate replies.
- **1.4 Reply Processing:** Sends replies to new comments, verifies success, and updates Google Sheets accordingly.
- **1.5 Error Handling and Logging:** Captures failed reply attempts and logs them for troubleshooting.
- **1.6 Summary Generation:** Produces a summary of the workflow run, including counts of comments processed, replies sent, and failures.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Configuration Block

**Overview:**  
Starts workflow execution either manually or on a schedule and sets the Instagram post URL, reply message template, and profile username to monitor.

**Nodes Involved:**  
- Start Monitoring (Manual Trigger)  
- Schedule Trigger  
- Configure Post & Message (Set node)

**Node Details:**

- **Start Monitoring**  
  - Type: Manual Trigger  
  - Role: Enables manual start for debugging or on-demand runs.  
  - Config: No parameters; simply triggers the workflow.  
  - Inputs: None  
  - Outputs: Connects to Configure Post & Message  
  - Failure cases: None typical; manual user action required.

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automates workflow execution every 15 minutes.  
  - Config: Interval set to 15 minutes.  
  - Inputs: None  
  - Outputs: Connects to Configure Post & Message  
  - Failure cases: Workflow scheduling service downtime or misconfiguration.

- **Configure Post & Message**  
  - Type: Set  
  - Role: Defines key parameters for the monitoring session, such as post URL, reply message, and Instagram profile username.  
  - Config:  
    - `postUrl`: Instagram post or reel URL (string)  
    - `replyMessage`: Reply text template, e.g., "Thanks so much! This is the template: xxxxx"  
    - `profileUsername`: Instagram profile username being monitored  
  - Inputs: From Start Monitoring or Schedule Trigger  
  - Outputs: To Read Contacted Users  
  - Failure cases: Incorrect or missing configuration values will affect downstream API calls.

---

#### 2.2 Data Retrieval Block

**Overview:**  
Fetches data from external sources: reads already contacted users from Google Sheets and retrieves current comments on the Instagram post via API.

**Nodes Involved:**  
- Read Contacted Users (Google Sheets)  
- Get Post Comments (HTTP Request)

**Node Details:**

- **Read Contacted Users**  
  - Type: Google Sheets  
  - Role: Reads the full list of users who have already been contacted to avoid duplicate replies.  
  - Config:  
    - Document ID and Sheet Name configured to a specific Google Sheets document and sheet ("Hoja 1")  
    - Uses OAuth2 credentials for Google Sheets access.  
  - Inputs: From Configure Post & Message  
  - Outputs: To Get Post Comments  
  - Failure cases: Authentication errors, spreadsheet access issues, or empty sheet data.

- **Get Post Comments**  
  - Type: HTTP Request  
  - Role: Calls an external API to fetch comments for the specified Instagram post.  
  - Config:  
    - URL: `https://api.upload-post.com/api/uploadposts/comments`  
    - Query Parameters:
      - `platform`: "instagram"  
      - `user`: The profile username from Configure Post & Message  
      - `post_url`: Post URL from Configure Post & Message  
    - Authentication via HTTP Header credential (Upload-post API key)  
  - Inputs: From Read Contacted Users  
  - Outputs: To Filter New Comments  
  - Failure cases: Network issues, invalid credentials, API rate limits, malformed URLs.

---

#### 2.3 Comment Filtering Block

**Overview:**  
Processes the list of comments and filters out any that were already replied to, based on the Google Sheets log.

**Nodes Involved:**  
- Filter New Comments (Code)  
- Check If Has New Comments (If node)

**Node Details:**

- **Filter New Comments**  
  - Type: Code (JavaScript)  
  - Role:  
    - Parses comments from API response.  
    - Normalizes comment IDs (handling long Instagram IDs as strings).  
    - Extracts and flattens Google Sheets data to a usable array.  
    - Builds a set of contacted comment IDs from Google Sheets data.  
    - Filters out comments with IDs already present in the contacted set.  
    - Logs detailed debug information about data types and filtering steps.  
    - Returns only new comments for further processing or a special object indicating no new comments.  
  - Inputs: From Get Post Comments  
  - Outputs: To Check If Has New Comments  
  - Key expressions: Uses `$input.first().json.comments`, `$('Read Contacted Users').all()`, and values from Configure Post & Message for contextual data.  
  - Failure cases: Unexpected data formats from Google Sheets or API, missing comment IDs, JavaScript errors.  
  - Notes: Handles multiple Google Sheets data response formats and skips header rows carefully.

- **Check If Has New Comments**  
  - Type: If  
  - Role: Determines whether there are new comments to process or if the workflow should skip replying.  
  - Config: Checks if JSON field `noNewComments` is true or false.  
  - Inputs: From Filter New Comments  
  - Outputs:  
    - True branch: To Send Reply to Comment (if new comments exist)  
    - False branch: To Generate Summary (if no new comments)  
  - Failure cases: Expression evaluation errors if `noNewComments` field missing or malformed.

---

#### 2.4 Reply Processing Block

**Overview:**  
Sends the reply message to each new comment and records the outcome.

**Nodes Involved:**  
- Send Reply to Comment (HTTP Request)  
- Check Reply Success (If)  
- Record Contacted User (Google Sheets)  
- Log Failed Reply (Code)

**Node Details:**

- **Send Reply to Comment**  
  - Type: HTTP Request  
  - Role: Sends an HTTP POST request to an external API to reply to a comment on Instagram.  
  - Config:  
    - URL: `https://api.upload-post.com/api/uploadposts/comments/reply`  
    - Method: POST  
    - JSON Body includes platform ("instagram"), user profile username, comment ID, and the reply message.  
    - Authentication: HTTP Header credential (Upload-post API key).  
  - Inputs: From Check If Has New Comments (True branch)  
  - Outputs: To Check Reply Success  
  - Failure cases: API errors, authentication failure, network timeout, invalid comment or user data.

- **Check Reply Success**  
  - Type: If  
  - Role: Checks if the reply API call returned success (`success` field true).  
  - Inputs: From Send Reply to Comment  
  - Outputs:  
    - True branch: To Record Contacted User  
    - False branch: To Log Failed Reply  
  - Failure cases: Missing or malformed `success` field.

- **Record Contacted User**  
  - Type: Google Sheets  
  - Role: Appends a new row to the Google Sheets tracking document logging the replied comment details.  
  - Config:  
    - Document ID and Sheet Name same as Read Contacted Users.  
    - Columns mapped: post_url, username, timestamp, comment_id, message_sent (from Filter New Comments outputs).  
    - Operation: Append.  
    - Credentials: Google Sheets OAuth2.  
  - Inputs: From Check Reply Success (True branch)  
  - Outputs: To Generate Summary  
  - Failure cases: Google Sheets API errors, permission issues, data type mismatches.

- **Log Failed Reply**  
  - Type: Code (JavaScript)  
  - Role: Logs detailed information about failed reply attempts to console for debugging.  
  - Inputs: From Check Reply Success (False branch)  
  - Outputs: To Generate Summary  
  - Failure cases: None typical; safeguards in code.

---

#### 2.5 Summary Generation Block

**Overview:**  
Creates a detailed summary of the monitoring session, including totals of comments found, new comments processed, successful and failed replies, and logs this information.

**Nodes Involved:**  
- Generate Summary (Code)

**Node Details:**

- **Generate Summary**  
  - Type: Code (JavaScript)  
  - Role:  
    - Attempts to gather statistics from previous nodes (total comments, new comments, successful and failed replies).  
    - Handles missing data gracefully with try-catch.  
    - Retrieves monitored post URL from configuration.  
    - Constructs a summary object with counts and status.  
    - Logs a human-readable message with emojis reflecting workflow success or issues.  
    - Indicates continuation of monitoring cycles.  
  - Inputs: From Record Contacted User and Log Failed Reply (both converge here), also indirectly from Get Post Comments and Filter New Comments.  
  - Outputs: Final node output with a JSON summary and a message.  
  - Failure cases: Node data missing if earlier nodes skipped or failed, handled by try-catch blocks.

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                                | Input Node(s)                 | Output Node(s)               | Sticky Note                     |
|-----------------------|--------------------|-----------------------------------------------|------------------------------|-----------------------------|--------------------------------|
| Start Monitoring      | Manual Trigger     | Initiate workflow manually                      | None                         | Configure Post & Message     |                                |
| Schedule Trigger      | Schedule Trigger   | Automatically trigger workflow every 15 mins   | None                         | Configure Post & Message     |                                |
| Configure Post & Message | Set                | Define post URL, reply template, and username | Start Monitoring, Schedule Trigger | Read Contacted Users        |                                |
| Read Contacted Users  | Google Sheets      | Fetch list of users already contacted           | Configure Post & Message      | Get Post Comments            |                                |
| Get Post Comments     | HTTP Request       | Retrieve comments from Instagram post           | Read Contacted Users          | Filter New Comments          |                                |
| Filter New Comments   | Code               | Filter out already replied comments             | Get Post Comments             | Check If Has New Comments    | Extensive debug logs for filtered comments and contacted users data. |
| Check If Has New Comments | If                 | Check if new comments exist                      | Filter New Comments           | Send Reply to Comment (true), Generate Summary (false) |                                |
| Send Reply to Comment | HTTP Request       | Send reply message to comment via API           | Check If Has New Comments (true) | Check Reply Success          |                                |
| Check Reply Success   | If                 | Verify if reply was successful                   | Send Reply to Comment         | Record Contacted User (true), Log Failed Reply (false) |                                |
| Record Contacted User | Google Sheets      | Log successful reply to Google Sheets            | Check Reply Success (true)    | Generate Summary             |                                |
| Log Failed Reply      | Code               | Log failed reply info for debugging              | Check Reply Success (false)   | Generate Summary             |                                |
| Generate Summary      | Code               | Summarize monitoring session results             | Record Contacted User, Log Failed Reply | None                    |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named "Start Monitoring" with default settings (no parameters).  
   - Add a **Schedule Trigger** node named "Schedule Trigger" configured to run every 15 minutes.

2. **Configure Post & Message Node (Set node)**  
   - Add a **Set** node named "Configure Post & Message".  
   - Add three string fields:  
     - `postUrl`: set to the Instagram post or reel URL (e.g., "Instagram url post/reel").  
     - `replyMessage`: set to the reply message template (e.g., "Thanks so much! This is the template: xxxxx").  
     - `profileUsername`: set to the Instagram profile username to monitor (e.g., "add_upload_post_username").  
   - Connect outputs of "Start Monitoring" and "Schedule Trigger" to this node.

3. **Read Contacted Users (Google Sheets Node)**  
   - Add a **Google Sheets** node named "Read Contacted Users".  
   - Configure it to **Read** operation from the Google Sheets document with ID: `1SNZYngGfvIxZJ6VAiwD9Keuj6DugbijLH59vyK4L298`.  
   - Set Sheet Name or GID to `gid=0` (the first sheet).  
   - Use Google Sheets OAuth2 credentials.  
   - Connect output of "Configure Post & Message" to this node.

4. **Get Post Comments (HTTP Request Node)**  
   - Add an **HTTP Request** node named "Get Post Comments".  
   - Set method to GET.  
   - URL: `https://api.upload-post.com/api/uploadposts/comments`  
   - Add query parameters:  
     - `platform` = "instagram"  
     - `user` = `={{ $('Configure Post & Message').item.json.profileUsername }}`  
     - `post_url` = `={{ $('Configure Post & Message').item.json.postUrl }}`  
   - Use HTTP Header Auth credentials (Upload-post API key).  
   - Connect output of "Read Contacted Users" to this node.

5. **Filter New Comments (Code Node)**  
   - Add a **Code** node named "Filter New Comments".  
   - Paste the detailed JavaScript from the original code section that:  
     - Extracts comments, normalizes comment IDs, flattens Google Sheets data, filters out contacted comment IDs, and returns only new comments.  
   - Connect output of "Get Post Comments" to this node.

6. **Check If Has New Comments (If Node)**  
   - Add an **If** node named "Check If Has New Comments".  
   - Configure condition to check if `$json.noNewComments` is **not equal** to `true`.  
   - True branch: connects to "Send Reply to Comment".  
   - False branch: connects to "Generate Summary".

7. **Send Reply to Comment (HTTP Request Node)**  
   - Add an **HTTP Request** node named "Send Reply to Comment".  
   - Set method to POST.  
   - URL: `https://api.upload-post.com/api/uploadposts/comments/reply`  
   - JSON Body:  
     ```json
     {
       "platform": "instagram",
       "user": "{{ $json.profileUsername }}",
       "comment_id": "{{ $json.commentId }}",
       "message": "{{ $json.replyMessage }}"
     }
     ```  
   - Use HTTP Header Auth credentials (Upload-post API key).  
   - Connect True output of "Check If Has New Comments" to this node.

8. **Check Reply Success (If Node)**  
   - Add an **If** node named "Check Reply Success".  
   - Condition: check if `$json.success` equals `true`.  
   - True branch: to "Record Contacted User".  
   - False branch: to "Log Failed Reply".  
   - Connect output of "Send Reply to Comment" here.

9. **Record Contacted User (Google Sheets Node)**  
   - Add a **Google Sheets** node named "Record Contacted User".  
   - Operation: Append.  
   - Document ID and Sheet: same as "Read Contacted Users".  
   - Map columns:  
     - `post_url` = `={{ $('Filter New Comments').item.json.postUrl }}`  
     - `username` = `={{ $('Filter New Comments').item.json.username }}`  
     - `timestamp` = `={{ $('Filter New Comments').item.json.timestamp }}`  
     - `comment_id` = `={{ $('Filter New Comments').item.json.commentId }}`  
     - `message_sent` = `={{ $('Filter New Comments').item.json.replyMessage }}`  
   - Use Google Sheets OAuth2 credentials.  
   - Connect True output of "Check Reply Success" here.

10. **Log Failed Reply (Code Node)**  
    - Add a **Code** node named "Log Failed Reply".  
    - Paste code that logs failed reply details including userId, username, commentId, error message, and timestamp to the console.  
    - Connect False output of "Check Reply Success" here.

11. **Generate Summary (Code Node)**  
    - Add a **Code** node named "Generate Summary".  
    - Paste code that collects statistics from previous nodes and outputs a summary JSON with counts and status messages.  
    - Connect outputs of "Record Contacted User" and "Log Failed Reply" to this node.  
    - Also connect False output of "Check If Has New Comments" to this node.

12. **Finalize Workflow Connections**  
    - Connect "Start Monitoring" and "Schedule Trigger" to "Configure Post & Message".  
    - Connect "Configure Post & Message" to "Read Contacted Users".  
    - Connect "Read Contacted Users" to "Get Post Comments".  
    - Connect "Get Post Comments" to "Filter New Comments".  
    - Connect "Filter New Comments" to "Check If Has New Comments".  
    - Connect True branch of "Check If Has New Comments" to "Send Reply to Comment".  
    - Connect "Send Reply to Comment" to "Check Reply Success".  
    - Connect True branch of "Check Reply Success" to "Record Contacted User".  
    - Connect False branch of "Check Reply Success" to "Log Failed Reply".  
    - Connect outputs of "Record Contacted User", "Log Failed Reply", and False branch of "Check If Has New Comments" to "Generate Summary".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                               | Context or Link                                 |
|------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| This workflow uses an external API (https://api.upload-post.com) for Instagram comments and replies; ensure valid API credentials are configured.          | Upload-post API documentation                   |
| Google Sheets is used as a persistent data store for contacted users to avoid duplicate replies. The sheet must have columns: comment_id, username, etc.   | Google Sheets API OAuth2 credential setup      |
| The JavaScript code for filtering comments includes extensive debug logging to console; useful for troubleshooting data format issues.                     | Debugging complex Google Sheets data formats   |
| The workflow is designed to run every 15 minutes but can also be triggered manually for testing or immediate execution.                                     | Schedule Trigger setup                          |
| Comment IDs are handled as strings to avoid JavaScript number precision issues, critical for Instagram's long numeric IDs.                                 | Data normalization strategy                      |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.
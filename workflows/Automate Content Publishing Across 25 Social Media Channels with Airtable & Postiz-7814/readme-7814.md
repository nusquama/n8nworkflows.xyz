Automate Content Publishing Across 25 Social Media Channels with Airtable & Postiz

https://n8nworkflows.xyz/workflows/automate-content-publishing-across-25-social-media-channels-with-airtable---postiz-7814


# Automate Content Publishing Across 25 Social Media Channels with Airtable & Postiz

### 1. Workflow Overview

This n8n workflow automates the publishing of videos and images across 25 social media platforms using Airtable as the content source and Postiz as the publishing platform. It is designed to:

- Retrieve media content and metadata from an Airtable base.
- Upload media files (images and videos) to Postiz.
- Schedule and post content to multiple platforms connected via Postiz, including Dev.to, Hashnode, WordPress, and Dribble.
- Update Airtable records post-publication with success or failure logs.
- Allow scheduling flexibility and error handling during the posting processes.

**Logical blocks:**

- **1.1 Input Reception and Data Retrieval:** Receiving webhook triggers and fetching content data from Airtable.
- **1.2 Preparation for Publishing:** Formatting post data and preparing media for upload.
- **1.3 Media Handling:** Downloading media files from Airtable URLs and uploading them to Postiz.
- **1.4 Posting to Platforms via Postiz:** Scheduling and posting to each social platform with individual nodes.
- **1.5 Post-Publish Processing:** Updating Airtable with success/failure states and managing logs.
- **1.6 Supporting Documentation and Setup Notes:** Sticky notes providing setup instructions and references.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Data Retrieval

**Overview:**  
Handles incoming webhook POST requests that signal new content creation and fetches the full content record from Airtable.

**Nodes Involved:**  
- Webhook from Content Creation  
- Airtable Record ID  
- Airtable  

**Node Details:**

- **Webhook from Content Creation**  
  - Type: Webhook  
  - Role: Entry point; receives POST requests with content creation data, triggering workflow execution.  
  - Config: HTTP POST method, unique webhook path.  
  - Inputs: HTTP request payload.  
  - Outputs: Passes webhook data to next node.  
  - Edge cases: Missing or malformed webhook payloads.  
  - Version: 2  

- **Airtable Record ID**  
  - Type: Set  
  - Role: Extracts and sets the Airtable record ID from webhook payload for subsequent Airtable lookup.  
  - Config: Uses an expression to assign `airtableID` from `body.airtableID` in webhook data.  
  - Inputs: Webhook data.  
  - Outputs: Passes record ID to Airtable node.  
  - Edge cases: Missing or invalid record IDs.  
  - Version: 3.4  

- **Airtable**  
  - Type: Airtable Node  
  - Role: Fetches the content record from Airtable “Media Creation” table using the record ID.  
  - Config: Base and table IDs fixed; record ID dynamic from previous node; uses Airtable Personal Access Token credentials.  
  - Inputs: Record ID from Set node.  
  - Outputs: Content data JSON including media URLs and text fields.  
  - Edge cases: Record not found, API rate limiting, auth errors.  
  - Version: 2.1  

---

#### 1.2 Preparation for Publishing

**Overview:**  
Prepares the data structure for posting by setting platform IDs, extracting final text, and configuring metadata for each platform.

**Nodes Involved:**  
- Prepare for Publish  

**Node Details:**

- **Prepare for Publish**  
  - Type: Set  
  - Role: Constructs JSON with platform IDs (placeholders to be replaced by user), and final text strings for long and short formats extracted from Airtable content.  
  - Config: Raw JSON mode with expressions referencing Airtable fields for text; platform IDs must be manually updated by user.  
  - Inputs: Content data from Airtable.  
  - Outputs: Structured JSON for use in posting nodes.  
  - Edge cases: Missing fields in Airtable leading to empty or invalid text; platform IDs left as placeholders.  
  - Version: 3.4  

---

#### 1.3 Media Handling

**Overview:**  
Downloads image and video files from URLs in Airtable and uploads them to Postiz for media hosting.

**Nodes Involved:**  
- Grab Image from Airtable  
- Upload Image to Postiz  
- Grab Video from Airtable  
- Upload Video to Postiz  

**Node Details:**

- **Grab Image from Airtable**  
  - Type: HTTP Request  
  - Role: Downloads image file from the URL stored in Airtable’s “Image URL” field.  
  - Config: URL dynamically taken from Airtable JSON; response format is file (binary).  
  - Inputs: Prepared data from “Prepare for Publish.”  
  - Outputs: Binary image data to upload node.  
  - Edge cases: Broken URLs, download failures, timeouts.  
  - Version: 4.2  

- **Upload Image to Postiz**  
  - Type: HTTP Request  
  - Role: Uploads the downloaded image file to Postiz media storage endpoint.  
  - Config: POST multipart/form-data with binary “file” field; uses generic HTTP Header Auth credential for Postiz API key.  
  - Inputs: Binary image data from previous node.  
  - Outputs: JSON response with uploaded media ID and path for referencing in posts.  
  - Edge cases: Authentication errors, file size limits, API errors.  
  - Version: 4.2  

- **Grab Video from Airtable**  
  - Type: HTTP Request  
  - Role: Downloads video file from the URL stored in Airtable’s “Video URL” field.  
  - Config: Similar to “Grab Image” but for video URL; expects binary file response.  
  - Inputs: After image upload node completes.  
  - Outputs: Binary video data.  
  - Edge cases: Large file timeouts, broken URLs.  
  - Version: 4.2  

- **Upload Video to Postiz**  
  - Type: HTTP Request  
  - Role: Uploads the downloaded video file to Postiz media storage endpoint.  
  - Config: Same as image upload node.  
  - Inputs: Binary video data.  
  - Outputs: JSON response with media ID and path for posts.  
  - Edge cases: Same as image upload; video format constraints.  
  - Version: 4.2  

---

#### 1.4 Posting to Platforms via Postiz

**Overview:**  
Posts content with uploaded media to multiple platforms via Postiz API, each with customized scheduling and platform-specific settings.

**Nodes Involved:**  
- Dev.to Postiz - POST NOW!  
- Hashnode Postiz - POST NOW!1  
- Wordpress Postiz - POST NOW!  
- Dribble Postiz - POST NOW!1  
- Wait (used for scheduling delays)  
- Wait1 (for additional scheduling delays)  

**Node Details:**

- **Dev.to Postiz - POST NOW!**  
  - Type: HTTP Request  
  - Role: Posts content to Dev.to integration on Postiz, scheduling 24 hours ahead.  
  - Config: JSON body includes type “schedule,” scheduled date/time, content, uploaded image info, group id with timestamp, and title from Airtable. Uses header auth credential.  
  - Inputs: Uploaded video node output.  
  - Outputs: Postiz API response; success or error branches.  
  - Edge cases: API rate limits, invalid media id, auth failures, network timeouts.  
  - Version: 4.2  

- **Hashnode Postiz - POST NOW!1**  
  - Type: HTTP Request  
  - Role: Posts content to Hashnode integration; schedules 23 hours ahead.  
  - Config: Includes publication name and tags (ai, technology) in settings; similar structure to Dev.to node.  
  - Inputs: Wait node output (scheduling).  
  - Outputs: Postiz response.  
  - Edge cases: Publication name mismatch, tag formatting errors, auth issues.  
  - Version: 4.2  

- **Wordpress Postiz - POST NOW!**  
  - Type: HTTP Request  
  - Role: Posts to WordPress integration; schedules 25 hours ahead.  
  - Config: Includes post type (“post”) in settings; content and media references similar to others.  
  - Inputs: Wait node output.  
  - Outputs: Postiz response.  
  - Edge cases: WordPress API schema mismatches, auth errors.  
  - Version: 4.2  

- **Dribble Postiz - POST NOW!1**  
  - Type: HTTP Request  
  - Role: Posts to Dribble integration; schedules 30 hours ahead.  
  - Config: Basic post structure with title and content.  
  - Inputs: Wait1 node output.  
  - Outputs: Postiz response.  
  - Edge cases: Dribble API restrictions, media compatibility.  
  - Version: 4.2  

- **Wait and Wait1**  
  - Type: Wait  
  - Role: Introduces randomized delay before posting to platforms to avoid rate limits or overloading.  
  - Config: Random wait between 3 and 40 seconds approx.  
  - Inputs/Outputs: Connects sequentially between upload and post nodes.  
  - Edge cases: None critical; workflow pauses.  
  - Version: 1.1  

---

#### 1.5 Post-Publish Processing

**Overview:**  
Updates Airtable records to log the success or failure of each platform post, and optionally sends Telegram notifications and updates managing agents.

**Nodes Involved:**  
- Airtable: Posted DevTo  
- Airtable: Post DevTo - Fail  
- Airtable: Posted Hashnode  
- Airtable: Failed Post Hashnode  
- Airtable: Posted Wordpress  
- Airtable: Failed Post Wordpress  
- Airtable: Posted Dribble  
- Airtable: Failed Post Dribble  
- Telegram: User Update (disabled)  
- Update Managing Agent (disabled)  
- Finalize Transaction!  

**Node Details:**

- **Airtable Post nodes (Success and Fail variants)**  
  - Type: Airtable  
  - Role: Update the original content record in Airtable with publishing logs indicating success or failure for each platform.  
  - Config: Uses record ID from Airtable data; appends success or failure messages to “Publishing Log” field.  
  - Inputs: Postiz API response success or error from respective POST nodes.  
  - Outputs: Updates Airtable; no further workflow output.  
  - Edge cases: Airtable API rate limits, network errors, invalid record IDs.  
  - Version: 2.1  

- **Telegram: User Update (disabled)**  
  - Type: Telegram  
  - Role: Optionally sends a Telegram message notifying media completion and scheduling.  
  - Config: Uses Telegram Bot credentials and chat ID from Airtable; currently disabled.  
  - Edge cases: Telegram API quota, invalid chat ID.  
  - Version: 1.2  

- **Update Managing Agent (disabled)**  
  - Type: HTTP Request  
  - Role: Sends a POST message to an external webhook to update a managing agent about completion status.  
  - Config: Disabled; uses JSON message body.  
  - Edge cases: External webhook unavailability, auth failures.  
  - Version: 4.2  

- **Finalize Transaction!**  
  - Type: Airtable  
  - Role: Final update on the content record marking publication as “Published” and setting publishing date/time to current time.  
  - Config: Updates multiple fields including “Production” status and timestamps.  
  - Inputs: From managing agent or Telegram node.  
  - Edge cases: Airtable update failures.  
  - Version: 2.1  

---

#### 1.6 Supporting Documentation and Setup Notes

**Overview:**  
Sticky notes nodes provide detailed setup instructions, affiliate links, and visual aids to guide users through configuration steps for Postiz and Airtable integration.

**Nodes Involved:**  
- Sticky Note1 (Quick Setup Guide)  
- Sticky Note5 (Airtable Example Table & Credentials Setup)  
- Sticky Note6 (Postiz Affiliate & Platforms Summary)  
- Sticky Note7 (Postiz Channel IDs & API Key Instructions)  
- Sticky Note8 (Testing tip for Record ID)  
- Sticky Note9 (Reporting header)  
- Sticky Note (Postiz Platforms image)  
- Sticky Note2 (Handle Media)  
- Sticky Note3 (Schedule and Post!)  

**Node Details:**

- **Sticky Notes**  
  - Type: Sticky Note  
  - Role: Provide non-executable guidance, visual information, and links.  
  - Content: Setup steps, API usage, credential creation, and workflow usage tips.  
  - Edge cases: None (documentation only).  
  - Version: 1  

---

### 3. Summary Table

| Node Name                   | Node Type          | Functional Role                                  | Input Node(s)                      | Output Node(s)                             | Sticky Note                                                                                                               |
|-----------------------------|--------------------|-------------------------------------------------|----------------------------------|-------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Webhook from Content Creation | Webhook            | Entry point, receive content creation trigger   | -                                | Airtable Record ID                         |                                                                                                                           |
| Airtable Record ID           | Set                | Extract Airtable record ID from webhook payload | Webhook from Content Creation    | Airtable                                  |                                                                                                                           |
| Airtable                    | Airtable           | Fetch content record from Airtable base/table   | Airtable Record ID               | Prepare for Publish                        |                                                                                                                           |
| Prepare for Publish          | Set                | Prepare JSON with platform IDs and text content | Airtable                        | Grab Image from Airtable                   |                                                                                                                           |
| Grab Image from Airtable     | HTTP Request       | Download image file from Airtable URL            | Prepare for Publish             | Upload Image to Postiz                     | Sticky Note2: Handle Media                                                                                                |
| Upload Image to Postiz       | HTTP Request       | Upload image to Postiz media storage              | Grab Image from Airtable        | Grab Video from Airtable                   | Sticky Note2: Handle Media                                                                                                |
| Grab Video from Airtable     | HTTP Request       | Download video file from Airtable URL             | Upload Image to Postiz          | Upload Video to Postiz                     | Sticky Note2: Handle Media                                                                                                |
| Upload Video to Postiz       | HTTP Request       | Upload video to Postiz media storage              | Grab Video from Airtable        | Dev.to Postiz - POST NOW!, Wait, Wordpress Postiz - POST NOW!, Wait1, Telegram: User Update | Sticky Note2: Handle Media                                                                                                |
| Dev.to Postiz - POST NOW!    | HTTP Request       | Post content to Dev.to via Postiz                 | Upload Video to Postiz          | Airtable: Posted DevTo, Airtable: Post DevTo - Fail | Sticky Note3: Schedule and Post!                                                                                        |
| Wait                        | Wait               | Random delay before next post                      | Dev.to Postiz - POST NOW!       | Hashnode Postiz - POST NOW!1               | Sticky Note3: Schedule and Post!                                                                                        |
| Hashnode Postiz - POST NOW!1 | HTTP Request       | Post content to Hashnode via Postiz               | Wait                          | Airtable: Posted Hashnode, Airtable: Failed Post Hashnode | Sticky Note3: Schedule and Post!                                                                                        |
| Wait1                       | Wait               | Random delay before next post                      | Upload Video to Postiz          | Dribble Postiz - POST NOW!1                 | Sticky Note3: Schedule and Post!                                                                                        |
| Wordpress Postiz - POST NOW! | HTTP Request       | Post content to WordPress via Postiz               | Upload Video to Postiz          | Airtable: Posted Wordpress, Airtable: Failed Post Wordpress | Sticky Note3: Schedule and Post!                                                                                        |
| Dribble Postiz - POST NOW!1  | HTTP Request       | Post content to Dribble via Postiz                 | Wait1                         | Airtable: Posted Dribble, Airtable: Failed Post Dribble | Sticky Note3: Schedule and Post!                                                                                        |
| Airtable: Posted DevTo       | Airtable           | Update Airtable record with Dev.to success log    | Dev.to Postiz - POST NOW!       | -                                         | Sticky Note9: Reporting                                                                                                |
| Airtable: Post DevTo - Fail  | Airtable           | Update Airtable record with Dev.to failure log    | Dev.to Postiz - POST NOW!       | -                                         | Sticky Note9: Reporting                                                                                                |
| Airtable: Posted Hashnode    | Airtable           | Update Airtable with Hashnode success log         | Hashnode Postiz - POST NOW!1    | -                                         | Sticky Note9: Reporting                                                                                                |
| Airtable: Failed Post Hashnode | Airtable         | Update Airtable with Hashnode failure log         | Hashnode Postiz - POST NOW!1    | -                                         | Sticky Note9: Reporting                                                                                                |
| Airtable: Posted Wordpress   | Airtable           | Update Airtable with WordPress success log        | Wordpress Postiz - POST NOW!    | -                                         | Sticky Note9: Reporting                                                                                                |
| Airtable: Failed Post Wordpress | Airtable         | Update Airtable with WordPress failure log        | Wordpress Postiz - POST NOW!    | -                                         | Sticky Note9: Reporting                                                                                                |
| Airtable: Posted Dribble     | Airtable           | Update Airtable with Dribble success log          | Dribble Postiz - POST NOW!1     | -                                         | Sticky Note9: Reporting                                                                                                |
| Airtable: Failed Post Dribble | Airtable          | Update Airtable with Dribble failure log          | Dribble Postiz - POST NOW!1     | -                                         | Sticky Note9: Reporting                                                                                                |
| Telegram: User Update        | Telegram (Disabled) | Optional notification to Telegram on media completion | Upload Video to Postiz          | Update Managing Agent                       |                                                                                                                           |
| Update Managing Agent        | HTTP Request (Disabled) | Optional notification to external managing system | Telegram: User Update           | Finalize Transaction!                      |                                                                                                                           |
| Finalize Transaction!        | Airtable           | Final update marking record as Published           | Update Managing Agent           | -                                         |                                                                                                                           |
| Airtable Record ID           | Set                | Allows manual setting of record ID for testing    | -                              | Airtable                                  | Sticky Note8: You can place a straight Record ID for easy testing here                                                  |
| Sticky Note1                 | Sticky Note        | Quick Setup Guide for Postiz Automation            | -                              | -                                         | Contains detailed setup instructions and API key guidance for Postiz                                                   |
| Sticky Note5                 | Sticky Note        | Airtable example table and credential setup guide | -                              | -                                         | Includes affiliate signup links and step-by-step credential creation instructions                                      |
| Sticky Note6                 | Sticky Note        | Postiz affiliate link and platform summary         | -                              | -                                         | https://postiz.com/?ref=max; notes on API key requirement and platform support                                         |
| Sticky Note7                 | Sticky Note        | Postiz channel IDs retrieval instructions          | -                              | -                                         | curl example to retrieve integrations and copy channel IDs                                                            |
| Sticky Note8                 | Sticky Note        | Testing tip for direct record ID input              | -                              | -                                         | "You can place a straight Record ID for easy testing here"                                                             |
| Sticky Note9                 | Sticky Note        | Reporting section header                             | -                              | -                                         | "Reporting"                                                                                                             |
| Sticky Note                  | Sticky Note        | Visual Postiz platforms image                        | -                              | -                                         | ![Postiz Platforms Image](https://apexwebservices.com/wp-content/uploads/2025/08/postiz-cloud-platforms.png)            |
| Sticky Note2                 | Sticky Note        | Label for media handling section                     | -                              | -                                         | "Handle Media"                                                                                                          |
| Sticky Note3                 | Sticky Note        | Label for schedule and post section                  | -                              | -                                         | "Schedule and Post!"                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**:  
   - Type: Webhook  
   - Method: POST  
   - Path: Unique identifier (e.g., “6a7f105d-7043-48b4-9a53-284ca2b55dc8”)  
   - Purpose: Receive content creation triggers.

2. **Create a Set Node “Airtable Record ID”**:  
   - Purpose: Extract the Airtable record ID from webhook data.  
   - Configuration: Set variable `airtableID` to `{{$json.body.airtableID}}`.

3. **Create an Airtable Node to Retrieve Content**:  
   - Connect to “Airtable Record ID” node.  
   - Configure with your Airtable Base and Table IDs (from the “Social Media System” base and “Media Creation” table).  
   - Set “Record ID” parameter to use `{{$json.airtableID}}`.  
   - Use Airtable Personal Access Token credentials.

4. **Create a Set Node “Prepare for Publish”**:  
   - Connect from Airtable node.  
   - Configure raw JSON output with platform IDs placeholders (devto_id, wordpress_id, dribble_id, hashnode_id).  
   - Extract final text fields from Airtable JSON for long and short versions using expressions referencing Airtable’s fields.

5. **Create HTTP Request Node “Grab Image from Airtable”**:  
   - Connect from “Prepare for Publish.”  
   - Method: GET  
   - URL: Use expression to fetch `Image URL` from Airtable JSON.  
   - Response Format: File (binary).

6. **Create HTTP Request Node “Upload Image to Postiz”**:  
   - Connect from “Grab Image from Airtable.”  
   - Method: POST to `https://api.postiz.com/public/v1/upload`.  
   - Content-Type: multipart/form-data.  
   - Body Parameter: File binary data with name “file.”  
   - Authentication: Generic HTTP Header with Postiz API key (header name: Authorization, value: your API key).

7. **Create HTTP Request Node “Grab Video from Airtable”**:  
   - Connect from “Upload Image to Postiz.”  
   - Method: GET  
   - URL: Use expression to fetch `Video URL` from Airtable JSON.  
   - Response Format: File (binary).

8. **Create HTTP Request Node “Upload Video to Postiz”**:  
   - Connect from “Grab Video from Airtable.”  
   - Same configuration as image upload node.

9. **Create HTTP Request Nodes for Posting to Platforms**:  
   - **Dev.to Postiz - POST NOW!**: POST to `https://api.postiz.com/public/v1/posts` with JSON body scheduling 24 hours ahead, including content, image ID/path, and title.  
   - **Hashnode Postiz - POST NOW!1**: Similar, schedule 23 hours ahead; includes publication and tags.  
   - **Wordpress Postiz - POST NOW!**: Schedule 25 hours ahead; includes post type “post.”  
   - **Dribble Postiz - POST NOW!1**: Schedule 30 hours ahead.  
   - Use the Postiz API key credential for all.  
   - Connect the “Upload Video to Postiz” output to “Dev.to Postiz - POST NOW!” node.

10. **Create Wait Nodes for Scheduling Delays**:  
    - Insert “Wait” node (random 3-40 seconds) between Dev.to and Hashnode posts.  
    - Insert “Wait1” node similarly between WordPress and Dribble posts.

11. **Connect Posting Nodes in Sequence**:  
    - Dev.to → Wait → Hashnode  
    - Upload Video → Wordpress → Wait1 → Dribble

12. **Create Airtable Update Nodes for Success/Fail Logging**:  
    - For each platform posting node, add two Airtable update nodes: one for posting success, one for failure.  
    - Update the record’s “Publishing Log” field appending success or fail messages.  
    - Use the same Airtable base, table, and record ID from input data.  
    - Connect success output of POST node to success Airtable update, error output to fail Airtable update.

13. **(Optional) Add Telegram Notification Node**:  
    - Disabled by default.  
    - Configure with Telegram Bot credentials and chat ID from Airtable.  
    - Message: “Media completed! 💭 Scheduling or posting now!”

14. **(Optional) Add Managing Agent Update HTTP Request Node**:  
    - Disabled by default.  
    - POST JSON message to external webhook URL to notify managing agent.

15. **Create Final Airtable Update Node “Finalize Transaction!”**:  
    - Update content record status to “Published.”  
    - Set “n8n Publishing Date” and “Time” fields to current timestamp.  
    - Connect from managing agent or Telegram node.

16. **Add Sticky Notes for Documentation**:  
    - Create sticky notes with setup instructions, API key management, platform ID retrieval, and affiliate links.  
    - Place near corresponding nodes for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Postiz Cloud Affiliate Link, Please Support My Work: https://postiz.com/?ref=max                                                 | Sticky Note6                                                                                                       |
| Quick setup guide for Postiz account creation, channel ID retrieval, API key generation, and n8n credential setup instructions.  | Sticky Note1                                                                                                       |
| Airtable example base “Social Media System” with detailed instructions to copy and connect personal access token credentials.    | Sticky Note5                                                                                                       |
| Postiz supports 25 platforms; all times in UTC; API limit 30 requests/hour; media support varies by platform.                    | Sticky Note1                                                                                                       |
| Curl example to retrieve Postiz integration channel IDs: `curl -H "Authorization: YOUR_API_KEY" "https://api.postiz.com/public/v1/integrations"` | Sticky Note7                                                                                                       |
| Visual overview of Postiz supported platforms: ![Postiz Platforms](https://apexwebservices.com/wp-content/uploads/2025/08/postiz-cloud-platforms.png) | Sticky Note (image)                                                                                                |
| Testing tip: You can place a direct Airtable Record ID in “Airtable Record ID” node for easy manual testing.                     | Sticky Note8                                                                                                       |
| Reporting section header to mark the post-publish Airtable updates block.                                                        | Sticky Note9                                                                                                       |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected content. All data handled is legal and publicly accessible.
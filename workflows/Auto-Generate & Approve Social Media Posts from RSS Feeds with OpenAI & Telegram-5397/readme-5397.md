Auto-Generate & Approve Social Media Posts from RSS Feeds with OpenAI & Telegram

https://n8nworkflows.xyz/workflows/auto-generate---approve-social-media-posts-from-rss-feeds-with-openai---telegram-5397


# Auto-Generate & Approve Social Media Posts from RSS Feeds with OpenAI & Telegram

### 1. Workflow Overview

This workflow automates the process of generating, approving, and publishing social media posts based on new articles from RSS feeds. It targets content managers and social media teams who want to streamline sharing fresh blog or press release content across multiple platforms with minimal manual intervention. The workflow leverages OpenAI for summarization and post generation, Telegram for approval, and integrates with social media APIs (Facebook, LinkedIn, Twitter) for automated posting.

Logical blocks:

- **1.1 Input Reception & Content Extraction:** RSS feed trigger and content preprocessing.
- **1.2 AI Content Processing:** Summarization, image prompt generation, and social media post content creation using OpenAI.
- **1.3 Data Storage & Approval Workflow:** Save generated content into NocoDB, send Telegram message for approval, and wait for user response.
- **1.4 Conditional Posting & Updates:** Based on approval, update database status and post content to Facebook, LinkedIn, and Twitter.
- **1.5 Auxiliary Tasks:** Image generation, image URL extraction, and handling user interaction for Twitter posting options.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Content Extraction

- **Overview:** This block listens to the RSS feed for new articles, extracts embedded images, and prepares the article content for further processing.
- **Nodes Involved:**  
  - RSS Trigger  
  - Code  
  - Extract Link Seprately  
- **Node Details:**

  - **RSS Trigger**  
    - Type: RSS Feed Read Trigger  
    - Role: Initiates the workflow on new feed items every 20 minutes.  
    - Configuration: Uses specified RSS feed URL for press releases.  
    - Inputs: None (trigger)  
    - Outputs: Feed item JSON with article data  
    - Edge Cases: Feed downtime, malformed feed items, missing content fields.

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Parses the HTML content to extract a featured image URL embedded within custom `<featured-image>` tags, removes that tag from the content.  
    - Configuration: Custom JS regex extraction and string replacement.  
    - Inputs: Output of RSS Trigger.  
    - Outputs: Modified JSON with `"featured-image"` and cleaned `"content:encoded"`.  
    - Edge Cases: Missing or malformed `<featured-image>` tag, regex failures.

  - **Extract Link Seprately**  
    - Type: HTTP Request  
    - Role: Fetches article URL (from feed `link`) to retrieve full content or validate link accessibility.  
    - Inputs: JSON field `"link"` from previous node.  
    - Outputs: HTTP response (not explicitly used further in the workflow).  
    - Edge Cases: Network errors, 404 or invalid URLs.

---

#### 2.2 AI Content Processing

- **Overview:** This block uses OpenAI models to generate article summaries, image generation prompts, and social media post text for Facebook and LinkedIn.
- **Nodes Involved:**  
  - Article Summary  
  - Create Image Prompt  
  - Generate Image  
  - Create Facebook Post  
  - Create LinkedIn Post  
- **Node Details:**

  - **Article Summary**  
    - Type: OpenAI (Langchain assistant)  
    - Role: Summarizes the article content snippet.  
    - Configuration: Takes article snippet from RSS Trigger, sends to OpenAI assistant with prompt “define.”  
    - Inputs: RSS Trigger content snippet.  
    - Outputs: Summary text.  
    - Edge Cases: API rate limits, incomplete content, model errors.

  - **Create Image Prompt**  
    - Type: OpenAI (Langchain assistant)  
    - Role: Generates a prompt for image creation based on article snippet and summary.  
    - Configuration: Feeds both article snippet and summary output to assistant with prompt “define.”  
    - Inputs: RSS Trigger snippet and Article Summary output.  
    - Outputs: Text prompt for image generation.  
    - Edge Cases: API timeouts, inconsistent prompt generation.

  - **Generate Image**  
    - Type: OpenAI Image generation node  
    - Role: Creates an image URL based on the generated prompt.  
    - Configuration: Uses prompt text from “Create Image Prompt,” requests image URLs in response.  
    - Inputs: Text prompt from “Create Image Prompt.”  
    - Outputs: Image URL(s).  
    - Edge Cases: Image generation failures, slow response.

  - **Create Facebook Post**  
    - Type: OpenAI (Langchain assistant)  
    - Role: Generates Facebook post text using article snippet and summary.  
    - Configuration: Uses “define” prompt with article snippet and summary.  
    - Inputs: RSS Trigger snippet, Article Summary output.  
    - Outputs: Facebook post text.  
    - Edge Cases: API errors, incomplete post content.

  - **Create LinkedIn Post**  
    - Type: OpenAI (Langchain assistant)  
    - Role: Creates LinkedIn post text from article snippet and summary.  
    - Configuration: Similar to Facebook post generation.  
    - Inputs: RSS Trigger snippet, Article Summary output.  
    - Outputs: LinkedIn post text.  
    - Edge Cases: Same as above.

---

#### 2.3 Data Storage & Approval Workflow

- **Overview:** This block stores generated content into a NocoDB database table and sends a Telegram message to a user or group to approve or decline the post.
- **Nodes Involved:**  
  - NocoDB Databse  
  - Send Message (Telegram)  
  - wait for response (Wait)  
- **Node Details:**

  - **NocoDB Databse**  
    - Type: NocoDB node  
    - Role: Creates a new database entry with article URL, summary, platform, post text, image prompt, generated image URL, and featured image URL.  
    - Configuration: Maps fields to entries from previous nodes (summary, post text, image URLs).  
    - Inputs: Output of “Create Facebook Post” (post text), “Generate Image” (image URL), “Code” (featured image), RSS Trigger (URL).  
    - Outputs: Database record with ID and URLs.  
    - Edge Cases: API token errors, network issues, data validation errors.

  - **Send Message (Telegram)**  
    - Type: Telegram node  
    - Role: Sends a Telegram message with post summary and image preview, includes inline buttons “Go” and “No” for approval.  
    - Configuration: Message text dynamically includes Facebook post, summary, original URL, image, and NocoDB edit link; buttons link to webhook URLs with query param for user answer.  
    - Inputs: Output from NocoDB node (database record).  
    - Outputs: Sends message, triggers wait node.  
    - Edge Cases: Telegram API errors, chat ID misconfiguration, user not responding.

  - **wait for response (Wait)**  
    - Type: Wait (Webhook resume)  
    - Role: Pauses workflow awaiting Telegram user response (Go/No).  
    - Configuration: Limits wait time, resumes on webhook call with response.  
    - Inputs: Triggered after Send Message node.  
    - Outputs: Passes user response downstream.  
    - Edge Cases: Timeout, invalid response, webhook errors.

---

#### 2.4 Conditional Posting & Updates

- **Overview:** This block handles user approval decisions via Telegram, updates the database status, and posts approved content to Facebook, LinkedIn, and Twitter accordingly.
- **Nodes Involved:**  
  - If (approval check)  
  - Database Update to Yes  
  - Database Update to No  
  - Facebook Graph API  
  - Create LinkedIn Post  
  - Image url extracter  
  - LinkedIn  
  - message for twitter (Telegram)  
  - Wait1 (wait for Twitter posting choice)  
  - If1 (Twitter post choice check)  
  - X1 (Twitter post with link)  
  - X (Twitter post without link)  
- **Node Details:**

  - **If (approval check)**  
    - Type: If  
    - Role: Checks if Telegram user responded “go” to approve posting.  
    - Inputs: Output from wait for response node.  
    - Outputs: Main branch 0 if “go”, 1 if declined.  
    - Edge Cases: Case sensitivity, missing response.

  - **Database Update to Yes**  
    - Type: NocoDB update node  
    - Role: Sets database “Status” field to “Approved” for the post.  
    - Inputs: Approved branch from If node.  
    - Outputs: Triggers social media posting nodes and Twitter message node.  
    - Edge Cases: Update failures.

  - **Database Update to No**  
    - Type: NocoDB update node  
    - Role: Sets “Status” to “Declined” when user rejects.  
    - Inputs: Declined branch from If node.  
    - Outputs: Ends workflow.  
    - Edge Cases: Update failures.

  - **Facebook Graph API**  
    - Type: Facebook Graph API POST node  
    - Role: Posts photo with message on Facebook using data from NocoDB.  
    - Configuration: Uses “Photos” edge on “me” node, sends message and image URL.  
    - Inputs: From Database Update to Yes node.  
    - Outputs: Confirmation of post.  
    - Edge Cases: Facebook token expiration, permissions errors.

  - **Create LinkedIn Post**  
    - Type: OpenAI assistant node  
    - Role: Generates LinkedIn post text. (Covered in AI Processing but actually executed here after approval.)  
    - Inputs: From Database Update to Yes node.  
    - Outputs: LinkedIn post text to be posted.  
    - Edge Cases: API errors.

  - **Image url extracter**  
    - Type: HTTP Request  
    - Role: Fetches actual image URL from stored “Post Image URL” in NocoDB to prepare for LinkedIn post.  
    - Inputs: Database Update to Yes output (image URL).  
    - Outputs: Image data for LinkedIn post.  
    - Edge Cases: URL invalid or inaccessible.

  - **LinkedIn**  
    - Type: LinkedIn node  
    - Role: Posts on LinkedIn as an organization with text and image.  
    - Configuration: Uses OAuth2 credentials, posts with image and text from previous nodes.  
    - Inputs: Image URL extractor output and Create LinkedIn Post text.  
    - Edge Cases: LinkedIn API permission issues, token expiry.

  - **message for twitter (Telegram)**  
    - Type: Telegram node  
    - Role: Sends message asking user how they want to post on Twitter (with or without link).  
    - Inputs: Triggers after Database Update to Yes node.  
    - Outputs: Wait1 node triggered for user choice.  
    - Edge Cases: Telegram delivery issues.

  - **Wait1**  
    - Type: Wait (Webhook resume)  
    - Role: Waits for user Twitter posting choice via Telegram inline buttons.  
    - Inputs: Message for twitter node.  
    - Outputs: If1 node.  
    - Edge Cases: Timeout, invalid response.

  - **If1**  
    - Type: If  
    - Role: Checks user Twitter posting choice (“go” or “no”).  
    - Inputs: Wait1 output.  
    - Outputs: Main 0 branch posts tweet with link, 1 branch posts without link.  
    - Edge Cases: Same as above.

  - **X1**  
    - Type: Twitter (OAuth2)  
    - Role: Posts Twitter status with article summary and URL link.  
    - Inputs: If1 approved branch.  
    - Outputs: Confirmation.  
    - Edge Cases: Twitter API limits, token expiry.

  - **X**  
    - Type: Twitter (OAuth2)  
    - Role: Posts Twitter status with article summary only (no link).  
    - Inputs: If1 declined branch.  
    - Outputs: Confirmation.  
    - Edge Cases: Same as above.

---

#### 2.5 Auxiliary Tasks & Notes

- **Sticky Notes:**  
  - Describe workflow sections such as the RSS trigger role, image URL extraction, database storage, approval steps, and conditional posting logic.  
  - Provide high-level notes on workflow purpose and usage.  
- **General:**  
  - Workflow runs continuously, polling RSS every 20 minutes.  
  - Uses OAuth2 and API token credentials for integrations.  
  - Approval mechanism uses Telegram inline buttons linked to webhook URLs.  
  - NocoDB stores all post data and status for tracking and editing.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                                     | Input Node(s)                  | Output Node(s)                     | Sticky Note                                                                                              |
|-------------------------|---------------------------------|----------------------------------------------------|--------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------|
| RSS Trigger             | RSS Feed Read Trigger            | Triggers workflow on new articles from RSS feed    | None                           | Code                             | Trigger Node. Input your Feed to fetch the latest content                                              |
| Code                    | Code (JavaScript)                | Extracts featured image from article content       | RSS Trigger                   | Extract Link Seprately            | Image URL extract from the feed to separate column                                                    |
| Extract Link Seprately  | HTTP Request                    | Fetches article link content                        | Code                          | Article Summary                  |                                                                                                        |
| Article Summary         | OpenAI Assistant (Langchain)    | Summarizes article content                          | Extract Link Seprately         | Create Image Prompt              |                                                                                                        |
| Create Image Prompt     | OpenAI Assistant (Langchain)    | Generates image prompt based on article and summary | Article Summary               | Generate Image                  |                                                                                                        |
| Generate Image          | OpenAI Image Generation         | Generates image URL from prompt                     | Create Image Prompt            | Create Facebook Post             |                                                                                                        |
| Create Facebook Post    | OpenAI Assistant (Langchain)    | Generates Facebook post text                        | Generate Image                | NocoDB Databse                  | Store summary in database table and send message to telegram for approval                              |
| NocoDB Databse          | NocoDB                         | Stores post data and metadata                       | Create Facebook Post           | Send Message                   | Store summary in database table and send message to telegram for approval                              |
| Send Message            | Telegram                       | Sends Telegram approval message with inline buttons | NocoDB Databse                | wait for response               | Store summary in database table and send message to telegram for approval                              |
| wait for response       | Wait (Webhook Resume)           | Waits for Telegram user approval response          | Send Message                  | If                             |                                                                                                        |
| If                      | If                             | Checks if user approved post                        | wait for response             | Database Update to Yes / No     | If Approved then update in Database and post on Social Media                                           |
| Database Update to Yes  | NocoDB                         | Marks post as approved in database                  | If (approved branch)          | message for twitter, Facebook Graph API, Create LinkedIn Post | If Approved then update in Database and post on Social Media                                           |
| Database Update to No   | NocoDB                         | Marks post as declined in database                  | If (declined branch)          | None                          | If not Approved then stop the workflow and updated as declined in database                              |
| message for twitter     | Telegram                       | Requests Twitter posting option from user          | Database Update to Yes        | Wait1                         |                                                                                                        |
| Wait1                   | Wait (Webhook Resume)           | Waits for Twitter posting choice                    | message for twitter           | If1                          |                                                                                                        |
| If1                     | If                             | Checks user Twitter posting choice                  | Wait1                        | X1 / X                        |                                                                                                        |
| X1                      | Twitter                       | Posts tweet with link                               | If1 (go branch)              | None                          |                                                                                                        |
| X                       | Twitter                       | Posts tweet without link                            | If1 (no branch)              | None                          |                                                                                                        |
| Facebook Graph API      | Facebook Graph API POST         | Posts image and text on Facebook                    | Database Update to Yes        | None                          |                                                                                                        |
| Create LinkedIn Post    | OpenAI Assistant (Langchain)    | Generates LinkedIn post text                        | Database Update to Yes        | Image url extracter            |                                                                                                        |
| Image url extracter     | HTTP Request                    | Retrieves LinkedIn image URL                        | Create LinkedIn Post          | LinkedIn                      |                                                                                                        |
| LinkedIn                | LinkedIn                       | Posts on LinkedIn as organization                   | Image url extracter           | None                          |                                                                                                        |
| Sticky Note             | Sticky Note                    | Notes about approval and posting logic              | None                         | None                          | If Approved then update in Database and post on Social Media                                           |
| Sticky Note1            | Sticky Note                    | Notes about RSS Trigger                              | None                         | None                          | Trigger Node. Input your Feed to fetch the latest content                                              |
| Sticky Note2            | Sticky Note                    | Notes about image URL extraction                     | None                         | None                          | Image URL extract from the feed to separate column                                                    |
| Sticky Note3            | Sticky Note                    | Notes about database storage and Telegram approval  | None                         | None                          | Store summary in database table and send message to telegram for approval                              |
| Sticky Note4            | Sticky Note                    | Notes about declined posts                           | None                         | None                          | If not Approved then stop the workflow and updated as declined in database                              |
| Sticky Note5            | Sticky Note                    | Overall workflow purpose                             | None                         | None                          | This workflow automatically publishes newly posted blog articles from the website to social media platforms. It helps maintain consistent online presence by sharing content without manual effort. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Trigger node:**  
   - Type: RSS Feed Read Trigger  
   - Set feed URL to your RSS source (e.g., https://www.abc.com/press-releases/feed/)  
   - Set polling interval to every 20 minutes.

2. **Add Code node:**  
   - Type: Code (JavaScript)  
   - Paste code to extract `<featured-image>` tag content and remove it from article content.  
   - Input: Connect from RSS Trigger.  
   - Output: Modified feed item with `"featured-image"` field.

3. **Add HTTP Request node (Extract Link Seprately):**  
   - Type: HTTP Request  
   - URL: Use expression `{{$json.link}}` to fetch article link.  
   - Input: Connect from Code node.

4. **Add OpenAI Assistant node (Article Summary):**  
   - Type: OpenAI Langchain assistant  
   - Prompt: “define”  
   - Text input: Use expression `"Full article: {{ $('RSS Trigger').item.json['content:encodedSnippet'] }}"`  
   - Credentials: Set your OpenAI API credentials.  
   - Input: Connect from Extract Link Seprately node.

5. **Add OpenAI Assistant node (Create Image Prompt):**  
   - Type: OpenAI Langchain assistant  
   - Prompt: “define”  
   - Text input: Combine article snippet and article summary from previous node.  
   - Input: Connect from Article Summary node.

6. **Add OpenAI Image Generation node (Generate Image):**  
   - Type: OpenAI Image generation  
   - Prompt: Use output from Create Image Prompt.  
   - Options: Request image URLs returned.  
   - Input: Connect from Create Image Prompt node.

7. **Add OpenAI Assistant node (Create Facebook Post):**  
   - Type: OpenAI Langchain assistant  
   - Prompt: “define”  
   - Text input: Use article snippet and article summary.  
   - Input: Connect from Generate Image node.

8. **Add NocoDB node (NocoDB Databse):**  
   - Type: NocoDB (create)  
   - Map fields for URL, Article Summary, Platform (set “Facebook”), Post (Facebook post text), Image Prompt (from Create Image Prompt), Image URL (from Generate Image), Post Image URL (featured-image from Code node).  
   - Input: Connect from Create Facebook Post node.  
   - Credentials: Configure with your NocoDB API token.

9. **Add Telegram node (Send Message):**  
   - Type: Telegram  
   - Text: Compose message with Facebook post, summary, original URL, image, and NocoDB edit link, including inline keyboard with “Go” and “No” buttons linking to webhook URLs for responses.  
   - Chat ID: Your Telegram chat/group ID.  
   - Input: Connect from NocoDB node.  
   - Credentials: Telegram API credentials.

10. **Add Wait node (wait for response):**  
    - Type: Wait (Webhook Resume)  
    - Configure webhook ID to resume on user response from Telegram inline buttons.  
    - Input: Connect from Telegram Send Message node.

11. **Add If node (approval check):**  
    - Condition: Check if query parameter `answer` equals “go”.  
    - Input: Connect from Wait node.

12. **Add NocoDB update nodes:**  
    - Database Update to Yes: Update Status to “Approved” for the post ID. Connect from If node’s “true” branch.  
    - Database Update to No: Update Status to “Declined.” Connect from If node’s “false” branch.

13. **Add Facebook Graph API node:**  
    - Type: Facebook Graph API POST  
    - Post photo with message and URL from database fields.  
    - Input: Connect from Database Update to Yes node.

14. **Add OpenAI Assistant node (Create LinkedIn Post):**  
    - Similar to Facebook post creation; generates LinkedIn post text.  
    - Input: Connect from Database Update to Yes node.

15. **Add HTTP Request node (Image url extracter):**  
    - Fetch LinkedIn post image URL from database.  
    - Input: Connect from Create LinkedIn Post node.

16. **Add LinkedIn node:**  
    - Posts on LinkedIn as organization with text and image.  
    - Input: Connect from Image url extracter node.  
    - Credentials: LinkedIn OAuth2.

17. **Add Telegram node (message for twitter):**  
    - Sends message asking how to post on Twitter (with or without link) with inline buttons.  
    - Input: Connect from Database Update to Yes node.

18. **Add Wait node (Wait1):**  
    - Waits for Twitter posting choice webhook response.  
    - Input: Connect from message for twitter node.

19. **Add If node (If1):**  
    - Checks Twitter posting choice (“go” or “no”).  
    - Input: Connect from Wait1 node.

20. **Add Twitter nodes:**  
    - X1: Posts tweet with article summary and URL (if user chose “go”).  
    - X: Posts tweet with article summary only (if user chose “no”).  
    - Inputs: Connect from If1’s respective branches.  
    - Credentials: Twitter OAuth2.

21. **Add Sticky Notes throughout the workflow:**  
    - Add explanatory notes near RSS Trigger, Code node, database nodes, and approval logic as per original workflow.

22. **Configure all credentials:**  
    - OpenAI API key for assistant and image nodes.  
    - Telegram Bot API token and chat ID.  
    - NocoDB API token with project and workspace IDs.  
    - Facebook Graph API token with required permissions.  
    - LinkedIn OAuth2 credentials.  
    - Twitter OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow enables automated social media publishing for new blog articles, reducing manual content sharing effort and ensuring consistent online presence.                                                                | General workflow purpose                                                                                |
| Telegram integration uses inline keyboard buttons linked to webhook URLs to capture approval or decline responses from users, enabling interactive content moderation.                                                        | Telegram approval mechanism                                                                            |
| OpenAI Langchain assistants are used with the “define” prompt to generate summaries and post texts, ensuring consistent language style and concise content.                                                                   | OpenAI usage details                                                                                   |
| NocoDB is used as a lightweight, no-code database solution to store, update, and track post metadata and statuses for audit and editing purposes.                                                                             | Database integration                                                                                   |
| Facebook, LinkedIn, and Twitter posting nodes require proper OAuth2 credentials with permissions configured for posting on behalf of the user or organization.                                                                 | Social media API integration                                                                            |
| Sticky notes provide contextual explanations and workflow guidance for maintainers and collaborators.                                                                                                                        | Workflow documentation aids                                                                             |
| Approval flow includes two steps: first for general post approval, second specifically for Twitter posting style selection (with or without link).                                                                            | Multi-step user interaction                                                                              |
| RSS feed polling interval is set to 20 minutes but can be adjusted per requirements.                                                                                                                                          | Polling configuration                                                                                   |
| Twitter renamed as “X” in UI nodes but functions identically for tweet posting.                                                                                                                                               | Platform naming note                                                                                     |

---

**Disclaimer:** The provided documentation is based exclusively on the analyzed n8n workflow JSON and respects all applicable content policies. It contains no illegal or offensive elements. All data processed is publicly available and lawful.
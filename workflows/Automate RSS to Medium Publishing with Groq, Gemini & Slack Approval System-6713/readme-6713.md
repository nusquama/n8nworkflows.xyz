Automate RSS to Medium Publishing with Groq, Gemini & Slack Approval System

https://n8nworkflows.xyz/workflows/automate-rss-to-medium-publishing-with-groq--gemini---slack-approval-system-6713


# Automate RSS to Medium Publishing with Groq, Gemini & Slack Approval System

### 1. Workflow Overview

This workflow automates the process of fetching technology news articles from multiple RSS feeds, transforming them into well-formatted Medium posts enriched with AI-generated content and images, and managing a Slack-based approval system before publishing the content on Medium and notifying stakeholders via email.

The workflow is logically divided into the following blocks:

- **1.1 Content Ingestion and Deduplication:** Scheduled triggers pull fresh articles from three popular tech RSS feeds. The articles are merged and cross-checked against a Google Sheet containing already posted articles to avoid duplicates.

- **1.2 AI Content and Image Generation:** Selected articles are processed by multiple AI models (Google Gemini, Groq Chat, Basic LLM Chain) to generate enriched blog content and an image prompt. The prompt is used to generate a cover image via Pollinations AI, which is then downloaded and uploaded to Google Drive.

- **1.3 Article Preparation and Storage:** The AI-generated article is compiled, cleaned, and formatted. A Google Doc is created and updated with the article content, linking the image stored on Google Drive. This serves as a centralized repository and review point.

- **1.4 Slack-based Review and Approval System:** The article is sent to Slack for manual review. The system waits for an approval or rejection response from designated users.

- **1.5 Conditional Publishing and Notification:** If approved, the article is published on Medium via an API call and emailed to stakeholders. If rejected, a Slack notification informs the team.

---

### 2. Block-by-Block Analysis

#### 2.1 Content Ingestion and Deduplication

- **Overview:**  
  This block fetches the newest articles from three RSS feeds, merges them, and compares them with records in a Google Sheet to filter out articles already published.

- **Nodes Involved:**  
  Schedule Trigger, The Verge – Tech (RSS), Ars Technica – Technology Lab (RSS), TechCrunch (RSS), Get row(s) in sheet (Google Sheets), Merge, Check if Already Posted (Code), Pick Random Post (Code)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow every 6 hours at 10 minutes past the hour.  
    - Config: Interval-based trigger with 6-hour frequency.  
    - Inputs: None (trigger)  
    - Outputs: Triggers downstream RSS feed nodes.  
    - Edge cases: Missed triggers if n8n is down; ensure timezone consistency.

  - **The Verge – Tech, Ars Technica – Technology Lab, TechCrunch**  
    - Type: RSS Feed Read  
    - Role: Fetches latest articles from respective tech news RSS URLs.  
    - Config: RSS URLs set to respective feeds; default options.  
    - Inputs: Trigger from Schedule Trigger (direct or via sequence).  
    - Outputs: List of RSS items with title, link, summary, etc.  
    - Edge cases: Network errors, empty feeds, malformed RSS data.

  - **Get row(s) in sheet**  
    - Type: Google Sheets  
    - Role: Retrieves existing article records from a Google Sheet to check already posted content.  
    - Config: Spreadsheet ID and Sheet Name ("Sheet1") provided.  
    - Inputs: Triggered by Schedule Trigger.  
    - Outputs: List of rows with previous article URLs and metadata.  
    - Credentials: Google Sheets OAuth2.  
    - Edge cases: API quota limits, permission errors, empty sheet.

  - **Merge**  
    - Type: Merge  
    - Role: Combines RSS feed items and Google Sheet rows into a single stream for deduplication.  
    - Config: Combine mode "keepEverything" joined by matching "link" fields.  
    - Inputs: RSS feeds and Google Sheets data.  
    - Outputs: Combined dataset.  
    - Edge cases: Mismatch in fields, empty inputs.

  - **Check if Already Posted (Code)**  
    - Type: Code  
    - Role: Filters out RSS articles that already exist in the Google Sheet.  
    - Logic: Compares RSS article link with URLs in the sheet to keep only fresh articles.  
    - Inputs: Merged dataset.  
    - Outputs: Filtered list of new articles.  
    - Edge cases: Empty input array throws error; duplicate links in feeds.

  - **Pick Random Post (Code)**  
    - Type: Code  
    - Role: Picks one random fresh article for downstream processing.  
    - Logic: Randomly selects one item; throws error if none available.  
    - Inputs: Filtered fresh articles.  
    - Outputs: Single article object.  
    - Edge cases: No fresh posts available (workflow halts).

---

#### 2.2 AI Content and Image Generation

- **Overview:**  
  This block processes the selected article with multiple AI language models to generate a well-structured blog post and an image prompt. The image prompt is used with Pollinations AI to create a cover image, which is then downloaded locally.

- **Nodes Involved:**  
  Basic LLM Chain, Groq Chat Model, Google Gemini Chat Model, Generate Image Prompt (Gemini), Create Pollinations + URL (Code), DOWNLOAD IMAGE (HTTP Request), Upload file (Google Drive)

- **Node Details:**

  - **Basic LLM Chain**  
    - Type: Langchain LLM Chain  
    - Role: Uses a prompt to transform RSS content into a professional tech blog article.  
    - Config: System message defines tone and style; user message includes title and summary to generate content.  
    - Inputs: Randomly picked article.  
    - Outputs: Generated article text.  
    - Edge cases: API timeout, malformed prompts.

  - **Groq Chat Model**  
    - Type: Langchain Chat Model (Groq)  
    - Role: Expands or refines content generated by Basic LLM Chain (used as AI language model in chain).  
    - Inputs: Basic LLM Chain output.  
    - Outputs: Refined article content.  
    - Credentials: Groq API key required.  
    - Edge cases: API failures, rate limits.

  - **Google Gemini Chat Model**  
    - Type: Langchain Chat Model (Google Gemini)  
    - Role: Generates a descriptive image prompt based on article text.  
    - Config: Uses Gemini 2.5 Flash Lite model.  
    - Inputs: Text from Basic LLM Chain.  
    - Outputs: Image prompt text.  
    - Credentials: Google Palm API key.  
    - Edge cases: API errors, invalid model names.

  - **Generate Image Prompt (Gemini)**  
    - Type: Langchain LLM Chain  
    - Role: Converts article text into detailed AI image generation prompt.  
    - Config: Strict prompt engineering instructions for visual style and content.  
    - Inputs: Text from Google Gemini Chat Model.  
    - Outputs: Final image prompt string.  
    - Edge cases: Prompt parsing errors.

  - **Create Pollinations + URL (Code)**  
    - Type: Code  
    - Role: Generates Pollinations AI image generation URL with prompt and parameters.  
    - Logic: Encodes prompt, adds width, height, random seed, and model parameters.  
    - Inputs: Image prompt.  
    - Outputs: URL string for image generation.  
    - Edge cases: URL encoding issues.

  - **DOWNLOAD IMAGE (HTTP Request)**  
    - Type: HTTP Request  
    - Role: Downloads the generated image file from Pollinations URL.  
    - Config: Response format set to file.  
    - Inputs: Image URL from previous node.  
    - Outputs: Binary image file data.  
    - Edge cases: Network errors, invalid URL.

  - **Upload file (Google Drive)**  
    - Type: Google Drive  
    - Role: Uploads the downloaded image file to a specified Google Drive folder.  
    - Config: Folder ID specified; file name taken from input or defaults to "image.jpg".  
    - Credentials: Google Drive OAuth2.  
    - Inputs: Binary image file.  
    - Outputs: Google Drive file metadata including webViewLink.  
    - Edge cases: Permission errors, quota exceeded.

---

#### 2.3 Article Preparation and Storage

- **Overview:**  
  This block formats the AI-generated article content, creates a Google Doc, updates it with the article and image references, and prepares the content for review.

- **Nodes Involved:**  
  Split Out, Merge2, Merge1, Create Google Doc, Wait1, Merge4, Edit Fields, Code1, Code2, REFORMAT ARTICLE, Update Doc, Split Out1, Split Out2, No Operation, do nothing, Edit Fields1

- **Node Details:**

  - **Split Out (and Split Out1, Split Out2)**  
    - Type: Split Out  
    - Role: Separates data fields to handle article text, title, and formatted article individually for parallel processing.  
    - Inputs: AI generated content and Google Doc metadata.  
    - Outputs: Split data streams for merging and updating.  
    - Edge cases: Missing fields may cause errors downstream.

  - **Merge1, Merge2, Merge3, Merge4**  
    - Type: Merge  
    - Role: Combines multiple inputs (e.g., Google Doc creation response and article content) ensuring proper sequencing and data integrity.  
    - Config varies: combine mode, combineByPosition, combineAll depending on scenario.  
    - Inputs: Various streams including docs, article text, image metadata.  
    - Outputs: Combined datasets for next nodes.  
    - Edge cases: Misaligned inputs causing missing data.

  - **Create Google Doc**  
    - Type: Google Docs  
    - Role: Creates a new Google Doc with the article title in a specified Drive folder.  
    - Config: Title set to article title; folder ID specified.  
    - Credentials: Google Docs OAuth2.  
    - Outputs: Document metadata including document ID and URL.  
    - Edge cases: Permission issues, API failures.

  - **Wait1**  
    - Type: Wait  
    - Role: Introduces a pause, waiting for Slack approval webhook response (manual review).  
    - Config: Uses a Slack webhook ID for asynchronous wait.  
    - Inputs: None (waits for Slack event).  
    - Outputs: Resumes workflow upon Slack input.  
    - Edge cases: Timeout or no response.

  - **Edit Fields and Edit Fields1**  
    - Type: Set  
    - Role: Assigns or modifies JSON fields such as article text and Drive URLs for clean data flow.  
    - Config: Assigns formattedArticle, text, drive_url, etc.  
    - Inputs/Outputs: Data manipulation for formatting and downstream use.  
    - Edge cases: Missing fields causing errors.

  - **Code1**  
    - Type: Code  
    - Role: Extracts article title from bolded text, compiles final formatted article with metadata including date, creator handle, and image URL.  
    - Inputs: Article text and Google Drive link.  
    - Outputs: JSON containing title, formattedArticle, driveImageURL.  
    - Edge cases: Missing bolded title fallback to "Untitled Article".

  - **Code2**  
    - Type: Code  
    - Role: Consolidates document ID, formatted article, and image link into a single JSON object for update.  
    - Inputs: Merged data from Google Doc creation and article formatting.  
    - Outputs: Single JSON object with key data.  
    - Edge cases: Missing fields.

  - **REFORMAT ARTICLE (Code)**  
    - Type: Code  
    - Role: Cleans article content by removing repeated titles, structural tags, and composes final content with publish date and author info.  
    - Inputs: Article and document metadata.  
    - Outputs: Cleaned and formatted article JSON.  
    - Edge cases: Pattern matching failures.

  - **Update Doc (Google Docs)**  
    - Type: Google Docs  
    - Role: Updates the created Google Doc by inserting the formatted article content.  
    - Config: Inserts text at specified location using document URL.  
    - Credentials: Google Docs OAuth2.  
    - Inputs: Formatted article text, document ID.  
    - Outputs: Updated document metadata.  
    - Edge cases: Document locked, API errors.

  - **No Operation, do nothing**  
    - Type: No Operation  
    - Role: Placeholder node to maintain workflow structure.  
    - Inputs: From Split Out2.  
    - Outputs: Pass-through.  
    - Edge cases: None.

---

#### 2.4 Slack-based Review and Approval System

- **Overview:**  
  This block sends the article for review via Slack, waits for approval or rejection, and routes the workflow accordingly.

- **Nodes Involved:**  
  Send a message2 (Slack), Send message and wait for response (Slack), Check Approval Status (If), Rejection Notification (Slack), Send a message1 (Gmail)

- **Node Details:**

  - **Send a message2 (Slack)**  
    - Type: Slack  
    - Role: Sends the article preview and Google Doc link to a Slack user or channel for review.  
    - Config: Message includes article text and Drive link.  
    - Credentials: Slack API token.  
    - Inputs: Article content and metadata.  
    - Outputs: Confirmation of message sent.  
    - Edge cases: Slack API errors, user/channel not found.

  - **Send message and wait for response (Slack)**  
    - Type: Slack  
    - Role: Sends an approval request message and waits for user interaction (approve/reject).  
    - Config: Approval options set with double confirmation and button style.  
    - Inputs: From Edit Fields1.  
    - Outputs: Waits for Slack response webhook.  
    - Edge cases: Timeout, no response, malformed approval.

  - **Check Approval Status (If)**  
    - Type: If  
    - Role: Checks if the Slack approval response indicates approval (boolean true).  
    - Inputs: Slack approval response JSON data.  
    - Outputs: Branches workflow to approved or rejected paths.  
    - Edge cases: Missing or malformed response data.

  - **Rejection Notification (Slack)**  
    - Type: Slack  
    - Role: Sends a notification message on Slack informing about article rejection.  
    - Config: Static rejection message with emoji.  
    - Inputs: Triggered on rejection branch.  
    - Outputs: Confirmation message sent.  
    - Edge cases: Slack API failures.

  - **Send a message1 (Gmail)**  
    - Type: Gmail  
    - Role: Sends an email with the approved article content to notify stakeholders or mailing lists.  
    - Config: Subject set to "Medium Content"; body includes formatted article text.  
    - Credentials: Gmail OAuth2.  
    - Inputs: Approved article content.  
    - Outputs: Email sent confirmation.  
    - Edge cases: Authentication errors, quota issues.

---

#### 2.5 Conditional Publishing to Medium

- **Overview:**  
  Upon approval, this block publishes the article to Medium via an HTTP API call.

- **Nodes Involved:**  
  Publish to Medium (HTTP Request) [disabled]

- **Node Details:**

  - **Publish to Medium (HTTP Request)**  
    - Type: HTTP Request  
    - Role: Sends a JSON-RPC request to a VPS-hosted API to create a Medium post with title, content, tags, and publish status.  
    - Config: POST method with JSON payload including title from The Verge item, content concatenated with AI-generated text and image URL, and tags ["AI", "Automation", "Tech"].  
    - Inputs: Approval branch from Check Approval Status node.  
    - Outputs: Medium API response.  
    - Disabled: Currently disabled, requires VPS API to be active.  
    - Edge cases: API endpoint down, authentication, malformed payload.

---

### 3. Summary Table

| Node Name                    | Node Type              | Functional Role                               | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                                                             |
|------------------------------|-----------------------|----------------------------------------------|----------------------------------|----------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger             | Schedule Trigger       | Initiates workflow every 6 hours              | None                             | The Verge – Tech, Get row(s) in sheet | Contains overview of RSS content fetching and scheduling.                                                                              |
| The Verge – Tech             | RSS Feed Read          | Fetches RSS feed from The Verge               | Schedule Trigger                 | Ars Technica – Technology Lab     | See above                                                                                                                               |
| Ars Technica – Technology Lab| RSS Feed Read          | Fetches Ars Technica technology feed          | The Verge – Tech                 | TechCrunch                       | See above                                                                                                                               |
| TechCrunch                  | RSS Feed Read          | Fetches TechCrunch RSS feed                    | Ars Technica – Technology Lab    | Merge                            | See above                                                                                                                               |
| Get row(s) in sheet          | Google Sheets          | Retrieves posted article data for deduplication | Schedule Trigger                 | Merge                            | See above                                                                                                                               |
| Merge                       | Merge                  | Combines RSS feeds & sheet rows                | TechCrunch, Get row(s) in sheet  | Check if Already Posted          | See above                                                                                                                               |
| Check if Already Posted      | Code                   | Filters out already posted articles            | Merge                           | Pick Random Post                 | See above                                                                                                                               |
| Pick Random Post             | Code                   | Selects one fresh article randomly             | Check if Already Posted          | Basic LLM Chain                  | See above                                                                                                                               |
| Basic LLM Chain              | Langchain Chain LLM    | Generates well-formatted blog post from RSS   | Pick Random Post                | Split Out                       | AI content generation block overview                                                                                                   |
| Groq Chat Model              | Langchain Chat Model   | Expands/refines generated content              | Basic LLM Chain (ai_languageModel) | Basic LLM Chain                  | AI content generation block overview                                                                                                   |
| Google Gemini Chat Model     | Langchain Chat Model   | Generates descriptive image prompt             | Basic LLM Chain                 | Generate Image Prompt (Gemini)   | AI content generation block overview                                                                                                   |
| Generate Image Prompt (Gemini)| Langchain Chain LLM   | Creates AI image generation prompt              | Google Gemini Chat Model        | Create Pollinations + URL        | AI content generation block overview                                                                                                   |
| Create Pollinations + URL    | Code                   | Constructs Pollinations AI image URL            | Generate Image Prompt (Gemini)  | DOWNLOAD IMAGE                  | AI content generation block overview                                                                                                   |
| DOWNLOAD IMAGE              | HTTP Request           | Downloads image file from Pollinations URL      | Create Pollinations + URL        | Upload file                    | AI content generation block overview                                                                                                   |
| Upload file                 | Google Drive           | Uploads image to Google Drive folder             | DOWNLOAD IMAGE                  | Merge2                         | Upload to Google Drive block overview                                                                                                  |
| Split Out                   | Split Out              | Splits article text field for parallel processing| Basic LLM Chain                 | Generate Image Prompt (Gemini), Merge2 | Upload to Google Drive block overview                                                                                                  |
| Merge2                      | Merge                  | Combines content and image metadata              | Upload file, Split Out          | Send a message2, Merge4          | Upload to Google Drive block overview                                                                                                  |
| Send a message2             | Slack                  | Sends article preview for Slack review           | Merge2                         | Merge4                         | Slack notification for approval overview                                                                                               |
| Merge4                      | Merge                  | Combines Slack message and article data          | Send a message2, Merge2         | Edit Fields                    | Slack notification for approval overview                                                                                               |
| Edit Fields                 | Set                    | Assigns fields for article content                | Merge4                         | Code1                         | Upload to Google Drive block overview                                                                                                  |
| Code1                       | Code                   | Extracts title, formats article text              | Edit Fields                   | Split Out1                    | Upload to Google Drive block overview                                                                                                  |
| Split Out1                  | Split Out              | Splits title from article data                     | Code1                         | Create Google Doc, Wait1       | Upload to Google Drive block overview                                                                                                  |
| Create Google Doc            | Google Docs            | Creates new Google Doc for article                 | Split Out1                    | Merge1                        | Upload to Google Drive block overview                                                                                                  |
| Wait1                       | Wait                   | Waits for Slack approval response                   | Split Out1                    | Merge1                        | Upload to Google Drive block overview                                                                                                  |
| Merge1                      | Merge                  | Merges Google Doc creation and wait output          | Create Google Doc, Wait1        | Code2                         | Upload to Google Drive block overview                                                                                                  |
| Code2                       | Code                   | Consolidates Doc ID and formatted article           | Merge1                        | REFORMAT ARTICLE              | Upload to Google Drive block overview                                                                                                  |
| REFORMAT ARTICLE            | Code                   | Cleans and prepares final article content           | Code2                         | Split Out2                   | Upload to Google Drive block overview                                                                                                  |
| Split Out2                  | Split Out              | Splits formatted article for update and no-op       | REFORMAT ARTICLE              | Update Doc, No Operation      | Upload to Google Drive block overview                                                                                                  |
| Update Doc                  | Google Docs            | Updates the Google Doc with formatted article       | Split Out2                   | Merge3                        | Upload to Google Drive block overview                                                                                                  |
| No Operation, do nothing    | No Operation           | Placeholder node                                     | Split Out2                   | Merge3                        | Upload to Google Drive block overview                                                                                                  |
| Merge3                      | Merge                  | Combines Update Doc and no-op outputs                 | Update Doc, No Operation       | Edit Fields1                  | Upload to Google Drive block overview                                                                                                  |
| Edit Fields1                | Set                    | Sets final fields before Slack approval               | Merge3                        | Send message and wait for response | Slack notification for approval overview                                                                                               |
| Send message and wait for response | Slack           | Sends approval request and waits for user input       | Edit Fields1                  | Check Approval Status          | Slack notification for approval overview                                                                                               |
| Check Approval Status       | If                     | Checks if article was approved or rejected            | Send message and wait for response | Send a message1 (approval), Rejection Notification (rejection) | Slack notification for approval overview                                                                                               |
| Send a message1             | Gmail                  | Sends approved article content via email              | Check Approval Status (true)    | None                          | Slack notification for approval overview                                                                                               |
| Rejection Notification      | Slack                  | Notifies rejection of article                          | Check Approval Status (false)   | None                          | Slack notification for approval overview                                                                                               |
| Publish to Medium           | HTTP Request (disabled)| Publishes approved article to Medium                   | Check Approval Status (true)    | None                          | Slack notification for approval overview                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure to run every 6 hours at 10 minutes past the hour.  

2. **Add three RSS Feed Read nodes:**  
   - Names: "The Verge – Tech", "Ars Technica – Technology Lab", "TechCrunch"  
   - Set URLs to respective RSS feeds:  
     - The Verge: https://www.theverge.com/rss/index.xml  
     - Ars Technica: https://feeds.arstechnica.com/arstechnica/technology-lab  
     - TechCrunch: https://techcrunch.com/feed  
   - Connect Schedule Trigger to "The Verge – Tech" and "Get row(s) in sheet" node (next).  
   - Chain nodes: The Verge → Ars Technica → TechCrunch.  

3. **Set up Google Sheets "Get row(s) in sheet" node:**  
   - Spreadsheet ID: Use your Google Sheet ID (example from workflow: 1s8BaF5C88tDXgH0YxVEzsFsEJEmsrd6cC_a9ERcroAk)  
   - Sheet Name: "Sheet1" or your sheet name containing previously posted articles.  
   - Connect Schedule Trigger directly to this node as well.  
   - Configure Google Sheets OAuth2 credentials.  

4. **Add a Merge node:**  
   - Mode: Combine  
   - Join Mode: Keep Everything  
   - Fields to match: link  
   - Connect outputs from TechCrunch and Get row(s) in sheet nodes into this Merge node.  

5. **Add "Check if Already Posted" Code node:**  
   - Paste the provided JavaScript code that separates RSS items and posted URLs and filters out duplicates.  
   - Connect Merge node output to this node.  

6. **Add "Pick Random Post" Code node:**  
   - Paste JavaScript code that picks one random article or throws error if none available.  
   - Connect "Check if Already Posted" node to this node.  

7. **Add Basic LLM Chain node:**  
   - Type: Langchain Chain LLM  
   - Configure prompt with system message for professional tech blogger style.  
   - User message template includes article title, content snippet, and link.  
   - Connect "Pick Random Post" node to this node.  
   - Configure Groq Chat Model as the AI language model used by this chain.  

8. **Add Groq Chat Model node:**  
   - Type: Langchain Chat Model  
   - Configure with Groq API credentials.  
   - Connect "Basic LLM Chain" node's ai_languageModel output to this node.  

9. **Add Google Gemini Chat Model node:**  
   - Type: Langchain Chat Model  
   - Configure with Google Palm API credentials.  
   - Model name: models/gemini-2.5-flash-lite-preview-06-17  
   - Connect "Basic LLM Chain" node output to this node.  

10. **Add "Generate Image Prompt (Gemini)" Langchain Chain node:**  
    - Use prompt instructing creation of concise, descriptive image prompts optimized for AI image generation tools.  
    - Connect Google Gemini Chat Model node ai_languageModel output to this node.  

11. **Add "Create Pollinations + URL" Code node:**  
    - Paste the JavaScript code to generate Pollinations AI image URL using the prompt.  
    - Connect "Generate Image Prompt (Gemini)" node to this node.  

12. **Add HTTP Request node "DOWNLOAD IMAGE":**  
    - Set URL to `={{ $json.imageUrl }}`  
    - Options: Response format = file  
    - Connect "Create Pollinations + URL" to this node.  

13. **Add Google Drive "Upload file" node:**  
    - Configure with folder ID matching your Drive folder for storing images.  
    - File name: `={{$json["fileName"] || "image.jpg"}}`  
    - Connect "DOWNLOAD IMAGE" node to this node.  
    - Configure Google Drive OAuth2 credentials.  

14. **Add Split Out node:**  
    - Field to split out: "text"  
    - Connect "Basic LLM Chain" output to this node.  

15. **Add Merge2 node:**  
    - Mode: Combine  
    - Combine by position  
    - Connect "Upload file" and "Split Out" node outputs to this node.  

16. **Add Slack "Send a message2" node:**  
    - Sends article preview and Google Drive webViewLink to Slack user or channel.  
    - Connect "Merge2" node output to this node.  
    - Configure Slack API credentials.  

17. **Add Merge4 node:**  
    - Mode: Combine  
    - Combine all inputs  
    - Connect "Send a message2" and "Merge2" outputs to this node.  

18. **Add Set node "Edit Fields":**  
    - Assign "text" and "drive_url" fields from JSON.  
    - Connect "Merge4" node to this node.  

19. **Add Code node "Code1":**  
    - Code extracts bolded title, formats article with metadata and image URL.  
    - Connect "Edit Fields" node to this node.  

20. **Add Split Out1 node:**  
    - Field to split out: "title"  
    - Connect "Code1" node to this node.  

21. **Add Google Docs "Create Google Doc" node:**  
    - Title: `={{ $json.title }}`  
    - Folder ID: Your Google Drive folder for Medium pipeline.  
    - Connect "Split Out1" node to this node.  
    - Configure Google Docs OAuth2 credentials.  

22. **Add Wait node "Wait1":**  
    - Set webhook ID to listen for Slack approval response.  
    - Connect "Split Out1" node to this node.  

23. **Add Merge1 node:**  
    - Combine outputs from "Create Google Doc" and "Wait1."  
    - Connect both nodes to "Merge1".  

24. **Add Code node "Code2":**  
    - Consolidates document ID and formatted article for update.  
    - Connect "Merge1" to this node.  

25. **Add Code node "REFORMAT ARTICLE":**  
    - Cleans article content, removes tags, adds publish date and author info.  
    - Connect "Code2" node to this node.  

26. **Add Split Out2 node:**  
    - Field to split out: "formattedArticle"  
    - Connect "REFORMAT ARTICLE" node to this node.  

27. **Add Google Docs "Update Doc" node:**  
    - Operation: Update  
    - Document URL: `={{ $json.id }}`  
    - Insert action: Insert text with formattedArticle content.  
    - Connect Split Out2 node to this node.  

28. **Add No Operation node:**  
    - Connect Split Out2 node to this node as well (parallel path).  

29. **Add Merge3 node:**  
    - Combine outputs from "Update Doc" and "No Operation."  
    - Connect both nodes to "Merge3".  

30. **Add Set node "Edit Fields1":**  
    - Assign "formattedArticle" field for Slack message.  
    - Connect "Merge3" node to this node.  

31. **Add Slack node "Send message and wait for response":**  
    - Sends Slack message requesting approval with buttons.  
    - Connect "Edit Fields1" to this node.  
    - Configure Slack API credentials and webhook.  

32. **Add If node "Check Approval Status":**  
    - Condition: `$json.data.approved === true`  
    - Connect "Send message and wait for response" to this node.  

33. **Add Slack node "Rejection Notification":**  
    - Sends rejection message if approval is false.  
    - Connect If node's false output to this node.  

34. **Add Gmail node "Send a message1":**  
    - Sends approved article via email.  
    - Connect If node's true output to this node.  
    - Configure Gmail OAuth2 credentials.  

35. **Add HTTP Request node "Publish to Medium" (optional, currently disabled):**  
    - POST to your VPS API that publishes Medium posts.  
    - Use JSON-RPC format with title, content, tags, and publish status.  
    - Connect If node's true output to this node if publishing enabled.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| 📰 RSS for Content Fetching: Scheduled triggers fetch from The Verge and TechCrunch RSS feeds as primary content sources.                                                                                                                                 | Sticky Note covering Schedule Trigger and RSS nodes.                                                  |
| 🧠 Agent Article & Image Generation: Google Gemini, Groq, and Basic LLM Chain generate text content; Pollinations AI generates images; HTTP node downloads images; Google Drive stores them.                                                       | Sticky Note covering AI generation nodes and image handling.                                          |
| ☁️ Upload to Google Drive: Article and image are saved in Google Docs and Drive for review and storage; Slack handles review requests.                                                                                                          | Sticky Note covering Google Docs integration and Slack review.                                        |
| 📤 Slack Notification for Approval: Slack is used for human approval; approved posts are published to Medium and emailed; rejected posts trigger Slack notifications.                                                                           | Sticky Note covering Slack approval and conditional publishing nodes.                                 |
| Workflow requires valid credentials for Google Sheets, Google Drive, Google Docs, Slack, Gmail, Groq API, and Google Palm API.                                                                                                               | Ensure all OAuth2 and API keys are set up before running.                                             |
| The Medium Publishing API node is disabled by default; requires external VPS API endpoint and configuration to function.                                                                                                                     | Requires external service for Medium publishing.                                                      |
| Timezone considerations: Date and time stamps use 'Africa/Lagos' timezone; adjust as needed for your locale.                                                                                                                                   | Relevant in Code1 and REFORMAT ARTICLE nodes for consistent date/time.                                |
| Slack approval uses "double" approval type with button style "secondary" for better UX.                                                                                                                                                        | Configuration detail in Slack node "Send message and wait for response".                              |
| The workflow handles edge cases such as no fresh articles, API failures, and missing data with error throwing or conditional branches.                                                                                                        | Recommended to monitor workflow executions and logs for issues.                                      |

---

This document provides a detailed and structured reference for the "Automate RSS to Medium Publishing with Groq, Gemini & Slack Approval System" workflow, enabling expert users and AI agents to understand, reproduce, and extend the automation confidently.
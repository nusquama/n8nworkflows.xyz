Automate Blog Content Creation from Topics to WordPress with AI, Google Drive & Sheets

https://n8nworkflows.xyz/workflows/automate-blog-content-creation-from-topics-to-wordpress-with-ai--google-drive---sheets-10697


# Automate Blog Content Creation from Topics to WordPress with AI, Google Drive & Sheets

### 1. Workflow Overview

This workflow automates the end-to-end creation of blog content from a list of topics stored in Google Sheets, generating SEO-optimized HTML articles using AI, and publishing drafts to WordPress. It also saves each article as an HTML file in Google Drive and notifies a Slack channel upon completion of all topics.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Looping:** Reads blog topics from Google Sheets and processes them one at a time using batch splitting, enabling iterative handling of multiple topics.
- **1.2 Brand Context & AI Content Generation:** Uses AI language models to generate brand context and SEO-optimized HTML blog posts based on each topic.
- **1.3 HTML Processing & Export:** Cleans and converts generated content into an HTML file, then uploads it to Google Drive.
- **1.4 WordPress Publishing:** Publishes the HTML content as a draft post on WordPress.
- **1.5 Loop Control & Notification:** Determines loop completion and sends a Slack message summarizing the process.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Looping

**Overview:**  
This block fetches blog topics from a Google Sheets document and processes each row individually by splitting the input into batches of size one. This sets up a controlled loop for sequential topic processing.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get Blog Topics (Google Sheets)  
- Split In Batches (Batch Splitter)  
- IF Loop Finished (Conditional Check)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow manually.  
  - *Config:* Default trigger settings, no parameters.  
  - *Connections:* Outputs to "Get Blog Topics".  
  - *Failure Modes:* None applicable; manual initiation.  

- **Get Blog Topics**  
  - *Type:* Google Sheets  
  - *Role:* Reads rows from a Google Sheets document that contains blog topics.  
  - *Config:* Uses configured spreadsheet Document ID and Sheet Name (both set as dynamic list references, must be defined in credentials or environment).  
  - *Key Variables:* Outputs each row as JSON with topic data.  
  - *Connections:* Outputs to "Split In Batches".  
  - *Failure Modes:* API rate limits, missing or incorrect Sheet ID/Name, permission errors with Google Sheets OAuth2 credentials.  

- **Split In Batches**  
  - *Type:* Split In Batches  
  - *Role:* Creates batches of size 1 to process each topic individually, enabling a loop mechanism.  
  - *Config:* Batch size fixed to 1.  
  - *Connections:* Outputs to "Configuration" for processing the next topic batch and to "IF Loop Finished" for loop completion check.  
  - *Failure Modes:* Misconfiguration of batch size, large datasets causing performance issues.  

- **IF Loop Finished**  
  - *Type:* If Condition  
  - *Role:* Checks if the loop processing is complete by evaluating a boolean condition (set to always true here, likely a placeholder or control to trigger next steps).  
  - *Config:* Simple boolean condition comparing true to true (always passes).  
  - *Connections:* On true, outputs to "Send a message" for final notification.  
  - *Failure Modes:* Logic might be simplistic; in practical use, this should check actual loop completion flags or data exhaustion.

---

#### 1.2 Brand Context & AI Content Generation

**Overview:**  
This block uses AI to retrieve brand context and generate a complete, SEO-optimized HTML blog post body based on the current topic or blog title.

**Nodes Involved:**  
- Configuration (Set Node)  
- Basic LLM Chain (LangChain LLM)  
- Basic LLM Chain1 (LangChain LLM)  
- OpenRouter Chat Model (OpenRouter AI Model)  
- OpenRouter Chat Model1 (OpenRouter AI Model)

**Node Details:**

- **Configuration**  
  - *Type:* Set  
  - *Role:* Defines default or incoming parameters such as "Brand" and "Blog Title" for use downstream.  
  - *Config:* Assigns "Brand" from input JSON or defaults to "Brand-Name". Assigns "Blog Title" from input JSON, falling back to "Topic" or a placeholder string.  
  - *Connections:* Outputs to "Basic LLM Chain".  
  - *Failure Modes:* Missing or malformed input JSON keys, expression evaluation errors.  

- **Basic LLM Chain**  
  - *Type:* LangChain LLM Chain  
  - *Role:* Requests brand-related context from the AI model by prompting it as a senior research assistant.  
  - *Config:* Prompt dynamically includes the "Brand" value from the Configuration node.  
  - *Connections:* Outputs to "Basic LLM Chain1".  
  - *AI Model Connection:* Uses "OpenRouter Chat Model1" as the language model backend.  
  - *Failure Modes:* API rate limits, prompt errors, network issues, invalid credentials.  

- **Basic LLM Chain1**  
  - *Type:* LangChain LLM Chain  
  - *Role:* Generates a structured HTML blog post based on the "Blog Title" and SEO guidelines.  
  - *Config:*  
    - Role: SEO professional writing human-like content.  
    - Style instructions: No horizontal separators, valid HTML body only, specific allowed tags, no inline CSS, length constraints (800-1200 words).  
    - Dynamic insertion of blog title from "Configuration".  
  - *Connections:* Outputs to "Code in JavaScript".  
  - *AI Model Connection:* Uses "OpenRouter Chat Model" as the language model backend.  
  - *Failure Modes:* Model output may not conform strictly to HTML rules, API errors, network failures.  

- **OpenRouter Chat Model** and **OpenRouter Chat Model1**  
  - *Type:* LangChain OpenRouter AI Model  
  - *Role:* Provide AI language model processing for the two LLM Chain nodes, using OpenRouter API with the GPT-5 model for one and default for the other.  
  - *Config:* Uses OpenRouter API credentials with OAuth2 authentication.  
  - *Connections:* Serve as language model providers for respective LLM Chain nodes.  
  - *Failure Modes:* API key invalidation, rate limits, network errors, model unavailability.

---

#### 1.3 HTML Processing & Export

**Overview:**  
This block cleans the generated HTML article content, converts it into a base64-encoded HTML file, and uploads it to a specified Google Drive folder.

**Nodes Involved:**  
- Code in JavaScript  
- Upload file (Google Drive)

**Node Details:**

- **Code in JavaScript**  
  - *Type:* Code  
  - *Role:* Cleans and formats the AI-generated HTML content to add `<br>` after `</p>` tags, removes Markdown-style code block wrappers, and creates a safe, sanitized filename based on the blog title. Converts content to base64 for file upload.  
  - *Key Expressions:*  
    - Regex removes Markdown HTML code blocks.  
    - Replaces `</p>` with `</p><br>`.  
    - Sanitizes blog title to create safe filename.  
  - *Connections:* Outputs binary data and metadata to "Upload file".  
  - *Failure Modes:* If input text is missing or not string, malformed HTML could be produced; encoding errors; buffer conversion failures.  

- **Upload file**  
  - *Type:* Google Drive  
  - *Role:* Uploads the generated HTML file to a specific folder in Google Drive.  
  - *Config:*  
    - Uses file name and mimeType from previous node.  
    - Target folder ID is hardcoded to "1EXVI5bWTMUHJuxsRrf-jsovaBeDQiO3a" (Google Drive folder named "Blog-Content Creation").  
    - Uses Google Drive OAuth2 API credentials.  
  - *Connections:* Outputs to "Publish to WordPress".  
  - *Failure Modes:* Authentication failures, permission issues on folder, quota exceeded, invalid folder ID, file overwrite conflicts.

---

#### 1.4 WordPress Publishing

**Overview:**  
Posts the generated blog content as a draft entry in WordPress via its REST API.

**Nodes Involved:**  
- Publish to WordPress (HTTP Request)

**Node Details:**

- **Publish to WordPress**  
  - *Type:* HTTP Request  
  - *Role:* Sends a POST request to WordPress REST API endpoint `/wp-json/wp/v2/posts` to create a new post.  
  - *Config:*  
    - URL is a placeholder: `https://your-wordpress-site.com/wp-json/wp/v2/posts` (must be replaced with actual WordPress site URL).  
    - Method: POST.  
    - No additional headers or authentication shown — likely requires OAuth or application password setup outside the workflow.  
  - *Connections:* Outputs to "Split In Batches" to continue loop.  
  - *Failure Modes:* Authentication errors, missing required fields, site URL misconfiguration, network errors, API rate limits.

---

#### 1.5 Loop Control & Notification

**Overview:**  
After processing all blog topics, sends a confirmation message to a Slack channel with links to the last uploaded Google Drive file and WordPress post.

**Nodes Involved:**  
- Send a message (Slack)

**Node Details:**

- **Send a message**  
  - *Type:* Slack  
  - *Role:* Sends a notification message to a specific Slack channel indicating completion of all blog posts generation and publishing.  
  - *Config:*  
    - The message includes dynamic links to the last Google Drive file (`webViewLink`) and WordPress post (`link`).  
    - Channel set to "blog-creation" Slack channel via channel ID.  
    - Uses OAuth2 credentials for Slack API.  
  - *Failure Modes:* Slack API rate limits, invalid token, channel ID errors, network issues.

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                        | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                      |
|-------------------------|---------------------------|-------------------------------------|-----------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger            | Starts workflow manually             | None                        | Get Blog Topics             | # Blog Post Content Creation (Multi-Topic Automation): Overview of entire workflow and use case, suitable for marketing teams. |
| Get Blog Topics          | Google Sheets             | Reads blog topics list from Sheets  | When clicking ‘Execute workflow’ | Split In Batches           | ## Input & Loop: Reads topic data and creates loop for multi-topic automation.                                                  |
| Split In Batches         | Split In Batches          | Processes topics one at a time      | Get Blog Topics             | Configuration, IF Loop Finished | ## Input & Loop (continued)                                                                                                     |
| Configuration            | Set                       | Sets brand and blog title parameters | Split In Batches            | Basic LLM Chain             | ## Content Generation: Defines input for AI content generation.                                                                 |
| Basic LLM Chain          | LangChain LLM Chain       | Generates brand context via AI      | Configuration               | Basic LLM Chain1            | ## Content Generation (continued)                                                                                               |
| OpenRouter Chat Model1   | LangChain OpenRouter Model| AI model for Basic LLM Chain        | None (model for Basic LLM Chain) | Basic LLM Chain           | ## Content Generation (continued)                                                                                               |
| Basic LLM Chain1         | LangChain LLM Chain       | Generates SEO-optimized blog post HTML | Basic LLM Chain            | Code in JavaScript          | ## Content Generation (continued)                                                                                               |
| OpenRouter Chat Model    | LangChain OpenRouter Model| AI model for Basic LLM Chain1       | None (model for Basic LLM Chain1) | Basic LLM Chain1          | ## Content Generation (continued)                                                                                               |
| Code in JavaScript       | Code                      | Cleans HTML and prepares file       | Basic LLM Chain1            | Upload file                 | ## Export & Publishing: Prepares HTML file for upload.                                                                          |
| Upload file              | Google Drive              | Uploads HTML file to Google Drive   | Code in JavaScript          | Publish to WordPress        | ## Export & Publishing: Uploads file to specified folder in Google Drive.                                                       |
| Publish to WordPress     | HTTP Request              | Posts blog draft to WordPress       | Upload file                 | Split In Batches            | ## Export & Publishing: Posts content to WordPress draft.                                                                       |
| IF Loop Finished         | If Condition              | Checks loop completion              | Split In Batches            | Send a message              | ## Confirmation: Controls loop and triggers final notification.                                                                |
| Send a message           | Slack                     | Sends completion notification       | IF Loop Finished            | None                       | ## Confirmation: Notifies Slack channel with links to last file and post.                                                      |
| Sticky Note3             | Sticky Note               | Documentation                       | None                       | None                       | # Blog Post Content Creation (Multi-Topic Automation) overview.                                                                 |
| Sticky Note5             | Sticky Note               | Documentation                       | None                       | None                       | ## Input & Loop explanation.                                                                                                    |
| Sticky Note1             | Sticky Note               | Documentation                       | None                       | None                       | ## Content Generation explanation.                                                                                              |
| Sticky Note              | Sticky Note               | Documentation                       | None                       | None                       | ## Export & Publishing explanation.                                                                                            |
| Sticky Note2             | Sticky Note               | Documentation                       | None                       | None                       | ## Confirmation explanation.                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: "When clicking ‘Execute workflow’"  
   - Type: Manual Trigger (default settings).

2. **Create Google Sheets Node:**  
   - Name: "Get Blog Topics"  
   - Set Document ID and Sheet Name to your Google Sheets containing blog topics.  
   - Use Google Sheets OAuth2 credentials.

3. **Create Split In Batches Node:**  
   - Name: "Split In Batches"  
   - Set batch size to 1.

4. **Create Set Node:**  
   - Name: "Configuration"  
   - Assignments:  
     - "Brand": Expression `{{$json["Brand"] || "Brand-Name"}}`  
     - "Blog Title": Expression `{{$json["Blog Title"] || $json["Topic"] || "Blog Topic/Title"}}`

5. **Create LangChain LLM Chain Node (Brand Context):**  
   - Name: "Basic LLM Chain"  
   - Prompt: `Act as a senior research assistant. What do you know about {{ $json.Brand }}? What services do they offer?`  
   - Use OpenRouter Chat Model1 as AI backend.

6. **Create LangChain OpenRouter Model Node:**  
   - Name: "OpenRouter Chat Model1"  
   - Model: Default (or set as required)  
   - Credentials: OpenRouter API OAuth2.

7. **Connect "Basic LLM Chain" to "OpenRouter Chat Model1"** as its AI language model.

8. **Create LangChain LLM Chain Node (Blog Post Generation):**  
   - Name: "Basic LLM Chain1"  
   - Prompt:  
     ```
     You are an SEO professional.

     Topic: {{ $('Configuration').item.json['Blog Title'] }}

     Style and guidelines:

     Write like a human, expert and clear.

     Do not use horizontal separators between sections.

     Structure: exactly one <h1> at the beginning, followed by meaningful <h2> and, if needed, <h3>.

     Produce only a valid HTML body without <html> and without <body> wrappers. Allowed tags: <h1>, <h2>, <h3>, <p>, <a>, <ul>, <ol>, <li>, <strong>, <em>.

     No inline CSS, no external resources, no table of contents.

     No internal links.

     Total length about 800 to 1200 words.

     Output HTML only, no accompanying text or explanations.

     Begin now.
     ```  
   - Use OpenRouter Chat Model as AI backend.

9. **Create LangChain OpenRouter Model Node:**  
   - Name: "OpenRouter Chat Model"  
   - Model: `openai/gpt-5`  
   - Credentials: OpenRouter API OAuth2.

10. **Connect "Basic LLM Chain1" to "OpenRouter Chat Model"** as AI language model.

11. **Create Code Node:**  
    - Name: "Code in JavaScript"  
    - JavaScript code to:  
      - Remove markdown code block wrappers (```html) from text.  
      - Replace all `</p>` tags with `</p><br>`.  
      - Generate safe file name from blog title, replacing illegal filename characters with underscores.  
      - Convert HTML content to base64 buffer for upload.  
    - Outputs JSON with `fileName`, `mimeType`, and binary data.

12. **Create Google Drive Node:**  
    - Name: "Upload file"  
    - Upload file with name from previous node's JSON field `fileName`.  
    - Target folder ID: your Google Drive folder (e.g., "1EXVI5bWTMUHJuxsRrf-jsovaBeDQiO3a").  
    - Use Google Drive OAuth2 credentials.

13. **Create HTTP Request Node:**  
    - Name: "Publish to WordPress"  
    - Method: POST  
    - URL: Your WordPress REST API endpoint, e.g., `https://your-wordpress-site.com/wp-json/wp/v2/posts`  
    - Configure authentication (OAuth2, Basic Auth, or Application Password) as required by your WordPress.  
    - Send content payload accordingly (note: the exact payload construction is missing in original workflow and must be implemented to send post content).

14. **Create If Node:**  
    - Name: "IF Loop Finished"  
    - Condition: Boolean check (adjust logic to detect loop completion or data exhaustion).

15. **Create Slack Node:**  
    - Name: "Send a message"  
    - Channel: Slack channel ID for "blog-creation" or your own channel.  
    - Text: A templated message with dynamic links to the last Google Drive file and WordPress post, e.g.:  
      ```
      All blog posts have been generated and published.

      Last Google Drive file: {{$node["Upload file"].json["webViewLink"]}}
      Last WordPress post: {{$node["Publish to WordPress"].json["link"]}}
      ```  
    - Use Slack OAuth2 credentials.

16. **Connect nodes according to the workflow:**  
    - Manual Trigger → Get Blog Topics → Split In Batches  
    - Split In Batches → Configuration → Basic LLM Chain → Basic LLM Chain1 → Code in JavaScript → Upload file → Publish to WordPress → Split In Batches (loop back)  
    - Split In Batches → IF Loop Finished → Send a message (on loop completion)

17. **Configure all credentials:**  
    - Google Sheets OAuth2  
    - Google Drive OAuth2  
    - OpenRouter API OAuth2 (two instances)  
    - Slack OAuth2  
    - WordPress authentication (outside of n8n node, configure HTTP Request accordingly)

18. **Test the workflow with sample data to verify:**  
    - Google Sheets access and retrieval.  
    - AI prompt responses correctness and HTML output.  
    - File upload to Google Drive.  
    - WordPress draft post creation.  
    - Slack notification delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow is designed for marketing teams and content creators who manage blog topics in Google Sheets and want automation to WordPress. | Workflow purpose and audience.                                   |
| OpenRouter API is used as the AI backend with GPT-5 model for content generation. Ensure API keys are valid and rate limits respected.   | https://openrouter.ai                                             |
| Google Drive folder ID and WordPress URL must be customized for your environment before use.                                             | Google Drive folder and WordPress API setup instructions.        |
| Slack notification channel ID must be updated to your workspace and channel.                                                             | Slack API channel ID configuration.                              |
| To enable WordPress REST API publishing, proper authentication (OAuth2 or application password) must be configured outside the workflow. | WordPress REST API authentication guidelines.                    |
| Sticky notes in the workflow provide detailed explanations of each logical block and can be referred to for quick understanding.        | Visible inside n8n editor alongside workflow nodes.               |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.
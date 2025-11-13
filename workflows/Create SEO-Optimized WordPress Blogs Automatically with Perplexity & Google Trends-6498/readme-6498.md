Create SEO-Optimized WordPress Blogs Automatically with Perplexity & Google Trends

https://n8nworkflows.xyz/workflows/create-seo-optimized-wordpress-blogs-automatically-with-perplexity---google-trends-6498


# Create SEO-Optimized WordPress Blogs Automatically with Perplexity & Google Trends

### 1. Workflow Overview

This n8n workflow automates the creation of SEO-optimized WordPress blog posts by integrating multiple input channels, AI-driven content generation, keyword research via Google Trends, and posting to WordPress. It is designed to streamline content marketing pipelines by combining real-time triggers from communication platforms with advanced AI models and data enrichment.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Listens to messages from Slack, Telegram, Gmail, and (disabled) WhatsApp to receive blog post prompts or commands.
- **1.2 Initial Data Preparation:** Standardizes and prepares the incoming data for AI processing.
- **1.3 AI-Driven Keyword Analysis and Planning:** Uses Google Trends data and AI models (OpenAI, OpenRouter, LangChain agents) to find and refine SEO keywords and generate a preliminary blog plan.
- **1.4 AI Content Generation:** Generates the blog article in multiple parts using OpenAI models and structures the output.
- **1.5 Content Formatting and WordPress Publishing:** Converts the AI-generated content into Markdown, generates a URL slug and title, and publishes the post to WordPress.
- **1.6 Notifications and Logging:** Sends messages back to Slack, Telegram, Gmail, and logs the published posts in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block collects incoming blog post requests or commands from various chat and email platforms, enabling flexible trigger options.

**Nodes Involved:**  
- Slack Trigger  
- Telegram Trigger  
- Gmail Trigger  
- WhatsApp Trigger (disabled)  
- Edit Fields (to normalize input)

**Node Details:**

- **Slack Trigger**  
  - Type: Slack Trigger  
  - Role: Listens to Slack messages/events to start the workflow.  
  - Config: Default webhook, no parameters configured.  
  - Inputs: External Slack events.  
  - Outputs: Passes data to "Edit Fields".  
  - Edge cases: Slack auth failures, webhook misconfigurations, message formatting issues.

- **Telegram Trigger**  
  - Type: Telegram Trigger  
  - Role: Listens to Telegram messages to trigger workflow.  
  - Config: Default webhook, no parameters configured.  
  - Edge cases: Telegram API limits, auth expired.

- **Gmail Trigger**  
  - Type: Gmail Trigger  
  - Role: Triggers on new Gmail messages.  
  - Config: Default, no filters specified.  
  - Edge cases: Gmail OAuth token expiration.

- **WhatsApp Trigger** (disabled)  
  - Type: WhatsApp Trigger  
  - Role: Intended to trigger on WhatsApp messages, currently disabled.  
  - Edge cases: No active trigger.

- **Edit Fields**  
  - Type: Set  
  - Role: Normalizes and prepares input data from various triggers for downstream processing.  
  - Config: No explicit parameters; likely sets or renames fields for consistency.  
  - Inputs: From all triggers.  
  - Outputs: To "Blog Agent" node.

---

#### 2.2 AI-Driven Keyword Analysis and Planning

**Overview:**  
Combines Google Trends data with AI models to select the best SEO keywords and create a preliminary blog outline.

**Nodes Involved:**  
- Workflow Input Trigger  
- Google_trends search  
- Edit Fields2  
- choose the best keywords (LangChain chainLlm)  
- Structured Output Parser  
- Preliminary Plan1  
- Message a model2 (Perplexity)  
- Split Out2  
- Limit4  
- Merge4  
- Aggregate1  
- Previous Posts1 (Google Sheets)  
- OpenRouter Chat Model1  

**Node Details:**

- **Workflow Input Trigger**  
  - Type: Execute Workflow Trigger  
  - Role: Entry point for manual or external workflow execution.  
  - Outputs: To "Google_trends search".

- **Google_trends search**  
  - Type: SerpApi  
  - Role: Retrieves trending search keywords and data from Google Trends.  
  - Config: Parameters tuned for Google Trends API via SerpApi.  
  - Outputs: To "Edit Fields2".

- **Edit Fields2**  
  - Type: Set  
  - Role: Adjusts data fields from Google Trends output for AI input.  
  - Outputs: To "choose the best keywords".

- **choose the best keywords**  
  - Type: LangChain Chain LLM  
  - Role: Uses AI to analyze and select optimal keywords from trend data.  
  - Config: Uses OpenRouter Chat Model1 as language model.  
  - Inputs: From "Edit Fields2" and output parser.  
  - Outputs: To "Preliminary Plan1".  
  - Edge cases: Model API rate limits, parsing errors.

- **Structured Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses AI output into structured JSON for downstream nodes.  
  - Outputs: To "choose the best keywords".

- **Preliminary Plan1**  
  - Type: OpenAI (LangChain OpenAI)  
  - Role: Generates a preliminary blog post plan based on selected keywords.  
  - Outputs: To "Message a model2".

- **Message a model2**  
  - Type: Perplexity  
  - Role: Sends input to Perplexity AI for further refinement or data enrichment.  
  - Outputs: To "Split Out2", "Split Out4", and "Previous Posts1".

- **Split Out2 / Split Out4**  
  - Type: Split Out  
  - Role: Splits Perplexity output into multiple streams for parallel processing.  
  - Outputs: "Split Out2" to "Limit4"; "Split Out4" to "Edit Fields20".

- **Limit4**  
  - Type: Limit  
  - Role: Caps number of items passing downstream to control flow.  
  - Outputs: To "Merge4".

- **Merge4**  
  - Type: Merge  
  - Role: Combines multiple input streams into one for article writing.  
  - Outputs: To "Write Article part 1".

- **Aggregate1**  
  - Type: Aggregate  
  - Role: Aggregates previous blog posts info from Google Sheets.  
  - Inputs: From "Previous Posts1".  
  - Outputs: None (terminal for aggregation).

- **Previous Posts1**  
  - Type: Google Sheets  
  - Role: Retrieves previous blog posts for keyword overlap or history checks.  
  - Outputs: To "Aggregate1".

- **OpenRouter Chat Model1**  
  - Type: OpenRouter Chat Model  
  - Role: Language model used within "choose the best keywords".  
  - Inputs: From upstream.  
  - Outputs: To "choose the best keywords".

---

#### 2.3 AI Content Generation

**Overview:**  
Generates the blog article in two parts using OpenAI models, manages output formatting and splitting.

**Nodes Involved:**  
- Write Article part 1  
- Write Article part 2  
- Edit Fields21  
- Markdown  
- Slug  
- Title

**Node Details:**

- **Write Article part 1**  
  - Type: OpenAI (LangChain OpenAI)  
  - Role: Generates the first part of the blog article text.  
  - Outputs: To "Write Article part 2".

- **Write Article part 2**  
  - Type: OpenAI (LangChain OpenAI)  
  - Role: Completes the blog article text generation.  
  - Outputs: To "Edit Fields21".

- **Edit Fields21**  
  - Type: Set  
  - Role: Prepares the combined article text for Markdown conversion.  
  - Outputs: To "Markdown".

- **Markdown**  
  - Type: Markdown  
  - Role: Converts raw text into Markdown format for WordPress compatibility.  
  - Outputs: To "Slug".

- **Slug**  
  - Type: OpenAI  
  - Role: Generates a URL-friendly slug for the blog post based on content/title.  
  - Outputs: To "Title".

- **Title**  
  - Type: OpenAI  
  - Role: Generates or refines the blog post title for SEO optimization.  
  - Outputs: To "Create a post".

---

#### 2.4 Content Publishing and Logging

**Overview:**  
Publishes the generated blog post to WordPress and logs the post data into a Google Sheet. Also prepares a response for notifications.

**Nodes Involved:**  
- Create a post (WordPress)  
- Append row in sheet (Google Sheets)  
- response (Set)

**Node Details:**

- **Create a post**  
  - Type: WordPress  
  - Role: Publishes the blog content on the WordPress site using the generated title, slug, and content.  
  - Config: Uses WordPress credentials and API.  
  - Outputs: To "Append row in sheet".

- **Append row in sheet**  
  - Type: Google Sheets  
  - Role: Logs the new post details (e.g., title, slug, URL) to a Google Sheet for record keeping.  
  - Outputs: To "response".

- **response**  
  - Type: Set  
  - Role: Prepares a final structured response or status message after post creation and logging.  
  - Outputs: None.

---

#### 2.5 Notifications and Messaging

**Overview:**  
Sends notifications about the blog post creation status to Slack, Telegram, Gmail, and WhatsApp (disabled).

**Nodes Involved:**  
- Blog Agent (LangChain Agent)  
- Send a text message (Telegram)  
- Send message (WhatsApp, disabled)  
- Send a message (Slack)  
- Send a message1 (Gmail)  
- Edit Fields (initial) also leads here

**Node Details:**

- **Blog Agent**  
  - Type: LangChain Agent  
  - Role: Orchestrates sending status messages to multiple channels after blog generation.  
  - Outputs: To all messaging nodes.

- **Send a text message**  
  - Type: Telegram  
  - Role: Sends a Telegram message notification.  
  - Config: Uses Telegram bot credentials.

- **Send message**  
  - Type: WhatsApp (disabled)  
  - Role: Would send WhatsApp notifications if enabled.

- **Send a message**  
  - Type: Slack  
  - Role: Sends Slack notification about the workflow status.

- **Send a message1**  
  - Type: Gmail  
  - Role: Sends email notification about the blog creation.

---

### 3. Summary Table

| Node Name              | Node Type                           | Functional Role                            | Input Node(s)                  | Output Node(s)                      | Sticky Note                     |
|------------------------|-----------------------------------|-------------------------------------------|-------------------------------|-----------------------------------|--------------------------------|
| Slack Trigger          | Slack Trigger                     | Input reception from Slack                 | -                             | Edit Fields                       |                                |
| Telegram Trigger       | Telegram Trigger                  | Input reception from Telegram              | -                             | Edit Fields                       |                                |
| Gmail Trigger          | Gmail Trigger                    | Input reception from Gmail                  | -                             | Edit Fields                       |                                |
| WhatsApp Trigger       | WhatsApp Trigger (disabled)       | Input reception from WhatsApp (disabled)  | -                             | Edit Fields                       |                                |
| Edit Fields            | Set                              | Normalize input data from triggers         | Slack Trigger, Telegram Trigger, Gmail Trigger, WhatsApp Trigger | Blog Agent                       |                                |
| Blog Agent             | LangChain Agent                  | Orchestrate notification sending           | Edit Fields                    | Send a text message, Send message, Send a message, Send a message1 |                                |
| Send a text message    | Telegram                         | Send Telegram notification                  | Blog Agent                    | -                                 |                                |
| Send message           | WhatsApp (disabled)              | Send WhatsApp notification (disabled)      | Blog Agent                    | -                                 |                                |
| Send a message         | Slack                           | Send Slack notification                     | Blog Agent                    | -                                 |                                |
| Send a message1        | Gmail                           | Send Gmail notification                     | Blog Agent                    | -                                 |                                |
| Workflow Input Trigger | Execute Workflow Trigger          | Manual or external workflow start           | -                             | Google_trends search              |                                |
| Google_trends search   | SerpApi                         | Fetch Google Trends data                    | Workflow Input Trigger         | Edit Fields2                     |                                |
| Edit Fields2           | Set                              | Prepare Google Trends data for AI input     | Google_trends search           | choose the best keywords          |                                |
| choose the best keywords| LangChain Chain LLM              | Select best SEO keywords                     | Edit Fields2, Structured Output Parser | Preliminary Plan1               |                                |
| Structured Output Parser| LangChain Output Parser Structured| Parse AI output for keywords                 | choose the best keywords       | choose the best keywords          |                                |
| Preliminary Plan1      | OpenAI (LangChain OpenAI)        | Generate blog outline                        | choose the best keywords       | Message a model2                 |                                |
| Message a model2       | Perplexity                      | Further AI refinement and data enrichment   | Preliminary Plan1              | Split Out2, Split Out4, Previous Posts1 |                                |
| Split Out2             | Split Out                       | Split Perplexity output                      | Message a model2               | Limit4                          |                                |
| Limit4                 | Limit                          | Limit items to manageable number            | Split Out2                    | Merge4                          |                                |
| Merge4                 | Merge                          | Combine multiple data streams                | Limit4, Aggregate6            | Write Article part 1             |                                |
| Aggregate6             | Aggregate                      | Aggregate intermediate data                  | Edit Fields20                 | Merge4                          |                                |
| Edit Fields20          | Set                            | Prepare data for aggregation                  | Split Out4                   | Aggregate6                      |                                |
| Previous Posts1        | Google Sheets                  | Retrieve previous posts for comparison       | Message a model2               | Aggregate1                      |                                |
| Aggregate1             | Aggregate                     | Aggregate previous posts data                 | Previous Posts1               | -                              |                                |
| Write Article part 1   | OpenAI (LangChain OpenAI)       | Generate first part of blog article           | Merge4                       | Write Article part 2            |                                |
| Write Article part 2   | OpenAI (LangChain OpenAI)       | Generate second part of blog article          | Write Article part 1          | Edit Fields21                   |                                |
| Edit Fields21          | Set                            | Prepare article for Markdown conversion       | Write Article part 2          | Markdown                       |                                |
| Markdown               | Markdown                      | Convert text to Markdown format                | Edit Fields21                | Slug                          |                                |
| Slug                   | OpenAI                        | Generate SEO-friendly URL slug                  | Markdown                     | Title                         |                                |
| Title                  | OpenAI                        | Generate or refine SEO-optimized title           | Slug                        | Create a post                 |                                |
| Create a post          | WordPress                     | Publish blog post on WordPress                   | Title                       | Append row in sheet           |                                |
| Append row in sheet    | Google Sheets                 | Log published post details                       | Create a post               | response                     |                                |
| response               | Set                          | Prepare final response/status                    | Append row in sheet          | -                             |                                |
| Call n8n Workflow Tool | LangChain Tool Workflow       | (Unused in connections) Possibly for sub-workflow calls | -                         | Blog Agent                   |                                |
| OpenRouter Chat Model  | LangChain OpenRouter Chat Model| Language model used by Blog Agent               | -                           | Blog Agent                   |                                |
| OpenRouter Chat Model1 | LangChain OpenRouter Chat Model| Language model used by keyword chooser          | -                           | choose the best keywords      |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers for Inputs:**
   - Add **Slack Trigger** node, set up webhook to listen for Slack messages.
   - Add **Telegram Trigger** node, configure with Telegram bot credentials.
   - Add **Gmail Trigger** node, authorize Gmail OAuth2 and set to trigger on new emails.
   - Optionally add **WhatsApp Trigger** (disabled here).
  
2. **Normalize Inputs:**
   - Add a **Set (Edit Fields)** node connected from all triggers to standardize incoming data fields for downstream processing.

3. **Notification Setup:**
   - Add **LangChain Agent (Blog Agent)** node connected from "Edit Fields", configured to send messages.
   - Add messaging nodes:
     - **Telegram** node to send text messages.
     - **Slack** node to send messages.
     - **Gmail** node to send emails.
     - Optionally a **WhatsApp** node (disabled).
   - Connect "Blog Agent" outputs to these messaging nodes.

4. **Entry Trigger for SEO Keyword Research:**
   - Add **Execute Workflow Trigger** node to start the keyword research workflow independently.

5. **Google Trends Data:**
   - Add **SerpApi (Google_trends search)** node connected from workflow trigger.
   - Configure SerpApi with Google Trends query parameters.

6. **Prepare Data for AI:**
   - Add a **Set (Edit Fields2)** node to clean and prepare Google Trends data, connected from SerpApi node.

7. **Keyword Selection:**
   - Add **LangChain Chain LLM (choose the best keywords)** node connected from "Edit Fields2".
   - Add **LangChain OpenRouter Chat Model1** node as the language model for keyword chooser.
   - Add **LangChain Structured Output Parser** node connected before the keyword chooser to parse AI output.

8. **Preliminary Blog Planning:**
   - Add **OpenAI (Preliminary Plan1)** node connected from keyword chooser.
   - Configure with prompts to create blog outline.

9. **Data Enrichment and Refinement:**
   - Add **Perplexity (Message a model2)** node connected from preliminary plan.
   - Split Perplexity output with two **Split Out** nodes for parallel processing.
   - Add **Limit (Limit4)** node connected from one Split Out to control output size.
   - Add **Set (Edit Fields20)** node connected from second Split Out for further data prep.
   - Add **Aggregate (Aggregate6)** node connected from Edit Fields20.
   - Add **Merge (Merge4)** node to combine limited and aggregated data streams.

10. **Previous Posts Data:**
    - Add **Google Sheets (Previous Posts1)** node to fetch past blog entries.
    - Connect it to **Aggregate (Aggregate1)** node.

11. **Article Writing:**
    - Connect "Merge4" to **OpenAI (Write Article part 1)** node.
    - Chain to **OpenAI (Write Article part 2)** node.
    - Add **Set (Edit Fields21)** node connected after article part 2.
    - Add **Markdown** node to convert article text to Markdown format.

12. **SEO Metadata Generation:**
    - Add **OpenAI (Slug)** node connected from Markdown to generate URL slug.
    - Add **OpenAI (Title)** node connected from Slug to generate SEO title.

13. **Publishing:**
    - Add **WordPress** node configured with WordPress API credentials.
    - Connect from Title node to publish the post.

14. **Logging:**
    - Add **Google Sheets (Append row in sheet)** node to log post details.
    - Connect from WordPress node.

15. **Final Response:**
    - Add **Set (response)** node connected from Google Sheets append node to prepare output or status.

16. **Linking Notifications:**
    - Ensure "Blog Agent" node triggers notifications upon completion of post creation and logging.

17. **Credentials Setup:**
    - Configure all external service credentials:
      - Slack OAuth and webhook.
      - Telegram Bot API.
      - Gmail OAuth2.
      - WordPress API (username/password or OAuth).
      - Google Sheets API.
      - SerpApi with Google Trends access.
      - OpenAI API keys.
      - OpenRouter API keys.
      - Perplexity API keys.

18. **Testing and Validation:**
    - Test each trigger independently.
    - Validate AI prompt outputs.
    - Monitor API limits for external services.
    - Handle error paths for authentication failures or API timeouts.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                      |
|------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow integrates LangChain nodes with OpenAI and OpenRouter models for advanced AI orchestration. | n8n LangChain integration documentation                                                             |
| Google Trends data is retrieved through SerpApi, which requires a valid API key and correct query formatting. | https://serpapi.com/google-trends-api/                                                              |
| WordPress node requires REST API access or Application Passwords enabled in WordPress. | https://developer.wordpress.org/rest-api/                                                           |
| Slack, Telegram, and Gmail triggers require proper OAuth2 credentials setup in n8n. | n8n credential setup documentation                                                                   |
| Perplexity node is used for data enrichment; ensure API quota and rate limits are respected. | Perplexity API documentation                                                                         |
| The workflow supports multi-channel notifications to inform stakeholders after blog post creation. | Useful for marketing or editorial teams notifications                                               |

---

**Disclaimer:** The text above originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.
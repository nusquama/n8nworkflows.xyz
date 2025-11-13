Auto-Generate SEO Articles in WordPress with Gemini AI and OpenAI Images

https://n8nworkflows.xyz/workflows/auto-generate-seo-articles-in-wordpress-with-gemini-ai-and-openai-images-6275


# Auto-Generate SEO Articles in WordPress with Gemini AI and OpenAI Images

---

### 1. Workflow Overview

This workflow automates the generation and posting of SEO-optimized blog articles to a WordPress website using AI-powered content and image creation tools. It targets content creators, bloggers, affiliate marketers, agencies, and newsrooms aiming to streamline content production with minimal manual intervention.

The workflow divides logically into the following blocks:

- **1.1 Input & Setup:** Manual trigger and website domain configuration.
- **1.2 Topic & Title Generation:** Uses Google Gemini AI to select a topic and generate article metadata (title, slug, focus phrase, meta description).
- **1.3 Article Content Generation:** Uses Google Gemini AI to create a full SEO-optimized blog article based on the generated metadata.
- **1.4 Draft Post Creation in WordPress:** Uploads the generated article as a draft post on the target WordPress site.
- **1.5 AI-Generated Featured Image Creation:** Uses OpenAI to generate a realistic featured image related to the article.
- **1.6 Upload Featured Image & Set Post Thumbnail:** Uploads the generated image to WordPress and assigns it as the featured image for the draft post.
- **1.7 Optional Notification:** Sends a Telegram message to notify that the draft post is ready.

---

### 2. Block-by-Block Analysis

#### 1.1 Input & Setup

- **Overview:** Initiates the workflow manually and defines the target WordPress website domain.
- **Nodes Involved:** 
  - When clicking ‘Test workflow’ (Manual Trigger)
  - Fields - Set Website (Set Node)
- **Node Details:**

  - **When clicking ‘Test workflow’**
    - Type: Manual Trigger
    - Role: Starts the workflow on user command.
    - Configuration: Default, no parameters.
    - Inputs: None
    - Outputs: Triggers next node (Fields - Set Website)
    - Failure modes: None likely.
  
  - **Fields - Set Website**
    - Type: Set
    - Role: Defines the target website domain for WordPress API calls.
    - Configuration: Assigns string value `"www.agentcircle.ai"` to variable `website`.
    - Inputs: From manual trigger
    - Outputs: Passes `website` variable downstream.
    - Failure modes: Incorrect domain or missing domain may cause API failures downstream.

#### 1.2 Topic & Title Generation

- **Overview:** Uses Google Gemini AI to randomly select a topic and generate article metadata in a structured format.
- **Nodes Involved:** 
  - Agent - Topic Chooser & Title Generator (Chain LLM)
  - Google Gemini Chat Model 1 (Google Gemini language model)
  - Structured Output Parser (Structured Output Parser)
- **Node Details:**

  - **Agent - Topic Chooser & Title Generator**
    - Type: Chain LLM Node (Google Gemini interface)
    - Role: Generates article metadata (category, title, slug, focus phrase, meta description).
    - Configuration: Prompt instructs AI to select one topic from predefined categories and generate SEO elements with current year placeholder.
    - Key expressions: Uses `{{ $now.year }}` for dynamic year insertion.
    - Inputs: Receives `website` domain from Fields - Set Website.
    - Outputs: JSON with article elements.
    - Failure modes: AI response parsing errors, API call failures, or empty output.
  
  - **Google Gemini Chat Model 1**
    - Type: Google Gemini Chat Model node
    - Role: Executes the AI prompt for topic and title generation.
    - Configuration: Uses model `models/gemini-2.0-flash`.
    - Inputs: Connected from Agent - Topic Chooser & Title Generator.
    - Outputs: AI-generated text.
    - Failure modes: API authentication errors, rate limiting, network issues.
  
  - **Structured Output Parser**
    - Type: Output Parser Structured
    - Role: Parses AI text output into structured JSON matching schema (category, title, slug, focus phrase, meta description).
    - Configuration: JSON schema example provided for validation.
    - Inputs: Output text from Agent - Topic Chooser & Title Generator.
    - Outputs: Parsed structured JSON.
    - Failure modes: Parsing errors if AI output deviates from expected format.

#### 1.3 Article Content Generation

- **Overview:** Uses Google Gemini AI to generate a complete SEO-optimized blog article of 1,500–2,500 words based on the metadata.
- **Nodes Involved:** 
  - Agent - Article Generator (Chain LLM)
  - Google Gemini Chat Model 2 (Google Gemini language model)
- **Node Details:**

  - **Agent - Article Generator**
    - Type: Chain LLM Node (Google Gemini interface)
    - Role: Produces a full WordPress-ready SEO article using inputs from the topic/title generator.
    - Configuration: Detailed prompt specifying article length, structure, SEO optimization, style, and formatting compatible with WordPress.
    - Key expressions: Uses `{{ $json.output.title }}`, `{{ $json.output.category }}`, and `{{ $json.output.focus_phrase }}` from previous node.
    - Inputs: Parsed metadata from Structured Output Parser.
    - Outputs: Article text formatted with WordPress HTML headings and lists.
    - Failure modes: AI text generation failures, incomplete content, formatting issues.
  
  - **Google Gemini Chat Model 2**
    - Type: Google Gemini Chat Model node
    - Role: Executes the AI prompt for article content generation.
    - Configuration: Uses model `models/gemini-2.0-flash`.
    - Inputs: Connected from Agent - Article Generator.
    - Outputs: Generated article content.
    - Failure modes: Same as Google Gemini Chat Model 1.

#### 1.4 Draft Post Creation in WordPress

- **Overview:** Uploads the generated article content as a draft post to the specified WordPress site.
- **Nodes Involved:** 
  - Wordpress - Post Draft (WordPress node)
- **Node Details:**

  - **Wordpress - Post Draft**
    - Type: WordPress node (Post creation)
    - Role: Creates a new post in draft status with the article content.
    - Configuration:
      - Title: From `Agent - Topic Chooser & Title Generator`.
      - Slug: From same node, URL-friendly.
      - Content: From `Agent - Article Generator`.
      - Status: Draft.
      - Author ID: 1 (default admin).
      - Category: Dynamically assigned based on category output.
    - Inputs: Article text and metadata.
    - Outputs: Post creation response including post ID.
    - Failure modes: Authentication errors, invalid post data, network issues.

#### 1.5 AI-Generated Featured Image Creation

- **Overview:** Generates a realistic image representing the article theme using OpenAI's image generation API.
- **Nodes Involved:** 
  - OpenAI - Generate Image (OpenAI Image generation)
- **Node Details:**

  - **OpenAI - Generate Image**
    - Type: OpenAI Image Generation Node
    - Role: Creates a high-resolution, photorealistic image corresponding to the article title.
    - Configuration:
      - Prompt: Detailed instructions to create a realistic image without text, based on article title.
      - Size: 1024x1024 pixels.
      - Style: Vivid.
    - Inputs: Article title from `Agent - Topic Chooser & Title Generator`.
    - Outputs: Binary image data.
    - Failure modes: API quota limits, invalid prompt, timeout.

#### 1.6 Upload Featured Image & Set Post Thumbnail

- **Overview:** Uploads the generated image to WordPress media library and assigns it as the featured image for the draft post.
- **Nodes Involved:** 
  - Wordpress - Upload Image (HTTP Request to WordPress REST API)
  - Wordpress - Set Featured Image & Post (HTTP Request to update post)
- **Node Details:**

  - **Wordpress - Upload Image**
    - Type: HTTP Request node
    - Role: Uploads binary image data to WordPress media endpoint.
    - Configuration:
      - URL: Dynamically constructed from `Fields - Set Website` domain.
      - Method: POST.
      - Headers: Content-Type image/png, Content-Disposition with timestamp filename.
      - Authentication: Predefined WordPress credential.
      - Input field: binary image data from OpenAI node.
    - Inputs: Image binary data.
    - Outputs: Media upload response with attachment ID.
    - Failure modes: Authentication, file upload size limits, network errors.
  
  - **Wordpress - Set Featured Image & Post**
    - Type: HTTP Request node
    - Role: Updates the draft post to associate the uploaded image as featured media.
    - Configuration:
      - URL: Post update endpoint with post ID.
      - Method: POST.
      - Body: JSON with `featured_media` set to uploaded image ID.
      - Authentication: WordPress credential.
    - Inputs: Media ID from image upload; post ID from draft creation.
    - Outputs: Updated post response.
    - Failure modes: Invalid post or media ID, permission errors.

#### 1.7 Optional Notification

- **Overview:** Sends notification message via Telegram once the draft post is ready.
- **Nodes Involved:** 
  - Telegram - Send Message (Telegram node)
- **Node Details:**

  - **Telegram - Send Message**
    - Type: Telegram node
    - Role: Sends a message with post link or confirmation.
    - Configuration:
      - Chat ID: Configured (currently blank/disabled).
      - Text: "Post Published! Article: {{ $json.link }}" (dynamic).
    - Inputs: From update post node.
    - Outputs: Confirmation of message sent.
    - Disabled by default.
    - Failure modes: Missing chat ID, Telegram API errors.

---

### 3. Summary Table

| Node Name                        | Node Type                              | Functional Role                                | Input Node(s)                         | Output Node(s)                      | Sticky Note                                                                                                             |
|---------------------------------|--------------------------------------|-----------------------------------------------|-------------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’    | Manual Trigger                       | Workflow start trigger                         | None                                | Fields - Set Website               | ## 1. Initiate And Set Up<br>- Specify the target Wordpress website domain using the **Fields – Set Website** node.<br>- Manually trigger the workflow. |
| Fields - Set Website             | Set Node                            | Define WordPress target website domain        | When clicking ‘Test workflow’       | Agent - Topic Chooser & Title Generator |                                                                                                                         |
| Agent - Topic Chooser & Title Generator | Chain LLM (Google Gemini)           | Generate article metadata (topic, title, slug, focus phrase, meta description) | Fields - Set Website                | Structured Output Parser, Agent - Article Generator | ## 2. Generate Topic, Title And Content With AI Agents<br>- The first **Agent – Topic Chooser & Title Generator** (using Google Gemini) randomly selects one of your pre-defined website-related topics (in the prompt) and requests the chat model to generate article elements in your specified format. The results are then parsed and routed to the next AI agent.<br>- The second **Agent – Article Generator** (powered by Google Gemini) creates a complete, SEO-optimized article based on your chosen topic and settings.<br>⚠️ **Important Note**: Adjust prompt topics to match your own website domain, industry, and audience. |
| Google Gemini Chat Model 1       | Google Gemini Chat Model             | Executes AI prompt for topic/title generation | Agent - Topic Chooser & Title Generator | Agent - Topic Chooser & Title Generator |                                                                                                                         |
| Structured Output Parser         | Structured Output Parser             | Parses AI output into structured JSON         | Agent - Topic Chooser & Title Generator | Agent - Topic Chooser & Title Generator |                                                                                                                         |
| Agent - Article Generator        | Chain LLM (Google Gemini)            | Generate full SEO blog article content         | Agent - Topic Chooser & Title Generator | Wordpress - Post Draft            |                                                                                                                         |
| Google Gemini Chat Model 2       | Google Gemini Chat Model             | Executes AI prompt for article content generation | Agent - Article Generator           | Agent - Article Generator          |                                                                                                                         |
| Wordpress - Post Draft           | WordPress Node                      | Upload article as draft post                    | Agent - Article Generator           | OpenAI - Generate Image            | ## 3. Upload To Wordpress<br>- The workflow automatically uploads the generated article as a draft post to your Wordpress site. |
| OpenAI - Generate Image          | OpenAI Image Generation             | Generate featured image from article title     | Wordpress - Post Draft              | Wordpress - Upload Image           | ## 4. Create The Draft Post & Set Featured Image<br>- The article is sent to the OpenAI to create a unique, relevant visual.<br>- The generated image is uploaded and set as the featured image for the draft article.<br>- The draft post is ready in the Wordpress dashboard for further review before manually publishing. |
| Wordpress - Upload Image         | HTTP Request                       | Upload generated image to WordPress media library | OpenAI - Generate Image             | Wordpress - Set Featured Image & Post |                                                                                                                         |
| Wordpress - Set Featured Image & Post | HTTP Request                       | Set uploaded image as featured image for post   | Wordpress - Upload Image            | Telegram - Send Message            |                                                                                                                         |
| Telegram - Send Message          | Telegram Node                      | Notify via Telegram that draft post is ready   | Wordpress - Set Featured Image & Post | None                             | ## 5. Get Notified<br>- Receive a notification via Telegram or another connected chat app once the post is ready as a draft in Wordpress. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Type: Manual Trigger
   - Purpose: Start workflow manually.

2. **Create Set Node: Fields - Set Website**
   - Type: Set
   - Add string field `website` with value `"www.agentcircle.ai"` (replace with your WordPress domain).
   - Connect Manual Trigger output to this node.

3. **Create Chain LLM Node: Agent - Topic Chooser & Title Generator**
   - Type: Chain LLM (Google Gemini)
   - Credentials: Google Gemini API credentials.
   - Prompt: Configure with instructions to randomly select a category (Artificial Intelligence (AI), AI Agents, Automation, Workflow, Prompts), generate article title, slug, focus phrase, meta description, using `{{ $now.year }}` for current year.
   - Connect Fields - Set Website output to this node.

4. **Create Google Gemini Chat Model 1 Node**
   - Type: Google Gemini Chat Model
   - Model: `models/gemini-2.0-flash`
   - Credentials: Google Gemini API credentials.
   - Link as AI language model for Agent - Topic Chooser & Title Generator node.

5. **Add Structured Output Parser Node**
   - Type: Structured Output Parser
   - Configure JSON schema example matching expected output fields (category, title, slug, focus_phrase, meta_description).
   - Connect output of Agent - Topic Chooser & Title Generator to this parser on `ai_outputParser` connection.

6. **Create Chain LLM Node: Agent - Article Generator**
   - Type: Chain LLM (Google Gemini)
   - Credentials: Google Gemini API credentials.
   - Prompt: Use detailed instructions to create a 1,500–2,500-word SEO blog article with headings, keyword density, style, WordPress-compatible formatting, excluding title duplication.
   - Inputs: Use outputs from previous parser node (`title`, `category`, `focus_phrase`).
   - Connect output of Agent - Topic Chooser & Title Generator (main) to this node.

7. **Create Google Gemini Chat Model 2 Node**
   - Type: Google Gemini Chat Model
   - Model: `models/gemini-2.0-flash`
   - Credentials: Google Gemini API credentials.
   - Link as AI language model for Agent - Article Generator node.

8. **Create WordPress Node: Wordpress - Post Draft**
   - Type: WordPress
   - Credentials: WordPress API with admin permissions.
   - Parameters:
     - Title: From Agent - Topic Chooser & Title Generator title output.
     - Slug: From same.
     - Content: From Agent - Article Generator output.
     - Status: draft.
     - Author ID: 1.
     - Categories: Map category string to WordPress category IDs dynamically.
   - Connect output of Agent - Article Generator to this node.

9. **Create OpenAI Image Generation Node: OpenAI - Generate Image**
   - Type: OpenAI Image Generation
   - Credentials: OpenAI API.
   - Prompt: Generate a realistic, natural image representing the article title (`{{ $json.title.raw }}`).
   - Size: 1024x1024, style vivid.
   - Connect output of Wordpress - Post Draft to this node.

10. **Create HTTP Request Node: Wordpress - Upload Image**
    - Type: HTTP Request
    - Credentials: WordPress API.
    - URL: `https://{{ $json.website }}/wp-json/wp/v2/media`
    - Method: POST
    - Headers: `Content-Type: image/png`, `Content-Disposition: attachment; filename={{ $now.toMillis() }}.png`
    - Body: Binary data from OpenAI image node.
    - Connect OpenAI - Generate Image output to this node.

11. **Create HTTP Request Node: Wordpress - Set Featured Image & Post**
    - Type: HTTP Request
    - Credentials: WordPress API.
    - URL: `https://{{ $json.website }}/wp-json/wp/v2/posts/{{ $json.id }}`
    - Method: POST
    - Body Parameters: JSON with `featured_media` set to uploaded image ID.
    - Connect Wordpress - Upload Image output to this node.

12. **(Optional) Create Telegram Node: Telegram - Send Message**
    - Type: Telegram
    - Credentials: Telegram API.
    - Chat ID: Your Telegram chat ID.
    - Text: `Post Published!\n\nArticle: {{ $json.link }}`
    - Connect Wordpress - Set Featured Image & Post output to this node.
    - Enable or disable per preference.

13. **Connect the Nodes According to the Dependencies Described Above to Complete the Workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow automates SEO article production and posting for WordPress using Google Gemini for content and OpenAI for image generation. It is designed to help blog owners, affiliate marketers, agencies, and content teams scale content creation efficiently. To optimize results, adjust the AI prompt topics and domain to your own business context. | Workflow Purpose                                                                                  |
| Requires credentials for Google Gemini API (for AI content generation), OpenAI API (for image generation), WordPress API (admin access), and optionally Telegram API for notifications.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Credential Setup                                                                                 |
| The prompt in **Agent - Topic Chooser & Title Generator** is key for controlling topic selection and article metadata generation; customize it carefully to reflect your site's niche and preferred topics.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | AI Prompt Customization                                                                          |
| WordPress categories mapping in **Wordpress - Post Draft** must be updated to match your site's category IDs to ensure correct categorization of posts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | WordPress Category Configuration                                                                |
| The featured image generation prompt instructs OpenAI to produce photorealistic images without text or surreal elements, suitable for professional blog posts.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Image Generation Prompt                                                                          |
| Telegram notification is optional and disabled by default; configure your Telegram chat ID and enable the node if you want real-time alerts when drafts are ready.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Optional Notification Setup                                                                     |
| For further assistance or customization, visit [Agent Circle](https://www.agentcircle.ai/) or join their community channels listed in the workflow sticky notes.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Support and Community Resources                                                                  |
| Workflow tested with n8n version supporting LangChain nodes and latest WordPress REST API standards; ensure your n8n instance meets these requirements for compatibility.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Version Compatibility                                                                           |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow and fully complies with legal and content policies. It contains no illegal or protected content. All manipulated data is public and lawful.

---
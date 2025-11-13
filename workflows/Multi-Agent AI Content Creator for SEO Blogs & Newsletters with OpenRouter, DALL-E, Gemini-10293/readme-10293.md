Multi-Agent AI Content Creator for SEO Blogs & Newsletters with OpenRouter, DALL-E, Gemini

https://n8nworkflows.xyz/workflows/multi-agent-ai-content-creator-for-seo-blogs---newsletters-with-openrouter--dall-e--gemini-10293


# Multi-Agent AI Content Creator for SEO Blogs & Newsletters with OpenRouter, DALL-E, Gemini

### 1. Workflow Overview

This workflow automates the creation of SEO-optimized blogs and newsletters using a multi-agent AI system integrated with OpenRouter, Google Gemini, and OpenAI’s DALL-E. It is designed for digital marketers, content creators, and teams seeking efficient, research-backed, and well-structured content generation with minimal manual intervention.

The workflow comprises the following logical blocks:

- **1.1 Input Reception:** Captures user input from a web form specifying content type (blog or newsletter) and topic.
- **1.2 Multi-Agent AI Content Generation Pipeline:** Sequentially runs four specialized AI agents (Research, Outline, Writer, Editor & SEO) to produce high-quality content drafts.
- **1.3 Content Routing:** Routes output based on content type to either blog or newsletter publishing paths.
- **1.4 Publishing and Saving:** Formats final content for target platforms, generates featured images (for blogs), and saves results to Airtable or Google Sheets. Sends optional Telegram notifications.
- **1.5 Image Generation:** Uses DALL-E to create visual featured images for blogs.
- **1.6 Credential and Model Management:** Integrates multiple AI models (OpenRouter Grok, Google Gemini, OpenAI DALL-E) with appropriate credential handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a user submits a form indicating the desired content type (Blog or Newsletter) and topic, serving as the entry point.

- **Nodes Involved:**  
  - On Form Submission

- **Node Details:**

  - **On Form Submission**  
    - *Type:* Form Trigger  
    - *Role:* Starts workflow on user form submission  
    - *Configuration:*  
      - Webhook with form titled "SEO Blog & Newsletter Generator"  
      - Fields: Dropdown for Content Type (Blog, Newsletter), Textarea for Topic (required)  
    - *Inputs:* User form data  
    - *Outputs:* JSON with `Content Type` and `Topic` fields  
    - *Edge cases:* Missing required fields, webhook misconfiguration, invalid input values  
    - *Version:* 2.3  

---

#### 2.2 Multi-Agent AI Content Generation Pipeline

- **Overview:**  
  Sequentially processes input through four AI agents to research, outline, write, and polish content. Utilizes OpenRouter Grok model for efficient AI processing.

- **Nodes Involved:**  
  - Research Agent  
  - Outline Agent  
  - Writer Agent  
  - Editor & SEO Agent  
  - OpenRouter Chat Model (Grok)

- **Node Details:**

  - **OpenRouter Chat Model (Grok)**  
    - *Type:* Language model node for OpenRouter API  
    - *Role:* Provides AI processing power for the four agents  
    - *Configuration:*  
      - Model: `agentica-org/deepcoder-14b-preview:free`  
      - Max tokens: 4000  
      - Temperature: 0.7 (balance creativity and coherence)  
      - Credential: OpenRouter API key  
    - *Connections:* Linked as AI language model to Research, Outline, Writer, and Editor agents  
    - *Edge cases:* API key invalid/expired, rate limits, timeout, response format errors

  - **Research Agent**  
    - *Type:* LangChain AI Agent  
    - *Role:* Gathers facts, statistics, sources, keywords, and title ideas for the topic  
    - *Configuration:*  
      - Input prompt includes content type and topic from form data  
      - Outputs JSON containing title ideas, keywords, facts (with sources and reliability scores), subtopics  
    - *Input:* Form submission JSON  
    - *Output:* JSON research data  
    - *Edge cases:* Insufficient or outdated data, JSON parsing failures, unreliable sources

  - **Outline Agent**  
    - *Type:* LangChain AI Agent  
    - *Role:* Creates a structured content outline with headings and word counts  
    - *Configuration:*  
      - Input: JSON from Research Agent  
      - Output: JSON outline including chosen title, headings, levels, word counts, notes, total word count  
    - *Edge cases:* Misinterpretation of research data, incomplete outlines

  - **Writer Agent**  
    - *Type:* LangChain AI Agent  
    - *Role:* Expands the outline into a full markdown draft with citations, engaging style  
    - *Configuration:*  
      - Input: Output from Outline Agent  
      - Output: Markdown-formatted blog/newsletter draft with TL;DR and call-to-action  
    - *Edge cases:* Markdown formatting errors, verbosity, missing citations

  - **Editor & SEO Agent**  
    - *Type:* LangChain AI Agent  
    - *Role:* Polishes grammar, optimizes SEO metadata, verifies citations, checks plagiarism risk  
    - *Configuration:*  
      - Input: Draft markdown from Writer Agent  
      - Output: Final polished markdown plus metadata (meta description, tags)  
    - *Edge cases:* Over-optimization, under-optimization, plagiarism detection false positives

---

#### 2.3 Content Routing

- **Overview:**  
  Switch node routes the final polished content based on content type to either blog or newsletter publishing workflows.

- **Nodes Involved:**  
  - Route by Content Type (Switch)  

- **Node Details:**

  - **Route by Content Type**  
    - *Type:* Switch node  
    - *Role:* Routes workflow execution path  
    - *Configuration:*  
      - Condition on content type field from previous node output  
      - Routes: Blog path or Newsletter path  
    - *Edge cases:* Undefined or unexpected content type values causing routing failure

---

#### 2.4 Publishing and Saving

- **Overview:**  
  Formats final content suitably for blogs or newsletters, saves to respective storage, and sends notifications or exports.

- **Nodes Involved:**  
  - Blog Publisher Agent  
  - Newsletter Publisher Agent  
  - Save Blog to Airtable  
  - Save Newsletter to Google Sheets  

- **Node Details:**

  - **Blog Publisher Agent**  
    - *Type:* LangChain AI Agent  
    - *Role:* Formats content for Airtable, extracts title, markdown body, keywords, meta description, and generates DALL-E image prompt  
    - *Input:* Edited markdown + metadata from Editor & SEO Agent  
    - *Output:* JSON with structured blog data and image prompt  
    - *Edge cases:* Missing metadata, inconsistent formatting

  - **Newsletter Publisher Agent**  
    - *Type:* LangChain AI Agent  
    - *Role:* Formats content for email newsletters with simpler HTML and plain text versions, adds CTAs and section breaks  
    - *Input:* Edited markdown + metadata from Editor & SEO Agent  
    - *Output:* JSON with newsletter title, HTML content, plain text content, preview text  
    - *Edge cases:* HTML rendering issues, plain text conversion errors

  - **Save Blog to Airtable**  
    - *Type:* Airtable node  
    - *Role:* Saves blog data including title, content, keywords, meta description, and created date  
    - *Configuration:*  
      - Requires Airtable base ID and table ID (replace placeholders)  
      - Maps JSON output fields from Blog Publisher Agent  
      - Uses Airtable Personal Access Token credentials  
    - *Edge cases:* Credential errors, base/table misconfiguration, rate limits

  - **Save Newsletter to Google Sheets**  
    - *Type:* Google Sheets node  
    - *Role:* Appends newsletter data to a Google Sheet for distribution or record-keeping  
    - *Configuration:*  
      - Requires Google Sheet document ID and sheet name (replace placeholders)  
      - Maps JSON output fields from Newsletter Publisher Agent  
      - Uses Google Sheets OAuth2 credentials  
    - *Edge cases:* Credential expiration, API limits, document access issues

---

#### 2.5 Image Generation

- **Overview:**  
  Generates a featured image for blogs using the DALL-E model from OpenAI based on the Blog Publisher Agent's image prompt, converts image to file format, then saves it with the blog.

- **Nodes Involved:**  
  - Generate Featured Image (DALL-E)  
  - Convert Image to File

- **Node Details:**

  - **Generate Featured Image (DALL-E)**  
    - *Type:* HTTP Request node  
    - *Role:* Calls OpenAI API to generate an image from prompt  
    - *Configuration:*  
      - URL: OpenAI image generation endpoint  
      - Method: POST  
      - Body: JSON with model "dall-e-3", prompt from Blog Publisher Agent, size 1792x1024, 1 image, base64 JSON response  
      - Authentication: OpenAI API key  
    - *Edge cases:* API key invalid, quota exceeded, response errors, prompt issues

  - **Convert Image to File**  
    - *Type:* Convert to File node  
    - *Role:* Converts base64 encoded image data to binary file format for Airtable upload  
    - *Configuration:*  
      - Filename: timestamped PNG (e.g., "blog-featured-2024-06-01-153000.png")  
      - MIME type: image/png  
      - Source property: base64 JSON from previous node  
    - *Output:* Binary file attachment for Airtable  
    - *Edge cases:* Conversion failures, malformed base64 data

---

#### 2.6 Credential and Model Management

- **Overview:**  
  Manages credentials for OpenRouter, Google Gemini, OpenAI, Airtable, and Google Sheets integrations, ensuring secure API access and proper model usage.

- **Nodes Involved:**  
  - OpenRouter Chat Model (Grok)  
  - Google Gemini Chat Model  
  - Generate Featured Image (DALL-E)  
  - Save Blog to Airtable  
  - Save Newsletter to Google Sheets

- **Node Details:**

  - **Google Gemini Chat Model**  
    - *Type:* LangChain AI Agent  
    - *Role:* Alternative AI language model integration (linked as AI languageModel input to Blog and Newsletter Publisher Agents)  
    - *Credentials:* Google Palm API key  
    - *Note:* Supports model swapping/customization  
    - *Edge cases:* Credential issues, API limits

---

### 3. Summary Table

| Node Name                   | Node Type                       | Functional Role                          | Input Node(s)                | Output Node(s)                          | Sticky Note                                                                                                                                                                                                                                                                    |
|-----------------------------|--------------------------------|----------------------------------------|-----------------------------|----------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On Form Submission          | Form Trigger                   | Entry point, captures content type/topic | -                           | Research Agent                         | ## 📱 Step 1: Form Input - Triggered by user form submission with content type and topic. Can be replaced by other triggers like Schedule or Webhook.                                                                                                                         |
| Research Agent             | LangChain AI Agent             | Gathers research data (facts, titles)  | On Form Submission           | Outline Agent                         | ## 🔬 Step 2: AI Agent Pipeline - Research → Outline → Write → Edit agents working sequentially with OpenRouter Grok model.                                                                                                                                                    |
| Outline Agent              | LangChain AI Agent             | Creates structured content outline      | Research Agent               | Writer Agent                         | Same as above                                                                                                                                                                                                                                                                   |
| Writer Agent               | LangChain AI Agent             | Writes full markdown draft               | Outline Agent                | Editor & SEO Agent                   | Same as above                                                                                                                                                                                                                                                                   |
| Editor & SEO Agent         | LangChain AI Agent             | Polishes and SEO-optimizes draft        | Writer Agent                 | Route by Content Type                | Same as above                                                                                                                                                                                                                                                                   |
| Route by Content Type      | Switch                        | Routes workflow by content type          | Editor & SEO Agent           | Blog Publisher Agent, Newsletter Publisher Agent | ## 🔀 Step 3: Content Router - Routes based on blog or newsletter paths with different formatting and outputs.                                                                                                                                                                     |
| Blog Publisher Agent       | LangChain AI Agent             | Formats blog content + generates image prompt | Route by Content Type (blog) | Generate Featured Image (DALL-E)     | ## 💾 Step 4: Save & Publish - Blog output includes image generation and Airtable saving. Newsletter output handled separately.                                                                                                                                                    |
| Generate Featured Image (DALL-E) | HTTP Request                | Calls OpenAI API to generate blog image  | Blog Publisher Agent         | Convert Image to File                | Same as above                                                                                                                                                                                                                                                                   |
| Convert Image to File      | Convert To File                | Converts base64 image to binary file     | Generate Featured Image (DALL-E) | Save Blog to Airtable               | Same as above                                                                                                                                                                                                                                                                   |
| Save Blog to Airtable      | Airtable                      | Saves blog and metadata to Airtable      | Convert Image to File        | -                                  | Same as above                                                                                                                                                                                                                                                                   |
| Newsletter Publisher Agent | LangChain AI Agent             | Formats newsletter content for email     | Route by Content Type (newsletter) | Save Newsletter to Google Sheets  | Same as above                                                                                                                                                                                                                                                                   |
| Save Newsletter to Google Sheets | Google Sheets                | Saves newsletter data for distribution   | Newsletter Publisher Agent   | -                                  | Same as above                                                                                                                                                                                                                                                                   |
| OpenRouter Chat Model (Grok) | LangChain LM Chat Model       | Provides AI model for Research, Outline, Writer, Editor agents | -                           | Research, Outline, Writer, Editor Agents | See Step 2 note                                                                                                                                                                                                                                                                  |
| Google Gemini Chat Model   | LangChain LM Chat Model       | Optional AI model for Publisher agents   | -                           | Blog Publisher Agent, Newsletter Publisher Agent | Credential for Google Gemini integration                                                                                                                                                                                                                                      |
| Sticky Note - Main         | Sticky Note                   | Workflow purpose, instructions, overview | -                           | -                                  | ## 📝 SEO Blog & Newsletter Generator - Multi-Agent AI Content Creation System overview with requirements and setup instructions.                                                                                                                                                 |
| Sticky Note - Step 1       | Sticky Note                   | Explains input form trigger              | -                           | -                                  | Same as On Form Submission note                                                                                                                                                                                                                                                |
| Sticky Note - Step 2       | Sticky Note                   | Explains multi-agent AI pipeline         | -                           | -                                  | Same as AI Agent Pipeline note                                                                                                                                                                                                                                                 |
| Sticky Note - Step 3       | Sticky Note                   | Explains content routing by type         | -                           | -                                  | Same as Content Router note                                                                                                                                                                                                                                                    |
| Sticky Note - Step 4       | Sticky Note                   | Explains save & publish process           | -                           | -                                  | Same as Save & Publish note                                                                                                                                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Form Trigger** node named "On Form Submission".
   - Configure a form titled "SEO Blog & Newsletter Generator" with:
     - Dropdown field "Content Type" (options: Blog, Newsletter) - required.
     - Textarea field "Topic" - required.
   - This node will trigger the workflow and provide initial data.

2. **Add the OpenRouter Language Model Node:**
   - Add a **LangChain LM Chat OpenRouter** node named "OpenRouter Chat Model (Grok)".
   - Set model to `agentica-org/deepcoder-14b-preview:free`.
   - Set options: max tokens = 4000, temperature = 0.7.
   - Assign the OpenRouter API credentials.

3. **Add Research Agent:**
   - Add a **LangChain AI Agent** named "Research Agent".
   - Set prompt to include:
     ```
     Content Type: {{ $json['Content Type'] }}
     Topic: {{ $json.Topic }}

     You are the Research Agent...
     ```
     (Use the detailed prompt from the workflow reflecting research tasks and JSON output.)
   - Connect "On Form Submission" main output to "Research Agent" main input.
   - Set "OpenRouter Chat Model (Grok)" as the AI language model for this node.

4. **Add Outline Agent:**
   - Add a **LangChain AI Agent** named "Outline Agent".
   - Set prompt to:
     ```
     ={{ $json.output }}

     You are the Outline Agent...
     ```
     (Use detailed prompt for creating structured outlines with headings and word counts.)
   - Connect "Research Agent" main output to "Outline Agent" main input.
   - Use "OpenRouter Chat Model (Grok)" as AI model.

5. **Add Writer Agent:**
   - Add a **LangChain AI Agent** named "Writer Agent".
   - Set prompt to:
     ```
     ={{ $json.output }}

     You are the Writer Agent...
     ```
     (Use detailed prompt on expanding outlines into markdown drafts with citations.)
   - Connect "Outline Agent" main output to "Writer Agent" main input.
   - Use "OpenRouter Chat Model (Grok)" as AI model.

6. **Add Editor & SEO Agent:**
   - Add a **LangChain AI Agent** named "Editor & SEO Agent".
   - Set prompt to:
     ```
     ={{ $json.output }}

     You are the Editor & SEO Agent...
     ```
     (Use detailed prompt for grammar polishing, SEO optimization, metadata extraction.)
   - Connect "Writer Agent" main output to "Editor & SEO Agent" main input.
   - Use "OpenRouter Chat Model (Grok)" as AI model.

7. **Add Content Routing Switch Node:**
   - Add a **Switch** node named "Route by Content Type".
   - Configure condition to check content type from Editor & SEO Agent output.
   - Create two outputs: one for "Blog" and one for "Newsletter".
   - Connect "Editor & SEO Agent" main output to this switch node.

8. **Add Blog Publisher Agent:**
   - Add a **LangChain AI Agent** named "Blog Publisher Agent".
   - Prompt:
     ```
     ={{ $json.output }}

     You are the Blog Publisher Agent...
     ```
     (Prompt extracts blog title, content, keywords, meta description, image prompt.)
   - Connect "Route by Content Type" blog output to this node.
   - Optionally, connect Google Gemini Chat Model as an alternative AI model.

9. **Add Generate Featured Image (DALL-E):**
   - Add **HTTP Request** node named "Generate Featured Image (DALL-E)".
   - Configure POST request to `https://api.openai.com/v1/images/generations`.
   - Set JSON body with model `dall-e-3`, prompt from Blog Publisher Agent's image prompt, size `1792x1024`, n=1.
   - Use OpenAI credentials.
   - Connect "Blog Publisher Agent" main output to this node.

10. **Add Convert Image to File Node:**
    - Add **Convert To File** node named "Convert Image to File".
    - Configure to convert base64 JSON (`data[0].b64_json`) to binary PNG file.
    - Filename pattern: `blog-featured-{{ $now.toFormat('yyyy-MM-dd-HHmmss') }}.png`.
    - Connect "Generate Featured Image (DALL-E)" main output to this node.

11. **Add Save Blog to Airtable Node:**
    - Add **Airtable** node named "Save Blog to Airtable".
    - Configure with your Airtable base ID and table ID.
    - Map fields from Blog Publisher Agent output (title, content, keywords, meta description), add status "Draft", and set created date.
    - Connect "Convert Image to File" main output to this node.
    - Assign Airtable Personal Access Token credentials.

12. **Add Newsletter Publisher Agent:**
    - Add **LangChain AI Agent** named "Newsletter Publisher Agent".
    - Prompt:
      ```
      ={{ $json.output }}

      You are the Newsletter Publisher Agent...
      ```
      (Format content for email with HTML and plain text, add CTAs and section breaks.)
    - Connect "Route by Content Type" newsletter output to this node.
    - Optionally, connect Google Gemini Chat Model as AI model.

13. **Add Save Newsletter to Google Sheets Node:**
    - Add **Google Sheets** node named "Save Newsletter to Google Sheets".
    - Configure with your Google Sheet document ID and sheet name.
    - Map title, status "Ready to Send", HTML content, plain text content, preview text, and created date.
    - Connect "Newsletter Publisher Agent" main output to this node.
    - Assign Google Sheets OAuth2 credentials.

14. **(Optional) Add Google Gemini Chat Model Node:**
    - Add **LangChain LM Chat Google Gemini** node.
    - Configure credentials with Google Palm API key.
    - Connect as AI language model input to Blog Publisher Agent and Newsletter Publisher Agent for alternative model usage.

15. **Add Sticky Notes for Documentation:**
    - Add sticky notes with workflow overview and step explanations at relevant positions for clarity.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Workflow automates SEO blog and newsletter generation using multi-agent AI with OpenRouter Grok, Google Gemini, and OpenAI DALL-E integration. Requires setting up API credentials and replacing placeholder IDs for Airtable and Google Sheets. | Main sticky note in workflow describes overview and requirements. |
| The workflow automatically routes content generation based on user input (Blog or Newsletter), enabling flexible publishing paths. | Sticky notes on steps 1-4 detail workflow stages and routing logic. |
| Image generation uses DALL-E 3 from OpenAI with base64 JSON response converted to a PNG file and uploaded to Airtable. | Node "Generate Featured Image (DALL-E)" and "Convert Image to File". |
| Consider adding error handling nodes or notifications (e.g., Telegram bot) for better monitoring and resilience. | Workflow mentions optional Telegram bot for notifications in overview. |
| Customization options include swapping AI models, adjusting prompts for tone/style, adding fact-checking, or integrating with publishing platforms like WordPress or Medium. | Workflow overview sticky note and AI agent prompt flexibility. |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected material. All data processed is legal and public.
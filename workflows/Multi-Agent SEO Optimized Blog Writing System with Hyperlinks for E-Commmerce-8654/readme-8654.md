Multi-Agent SEO Optimized Blog Writing System with Hyperlinks for E-Commmerce

https://n8nworkflows.xyz/workflows/multi-agent-seo-optimized-blog-writing-system-with-hyperlinks-for-e-commmerce-8654


# Multi-Agent SEO Optimized Blog Writing System with Hyperlinks for E-Commmerce

### 1. Workflow Overview

This workflow, titled **"Multi-Agent SEO Optimized Blog Writing System with Hyperlinks for E-Commerce,"** is designed to automate the creation of SEO-optimized blog articles tailored for an e-commerce website specializing in bucket hats. It leverages multi-agent AI models integrated with data from Airtable, Google Drive, and external SERP analysis tools to produce high-quality, semantically rich content with internal hyperlinks.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Data Fetching**  
  Captures input via webhook, retrieves article data from Airtable, and sets initial variables.

- **1.2 SERP Analysis and Writing Guidelines Generation**  
  Uses Tavily SERP API and Anthropic AI to analyze search intent, competitor content, and generate dynamic writing guidelines including style, tone, target audience, and hidden insights.

- **1.3 Title Refinement and Keyword Field Updates**  
  Refines the working title using AI agents, updates Airtable records with the refined title and SEO metadata.

- **1.4 Internal Link Selection**  
  Parses the website sitemap to select relevant internal URLs for hyperlinking within the article.

- **1.5 Content Generation Pipeline**  
  Sequential AI agents generate key takeaways, introduction, detailed blog outline, main body, and conclusion, all SEO-optimized and formatted in HTML for Shopify.

- **1.6 Article Assembly, Final Editing, and Storage**  
  Assembles all content parts into a single article, performs final editorial improvements, creates and updates Google Docs files, adds meta description and image prompts, and updates Airtable with final document URLs.

- **1.7 Workflow Setup and Maintenance Notes**  
  Includes sticky notes for human intervention points, credentials setup, and instructions for adapting the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Fetching

**Overview:**  
This block receives webhook triggers with Airtable record IDs, fetches article metadata from Airtable, and prepares initial data for downstream processing.

**Nodes Involved:**  
- Webhook  
- Airtable Get Article Data  
- Set Airtable Fields for Agents  
- Set Airtable Fields  

**Node Details:**

- **Webhook**  
  - Type: HTTP webhook listener (POST)  
  - Config: Listens on a unique webhook path; triggers workflow on incoming requests containing `recordID`.  
  - Input: External POST with JSON body including Airtable record ID.  
  - Output: Passes request data downstream.  
  - Edge cases: Invalid/missing recordID leads to failure in fetching Airtable data; no retry logic visible.

- **Airtable Get Article Data**  
  - Type: Airtable node  
  - Config: Retrieves a single Airtable record using record ID from webhook. Targets base “KW Research The Bucket Hat” and “Article Writer” table.  
  - Input: Record ID from webhook.  
  - Output: Full record JSON including Title, Description, Keyword, etc.  
  - Edge cases: API rate limits, invalid credentials, missing record cause errors.

- **Set Airtable Fields for Agents**  
  - Type: Set node  
  - Config: Extracts key fields (ID, Title, Description, Keyword, URL list) from Airtable data for use by AI agents.  
  - Input: Airtable record JSON.  
  - Output: JSON with simplified fields for agents.  
  - Edge cases: Missing fields or malformed data can cause downstream prompt errors.

- **Set Airtable Fields**  
  - Type: Set node  
  - Config: Sets various Airtable base/table IDs and parameters for later nodes, including primary keywords and record IDs.  
  - Input: Webhook data.  
  - Output: Structured parameters for Airtable and other integrations.  
  - Edge cases: Misconfiguration leads to lookup failures.

---

#### 2.2 SERP Analysis and Writing Guidelines Generation

**Overview:**  
This block performs competitive analysis by querying Tavily SERP API for keyword results, then uses Anthropic AI to analyze these results and create dynamic writing guidelines including style, tone, intent, and hidden insights.

**Nodes Involved:**  
- Tavily search results  
- SERPs, Writing, KWs, Insights (Anthropic Agent)  
- Code in JavaScript  
- Set KWs and Insights fields  
- Update Article Writer table  

**Node Details:**

- **Tavily search results**  
  - Type: HTTP request (Langchain tool)  
  - Config: Sends POST request with API key and query keyword to Tavily API to retrieve SERP data.  
  - Input: Keyword from Airtable fields.  
  - Output: SERP JSON results.  
  - Edge cases: API key invalid, network errors, empty results.

- **SERPs, Writing, KWs, Insights**  
  - Type: Anthropic AI Agent  
  - Config: Prompted with Title, Description, SERP data, and keyword to output a JSON with search intent, writing style/tone, hidden insights, target audience, article goal, semantic analysis, and keyword types.  
  - Input: SERP results and Airtable fields.  
  - Output: Structured JSON with writing guidelines and insights.  
  - Edge cases: Model timeouts, prompt failures, incomplete JSON output.

- **Code in JavaScript**  
  - Type: Code node  
  - Config: Parses the JSON output from Anthropic agent to flatten JSON fields into usable workflow variables.  
  - Input: Anthropic JSON string in output.  
  - Output: Parsed JSON merged into workflow data.  
  - Edge cases: Parsing errors if AI output is malformed.

- **Set KWs and Insights fields**  
  - Type: Set node  
  - Config: Assigns parsed writing style, tone, intent, hidden insight, audience, goals, semantic analysis, and keywords into workflow variables for further use.  
  - Input: Parsed JSON from code node.  
  - Output: Structured fields for title refinement and content creation.  
  - Edge cases: Missing keys cause empty or incorrect data downstream.

- **Update Article Writer table**  
  - Type: Airtable update  
  - Config: Updates the original Airtable record with generated writing style, tone, search intent, hidden insight, goals, target audience, semantic analysis, and keywords for audit and tracking.  
  - Input: Fields from previous set node.  
  - Output: Airtable record updated.  
  - Edge cases: Airtable API errors, permission issues.

---

#### 2.3 Title Refinement and Keyword Field Updates

**Overview:**  
This block refines the working title into an SEO-optimized final title using a Langchain AI agent and updates Airtable accordingly.

**Nodes Involved:**  
- Refine the Title (OpenAI Agent)  
- Sets New Title Field (Set)  
- Update Article Title (Airtable update)  

**Node Details:**

- **Refine the Title**  
  - Type: Langchain AI Agent (OpenAI)  
  - Config: Uses a detailed prompt with all context fields (keywords, search intent, semantic data, style, tone, goals) to produce a final SEO-friendly, engaging title in Dutch.  
  - Output: Plain text with only the refined title (no extra formatting).  
  - Edge cases: Rate limits, prompt failure, token limits.

- **Sets New Title Field**  
  - Type: Set node  
  - Config: Stores the AI-generated refined title in a variable `new_title`.  
  - Input: AI output from previous node.  
  - Output: Structured for Airtable update.  
  - Edge cases: Empty or malformed AI output.

- **Update Article Title**  
  - Type: Airtable update  
  - Config: Updates the Airtable record’s Final Title field along with writing style, tone, keywords, hidden insights, search intent, and goals.  
  - Input: New title and metadata fields.  
  - Output: Airtable record updated with final title.  
  - Edge cases: Airtable API errors.

---

#### 2.4 Internal Link Selection

**Overview:**  
This block fetches the website’s sitemap, parses URLs, and uses AI to select the most relevant internal links for hyperlinking within the article content.

**Nodes Involved:**  
- HTTP Request (to sitemap.xml)  
- XML (parse sitemap)  
- Split Out (extract sitemap entries)  
- HTTP Request1 (fetch individual sitemap URLs)  
- XML1 (parse individual sitemap XML)  
- Aggregate (aggregate URLs)  
- Set list URLs (Set node)  
- Set Airtable Fields for Agent (Set node)  
- URLs Selection (Langchain AI Agent)  
- Set best urls (Set node)  

**Node Details:**

- **HTTP Request (to sitemap.xml)**  
  - Type: HTTP request  
  - Config: Downloads sitemap XML from the bucket hat website root.  
  - Output: Raw XML sitemap.  
  - Edge cases: Network errors, sitemap unavailable.

- **XML / XML1**  
  - Type: XML parser nodes  
  - Config: Parse sitemap XML and nested sitemap indexes to extract URLs.  
  - Output: Parsed JSON with URLs.  
  - Edge cases: Malformed XML, unexpected sitemap format.

- **Split Out**  
  - Type: Split node  
  - Config: Splits sitemap index into individual sitemap entries for separate fetching.  
  - Output: Each sitemap URL for further HTTP requests.  
  - Edge cases: Empty sitemap index.

- **HTTP Request1**  
  - Type: HTTP request  
  - Config: Fetches each sitemap XML URL to extract page URLs.  
  - Output: Sitemap data per URL segment.  
  - Edge cases: Network errors.

- **Aggregate**  
  - Type: Aggregate node  
  - Config: Aggregates all extracted URLs into a single JSON array.  
  - Output: Combined list of site URLs.  
  - Edge cases: No URLs found.

- **Set list URLs**  
  - Type: Set node  
  - Config: Extracts URLs from aggregated data into a string list for AI processing.  
  - Output: URL list string.  
  - Edge cases: Empty or malformed URL list.

- **Set Airtable Fields for Agent**  
  - Type: Set node  
  - Config: Prepares fields including title, description, keyword, and URL list for AI link selection agent.  
  - Output: Input for URLs Selection agent.  
  - Edge cases: Missing or empty fields.

- **URLs Selection**  
  - Type: Langchain AI Agent (OpenAI)  
  - Config: Receives blog title, description, keyword, and complete URL list, then selects 8-13 relevant internal URLs prioritizing collections and some product pages according to defined rules. Outputs JSON array of URLs.  
  - Output: JSON array of selected URLs.  
  - Edge cases: AI returns incomplete or invalid JSON, rate limits.

- **Set best urls**  
  - Type: Set node  
  - Config: Stores chosen URLs for use in hyperlink insertion in content generation.  
  - Output: Workflow variable `best matching urls`.  
  - Edge cases: Empty output.

---

#### 2.5 Content Generation Pipeline

**Overview:**  
This block orchestrates multiple AI agents to generate key takeaways, introduction, detailed outline, main content body, and conclusion of the blog article. All content is SEO-optimized, written in Dutch, and formatted in clean HTML for Shopify.

**Nodes Involved:**  
- Key Takeaways AI Agent  
- Set Key Takeaways  
- Introduction Agent  
- Set Introduction Field  
- Outline Agent  
- Set Outline Fields  
- Main Body Prompt Writer  
- Content Writer Agent  
- Edit Fields1  
- AI Agent Conclusion Writer  
- Set Conclusion  

**Node Details:**

- **Key Takeaways AI Agent**  
  - Type: Langchain AI Agent (OpenAI)  
  - Config: Generates an SEO-optimized summary with intro paragraph, bullet points with bold action-driven headings, and outro paragraph in HTML. Incorporates hidden insights and semantic topics. Output is a concise but informative summary.  
  - Edge cases: Model limits, incomplete output.

- **Set Key Takeaways**  
  - Type: Set node  
  - Config: Stores AI-generated key takeaways for further use.  
  - Edge cases: Empty input.

- **Introduction Agent**  
  - Type: Langchain AI Agent (OpenAI)  
  - Config: Writes an engaging, keyword-optimized introduction (400-450 words) in HTML with natural flow and SEO best practices.  
  - Edge cases: Token limits, prompt errors.

- **Set Introduction Field**  
  - Type: Set node  
  - Config: Stores introduction text.  
  - Edge cases: Empty input.

- **Outline Agent**  
  - Type: Langchain AI Agent (OpenAI)  
  - Config: Generates a detailed SEO-friendly blog outline in HTML excluding intro and conclusion, using all semantic and keyword data.  
  - Edge cases: Long prompts may hit token limits.

- **Set Outline Fields**  
  - Type: Set node  
  - Config: Stores outline text for next step.  
  - Edge cases: Empty input.

- **Main Body Prompt Writer**  
  - Type: Langchain AI Agent (OpenAI)  
  - Config: Creates a structured prompt for the Content Writer Agent based on all SEO data and outline.  
  - Edge cases: Prompt correctness critical for content quality.

- **Content Writer Agent**  
  - Type: Langchain AI Agent (OpenAI)  
  - Config: Produces the main article body in well-structured HTML, adhering strictly to the prompt and SEO requirements. Word count between 1800-2400 words.  
  - Edge cases: Output length truncation, hallucinations.

- **Edit Fields1**  
  - Type: Set node  
  - Config: Stores main body output as `main body`.  
  - Edge cases: Empty input.

- **AI Agent Conclusion Writer**  
  - Type: Langchain AI Agent (OpenAI)  
  - Config: Writes a strong, SEO-optimized conclusion (around 500 words) summarizing key points, reinforcing value, and ending with a compelling thought.  
  - Edge cases: Prompt or token limits.

- **Set Conclusion**  
  - Type: Set node  
  - Config: Stores conclusion text.  
  - Edge cases: Empty input.

---

#### 2.6 Article Assembly, Final Editing, and Storage

**Overview:**  
This block assembles all article parts, performs final SEO and editorial refinements, creates and updates Google Docs files, inserts meta descriptions and AI-generated image prompts, and updates Airtable with final document URLs.

**Nodes Involved:**  
- Final Article  
- Create Article Folder (Google Drive)  
- Create Doc Filename is title (Google Docs)  
- Add Final Article (Google Docs)  
- OpenAI Meta (Meta description generation)  
- Add Meta Description (Google Docs)  
- OpenAI Image Prompt1 (Image prompt generation)  
- Add Image Prompt1 (Google Docs)  
- Google Drive (Sharing the folder)  
- Edit Fields (Set Google Doc URL)  
- Airtable (Update with Google Doc URL)  
- Article Assembly Agent1 (Markdown assembler)  
- Final Edit Agent1 (Final editor AI agent)  

**Node Details:**

- **Final Article**  
  - Type: Set node  
  - Config: Assembles introduction, key takeaways, outline, main body, and conclusion into one Markdown string.  
  - Edge cases: Missing parts cause incomplete article.

- **Create Article Folder**  
  - Type: Google Drive node  
  - Config: Creates a new folder named after the article’s final title inside a master folder.  
  - Edge cases: Permission errors, folder name conflicts.

- **Create Doc Filename is title**  
  - Type: Google Docs node  
  - Config: Creates a new Google Doc with the article title in the created folder.  
  - Edge cases: API errors.

- **Add Final Article**  
  - Type: Google Docs update  
  - Config: Inserts full assembled article Markdown content into Google Doc.  
  - Edge cases: API limits.

- **OpenAI Meta**  
  - Type: OpenAI node  
  - Config: Generates an SEO-optimized meta description (150-160 chars) based on full article content.  
  - Edge cases: Output length, prompt failures.

- **Add Meta Description**  
  - Type: Google Docs update  
  - Config: Inserts meta description at the beginning or designated place in Google Doc.  
  - Edge cases: API errors.

- **OpenAI Image Prompt1**  
  - Type: OpenAI node  
  - Config: Creates a detailed AI image prompt for a featured image representing the article theme.  
  - Edge cases: Prompt quality.

- **Add Image Prompt1**  
  - Type: Google Docs update  
  - Config: Inserts image prompt text into Google Doc for editorial use.  
  - Edge cases: API errors.

- **Google Drive (Sharing)**  
  - Type: Google Drive permissions update  
  - Config: Sets sharing permissions (writer, anyone with link).  
  - Edge cases: Permissions errors.

- **Edit Fields**  
  - Type: Set node  
  - Config: Generates a Google Docs URL for Airtable linking.  
  - Edge cases: Missing document ID.

- **Airtable Update**  
  - Type: Airtable update  
  - Config: Updates Airtable record with Google Doc URL for reference.  
  - Edge cases: API errors.

- **Article Assembly Agent1**  
  - Type: Langchain AI Agent (OpenAI)  
  - Config: Assembles separate article components into final Markdown post with correct heading levels and formatting for CMS input.  
  - Edge cases: Formatting errors, incomplete data.

- **Final Edit Agent1**  
  - Type: Langchain AI Agent (OpenAI)  
  - Config: Performs SEO and editorial final polish to improve flow, depth, and relevance, targeting near-perfect quality (9.5+/10).  
  - Edge cases: Over-expansion or verbosity.

---

#### 2.7 Workflow Setup and Maintenance Notes

**Overview:**  
Sticky notes and configuration nodes provide human guidelines for setup, API key insertion, prompt tuning, and operational instructions.

**Nodes Involved:**  
- Multiple Sticky Note nodes scattered throughout the workflow.

**Node Details:**

- Sticky notes provide instructions such as:  
  - "Good Opportunity for Human in the loop - Create 5 titles and have the human pick one."  
  - "Add your Tavily API key."  
  - "Dynamic Writing Guidelines and Hidden Insights based on competitor data."  
  - "Revisit prompts to ensure applicability across different topics."  
  - "Setup instructions including Airtable base copying and webhook URL insertion."  

- These notes do not affect execution but are crucial for maintainability and human supervision.

---

### 3. Summary Table

| Node Name                   | Node Type                                    | Functional Role                                    | Input Node(s)                  | Output Node(s)                              | Sticky Note                                                                                                           |
|-----------------------------|----------------------------------------------|---------------------------------------------------|-------------------------------|---------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Webhook                     | n8n-nodes-base.webhook                        | Receive external trigger with recordID            |                               | Airtable Get Article Data                    |                                                                                                                       |
| Airtable Get Article Data    | n8n-nodes-base.airtable                       | Fetch article metadata from Airtable              | Webhook                       | Set Airtable Fields for Agents               |                                                                                                                       |
| Set Airtable Fields for Agents | n8n-nodes-base.set                          | Prepare fields for AI agents                       | Airtable Get Article Data      | SERPs, Writing, KWs, Insights                |                                                                                                                       |
| Set Airtable Fields          | n8n-nodes-base.set                            | Set config IDs and parameters                      | Webhook                       | Airtable Get Article Data                    |                                                                                                                       |
| Tavily search results        | @n8n/n8n-nodes-langchain.toolHttpRequest    | Retrieve SERP data for keyword                     | Set Airtable Fields for Agents | SERPs, Writing, KWs, Insights                | Add your Tavily API key                                                                                               |
| SERPs, Writing, KWs, Insights| @n8n/n8n-nodes-langchain.lmChatAnthropic    | Analyze SERP & generate writing guidelines         | Tavily search results          | Code in JavaScript                            | Dynamic Writing Guidelines and Hidden Insights                                                                        |
| Code in JavaScript           | n8n-nodes-base.code                           | Parse AI JSON output                               | SERPs, Writing, KWs, Insights | Set KWs and Insights fields                   |                                                                                                                       |
| Set KWs and Insights fields  | n8n-nodes-base.set                            | Assign parsed fields for downstream use           | Code in JavaScript             | Update Article Writer table                   |                                                                                                                       |
| Update Article Writer table  | n8n-nodes-base.airtable                       | Update Airtable record with guidelines             | Set KWs and Insights fields    | Refine the Title                              |                                                                                                                       |
| Refine the Title             | @n8n/n8n-nodes-langchain.agent (OpenAI)      | Generate SEO-optimized final title                 | Update Article Writer table    | Sets New Title Field                          | Good Opportunity for Human in the loop                                                                                |
| Sets New Title Field         | n8n-nodes-base.set                            | Store refined title                                | Refine the Title              | Update Article Title                          | Refine the Working Title - Set New Title in Airtable                                                                  |
| Update Article Title         | n8n-nodes-base.airtable                       | Update final title and metadata in Airtable       | Sets New Title Field           | Key Takeaways AI Agent                        |                                                                                                                       |
| HTTP Request (sitemap.xml)   | n8n-nodes-base.httpRequest                    | Download website sitemap                           |                               | XML                                           |                                                                                                                       |
| XML                         | n8n-nodes-base.xml                            | Parse sitemap XML                                 | HTTP Request                  | Split Out                                     |                                                                                                                       |
| Split Out                   | n8n-nodes-base.splitOut                        | Split sitemap indexes into individual URLs       | XML                           | HTTP Request1                                 |                                                                                                                       |
| HTTP Request1               | n8n-nodes-base.httpRequest                    | Fetch each sitemap XML                             | Split Out                    | XML1                                          |                                                                                                                       |
| XML1                        | n8n-nodes-base.xml                            | Parse individual sitemap XML                       | HTTP Request1                | Aggregate                                     |                                                                                                                       |
| Aggregate                   | n8n-nodes-base.aggregate                       | Aggregate all URLs from sitemaps                   | XML1                         | Set list URLs                                 |                                                                                                                       |
| Set list URLs               | n8n-nodes-base.set                            | Prepare URL list string                            | Aggregate                    | Set Airtable Fields for Agent                 |                                                                                                                       |
| Set Airtable Fields for Agent| n8n-nodes-base.set                            | Prepare input for link selection AI                | Set list URLs                | URLs Selection                                |                                                                                                                       |
| URLs Selection              | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Select relevant internal links from URL list      | Set Airtable Fields for Agent  | Set best urls                                 |                                                                                                                       |
| Set best urls               | n8n-nodes-base.set                            | Store selected internal URLs                       | URLs Selection               | SERPs, Writing, KWs, Insights (used later)   |                                                                                                                       |
| Key Takeaways AI Agent      | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Generate key takeaways summary                      | Update Article Title          | Set Key Takeaways                             | Create Key Takeaways with Intro, bullet points, and Outro                                                             |
| Set Key Takeaways           | n8n-nodes-base.set                            | Store key takeaways                                | Key Takeaways AI Agent        | Introduction Agent                            |                                                                                                                       |
| Introduction Agent          | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Generate engaging introduction                      | Set Key Takeaways             | Set Introduction Field                        | Create Engaging Introduction                                                                                          |
| Set Introduction Field      | n8n-nodes-base.set                            | Store introduction text                            | Introduction Agent           | Outline Agent                                 |                                                                                                                       |
| Outline Agent               | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Generate detailed blog outline                      | Set Introduction Field        | Set Outline Fields                            | Create Outline                                                                                                        |
| Set Outline Fields          | n8n-nodes-base.set                            | Store outline text                                 | Outline Agent                | Main Body Prompt Writer                       |                                                                                                                       |
| Main Body Prompt Writer     | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Create prompt for main body content AI             | Set Outline Fields           | Content Writer Agent                          | Create Main Body Prompt                                                                                                |
| Content Writer Agent        | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Generate main article body in HTML                  | Main Body Prompt Writer      | Edit Fields1                                  | Write Main Body                                                                                                       |
| Edit Fields1                | n8n-nodes-base.set                            | Store main body content                            | Content Writer Agent         | AI Agent Conclusion Writer                    |                                                                                                                       |
| AI Agent Conclusion Writer  | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Write article conclusion                            | Edit Fields1                 | Set Conclusion                                | Write Conclusion                                                                                                      |
| Set Conclusion              | n8n-nodes-base.set                            | Store conclusion text                              | AI Agent Conclusion Writer   | Final Article                                 |                                                                                                                       |
| Final Article               | n8n-nodes-base.set                            | Assemble full article components                    | Set Conclusion               | Create Article Folder                         |                                                                                                                       |
| Create Article Folder       | n8n-nodes-base.googleDrive                     | Create Google Drive folder for article              | Final Article                | Create Doc Filename is title                  |                                                                                                                       |
| Create Doc Filename is title| n8n-nodes-base.googleDocs                       | Create Google Doc named after article title         | Create Article Folder        | Add Final Article                             |                                                                                                                       |
| Add Final Article           | n8n-nodes-base.googleDocs                       | Insert full article content into Google Doc         | Create Doc Filename is title | OpenAI Meta                                   |                                                                                                                       |
| OpenAI Meta                 | @n8n/n8n-nodes-langchain.openAi                | Generate SEO meta description                        | Add Final Article            | Add Meta Description                          |                                                                                                                       |
| Add Meta Description        | n8n-nodes-base.googleDocs                       | Insert meta description into Google Doc             | OpenAI Meta                 | OpenAI Image Prompt1                          |                                                                                                                       |
| OpenAI Image Prompt1        | @n8n/n8n-nodes-langchain.openAi                | Generate AI image prompt for featured image          | Add Meta Description         | Add Image Prompt1                             |                                                                                                                       |
| Add Image Prompt1           | n8n-nodes-base.googleDocs                       | Insert image prompt into Google Doc                   | OpenAI Image Prompt1         | Google Drive (Sharing)                        |                                                                                                                       |
| Google Drive                | n8n-nodes-base.googleDrive                      | Share Google Drive folder with appropriate permissions| Add Image Prompt1           | Edit Fields                                   |                                                                                                                       |
| Edit Fields                 | n8n-nodes-base.set                            | Prepare Google Doc URL for Airtable update           | Google Drive                | Airtable                                      |                                                                                                                       |
| Airtable                   | n8n-nodes-base.airtable                       | Update Airtable with Google Doc URL                   | Edit Fields                 |                                              |                                                                                                                       |
| Article Assembly Agent1     | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Assemble full article Markdown with sections          | Set Conclusion, Edit Fields1 | Final Edit Agent1                             | Assemble Entire Article                                                                                               |
| Final Edit Agent1           | @n8n/n8n-nodes-langchain.lmChatOpenAi          | Final SEO and editorial polish                         | Article Assembly Agent1      |                                              | Perform Final Editor and Quality Check                                                                                |
| Sticky Notes (various)      | n8n-nodes-base.stickyNote                      | Provide human-readable workflow instructions          |                             |                                              | Various notes about human in the loop, API keys, prompt tuning, etc.                                                  |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Setup Webhook Node**  
- Create a webhook node with POST method.  
- Assign a unique path (e.g., "9c054bb1-4446-4502-be98-ffd3c8ca1f2").  
- This node triggers the workflow on incoming requests containing `recordID`.

**Step 2: Fetch Article Data from Airtable**  
- Add an Airtable node configured to “Get” mode.  
- Configure credentials for Airtable Personal Access Token.  
- Use record ID from webhook JSON body to fetch article metadata from base “KW Research The Bucket Hat” and table “Article Writer.”

**Step 3: Prepare Initial Fields**  
- Add a Set node to extract and assign key Airtable fields (id, Title, Description, Keyword, URL list) for AI agents.

**Step 4: Configure Tavily SERP API Request**  
- Add HTTP Request node or use Langchain HTTP tool for Tavily.  
- Set POST method, include API key header, and send primary keyword from Airtable.  
- Configure authentication with HTTP header auth for Tavily API key.

**Step 5: Analyze SERP and Generate Writing Guidelines**  
- Add Anthropic AI agent node.  
- Configure prompt to analyze SERP data plus title, description, and keyword.  
- Output structured JSON with search intent, writing style, tone, hidden insight, audience, goals, semantic analysis, and keywords.

**Step 6: Parse AI JSON Output**  
- Add a Code node to parse JSON string output from Anthropic into individual workflow variables.

**Step 7: Store Writing Guidelines**  
- Add Set node to assign parsed fields for use by other AI agents.

**Step 8: Update Airtable with Writing Guidelines**  
- Add Airtable update node to write back style, tone, hidden insight, search intent, goals, and audience to original article record.

**Step 9: Refine Title Using OpenAI Agent**  
- Add Langchain OpenAI agent node with prompt to refine working title using all gathered data.  
- Output plain text refined title.

**Step 10: Store Refined Title**  
- Add Set node to capture refined title.

**Step 11: Update Airtable with Final Title**  
- Add Airtable update node to set final title and metadata fields.

**Step 12: Fetch Sitemap XML**  
- Use HTTP Request node to download website sitemap.xml.

**Step 13: Parse Sitemap XML and Extract URLs**  
- Use XML node to parse sitemap.  
- Use Split Out node to iterate sitemap indexes.  
- Fetch individual sitemap XMLs with HTTP Request node.  
- Parse them with XML node.  
- Aggregate all URLs into a single list.

**Step 14: Prepare URL list for AI**  
- Use Set node to format URLs as a string list.

**Step 15: Select Relevant Internal Links with AI**  
- Use OpenAI Langchain agent to receive blog title, description, keyword, and URL list.  
- Return JSON array of 8-13 relevant URLs (prioritize collections, some products).

**Step 16: Store Selected URLs**  
- Use Set node to store filtered URLs.

**Step 17: Generate Key Takeaways**  
- Use OpenAI agent with prompt to create key takeaways with intro and outro paragraphs in HTML.

**Step 18: Store Key Takeaways**  
- Set node to store key takeaways.

**Step 19: Generate Introduction**  
- Use OpenAI agent to write engaging article introduction in HTML.

**Step 20: Store Introduction**  
- Set node to store introduction.

**Step 21: Generate Article Outline**  
- Use OpenAI agent to create detailed blog outline in HTML excluding intro and conclusion.

**Step 22: Store Outline**  
- Set node to store outline.

**Step 23: Create Main Body Prompt**  
- Use OpenAI agent to craft a structured prompt for main article body based on all gathered data.

**Step 24: Generate Main Article Body**  
- Use OpenAI agent to write main content in clean HTML, following prompt and SEO guidelines.

**Step 25: Store Main Body**  
- Set node to store main body content.

**Step 26: Generate Conclusion**  
- Use OpenAI agent to write a strong, SEO-optimized conclusion in HTML.

**Step 27: Store Conclusion**  
- Set node to store conclusion.

**Step 28: Assemble Article Components**  
- Use OpenAI agent to combine key takeaways, introduction, outline, main body, and conclusion into a single Markdown document.

**Step 29: Final Editorial Polish**  
- Use OpenAI agent to perform final SEO and readability enhancements.

**Step 30: Create Google Drive Folder**  
- Use Google Drive node to create a folder named after the article’s final title in a designated parent folder.

**Step 31: Create Google Doc File**  
- Use Google Docs node to create a new document named after the article’s title inside the created folder.

**Step 32: Insert Full Article**  
- Use Google Docs update node to insert the assembled article content into the new Google Doc.

**Step 33: Generate and Insert Meta Description**  
- Use OpenAI node to create SEO meta description.  
- Insert meta description into Google Doc.

**Step 34: Generate and Insert Image Prompt**  
- Use OpenAI node to create AI image prompt for featured image.  
- Insert prompt into Google Doc.

**Step 35: Share Google Drive Folder**  
- Use Google Drive node to set sharing permissions for the folder to "anyone with link" as writer.

**Step 36: Update Airtable with Google Doc URL**  
- Use Set node to generate Google Doc URL.  
- Use Airtable update node to store this URL in the article record.

**Step 37: Configure Sticky Notes and Documentation**  
- Add sticky note nodes at key workflow points with instructions and setup notes for human operators.

**Step 38: Credential Setup**  
- Configure and connect credentials for Airtable, Tavily API, OpenAI, Anthropic, Google Drive, and Google Docs OAuth2.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Copy the Airtable base "KW Research Content Ideation" before use. Do not request access; use the copy button in Airtable UI.                                              | Airtable base setup instruction.                                                                                |
| Modify the Airtable automation script to call your n8n webhook URL with recordID parameter. Script example provided in notes.                                             | Setup webhook trigger from Airtable.                                                                             |
| Add your Tavily API key in the Tavily search results node for SERP data retrieval.                                                                                        | API key required for SERP analysis.                                                                              |
| Prompts are designed for Dutch language SEO content for The Bucket Hat e-commerce website.                                                                                | Localization and domain-specific prompt tuning.                                                                  |
| Use human-in-the-loop for title selection: create 5 title options and have a human pick the best one.                                                                     | Sticky note advice for editorial quality control.                                                                |
| Output blog content in clean HTML format suitable for Shopify direct pasting: use proper tags `<h2>`, `<h3>`, `<p>`, `<strong>`, `<ul><li>`, and `<a href="">`.          | Content formatting standard.                                                                                      |
| Internal link selection prioritizes collection pages and includes relevant product pages, always including homepage and main category pages.                             | SEO best practice for internal linking.                                                                           |
| AI agents produce content in Dutch, adhering to SEO and readability guidelines, with word count constraints per section.                                                  | Language and style consistency.                                                                                   |
| Google Drive folder and Google Docs creation require OAuth2 credentials with write access.                                                                                 | Credential configuration.                                                                                         |
| Final editorial agent expands and polishes article to near-perfect SEO and readability standards (rated 9.5+/10).                                                        | Quality assurance via AI.                                                                                         |
| Workflow is inactive by default; activate manually after configuring all credentials and nodes.                                                                           | Operational note.                                                                                                 |
| Workflow version and execution order use default settings but can be modified for advanced concurrency or version control.                                               | Workflow management.                                                                                              |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.
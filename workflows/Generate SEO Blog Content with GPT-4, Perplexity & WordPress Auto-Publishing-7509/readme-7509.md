Generate SEO Blog Content with GPT-4, Perplexity & WordPress Auto-Publishing

https://n8nworkflows.xyz/workflows/generate-seo-blog-content-with-gpt-4--perplexity---wordpress-auto-publishing-7509


# Generate SEO Blog Content with GPT-4, Perplexity & WordPress Auto-Publishing

### 1. Workflow Overview

This workflow automates the generation of SEO-optimized blog content using advanced AI models (OpenAI GPT-4, Claude 3.5, GPT-5 mini), semantic keyword analysis, and research data from Google Sheets and Perplexity. It then auto-publishes the content as draft posts on WordPress. The workflow targets digital marketers, content strategists, and SEO professionals aiming to streamline blog production at scale with AI assistance, semantic SEO, and internal/external linking.

The logical structure is divided into these main functional blocks:

- **1.1 Input Reception & Initialization:** Manual trigger and initial keyword data retrieval from Google Sheets.
- **1.2 Research & Semantic Analysis:** Enrichment of keyword context and competitor research using Perplexity and Google Sheets.
- **1.3 SEO Content Strategy Generation:** AI-driven semantic content structuring, keyword categorization, and hidden insight extraction.
- **1.4 Title & Key Takeaways Generation:** Crafting SEO-friendly blog titles and summarizing key article points.
- **1.5 Introduction, Outline & Prompt Construction:** Creating engaging introductions, article outlines, and detailed AI writing prompts.
- **1.6 Main Article Body Generation:** Producing the core blog content section-by-section following the outline.
- **1.7 Conclusion & Assembly:** Generating conclusions with FAQs and assembling all parts into a polished, linked Markdown article.
- **1.8 Final Editing & Formatting:** Enhancing article quality and converting Markdown into HTML.
- **1.9 Publishing:** Posting the final article draft to a WordPress site.
- **1.10 Utility & Data Handling:** Various set and code nodes for JSON parsing, field assignments, and data transformations.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initialization

- **Overview:** This block starts the workflow manually and retrieves initial keyword data with search intent from a Google Sheet.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Get row(s) in sheet1 (Google Sheets)  
  - Edit Fields2 (Set)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual trigger  
    - Role: Initiates the workflow manually for testing or production runs.  
    - Connections: Outputs to "Get row(s) in sheet1".  
    - Failure Modes: None specific; user initiation only.

  - **Get row(s) in sheet1**  
    - Type: Google Sheets node  
    - Role: Retrieves keyword data filtered where column "blog" equals "no" (i.e., keywords needing blog posts).  
    - Config: Returns first match only from sheet ID 1932135411 of the specified spreadsheet.  
    - Output: JSON with keyword, related keywords, search intent, suggested keywords.  
    - Failure Modes: Auth errors, empty results, sheet changes.

  - **Edit Fields2**  
    - Type: Set node  
    - Role: Normalizes and assigns input fields for downstream use (keyword, search intent, related keywords, suggested keyword).  
    - Expressions: Maps data from Google Sheets output JSON.  
    - Connections: Outputs to "Research".  
    - Failure Modes: Missing fields or empty values.

---

#### 2.2 Research & Semantic Analysis

- **Overview:** Enriches keyword data with external research via Perplexity, limits results, and extracts semantic content structure.
- **Nodes Involved:**  
  - Research (Perplexity)  
  - research data (Code)  
  - Get row(s) in sheet (Google Sheets)  

- **Node Details:**

  - **Research**  
    - Type: Perplexity API node  
    - Role: Performs deep research for the keyword topic, fetching summarized news and articles.  
    - Config: Uses "sonar-deep-research" model with the keyword data as input.  
    - Output: Raw research JSON with search results and citations.  
    - Failure Modes: API quota limits, connection errors.

  - **research data**  
    - Type: Code node  
    - Role: Processes Perplexity output; limits to first 5 search results & citations for efficiency.  
    - Code: Extracts titles, URLs, dates, last updated fields; prepares trimmed data for AI input.  
    - Connections: Outputs to "Get row(s) in sheet" for internal linking.  
    - Failure Modes: JSON parsing errors, missing fields.

  - **Get row(s) in sheet**  
    - Type: Google Sheets  
    - Role: Retrieves URLs of completed blog posts for internal linking.  
    - Config: Reads from a separate sheet with published blog URLs.  
    - Failure Modes: Auth issues, sheet changes.

---

#### 2.3 SEO Content Strategy Generation

- **Overview:** Generates a detailed, semantically rich SEO content strategy including writing style, hidden insights, semantic structure, and FAQ.
- **Nodes Involved:**  
  - structure (OpenAI)  
  - Code (JSON parsing)  
  - Set KWs and Insights fields (Set)

- **Node Details:**

  - **structure**  
    - Type: OpenAI GPT-4o-mini  
    - Role: Creates an SEO content strategy brief from keyword inputs (primary, related, suggested, search intent).  
    - Input: Keyword data from "Get row(s) in sheet" processed JSON.  
    - Output: JSON with content structure, target audience, hidden insights, semantic sections, FAQs.  
    - Failure Modes: API limits, malformed inputs.

  - **Code**  
    - Type: Code node  
    - Role: Parses the raw JSON string output from "structure" into usable JSON for later nodes.  
    - Code: JSON.parse on the AI response content.  
    - Connections: Outputs to "Set KWs and Insights fields".  
    - Failure Modes: JSON parsing errors if AI output malformed.

  - **Set KWs and Insights fields**  
    - Type: Set node  
    - Role: Extracts and stores specific fields from the structured JSON (search intent, hidden insight, writing style, semantic content sections, keywords).  
    - Expressions: Uses JSON path expressions on parsed AI output.  
    - Connections: Outputs to "blog tittle".  
    - Failure Modes: Missing fields or empty values.

---

#### 2.4 Title & Key Takeaways Generation

- **Overview:** Refines the blog post title for SEO and generates key takeaways summarizing article value.
- **Nodes Involved:**  
  - blog tittle (OpenAI)  
  - genrate key takeaways (OpenAI)

- **Node Details:**

  - **blog tittle**  
    - Type: OpenAI GPT-4o-mini-2024-07-18  
    - Role: Revises and optimizes the blog title based on keywords, writing style, and semantic sections.  
    - Input: Extracted semantic data from "Set KWs and Insights fields".  
    - Output: JSON string with refined title.  
    - Failure Modes: API failure, incomplete input.

  - **genrate key takeaways**  
    - Type: OpenAI GPT-4o-2024-11-20  
    - Role: Creates a concise, markdown-formatted list of key takeaways highlighting main ideas, hidden insights, and subtopics.  
    - Input: Title and semantic content fields.  
    - Output: Markdown text with key takeaways.  
    - Failure Modes: API errors, incomplete data.

---

#### 2.5 Introduction, Outline & Prompt Construction

- **Overview:** Produces an engaging introduction, detailed article outline, and a comprehensive AI prompt for writing the article body.
- **Nodes Involved:**  
  - introduction (Anthropic Claude 3.5 Sonnet)  
  - outline (OpenAI GPT-4o-mini)  
  - blog prompet (OpenAI GPT-4o-mini)

- **Node Details:**

  - **introduction**  
    - Type: Anthropic Claude 3.5 Sonnet  
    - Role: Crafts a compelling, keyword-rich introduction paragraph in Markdown.  
    - Input: Title, key takeaways, keywords, and semantic sections.  
    - Output: Markdown text introduction.  
    - Failure Modes: API limits, missing inputs.

  - **outline**  
    - Type: OpenAI GPT-4o-mini  
    - Role: Generates a detailed, SEO-optimized article outline with hierarchical headings (##, ###).  
    - Input: Title, key takeaways, introduction, semantic sections, writing style.  
    - Output: Markdown outline text.  
    - Failure Modes: API errors.

  - **blog prompet**  
    - Type: OpenAI GPT-4o-mini  
    - Role: Builds a detailed prompt for the article body writer AI, including all semantic data, outline, and keywords.  
    - Input: Title, key takeaways, introduction, outline, semantic content, writing style.  
    - Output: Markdown prompt text for article generation.  
    - Failure Modes: API limitations.

---

#### 2.6 Main Article Body Generation

- **Overview:** Generates the main blog content body strictly following the outline and SEO strategy.
- **Nodes Involved:**  
  - body of article (OpenAI GPT-5 mini)

- **Node Details:**

  - **body of article**  
    - Type: OpenAI GPT-5 mini  
    - Role: Writes the full main content body with structured headings, bullet points, and SEO keyword integration.  
    - Input: Detailed AI prompt from "blog prompet".  
    - Output: Markdown formatted article body.  
    - Failure Modes: API errors, incomplete or inconsistent prompts.

---

#### 2.7 Conclusion & Assembly

- **Overview:** Creates a conclusion with FAQs and assembles all parts into a single cohesive Markdown article, embedding internal and external links.
- **Nodes Involved:**  
  - conclusion (OpenAI ChatGPT-4o latest)  
  - assemble (OpenAI ChatGPT-4o latest)

- **Node Details:**

  - **conclusion**  
    - Type: OpenAI ChatGPT-4o latest  
    - Role: Produces a concise conclusion summarizing key points and adding FAQs in Markdown format.  
    - Input: Main article body content.  
    - Output: Markdown conclusion text.  
    - Failure Modes: API failure, insufficient input.

  - **assemble**  
    - Type: OpenAI ChatGPT-4o latest  
    - Role: Combines introduction, key takeaways, main content, and conclusion into a single Markdown article.  
    - Embeds internal links (from completed blogs sheet) and external links (from Perplexity search results).  
    - Output: Final Markdown article.  
    - Failure Modes: API errors, missing inputs, link insertion issues.

---

#### 2.8 Final Editing & Formatting

- **Overview:** Performs a final editorial pass to enhance quality, readability, and SEO, then converts Markdown to HTML.
- **Nodes Involved:**  
  - edit (OpenAI ChatGPT-4o latest)  
  - HTML (Code)

- **Node Details:**

  - **edit**  
    - Type: OpenAI ChatGPT-4o latest  
    - Role: Final content polishing, expanding depth, improving transitions, and embedding strategic internal/external links.  
    - Input: Assembled Markdown article.  
    - Output: Enhanced Markdown article.  
    - Failure Modes: API errors.

  - **HTML**  
    - Type: Code node  
    - Role: Converts Markdown content into HTML using regex replacements for headings, bold, italics, links, lists, and paragraphs.  
    - Output: HTML string wrapped in styled container for WordPress.  
    - Failure Modes: Markdown edge cases not converting correctly.

---

#### 2.9 Publishing

- **Overview:** Publishes the final HTML content as a draft blog post in WordPress.
- **Nodes Involved:**  
  - Create a post (WordPress)

- **Node Details:**

  - **Create a post**  
    - Type: WordPress node  
    - Role: Creates a new post with the generated title, content (HTML), author ID, and status as draft, with comments disabled.  
    - Credentials: Requires WordPress API credentials with posting rights.  
    - Failure Modes: Auth errors, API limits, malformed HTML.

---

#### 2.10 Utility & Data Handling

- **Overview:** Miscellaneous nodes managing JSON parsing, assignments, and workflow notes.
- **Nodes Involved:**  
  - Code node (parsing JSON for semantic structure)  
  - Sticky Notes (guidance for user)

- **Node Details:**

  - **Code** (parsing AI JSON output)  
    - Parses raw JSON strings from AI responses into structured JSON objects for downstream nodes.

  - **Sticky Notes**  
    - Provide user guidance on prerequisites like Google Sheets setup, credential configuration, and workflow usage tips with helpful links.

---

### 3. Summary Table

| Node Name                   | Node Type                    | Functional Role                        | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                   |
|-----------------------------|------------------------------|-------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger               | Starts the workflow manually         |                             | Get row(s) in sheet1         | #add a google sheet with keyword +related keyword+search intent also create google sheet cerdentiols **Double click** to edit me. [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| Get row(s) in sheet1         | Google Sheets                | Retrieves keyword data for writing   | When clicking ‘Test workflow’| Edit Fields2                 |                                                                                                |
| Edit Fields2                 | Set                         | Normalizes keyword and search intent | Get row(s) in sheet1          | Research                    |                                                                                                |
| Research                    | Perplexity API               | Performs research on keyword topic   | Edit Fields2                 | research data               | ## this perpelexity node will search the top ten article **Double click** to edit me. [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| research data               | Code                        | Limits research results and citations| Research                    | Get row(s) in sheet          |                                                                                                |
| Get row(s) in sheet          | Google Sheets                | Fetches completed blog URLs          | research data                | structure                   | ## add sheet with your completed blogs url its for internal linking **Double click** to edit me. [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| structure                   | OpenAI GPT-4o-mini           | Generates SEO content strategy brief | Get row(s) in sheet          | Code                       |                                                                                                |
| Code                        | Code                        | Parses JSON SEO strategy              | structure                   | Set KWs and Insights fields  |                                                                                                |
| Set KWs and Insights fields  | Set                         | Extracts semantic fields and insights| Code                        | blog tittle                 |                                                                                                |
| blog tittle                 | OpenAI GPT-4o-mini           | Refines blog post title               | Set KWs and Insights fields  | genrate key takeaways       |                                                                                                |
| genrate key takeaways       | OpenAI GPT-4o                | Creates article key takeaways         | blog tittle                 | introduction                |                                                                                                |
| introduction                | Anthropic Claude 3.5 Sonnet  | Writes engaging blog introduction     | genrate key takeaways       | outline                    |                                                                                                |
| outline                    | OpenAI GPT-4o-mini           | Generates detailed article outline    | introduction                | blog prompet                |                                                                                                |
| blog prompet               | OpenAI GPT-4o-mini           | Builds detailed article writing prompt| outline                    | body of article             |                                                                                                |
| body of article            | OpenAI GPT-5 mini            | Generates main blog article content   | blog prompet                | conclusion                  |                                                                                                |
| conclusion                 | OpenAI ChatGPT-4o latest     | Writes article conclusion and FAQs    | body of article             | assemble                   |                                                                                                |
| assemble                   | OpenAI ChatGPT-4o latest     | Assembles full article with links     | conclusion                  | edit                       |                                                                                                |
| edit                       | OpenAI ChatGPT-4o latest     | Final editorial pass and enhancement  | assemble                    | HTML                       | ## this is edit model for article you can change it prompet on your gauidlines **Double click** to edit me. [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| HTML                       | Code                        | Converts Markdown to HTML for WP      | edit                       | Create a post              | ## this will convert your article into html that will posted on wordpress **Double click** to edit me. [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| Create a post              | WordPress                   | Publishes draft blog post              | HTML                       |                             | ## just create the wordpress user cerdentiols and select create post **Double click** to edit me. [Guide](https://docs.n8n.io/workflows/sticky-notes/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "When clicking ‘Test workflow’" to start the workflow manually.

2. **Add a Google Sheets node** "Get row(s) in sheet1":  
   - Configure to connect to your Google Sheets account.  
   - Select the spreadsheet with keyword data (e.g., "analyzed_keywords_with_volume.csv").  
   - Sheet name: The sheet containing keywords (ID 1932135411 in the original).  
   - Add filter: column "blog" equals "no".  
   - Return the first matching row only.

3. **Add a Set node "Edit Fields2"**:  
   - Map these fields from the previous node:  
     - keyword ← $json.keyword  
     - search intent ← $json['search intent']  
     - related keyword ← $json['related keyword']  
     - suggested keyword ← $json['suggested keyword']

4. **Add a Perplexity node "Research"**:  
   - Authenticate with Perplexity API credentials.  
   - Use the "sonar-deep-research" model.  
   - Pass the JSON data from "Edit Fields2" as input for research query.

5. **Add a Code node "research data"**:  
   - Write JavaScript to limit Perplexity results to first 5 search results and citations.  
   - Output trimmed JSON with titles, URLs, dates.

6. **Add a Google Sheets node "Get row(s) in sheet"**:  
   - Connect to your spreadsheet of completed blogs with URLs for internal linking.  
   - Use the proper spreadsheet and sheet IDs.

7. **Add an OpenAI node "structure" (GPT-4o-mini)**:  
   - Configure with OpenAI API credentials.  
   - Input the primary keyword, search intent, related and suggested keywords from "Get row(s) in sheet".  
   - Request a full SEO content strategy JSON including writing style, hidden insights, semantic content structure.

8. **Add a Code node "Code"**:  
   - Parse the JSON string from "structure" AI output to usable JSON object.

9. **Add a Set node "Set KWs and Insights fields"**:  
   - Extract from parsed JSON: search_intent, hidden_insight, target_audience, semantic_content_structure sections (introduction, conclusion, section 1-3), primary and related keywords, writing style.

10. **Add an OpenAI node "blog tittle" (GPT-4o-mini-2024-07-18)**:  
    - Input extracted semantic fields from "Set KWs and Insights fields".  
    - Output refined blog title in JSON.

11. **Add an OpenAI node "genrate key takeaways" (GPT-4o-2024-11-20)**:  
    - Use the refined title and semantic fields.  
    - Output key takeaways in Markdown.

12. **Add an Anthropic Claude 3.5 node "introduction"**:  
    - Input title, key takeaways, keywords, search intent, and semantic sections.  
    - Output introduction paragraph in Markdown.

13. **Add an OpenAI node "outline" (GPT-4o-mini)**:  
    - Input title, key takeaways, introduction, primary & related keywords, semantic sections, writing style.  
    - Output a detailed article outline in Markdown with ## and ### headings.

14. **Add an OpenAI node "blog prompet" (GPT-4o-mini)**:  
    - Input title, key takeaways, introduction, outline, semantic sections, writing style.  
    - Output a full AI prompt for writing the article body.

15. **Add an OpenAI node "body of article" (GPT-5 mini)**:  
    - Input the prompt from "blog prompet".  
    - Output the main article body in Markdown.

16. **Add an OpenAI node "conclusion" (ChatGPT-4o latest)**:  
    - Input main article body content.  
    - Output conclusion and FAQs in Markdown.

17. **Add an OpenAI node "assemble" (ChatGPT-4o latest)**:  
    - Input introduction, key takeaways, main content, conclusion.  
    - Include internal link URLs from "Get row(s) in sheet" and external links from "research data".  
    - Output the full cohesive Markdown article.

18. **Add an OpenAI node "edit" (ChatGPT-4o latest)**:  
    - Input assembled article Markdown.  
    - Perform final editing and enrichment.

19. **Add a Code node "HTML"**:  
    - Convert Markdown article to HTML using regex replacements for headings, bold, italics, links, lists, paragraphs.  
    - Wrap in styled div for WordPress.

20. **Add a WordPress node "Create a post"**:  
    - Configure with WordPress API credentials.  
    - Post title from "blog tittle".  
    - Content from "HTML" output.  
    - Set status as "draft", author ID 1, and disable comments.

21. **Connect nodes sequentially** following the input-to-output flow described above.

22. **Ensure all API credentials for Google Sheets, OpenAI, Perplexity, and WordPress are properly set.**

23. **Add Sticky Notes** at key workflow points to remind users about sheet setup, credential creation, and usage tips.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                                               |
|-------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Add a Google Sheet with columns: keyword, related keyword, search intent. Also create Google Sheets credentials. | Sticky Note near initial Google Sheets node. Guide: [n8n Sticky Notes](https://docs.n8n.io/workflows/sticky-notes/)            |
| This Perplexity node searches the top ten articles for research context.                                         | Sticky Note near Perplexity node. Guide: [n8n Sticky Notes](https://docs.n8n.io/workflows/sticky-notes/)                       |
| Add a sheet with your completed blog URLs to enable internal linking.                                            | Sticky Note near completed blogs Google Sheets node. Guide: [n8n Sticky Notes](https://docs.n8n.io/workflows/sticky-notes/)    |
| This node converts your article Markdown into HTML for WordPress posting.                                        | Sticky Note near HTML code node. Guide: [n8n Sticky Notes](https://docs.n8n.io/workflows/sticky-notes/)                         |
| Create WordPress user credentials and select "create post" action.                                               | Sticky Note near WordPress node. Guide: [n8n Sticky Notes](https://docs.n8n.io/workflows/sticky-notes/)                         |
| The Edit node uses an advanced prompt for final article polishing and SEO optimization.                           | Sticky Note near Edit node. Guide: [n8n Sticky Notes](https://docs.n8n.io/workflows/sticky-notes/)                              |

---

**Disclaimer:** The provided text is exclusively sourced from an n8n automated workflow, fully compliant with content policies, and contains only legal, public data.
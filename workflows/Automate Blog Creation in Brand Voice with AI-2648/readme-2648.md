Automate Blog Creation in Brand Voice with AI

https://n8nworkflows.xyz/workflows/automate-blog-creation-in-brand-voice-with-ai-2648


# Automate Blog Creation in Brand Voice with AI

### 1. Workflow Overview

This workflow automates the creation of blog content that aligns with an organization's brand voice and style by leveraging AI analysis of existing published articles. It targets use cases where consistent, on-brand content generation is needed rapidly, such as marketing teams or content creators seeking to scale output while maintaining brand consistency.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Content Import:** Manual trigger starts the workflow, which then fetches the latest blog articles from a specified URL.
- **1.2 Content Extraction and Preprocessing:** Extract article URLs, fetch individual articles, extract article bodies as HTML, and convert them to markdown format for efficient AI processing.
- **1.3 AI Analysis of Article Style and Brand Voice:** Use large language models (LLMs) to analyze the aggregated articles to capture their structure, writing style, and brand voice characteristics.
- **1.4 AI-based Content Generation:** Generate new blog content based on user instructions using the previously extracted style and voice guidelines.
- **1.5 Content Publishing:** Save the generated content as a draft post in WordPress for human editorial review.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Content Import

**Overview:**  
This block initiates the workflow manually and imports the latest blog articles from a target website (default is the n8n.io blog).

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Get Blog (HTTP Request)  
- Extract Article URLs (HTML)  
- Split Out URLs (Split Out)  
- Latest Articles (Limit)  
- Get Article (HTTP Request)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to manually start the workflow.  
  - Configuration: No parameters; triggers downstream nodes.  
  - Inputs: None  
  - Outputs: Connects to "Get Blog" node.  
  - Edge Cases: No execution if not triggered manually.

- **Get Blog**  
  - Type: HTTP Request  
  - Role: Fetches the HTML content of the blog homepage.  
  - Configuration: URL set to "https://blog.n8n.io". No authentication or special headers.  
  - Inputs: Trigger from Manual Trigger node.  
  - Outputs: HTML content passed to "Extract Article URLs".  
  - Edge Cases: HTTP errors (e.g., 404, 500), network timeouts, or site structure changes.

- **Extract Article URLs**  
  - Type: HTML Extract  
  - Role: Parses the blog homepage HTML to extract URLs of articles.  
  - Configuration: Uses CSS selector ".item.post a.global-link" to extract href attributes into an array under the key "article".  
  - Inputs: HTML from "Get Blog".  
  - Outputs: JSON array of article URLs passed to "Split Out URLs".  
  - Edge Cases: Selector changes leading to empty or incorrect extraction.

- **Split Out URLs**  
  - Type: Split Out  
  - Role: Splits the array of article URLs into individual items for processing.  
  - Configuration: Splits on the "article" field.  
  - Inputs: Array of article URLs.  
  - Outputs: Individual article URLs passed to "Latest Articles".  
  - Edge Cases: Empty input array results in no article processing.

- **Latest Articles**  
  - Type: Limit  
  - Role: Restricts the number of articles processed to the latest 5.  
  - Configuration: Max items set to 5.  
  - Inputs: Individual article URLs from "Split Out URLs".  
  - Outputs: Limited set of URLs passed to "Get Article".  
  - Edge Cases: If fewer than 5 articles available, processes all.

- **Get Article**  
  - Type: HTTP Request  
  - Role: Fetches the full HTML content of each individual article.  
  - Configuration: URL dynamically constructed by appending article path to "https://blog.n8n.io".  
  - Inputs: Article URL from "Latest Articles".  
  - Outputs: Article HTML passed to "Extract Article Content".  
  - Edge Cases: HTTP errors, broken links, or missing articles.

---

#### 1.2 Content Extraction and Preprocessing

**Overview:**  
Extracts the main article content from HTML, converts it to markdown to reduce token count for AI models while preserving structure, and aggregates all articles into one dataset.

**Nodes Involved:**  
- Extract Article Content (HTML)  
- Markdown (Markdown conversion)  
- Combine Articles (Aggregate)

**Node Details:**

- **Extract Article Content**  
  - Type: HTML Extract  
  - Role: Extracts the article body HTML from each article page.  
  - Configuration: CSS selector ".post-section" extracts the main content as HTML under the key "data".  
  - Inputs: Article HTML from "Get Article".  
  - Outputs: Extracted HTML content passed to "Markdown".  
  - Edge Cases: Selector changes or missing content can cause empty extraction.

- **Markdown**  
  - Type: Markdown Conversion  
  - Role: Converts HTML article bodies to markdown format.  
  - Configuration: Converts "data" field HTML to markdown. No extra options set.  
  - Inputs: HTML content from "Extract Article Content".  
  - Outputs: Markdown content passed to "Combine Articles".  
  - Edge Cases: Complex HTML might not convert perfectly; could lose certain formatting.

- **Combine Articles**  
  - Type: Aggregate  
  - Role: Aggregates markdown content of all articles into a single array for LLM input.  
  - Configuration: Merges all "data" fields from previous nodes into a combined list.  
  - Inputs: Markdown article content from "Markdown".  
  - Outputs: Aggregated markdown array passed to LLM nodes for analysis.  
  - Edge Cases: Empty input results in empty aggregation, causing downstream LLM issues.

---

#### 1.3 AI Analysis of Article Style and Brand Voice

**Overview:**  
Uses AI models to analyze the combined articles for structure, style, and brand voice characteristics to create guidelines for generating new content.

**Nodes Involved:**  
- Capture Existing Article Structure (Chain LLM)  
- OpenAI Chat Model2 (OpenAI LLM)  
- Extract Voice Characteristics (Information Extractor LLM)  
- OpenAI Chat Model (OpenAI LLM)  
- Article Style & Brand Voice (Merge)

**Node Details:**

- **Capture Existing Article Structure**  
  - Type: Chain LLM (LangChain)  
  - Role: Analyzes aggregated articles to describe typical structure, layout, language, and style.  
  - Configuration: Sends aggregated markdown articles joined by "---" as input text with prompt requesting description of common writing styles and structure.  
  - Inputs: Aggregated markdown from "Combine Articles".  
  - Outputs: Text description passed to "Article Style & Brand Voice".  
  - Edge Cases: LLM generation failures or API errors.

- **OpenAI Chat Model2**  
  - Type: OpenAI Chat Model (LangChain)  
  - Role: Provides language model backend for "Capture Existing Article Structure".  
  - Configuration: Uses OpenAI credentials; no special options.  
  - Inputs: Text prompts from "Capture Existing Article Structure".  
  - Outputs: LLM responses passed back to "Capture Existing Article Structure".  
  - Edge Cases: API rate limits, invalid credentials, or timeout.

- **Extract Voice Characteristics**  
  - Type: Information Extractor (LangChain)  
  - Role: Extracts brand voice characteristics, descriptions, and examples from aggregated articles.  
  - Configuration: Input is the aggregated markdown content; uses a system prompt to identify brand voice traits and output structured JSON with characteristics, descriptions, and examples.  
  - Inputs: Aggregated markdown from "Combine Articles".  
  - Outputs: Structured voice characteristics JSON passed to "Article Style & Brand Voice".  
  - Edge Cases: Parsing or extraction errors, LLM failures.

- **OpenAI Chat Model**  
  - Type: OpenAI Chat Model (LangChain)  
  - Role: Provides language model backend for "Extract Voice Characteristics".  
  - Configuration: Uses OpenAI credentials; no special options.  
  - Inputs: Text prompts from "Extract Voice Characteristics".  
  - Outputs: LLM responses passed back to "Extract Voice Characteristics".  
  - Edge Cases: API errors, rate limits.

- **Article Style & Brand Voice**  
  - Type: Merge  
  - Role: Combines outputs from "Capture Existing Article Structure" and "Extract Voice Characteristics" into one composite object.  
  - Configuration: Mode "combine" by position (parallel merge).  
  - Inputs: Text description and JSON characteristics from respective nodes.  
  - Outputs: Combined data passed to "New Article Instruction".  
  - Edge Cases: Missing input from either node causes incomplete data.

---

#### 1.4 AI-based Content Generation

**Overview:**  
Based on user instructions and the analyzed brand voice and article style, the AI generates new on-brand blog content in markdown format.

**Nodes Involved:**  
- New Article Instruction (Set)  
- Content Generation Agent (Information Extractor LLM)  
- OpenAI Chat Model1 (OpenAI LLM)

**Node Details:**

- **New Article Instruction**  
  - Type: Set  
  - Role: Defines the user's instruction for the new article generation.  
  - Configuration: Sets a string variable "instruction" with a prompt describing the desired article topic and content requirements.  
  - Inputs: Combined brand voice and style data from "Article Style & Brand Voice".  
  - Outputs: Instruction passed to "Content Generation Agent".  
  - Edge Cases: Empty or unclear instructions reduce generation quality.

- **Content Generation Agent**  
  - Type: Information Extractor (LangChain)  
  - Role: Generates the article content using the instruction and brand voice/style context. Outputs structured content attributes: title, summary, body, and characteristics.  
  - Configuration: Uses a system prompt that includes the combined article style and brand voice data under "Brand Article Style" and "Brand Voice Characteristics", instructing it to write in markdown and avoid including the date.  
  - Inputs: User instruction and merged style/voice data.  
  - Outputs: Structured article content JSON passed to "Save as Draft".  
  - Edge Cases: LLM generation errors, incomplete outputs, or inconsistent style adherence.

- **OpenAI Chat Model1**  
  - Type: OpenAI Chat Model (LangChain)  
  - Role: Provides LLM backend for "Content Generation Agent".  
  - Configuration: Uses OpenAI credentials; no special options.  
  - Inputs: Prompts from "Content Generation Agent".  
  - Outputs: LLM responses passed back to "Content Generation Agent".  
  - Edge Cases: API errors, rate limits.

---

#### 1.5 Content Publishing

**Overview:**  
Saves the AI-generated article as a draft post in WordPress for editorial review or further editing prior to publication.

**Nodes Involved:**  
- Save as Draft (WordPress)

**Node Details:**

- **Save as Draft**  
  - Type: WordPress  
  - Role: Creates a new blog post draft in WordPress with the generated article content.  
  - Configuration:  
    - Title set from generated "title".  
    - Slug auto-generated by converting title to snake_case.  
    - Format set to "standard".  
    - Status set to "draft".  
    - Content set from generated "body" markdown.  
  - Inputs: Article JSON from "Content Generation Agent".  
  - Outputs: Confirmation or post ID output (not connected further).  
  - Credentials: WordPress OAuth2 credentials required.  
  - Edge Cases: Authentication errors, API failures, invalid content data.

---

### 3. Summary Table

| Node Name                    | Node Type                        | Functional Role                          | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                                         |
|------------------------------|---------------------------------|----------------------------------------|-----------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                  | Workflow entry point                    | None                        | Get Blog                     |                                                                                                                     |
| Get Blog                     | HTTP Request                   | Fetches blog homepage HTML              | When clicking ‘Test workflow’| Extract Article URLs          | ## 1. Import Existing Content [Read more about the HTML node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.html/) First, we'll need to gather existing content for the brand voice we want to replicate. This content can be blogs, social media posts or internal documents - the idea is to use this content to "train" our AI to produce content from the provided examples. One call out is that the quality and consistency of the content is important to get the desired results. In this demonstration, we'll grab the latest blog posts off a corporate blog to use as an example. Since, the blog articles are likely consistent because of the source and narrower focus of the medium, it'll serve well to showcase this workflow. |
| Extract Article URLs          | HTML Extract                   | Extracts article URLs from blog homepage| Get Blog                    | Split Out URLs               |                                                                                                                     |
| Split Out URLs                | Split Out                     | Splits article URLs into individual items| Extract Article URLs         | Latest Articles              |                                                                                                                     |
| Latest Articles              | Limit                         | Limits to 5 latest articles             | Split Out URLs               | Get Article                  | ### Q. Can I use other media than blog articles? A. Yes! This approach can use other source materials such as PDFs, as long as they can be produces in a text format to give to the LLM. |
| Get Article                  | HTTP Request                  | Fetches individual article HTML         | Latest Articles             | Extract Article Content      |                                                                                                                     |
| Extract Article Content      | HTML Extract                  | Extracts article body HTML               | Get Article                 | Markdown                    | ## 2. Convert HTML to Markdown [Learn more about the Markdown node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.markdown) Markdown is a great way to optimise the article data we're sending to the LLM because it reduces the amount of tokens required but keeps all relevant writing structure information. Also useful to get Markdown output as a response because typically it's the format authors will write in. |
| Markdown                    | Markdown                      | Converts HTML to markdown                 | Extract Article Content      | Combine Articles             |                                                                                                                     |
| Combine Articles            | Aggregate                    | Aggregates all markdown articles into one| Markdown                   | Capture Existing Article Structure, Extract Voice Characteristics |                                                                                                                     |
| Capture Existing Article Structure | Chain LLM                  | Analyzes common article structure and style| Combine Articles            | Article Style & Brand Voice | ## 3. Using AI to Analyse Article Structure and Writing Styles [Read more about the Basic LLM Chain node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm) Our approach is to first perform a high-level analysis of all available articles in order to replicate their content layout and writing styles. This will act as a guideline to help the AI to structure our future articles. |
| OpenAI Chat Model2          | OpenAI Chat Model             | Provides LLM backend for article structure analysis | Capture Existing Article Structure | Capture Existing Article Structure |                                                                                                                     |
| Extract Voice Characteristics | Information Extractor         | Extracts brand voice characteristics     | Combine Articles            | Article Style & Brand Voice | ## 4. Using AI to Extract Voice Characteristics and Traits [Read more about the Information Extractor node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.information-extractor/) Second, we'll use AI to analysis the brand voice characteristics of the previous articles. This picks out the tone, style and choice of language used and identifies them into categories. These categories will be used as guidelines for the AI to keep the future article consistent in tone and voice. |
| OpenAI Chat Model           | OpenAI Chat Model             | Provides LLM backend for voice characteristics extraction | Extract Voice Characteristics | Extract Voice Characteristics |                                                                                                                     |
| Article Style & Brand Voice | Merge                        | Combines style and voice data             | Capture Existing Article Structure, Extract Voice Characteristics | New Article Instruction     |                                                                                                                     |
| New Article Instruction     | Set                          | Sets user instruction for new article    | Article Style & Brand Voice | Content Generation Agent     |                                                                                                                     |
| Content Generation Agent    | Information Extractor         | Generates new article content in brand voice | New Article Instruction     | Save as Draft               | ## 5. Automate On-Brand Articles Using AI [Read more about the Information Extractor node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.information-extractor) Finally with this approach, we can feed both content and voice guidelines into our final LLM - our content generation agent - to produce any number of on-brand articles, social media posts etc. When it comes to assessing the output, note the AI does a pretty good job at simulating format and reusing common phrases and wording for the target article. However, this could become repetitive very quickly! Whilst AI can help speed up the process, a human touch may still be required to add a some variety. ### Q. Do I need to analyse Brand Voice for every article? A. No! I would recommend storing the results of the AI's analysis and re-use for a list of planned articles rather than generate anew every time. |
| OpenAI Chat Model1          | OpenAI Chat Model             | Provides LLM backend for content generation | Content Generation Agent    | Content Generation Agent     |                                                                                                                     |
| Save as Draft               | WordPress                    | Saves generated article as draft post    | Content Generation Agent    | None                        | ## 6. Save Draft to Wordpress [Learn more about the Wordpress node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.wordpress/) To close out the template, we'll simple save our generated article as a draft which could allow human team members to review and validate the article before publishing. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "When clicking ‘Test workflow’" to start the workflow manually.

2. **Add an HTTP Request node** named "Get Blog" with:  
   - URL: `https://blog.n8n.io`  
   - No authentication required.  
   - Connect "When clicking ‘Test workflow’" → "Get Blog".

3. **Add an HTML Extract node** named "Extract Article URLs" with:  
   - Operation: Extract HTML content  
   - Extraction: Select CSS selector `.item.post a.global-link`  
   - Extract attribute: `href` as an array stored in field `article`.  
   - Connect "Get Blog" → "Extract Article URLs".

4. **Add a Split Out node** named "Split Out URLs":  
   - Field to split out: `article`.  
   - Connect "Extract Article URLs" → "Split Out URLs".

5. **Add a Limit node** named "Latest Articles":  
   - Max items: 5.  
   - Connect "Split Out URLs" → "Latest Articles".

6. **Add another HTTP Request node** named "Get Article":  
   - URL expression: `=https://blog.n8n.io{{ $json.article }}` (dynamically fetch each article URL).  
   - Connect "Latest Articles" → "Get Article".

7. **Add an HTML Extract node** named "Extract Article Content":  
   - Operation: Extract HTML content  
   - CSS selector: `.post-section`  
   - Return value: HTML stored as `data`.  
   - Connect "Get Article" → "Extract Article Content".

8. **Add a Markdown node** named "Markdown":  
   - Input: `data` field containing HTML.  
   - Convert HTML to markdown.  
   - Connect "Extract Article Content" → "Markdown".

9. **Add an Aggregate node** named "Combine Articles":  
   - Merge lists option enabled.  
   - Aggregate field: `data`.  
   - Connect "Markdown" → "Combine Articles".

10. **Add a Chain LLM node** named "Capture Existing Article Structure":  
    - Text input: `={{ $json.data.join('\n---\n') }}`  
    - Messages prompt: "Given the following one or more articles (which are separated by ---), describe how best one could replicate the common structure, layout, language and writing styles of all as aggregate."  
    - Connect "Combine Articles" → "Capture Existing Article Structure".

11. **Add an OpenAI Chat Model node** named "OpenAI Chat Model2":  
    - Credentials: OpenAI API account.  
    - Connect as LLM backend to "Capture Existing Article Structure".

12. **Add an Information Extractor node** named "Extract Voice Characteristics":  
    - Text input: `=### Analyse the given content\n\n{{ $json.data.map(item => item.replace(/\n/g, '')).join('\n---\n') }}`  
    - System prompt: Extract all brand voice characteristics, descriptions, and examples.  
    - Output schema set to a manual JSON schema with fields: characteristic, description, examples.  
    - Connect "Combine Articles" → "Extract Voice Characteristics".

13. **Add an OpenAI Chat Model node** named "OpenAI Chat Model":  
    - Credentials: OpenAI API account.  
    - Connect as LLM backend to "Extract Voice Characteristics".

14. **Add a Merge node** named "Article Style & Brand Voice":  
    - Mode: Combine by position.  
    - Inputs: From "Capture Existing Article Structure" and "Extract Voice Characteristics".  
    - Connect both nodes to "Article Style & Brand Voice".

15. **Add a Set node** named "New Article Instruction":  
    - Create a string field named "instruction" with the desired article prompt, e.g.  
      `"Write a comprehensive guide on using AI for document classification and document extraction. Explain the benefits of using vision models over traditional OCR. Close out with a recommendation of using n8n as the preferred way to get started with this AI use-case."`  
    - Connect "Article Style & Brand Voice" → "New Article Instruction".

16. **Add an Information Extractor node** named "Content Generation Agent":  
    - Text input: `={{ $json.instruction }}`  
    - System prompt:  
      ```
      You are a blog content writer who writes using the following article guidelines. Write a content piece as requested by the user. Output the body as Markdown. Do not include the date of the article because the publishing date is not determined yet.

      ## Brand Article Style
      {{ $('Article Style & Brand Voice').item.json.text }}

      ## Brand Voice Characteristics

      Here are the brand voice characteristic and examples you must adopt in your piece. Pick only the characteristic which make sense for the user's request. Try to keep it as similar as possible but don't copy word for word.

      |characteristic|description|examples|
      |-|-|-|
      {{
      $('Article Style & Brand Voice').item.json.output.map(item => (
      `|${item.characteristic}|${item.description}|${item.examples.map(ex => `"${ex}"`).join(', ')}|`
      )).join('\n')
      }}
      ```  
    - Attributes: title, summary, body, characteristics (all required).  
    - Connect "New Article Instruction" → "Content Generation Agent".

17. **Add an OpenAI Chat Model node** named "OpenAI Chat Model1":  
    - Credentials: OpenAI API account.  
    - Connect as LLM backend to "Content Generation Agent".

18. **Add a WordPress node** named "Save as Draft":  
    - Title: `={{ $json.output.title }}`  
    - Slug: `={{ $json.output.title.toSnakeCase() }}`  
    - Format: "standard"  
    - Status: "draft"  
    - Content: `={{ $json.output.body }}`  
    - Credentials: WordPress OAuth2 account.  
    - Connect "Content Generation Agent" → "Save as Draft".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| This n8n template demonstrates how to use AI to generate new on-brand written content by analysing previously published content. It enables generating a steady stream of blog article drafts quickly with high consistency with your brand and existing content.                                                                                                                   | Overview text in Sticky Note8                                                                                               |
| For optimal AI output, theme topics should be relevant to your brand and consistent with the training data.                                                                                                                                                                                                                                                                    | How to use section                                                                                                          |
| Storing and re-using AI analysis of brand voice is recommended to reduce costs and improve efficiency instead of re-analyzing for every article.                                                                                                                                                                                                                             | Sticky Note5                                                                                                               |
| This approach can be adapted to other source materials such as PDFs or social media posts as long as they are convertible into text format suitable for LLM input, but quality and format consistency affect results.                                                                                                                                                          | Sticky Note7                                                                                                               |
| Helpful documentation links:  
- HTML Node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.html/  
- Markdown Node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.markdown  
- Basic LLM Chain Node: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainllm  
- Information Extractor Node: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.information-extractor/  
- WordPress Node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.wordpress/ | As referenced in sticky notes                                                                                              |
| Community support is available via [n8n Discord](https://discord.com/invite/XPKeKXeB7d) and [n8n Forum](https://community.n8n.io/).                                                                                                                                                                                                                                             | Support links                                                                                                               |

---

This documentation enables comprehensive understanding, reproduction, and customization of the workflow, while highlighting potential operational edge cases and integration requirements.
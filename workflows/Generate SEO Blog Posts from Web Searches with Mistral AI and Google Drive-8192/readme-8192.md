Generate SEO Blog Posts from Web Searches with Mistral AI and Google Drive

https://n8nworkflows.xyz/workflows/generate-seo-blog-posts-from-web-searches-with-mistral-ai-and-google-drive-8192


# Generate SEO Blog Posts from Web Searches with Mistral AI and Google Drive

### 1. Workflow Overview

This workflow automates the generation of SEO-optimized blog posts based on web search results, leveraging AI models and Google Drive for storage.  
It is designed for content creators, marketers, and SEO professionals aiming to produce high-quality blog content by scraping top-ranking articles on a keyword, extracting and summarizing key information, and enhancing the text with AI writing assistance.

The workflow consists of the following logical blocks:

- **1.1 Input Reception and Web Search**: Receives a chat message containing a keyword or query, triggers a Google Search via RapidAPI to fetch top articles.  
- **1.2 URL Extraction and Iterative Content Retrieval**: Extracts URLs from search results and loops over them to scrape their HTML content.  
- **1.3 Content Cleaning and Aggregation**: Cleans HTML to extract plain text from key tags, then aggregates all cleaned page texts into a single combined body of text.  
- **1.4 AI-Driven Summarization and SEO Content Generation**: Uses Mistral AI (via Ollama) to summarize key insights from combined content, then generates a detailed SEO blog article, and refines it to a conversational, engaging tone.  
- **1.5 Output Storage**: Saves the finalized blog post content as a text file in a specified Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Web Search

- **Overview:**  
  This block listens for chat messages as triggers, uses the message content as a search query, and performs a Google Search to retrieve top 3 relevant articles.

- **Nodes Involved:**  
  - When chat message received  
  - Google Search (rapid api)  
  - Code (URL extraction)  

- **Node Details:**  

  - **When chat message received**  
    - *Type & Role:* Langchain chat trigger node, entry point for workflow on chat input reception.  
    - *Configuration:* Default with webhook to catch incoming chat messages.  
    - *Input/Output:* No input; outputs chat message content as JSON.  
    - *Edge Cases:* Network issues, missing webhook calls, malformed chat input.  
    - *Version:* 1.1  

  - **Google Search (rapid api)**  
    - *Type & Role:* HTTP Request node calling Google Search API via RapidAPI to get search results.  
    - *Config:* Uses query parameter `query` set dynamically from chat message input; limits results to 3. Authenticated via header auth credentials.  
    - *Input/Output:* Input from chat message; outputs JSON with search results including URLs, titles, descriptions.  
    - *Edge Cases:* API quota limits, auth failures, empty results, rate limiting.  
    - *Version:* 4.2  

  - **Code (URL extraction)**  
    - *Type & Role:* Code node to process Google Search API output and extract an array of URLs with titles and descriptions.  
    - *Config:* Extracts `url`, `title`, and `description` from each search result item, returns structured array.  
    - *Input/Output:* Takes search results JSON; outputs array of simplified URL objects.  
    - *Edge Cases:* Unexpected API response format, missing fields.  
    - *Version:* 2  

---

#### 2.2 URL Extraction and Iterative Content Retrieval

- **Overview:**  
  This block processes each URL one by one to retrieve full HTML content of the articles.

- **Nodes Involved:**  
  - Loop Over Items (splitInBatches)  
  - Get content from URL (HTTP Request)  
  - Loop Over Items1 (splitInBatches, nested)  

- **Node Details:**  

  - **Loop Over Items**  
    - *Type & Role:* SplitInBatches node to iterate over extracted URLs individually to control processing flow.  
    - *Config:* Default batching, processes one URL per iteration.  
    - *Input/Output:* Input from Code node; outputs single URL JSON object per iteration.  
    - *Edge Cases:* Empty input array, batch size misconfiguration.  
    - *Version:* 3  

  - **Get content from URL**  
    - *Type & Role:* HTTP Request node to fetch the HTML content of each URL.  
    - *Config:* URL dynamically set from current item's `url` field. No special headers or auth configured.  
    - *Input/Output:* Input one URL; outputs raw HTML content in response body.  
    - *Edge Cases:* URL unreachable, timeouts, HTTP errors, anti-bot protections.  
    - *Version:* 4.2  

  - **Loop Over Items1**  
    - *Type & Role:* Another SplitInBatches node inside the first loop, used to further process batches of HTML content items.  
    - *Config:* Default batching.  
    - *Input/Output:* Input from Loop Over Items, output to content cleaning.  
    - *Edge Cases:* Similar to above.  
    - *Version:* 3  

---

#### 2.3 Content Cleaning and Aggregation

- **Overview:**  
  Cleans HTML content to extract meaningful textual data, then aggregates all page texts into a single combined text for AI processing.

- **Nodes Involved:**  
  - Clean body text (Code)  
  - Create Body Text Code (Code)  

- **Node Details:**  

  - **Clean body text**  
    - *Type & Role:* Code node to parse HTML and extract plain text from `<p>` and `<span>` tags.  
    - *Config:* Uses regex to find tags, strips nested HTML tags, trims text, returns array `bodyText`. Throws error if HTML missing.  
    - *Input/Output:* Input raw HTML; output array of cleaned text snippets.  
    - *Edge Cases:* Malformed HTML, empty content, regex failures.  
    - *Version:* 2  

  - **Create Body Text Code**  
    - *Type & Role:* Code node that concatenates the cleaned text arrays from all pages into a single string separated by a page break marker.  
    - *Config:* Joins array of `bodyText` with separator `"\n\n─── Page Break ───\n\n"`.  
    - *Input/Output:* Input multiple items with `bodyText`; outputs single object with `combinedBodyText` string.  
    - *Edge Cases:* Empty input arrays, missing fields.  
    - *Version:* 2  

---

#### 2.4 AI-Driven Summarization and SEO Content Generation

- **Overview:**  
  This critical block uses Mistral AI to analyze combined text, extract key insights, then generate and refine an SEO-optimized blog article.

- **Nodes Involved:**  
  - Data extractor and summarizer1 (Langchain agent)  
  - Seo Content1 (Langchain agent)  
  - Refine Content1 (Langchain agent)  
  - Ollama Chat Model (LM Chat Mistral)  

- **Node Details:**  

  - **Ollama Chat Model**  
    - *Type & Role:* AI language model node interfacing with Ollama API running Mistral 7b model.  
    - *Config:* Model `mistral:7b` selected, no additional options. Requires Ollama API credentials.  
    - *Input/Output:* Receives and sends messages to AI agents in chain.  
    - *Edge Cases:* API errors, model unavailability, rate limiting.  
    - *Version:* 1  

  - **Data extractor and summarizer1**  
    - *Type & Role:* Langchain agent node acting as data extractor and summarizer to produce structured summary from combined body text.  
    - *Config:* Detailed prompt instructing to analyze combined text from 3 pages, extract topics, key insights, supporting evidence, and implications formatted for blog writing.  
    - *Input/Output:* Input combined text string; output structured summary content.  
    - *Edge Cases:* Incoherent AI output, timeout, prompt misinterpretation.  
    - *Version:* 2.2  

  - **Seo Content1**  
    - *Type & Role:* Langchain SEO Content Writer agent that uses the summary to generate a superior, SEO-optimized blog article.  
    - *Config:* Prompt stresses improving and expanding on competitor content, structuring article with intro/body/conclusion, SEO keyword integration, and readability.  
    - *Input/Output:* Input summary from previous node; outputs blog article text.  
    - *Edge Cases:* AI hallucinations, keyword stuffing, off-topic content.  
    - *Version:* 2.2  

  - **Refine Content1**  
    - *Type & Role:* Langchain agent refining the blog content to a human-like, engaging, conversational style without altering facts or structure.  
    - *Config:* Prompt with detailed guidelines on tone, style, prohibitions on certain words, and examples.  
    - *Input/Output:* Input raw blog article; output polished final blog post content.  
    - *Edge Cases:* Style drift, loss of factual accuracy.  
    - *Version:* 2.2  

---

#### 2.5 Output Storage

- **Overview:**  
  Saves the refined blog post content as a text file in a designated Google Drive folder for easy access and further use.

- **Nodes Involved:**  
  - Google Drive  

- **Node Details:**  

  - **Google Drive**  
    - *Type & Role:* Google Drive node to create a text file from the blog post content.  
    - *Config:* File name dynamically generated with timestamp prefix `blog_post_({{ $now }})`. Content set from refined content node output. Saves to folder with ID `1VWWC9wby04xopZAatnTu4YPywLjCRIj5` (named 'Seo content'). Uses OAuth2 credentials.  
    - *Input/Output:* Input final blog text; outputs metadata of created file.  
    - *Edge Cases:* Auth failures, quota exceeded, file name conflicts.  
    - *Version:* 3  

---

### 3. Summary Table

| Node Name                     | Node Type                           | Functional Role                            | Input Node(s)                  | Output Node(s)                | Sticky Note                                         |
|-------------------------------|-----------------------------------|--------------------------------------------|-------------------------------|------------------------------|----------------------------------------------------|
| When chat message received     | @n8n/n8n-nodes-langchain.chatTrigger | Entry point, receives chat messages        | —                             | Google Search (rapid api)      |                                                    |
| Google Search (rapid api)      | n8n-nodes-base.httpRequest         | Performs Google Search (top 3 results)     | When chat message received      | Code                         |                                                    |
| Code                          | n8n-nodes-base.code                | Extracts URLs, titles, descriptions         | Google Search (rapid api)       | Loop Over Items              | ## Scraping top 3 articles                          |
| Loop Over Items               | n8n-nodes-base.splitInBatches      | Iterates over URLs to fetch HTML content    | Code                          | Get content from URL, Loop Over Items1 |                                                    |
| Get content from URL           | n8n-nodes-base.httpRequest         | Retrieves HTML content from URLs            | Loop Over Items                | Loop Over Items              |                                                    |
| Loop Over Items1              | n8n-nodes-base.splitInBatches      | Further iterates over HTML content items    | Loop Over Items                | Create Body Text Code, Clean body text | ## Cleaning the data                                |
| Clean body text               | n8n-nodes-base.code                | Extracts clean text from HTML                | Loop Over Items1               | Loop Over Items1             |                                                    |
| Create Body Text Code          | n8n-nodes-base.code                | Aggregates all cleaned texts into one string| Loop Over Items1               | Data extractor and summarizer1 |                                                    |
| Data extractor and summarizer1| @n8n/n8n-nodes-langchain.agent    | Summarizes combined text to structured info | Create Body Text Code          | Seo Content1                 | ## Creating the improved article                    |
| Seo Content1                  | @n8n/n8n-nodes-langchain.agent    | Generates SEO blog article from summary     | Data extractor and summarizer1 | Refine Content1              |                                                    |
| Refine Content1               | @n8n/n8n-nodes-langchain.agent    | Polishes article to conversational style   | Seo Content1                  | Google Drive                 |                                                    |
| Google Drive                  | n8n-nodes-base.googleDrive         | Saves final blog post as text file           | Refine Content1               | —                            |                                                    |
| Ollama Chat Model             | @n8n/n8n-nodes-langchain.lmChatOllama | Provides Mistral AI model interface          | — (used by AI agents)          | Data extractor and summarizer1, Seo Content1, Refine Content1 |                                                    |
| Sticky Note                  | n8n-nodes-base.stickyNote          | Visual comment: Scraping top 3 articles     | —                             | —                            | ## Scraping top 3 articles                          |
| Sticky Note1                 | n8n-nodes-base.stickyNote          | Visual comment: Cleaning the data            | —                             | —                            | ## Cleaning the data                                |
| Sticky Note2                 | n8n-nodes-base.stickyNote          | Visual comment: Creating the improved article| —                             | —                            | ## Creating the improved article                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add **When chat message received** node (Langchain Chat Trigger).  
   - Use default settings to capture incoming chat messages via webhook.

2. **Setup Google Search Node:**  
   - Add **HTTP Request** node named `Google Search (rapid api)`.  
   - Configure URL to `https://google-search74.p.rapidapi.com/?`.  
   - Set Query Parameters:  
     - `query` = Expression: `{{$json.chatInput}}`  
     - `limit` = `3`  
   - Set Header Parameters:  
     - `x-rapidapi-host` = `google-search74.p.rapidapi.com`  
   - Authentication: Use HTTP Header Auth credentials with RapidAPI key.  
   - Connect **When chat message received** → **Google Search (rapid api)**.

3. **Add Code Node to Extract URLs:**  
   - Add **Code** node named `Code`.  
   - Paste JavaScript code to map `results` array extracting `url`, `title`, and `description`.  
   - Connect **Google Search (rapid api)** → **Code**.

4. **Set up Loop Over URLs:**  
   - Add **SplitInBatches** node named `Loop Over Items`.  
   - Default batch size (1) is fine.  
   - Connect **Code** → **Loop Over Items**.

5. **Add HTTP Request to Get Page Content:**  
   - Add **HTTP Request** node named `Get content from URL`.  
   - Set URL parameter to expression: `{{$json.url}}`.  
   - Connect **Loop Over Items** → **Get content from URL**.

6. **Loop Over Fetched Contents for Cleaning:**  
   - Add **SplitInBatches** node named `Loop Over Items1`.  
   - Connect **Loop Over Items** main output (index 1) → **Loop Over Items1** and also **Get content from URL** main output → **Loop Over Items** (to process next URL).  
   - Connect **Get content from URL** → **Loop Over Items** for iterative fetching.

7. **Add Code Node to Clean HTML Content:**  
   - Add **Code** node named `Clean body text`.  
   - Paste JavaScript code that extracts text from `<p>` and `<span>` tags, strips HTML tags, and returns array `bodyText`.  
   - Connect **Loop Over Items1** → **Clean body text**.

8. **Add Code Node to Aggregate Text:**  
   - Add **Code** node named `Create Body Text Code`.  
   - Paste JavaScript code to join all `bodyText` arrays with a page break separator into `combinedBodyText`.  
   - Connect **Loop Over Items1** → **Create Body Text Code**.

9. **Add Ollama Chat Model Node:**  
   - Add **@n8n/n8n-nodes-langchain.lmChatOllama** node named `Ollama Chat Model`.  
   - Select model `mistral:7b`.  
   - Configure Ollama API credentials.  
   - This node will be linked as AI language model provider for subsequent agent nodes.

10. **Add Data Extractor and Summarizer Agent:**  
    - Add **Langchain Agent** node named `Data extractor and summarizer1`.  
    - Set prompt text per instructions to analyze `combinedBodyText` and produce structured summary.  
    - Use `Ollama Chat Model` for AI model.  
    - Connect **Create Body Text Code** → **Data extractor and summarizer1**.

11. **Add SEO Content Writer Agent:**  
    - Add **Langchain Agent** node named `Seo Content1`.  
    - Set prompt to generate SEO-optimized blog article from summary content.  
    - Use `Ollama Chat Model`.  
    - Connect **Data extractor and summarizer1** → **Seo Content1**.

12. **Add Content Refinement Agent:**  
    - Add **Langchain Agent** node named `Refine Content1`.  
    - Set prompt to humanize and polish the blog text.  
    - Use `Ollama Chat Model`.  
    - Connect **Seo Content1** → **Refine Content1**.

13. **Add Google Drive Node:**  
    - Add **Google Drive** node named `Google Drive`.  
    - Operation: `Create From Text`.  
    - Name: Expression `=blog_post_({{ $now }})` for timestamped file name.  
    - Content: Set to output from `Refine Content1`.  
    - Folder ID: Use your target Google Drive folder ID (e.g., `1VWWC9wby04xopZAatnTu4YPywLjCRIj5`).  
    - Set Google Drive OAuth2 credentials.  
    - Connect **Refine Content1** → **Google Drive**.

14. **Add Sticky Notes (Optional):**  
    - Add sticky notes to document workflow sections for clarity:  
      - "Scraping top 3 articles" near search and extraction nodes.  
      - "Cleaning the data" near HTML cleaning nodes.  
      - "Creating the improved article" near AI content generation nodes.

15. **Test and Activate Workflow:**  
    - Validate node connections and credentials.  
    - Activate webhook and test by sending a chat message with a keyword.  
    - Monitor execution for errors and performance.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                  |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow uses Mistral 7b AI model via Ollama for advanced natural language processing and content generation. | Ollama API and model integration requires valid credentials.    |
| Google Search API is accessed through RapidAPI with HTTP header authentication; ensure API keys are valid.   | https://rapidapi.com/google-search74-google-search74-default/api/google-search74 |
| Google Drive folder ID must be set correctly to store blog posts; ensure OAuth2 credentials have write access.| Google Drive folder "Seo content" with ID `1VWWC9wby04xopZAatnTu4YPywLjCRIj5` |
| Prompt texts are detailed and tailored for SEO and humanized writing to maximize content quality and ranking. | Prompts embedded in Langchain agent nodes.                      |

---

**Disclaimer:** The provided content is solely derived from an automated n8n workflow. It adheres strictly to content policies and handles only legal and public data.
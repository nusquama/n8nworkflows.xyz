Website Content Scraper & SEO Keyword Extractor with GPT-4o-mini and Airtable

https://n8nworkflows.xyz/workflows/website-content-scraper---seo-keyword-extractor-with-gpt-4o-mini-and-airtable-5657


# Website Content Scraper & SEO Keyword Extractor with GPT-4o-mini and Airtable

### 1. Workflow Overview

This n8n workflow is designed to scrape website content based on user input, extract SEO-relevant keywords using GPT-4o-mini, and store refined data into Airtable. It targets digital marketers, SEO specialists, or content teams seeking automated keyword extraction and content insights from websites.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Capture website URL input from user via form trigger.
- **1.2 Website Content Retrieval & Cleaning:** Fetch raw HTML content and clean it to extract meaningful text.
- **1.3 Topic-Wise Information Extraction:** Use GPT-4o-mini to analyze cleaned content and organize it by topic.
- **1.4 Keyword List Generation:** Generate a curated list of 90 SEO keywords from cleaned text using GPT-4o-mini.
- **1.5 Data Cleaning & Synchronization:** Clean GPT outputs and upsert structured data with keywords into Airtable.
- **1.6 Controlled Timing & Data Merging:** Manage asynchronous data flow and merging to ensure data integrity before storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  The workflow begins by gathering a website name (URL) from the user through an interactive form.

- **Nodes Involved:**  
  - Website Name (formTrigger)

- **Node Details:**

  - **Website Name**  
    - Type: Form Trigger  
    - Role: Captures the website URL input from user interaction.  
    - Configuration: Single required field labeled "Website Name" with a submit button.  
    - Key Expressions: None; direct user input.  
    - Input: External user form submission.  
    - Output: JSON containing the website URL under the key `"Website Name SEO"`.  
    - Edge Cases: User submits invalid URL or empty input (form enforces required field).  
    - Notes: The form response is set to `"lastNode"` indicating synchronous flow.

---

#### 2.2 Website Content Retrieval & Cleaning

- **Overview:**  
  Fetches the raw HTML content of the input website and processes it to extract clean, plain text for further analysis.

- **Nodes Involved:**  
  - HTTP (httpRequest)  
  - HTML (code)  
  - Split Out1 (splitOut)  

- **Node Details:**

  - **HTTP**  
    - Type: HTTP Request  
    - Role: Retrieves the raw HTML source of the user-provided website URL.  
    - Configuration: URL dynamically set to the `"Website Name SEO"` field from the input node.  
    - Input: Website URL from "Website Name" node.  
    - Output: Raw HTML content in JSON field `data`.  
    - Edge Cases: Network errors, invalid URL, timeouts, HTTP errors (404, 500).  
    - Version: v4.2

  - **HTML**  
    - Type: Code (JavaScript)  
    - Role: Cleans the raw HTML by removing tags, styles, and excessive whitespace to produce readable plain text.  
    - Configuration: Custom JavaScript that strips `<style>` tags and all HTML tags, condenses whitespace, and trims result.  
    - Key Expressions: Uses `$(“HTTP”).all()[0]?.json?.data` to fetch raw HTML from previous node.  
    - Input: Output JSON of HTTP node.  
    - Output: JSON with property `cleanedData` containing plain text.  
    - Edge Cases: Empty or malformed HTML input; possible null reference if HTTP output missing.  
    - Version: v2

  - **Split Out1**  
    - Type: Split Out  
    - Role: Splits array or object fields; here used to pass `cleanedData` as individual items.  
    - Configuration: Splits out the field `"cleanedData"` from the HTML node output.  
    - Input: Output from HTML node.  
    - Output: Items with `cleanedData` field isolated.  
    - Edge Cases: If `cleanedData` is missing or empty, downstream nodes receive empty inputs.

---

#### 2.3 Topic-Wise Information Extraction

- **Overview:**  
  Processes the cleaned website text with GPT-4o-mini to extract topic-wise structured information about the website content.

- **Nodes Involved:**  
  - Topic Wise information. (langchain.agent)  
  - Cleaned ## (code)  
  - Split Out2 (splitOut)  
  - Wait1 (wait)  

- **Node Details:**

  - **Topic Wise information.**  
    - Type: LangChain Agent  
    - Role: Sends cleaned text to GPT-4o-mini to extract organized topic-wise website information.  
    - Configuration:  
      - Text input is the website URL from the user input node `"Website Name SEO"`.  
      - System message set dynamically to the cleaned text (`$json.cleanedData`) instructing GPT to find topic-wise information.  
      - Prompt type: define.  
    - Input: Cleaned data from Split Out1.  
    - Output: JSON with GPT response under `output` or similar key.  
    - Edge Cases: GPT API failures, rate limits, incomplete or irrelevant responses.  
    - Version: v1.8

  - **Cleaned ##**  
    - Type: Code (JavaScript)  
    - Role: Cleans GPT output by removing markdown formatting like asterisks and heading hashes to produce plain text.  
    - Configuration: Regex operations removing `**`, `###`, `##` at line starts.  
    - Key Expressions: Works on `"output"` field from previous GPT node.  
    - Input: Output from Topic Wise information.  
    - Output: JSON with cleaned text field `"cleaned"`.  
    - Edge Cases: Empty or malformed GPT output.  
    - Version: v2

  - **Split Out2**  
    - Type: Split Out  
    - Role: Splits out `"cleaned"` field for downstream processing.  
    - Input: Cleaned ## node output.  
    - Output: Items with `"cleaned"` field isolated.  
    - Edge Cases: Empty split if no cleaned data.

  - **Wait1**  
    - Type: Wait  
    - Role: Adds a 20-second pause before merging data to manage pacing and API rate limits.  
    - Configuration: Wait 20 seconds (amount: 20).  
    - Input: Output from Split Out2.  
    - Output: Delayed pass-through.  
    - Edge Cases: Delay might cause timeout in long workflows or affect responsiveness.

---

#### 2.4 Keyword List Generation

- **Overview:**  
  Generates a curated list of 90 SEO keywords from the cleaned website content using GPT-4o-mini.

- **Nodes Involved:**  
  - OpenAI Chat Model1 (lmChatOpenAi)  
  - list (langchain.agent)  

- **Node Details:**

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4o-mini language model interface for keyword extraction.  
    - Configuration: Model set to "gpt-4o-mini", no extra options.  
    - Credentials: Uses OpenAI API key "OpenAi account 2".  
    - Input: Receives from Wait1 node after cleaning.  
    - Output: GPT output for keyword extraction.  
    - Edge Cases: API failures, rate limits.  
    - Version: v1.2

  - **list**  
    - Type: LangChain Agent  
    - Role: Prompts GPT to produce an "Important Keyword List for SEO" limited to 90 keywords from cleaned text.  
    - Configuration:  
      - Text input is the cleaned data `"cleaned"`.  
      - System message: instructs GPT to generate a list of 90 SEO keywords only.  
      - Prompt type: define.  
    - Input: Output from OpenAI Chat Model1.  
    - Output: JSON with keyword list in `"output"` or similar.  
    - Edge Cases: GPT may generate fewer or irrelevant keywords if input is poor or prompt misunderstood.  
    - Version: v2

---

#### 2.5 Data Cleaning & Synchronization

- **Overview:**  
  Cleans final keyword list output and merges with topic information for structured storage. Then updates or creates records in Airtable.

- **Nodes Involved:**  
  - Merge (merge)  
  - Airtable (airtable)  

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines two streams of data by SQL-like join to align topic-wise info with keyword list.  
    - Configuration: Mode set to "combineBySql" to merge inputs from Wait1 (topic info) and list (keywords).  
    - Input: Two inputs: Wait1 (cleaned topic info) and list (keyword list).  
    - Output: Merged JSON combining topic data and keywords.  
    - Edge Cases: Mismatch in data items causing empty merges or duplicates.  
    - Version: v3.1

  - **Airtable**  
    - Type: Airtable node  
    - Role: Upserts data into Airtable base and table for persistent storage.  
    - Configuration:  
      - Base: appxR9kySQVhhjSZ9 (named "website")  
      - Table: tblirvzTvL2ShdbR1 ("Table 1")  
      - Columns:  
        - Data: mapped from merged field `cleaned` (topic info text)  
        - Status: set statically to "Done"  
        - Keyword: mapped from merged field `output` (keyword list)  
        - Website Name: mapped from original form input `"Website Name SEO"`  
      - Matching Columns: Data (for upsert)  
      - Operation: Upsert  
      - Authentication: Airtable OAuth2 Personal Access Token  
    - Input: Output from Merge node.  
    - Output: Airtable operation result.  
    - Edge Cases: Authentication failures, Airtable API limits, data schema mismatches.  
    - Version: v2.1

---

#### 2.6 Controlled Timing & Data Flow Management

- **Overview:**  
  Manages timing and data outputs using splitting and wait nodes to ensure correct sequencing and data isolation.

- **Nodes Involved:**  
  - Split Out1  
  - Split Out2  
  - Wait1  

- **Node Details:**  

  - **Split Out1**  
    - Splits cleaned HTML data for topic-wise processing.

  - **Split Out2**  
    - Splits cleaned GPT output for waiting and keyword extraction.

  - **Wait1**  
    - Adds delay to prevent race conditions or API overuse, ensuring data integrity before merging.

---

### 3. Summary Table

| Node Name            | Node Type                          | Functional Role                                 | Input Node(s)        | Output Node(s)         | Sticky Note                                  |
|----------------------|----------------------------------|------------------------------------------------|----------------------|------------------------|----------------------------------------------|
| Website Name         | formTrigger                      | Receives website URL input from user           | (external)           | HTTP                   | ## READING WEBSITE user input                |
| HTTP                 | httpRequest                     | Fetches raw HTML content from website           | Website Name         | HTML                   |                                              |
| HTML                 | code                            | Cleans raw HTML to plain text                    | HTTP                 | Split Out1             | ## cleaned HTML code                          |
| Split Out1           | splitOut                        | Splits cleaned text for topic-wise processing   | HTML                 | Topic Wise information. |                                              |
| Topic Wise information.| langchain.agent                 | Extracts topic-wise info via GPT-4o-mini        | Split Out1            | Cleaned ##             | ## Topic wise information. website name.     |
| Cleaned ##           | code                            | Cleans GPT markdown output                        | Topic Wise information.| Split Out2             |                                              |
| Split Out2           | splitOut                        | Splits cleaned output for delay and keywords    | Cleaned ##           | Wait1, list            |                                              |
| Wait1                | wait                            | Adds 20-second delay to manage pacing            | Split Out2           | Merge                  |                                              |
| OpenAI Chat Model1   | lmChatOpenAi                    | Provides GPT-4o-mini interface for keyword gen  | (implicit via list)   | list                   |                                              |
| list                 | langchain.agent                 | Generates SEO keyword list with GPT               | OpenAI Chat Model1, Split Out2 | Merge        |                                              |
| Merge                | merge                           | Combines topic info and keyword list             | Wait1, list          | Airtable               |                                              |
| Airtable             | airtable                        | Upserts combined data into Airtable               | Merge                | (end)                  |                                              |
| OpenAI Chat Model    | lmChatOpenAi                    | Supports Topic Wise information node (AI LM)    | (system)             | Topic Wise information. |                                              |
| Sticky Note          | stickyNote                      | Visual comment: "## READING WEBSITE user input" | None                 | None                   | ## READING WEBSITE user input                |
| Sticky Note1         | stickyNote                      | Visual comment: "## cleaned HTML code"            | None                 | None                   | ## cleaned HTML code                          |
| Sticky Note2         | stickyNote                      | Visual comment: "## Topic wise information. website name." | None           | None                   | ## Topic wise information. website name.     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Website Name" node**  
   - Type: Form Trigger  
   - Configure form with one required field labeled "Website Name" (text input).  
   - Set button label to "Submit".  
   - Set response mode to "lastNode".

2. **Create "HTTP" node**  
   - Type: HTTP Request  
   - Set URL to expression: `{{$json["Website Name SEO"]}}` (the input URL).  
   - Use default GET method.

3. **Create "HTML" node**  
   - Type: Code (JavaScript)  
   - Paste the following JavaScript code to clean HTML:  
     ```js
     const data = $("HTTP").all()[0]?.json?.data;

     function extractTextFromHTML(html) {
       const cleanedHTML = html
         .replace(/<style[\s\S]*?>[\s\S]*?<\/style>/gi, "")
         .replace(/<[^>]+>/g, "")
         .replace(/\s+/g, " ")
         .trim();
       return cleanedHTML;
     }

     const cleanedData = extractTextFromHTML(data);

     return { cleanedData };
     ```
   - Input: Connect from HTTP node.

4. **Create "Split Out1" node**  
   - Type: Split Out  
   - Configure to split out field `"cleanedData"`.  
   - Connect from HTML node.

5. **Create "Topic Wise information." node**  
   - Type: LangChain Agent  
   - Set `text` parameter to expression: `{{$json["Website Name SEO"]}}` (website URL).  
   - Set system message to expression: `{{$json.cleanedData}} + "\n\nfind it topic wise information.\n"`  
   - Set prompt type to 'define'.  
   - Connect from Split Out1.

6. **Create "Cleaned ##" node**  
   - Type: Code (JavaScript)  
   - Paste the following to clean markdown from GPT output:  
     ```js
     const input = $json["output"];
     const cleaned = input
       .replace(/\*\*/g, '')
       .replace(/^###\s?/gm, '')
       .replace(/^##\s?/gm, '');
     return { json: { cleaned } };
     ```
   - Connect from Topic Wise information. node.

7. **Create "Split Out2" node**  
   - Type: Split Out  
   - Configure to split out field `"cleaned"`.  
   - Connect from Cleaned ## node.

8. **Create "Wait1" node**  
   - Type: Wait  
   - Set time delay to 20 seconds.  
   - Connect from Split Out2.

9. **Create "OpenAI Chat Model1" node**  
   - Type: LangChain OpenAI Chat Model  
   - Select model: "gpt-4o-mini".  
   - Configure credentials with OpenAI API key (e.g., "OpenAi account 2").  
   - No additional options.  
   - Connect from Wait1 node (or to be used as AI language model for next node).

10. **Create "list" node**  
    - Type: LangChain Agent  
    - Set `text` to expression: `{{$json.cleaned}}` (cleaned text).  
    - System message: `only for list number of 90 keyword data """Important Keyword List for SEO"""`  
    - Prompt type: 'define'.  
    - Connect AI Language Model input from OpenAI Chat Model1 node.

11. **Create "Merge" node**  
    - Type: Merge  
    - Set mode to "combineBySql".  
    - Connect first input from Wait1 node (topic-wise cleaned data).  
    - Connect second input from list node (keyword list).

12. **Create "Airtable" node**  
    - Type: Airtable  
    - Configure with Airtable OAuth2 Personal Access Token credentials.  
    - Set Base to "appxR9kySQVhhjSZ9" (website base).  
    - Set Table to "tblirvzTvL2ShdbR1" (Table 1).  
    - Mapping:  
      - Data: `={{ $json.cleaned }}` (from merged data)  
      - Status: "Done" (static value)  
      - Keyword: `={{ $json.output }}` (keyword list)  
      - Website Name: `={{ $('Website Name').item.json['Website Name SEO'] }}` (from form input)  
    - Operation: Upsert, matching on "Data" column.  
    - Connect from Merge node.

13. **Add Sticky Notes for visual clarity** (optional)  
    - Place Sticky Notes near groups of nodes with contents:  
      - "## READING WEBSITE user input" near Website Name node  
      - "## cleaned HTML code" near HTML node  
      - "## Topic wise information. website name." near Topic Wise information. node

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow uses GPT-4o-mini (a lightweight GPT-4 variant) for cost-effective, fast AI steps.   | Model choice balances performance and cost for SEO keyword generation and topic extraction.       |
| Airtable personal access token credentials must have permission to upsert data in the specified base and table. | Ensure token scopes include read/write access to the base "website" and table "Table 1".          |
| The workflow includes a 20-second wait node to regulate API calls and prevent race conditions.   | Adjust wait time based on API rate limits and workflow responsiveness requirements.                |
| The Split Out nodes are crucial for isolating fields to process array elements cleanly.          | Improper splitting may cause data loss or failed merges downstream.                               |
| See n8n documentation on LangChain nodes for advanced prompt engineering and OpenAI API setup.  | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/                               |

---

This completes the full technical reference documentation for the "Website Content Scraper & SEO Keyword Extractor with GPT-4o-mini and Airtable" workflow. It is designed to be comprehensive for experienced users and AI agents to understand, reproduce, and extend the workflow reliably.
AI Agent: Find the Right LinkedIn Profiles in Seconds

https://n8nworkflows.xyz/workflows/ai-agent--find-the-right-linkedin-profiles-in-seconds-2898


# AI Agent: Find the Right LinkedIn Profiles in Seconds

### 1. Workflow Overview

This workflow automates LinkedIn prospecting by transforming natural language queries into structured search parameters, performing targeted Google Custom Searches for LinkedIn profiles, filtering relevant results, and saving them into Google Sheets for easy access and outreach.

It is organized into three main logical blocks:

- **1.1 Input Reception and AI Processing**: Receives natural language queries via chat, uses an AI agent with memory and language models to extract structured search parameters such as job titles, industries, and locations.

- **1.2 Search Execution and Filtering**: Uses the extracted parameters to perform paginated Google Custom Search API requests, then filters the results to retain only valid LinkedIn profile URLs.

- **1.3 Data Storage**: Saves the filtered LinkedIn profile URLs and titles into a configured Google Sheet for further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and AI Processing

**Overview:**  
This block handles incoming user queries, processes them with an AI agent that leverages OpenAI language models and memory buffers, and prepares structured search parameters for the search phase.

**Nodes Involved:**  
- When chat message received  
- AI Agent  
- OpenAI Chat Model  
- Window Buffer Memory  
- Call n8n Workflow Tool  

**Node Details:**

- **When chat message received**  
  - *Type:* Chat Trigger (LangChain)  
  - *Role:* Entry point that listens for incoming chat messages containing prospecting queries.  
  - *Configuration:* Uses webhook to receive messages; no parameters set explicitly.  
  - *Connections:* Output to AI Agent node.  
  - *Edge Cases:* Webhook failures, malformed input, or unsupported message formats.

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Core AI processor that interprets natural language queries into structured search parameters.  
  - *Configuration:* Utilizes linked OpenAI Chat Model, Window Buffer Memory for context retention, and can invoke sub-workflows via Call n8n Workflow Tool.  
  - *Key Expressions:* Uses ai_languageModel, ai_memory, and ai_tool inputs to integrate language model, memory, and external workflow calls.  
  - *Connections:* Inputs from When chat message received, OpenAI Chat Model, Window Buffer Memory, Call n8n Workflow Tool; outputs structured data to next block.  
  - *Edge Cases:* AI model timeouts, API quota limits, memory overflow, or unexpected input causing extraction errors.

- **OpenAI Chat Model**  
  - *Type:* Language Model (OpenAI Chat)  
  - *Role:* Provides conversational AI capabilities to the AI Agent for natural language understanding.  
  - *Configuration:* Uses OpenAI API credentials; default or customized model parameters.  
  - *Connections:* Output to AI Agent’s language model input.  
  - *Edge Cases:* API authentication errors, rate limits, or model unavailability.

- **Window Buffer Memory**  
  - *Type:* Memory Buffer (LangChain)  
  - *Role:* Maintains conversation context to improve parameter extraction accuracy over multiple interactions.  
  - *Configuration:* Sliding window memory with default size; no explicit parameters shown.  
  - *Connections:* Output to AI Agent’s memory input.  
  - *Edge Cases:* Memory size limits, context loss on restart.

- **Call n8n Workflow Tool**  
  - *Type:* Tool Workflow (LangChain)  
  - *Role:* Allows the AI Agent to invoke additional workflows if needed for extended processing or data enrichment.  
  - *Configuration:* No parameters specified; presumably calls a sub-workflow for advanced tasks.  
  - *Connections:* Output to AI Agent’s tool input.  
  - *Edge Cases:* Sub-workflow failures, missing workflows, or parameter mismatches.

---

#### 2.2 Search Execution and Filtering

**Overview:**  
This block takes the structured search parameters, generates paginated search requests to Google Custom Search API, and filters the results to isolate valid LinkedIn profile URLs.

**Nodes Involved:**  
- Execute Workflow Trigger  
- OpenAI  
- Set Search Page Numbers  
- Split Search Pages  
- Google Custom Search API Request  
- Filter LinkedIn Profiles  

**Node Details:**

- **Execute Workflow Trigger**  
  - *Type:* Execute Workflow Trigger  
  - *Role:* Initiates the search process with preset or dynamic query parameters (e.g., jobTitle, companyIndustry, location).  
  - *Configuration:* Contains example query parameters in pinData (jobTitle=CEO, companyIndustry=Finance, location=London).  
  - *Connections:* Output to OpenAI node.  
  - *Edge Cases:* Trigger misfires, missing parameters.

- **OpenAI**  
  - *Type:* OpenAI Node (LangChain)  
  - *Role:* Possibly used here to refine or generate search queries or parameters before pagination setup.  
  - *Configuration:* Uses OpenAI API credentials; exact prompt not shown.  
  - *Connections:* Output to Set Search Page Numbers.  
  - *Edge Cases:* API errors, prompt failures.

- **Set Search Page Numbers**  
  - *Type:* Set Node  
  - *Role:* Defines the pagination range for Google Custom Search (e.g., pages 1 to N).  
  - *Configuration:* Sets an array or list of page numbers to iterate over.  
  - *Connections:* Output to Split Search Pages.  
  - *Edge Cases:* Incorrect page number ranges causing empty or excessive requests.

- **Split Search Pages**  
  - *Type:* SplitOut Node  
  - *Role:* Splits the pagination array into individual items to trigger separate API requests per page.  
  - *Configuration:* Default splitting behavior.  
  - *Connections:* Output to Google Custom Search API Request.  
  - *Edge Cases:* Empty input array, splitting errors.

- **Google Custom Search API Request**  
  - *Type:* HTTP Request  
  - *Role:* Performs the actual search queries against Google Custom Search API using the parameters and page numbers.  
  - *Configuration:* Uses Google Custom Search API credentials; query parameters include job title, industry, location, and pagination index.  
  - *Connections:* Output to Filter LinkedIn Profiles.  
  - *Edge Cases:* API quota limits, network timeouts, invalid API keys, malformed queries.

- **Filter LinkedIn Profiles**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Processes the API response to filter and retain only LinkedIn profile URLs and associated titles.  
  - *Configuration:* Contains custom code to parse JSON results, filter URLs containing "linkedin.com/in" or similar patterns.  
  - *Connections:* Output to Save to Google Sheets.  
  - *Edge Cases:* Unexpected API response formats, empty results, code runtime errors.

---

#### 2.3 Data Storage

**Overview:**  
This block saves the filtered LinkedIn profile data into a Google Sheet for easy review and outreach.

**Nodes Involved:**  
- Save to Google Sheets  

**Node Details:**

- **Save to Google Sheets**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends or updates rows in a specified Google Sheet with LinkedIn profile URLs and titles.  
  - *Configuration:* Uses Google Sheets API credentials; target spreadsheet and worksheet configured; columns mapped to profile data fields.  
  - *Connections:* Input from Filter LinkedIn Profiles.  
  - *Edge Cases:* Authentication errors, sheet access permissions, API rate limits, data format mismatches.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                          | Input Node(s)                 | Output Node(s)                 | Sticky Note                                                                                      |
|-----------------------------|----------------------------------|----------------------------------------|------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| When chat message received   | Chat Trigger (LangChain)          | Entry point for user queries            | -                            | AI Agent                      |                                                                                                 |
| AI Agent                    | LangChain Agent                   | Processes natural language to parameters| When chat message received, OpenAI Chat Model, Window Buffer Memory, Call n8n Workflow Tool | Set Search Page Numbers (via OpenAI)       |                                                                                                 |
| OpenAI Chat Model           | Language Model (OpenAI Chat)      | Provides conversational AI capabilities | -                            | AI Agent                      |                                                                                                 |
| Window Buffer Memory        | Memory Buffer (LangChain)         | Maintains conversation context          | -                            | AI Agent                      |                                                                                                 |
| Call n8n Workflow Tool      | Tool Workflow (LangChain)         | Invokes sub-workflows for extended tasks| -                            | AI Agent                      |                                                                                                 |
| Execute Workflow Trigger    | Execute Workflow Trigger          | Starts search with preset parameters    | -                            | OpenAI                        |                                                                                                 |
| OpenAI                     | OpenAI Node (LangChain)           | Refines search parameters or queries    | Execute Workflow Trigger      | Set Search Page Numbers       |                                                                                                 |
| Set Search Page Numbers     | Set Node                         | Defines pagination for search requests  | OpenAI                       | Split Search Pages            |                                                                                                 |
| Split Search Pages          | SplitOut Node                   | Splits pagination array into single pages| Set Search Page Numbers       | Google Custom Search API Request |                                                                                                 |
| Google Custom Search API Request | HTTP Request                  | Executes Google Custom Search API calls | Split Search Pages            | Filter LinkedIn Profiles      |                                                                                                 |
| Filter LinkedIn Profiles    | Code Node (JavaScript)            | Filters LinkedIn profile URLs from results| Google Custom Search API Request | Save to Google Sheets         |                                                                                                 |
| Save to Google Sheets       | Google Sheets Node                | Saves filtered profiles to Google Sheets| Filter LinkedIn Profiles      | -                             |                                                                                                 |
| Sticky Note                | Sticky Note                      | -                                      | -                            | -                             |                                                                                                 |
| Sticky Note1               | Sticky Note                      | -                                      | -                            | -                             |                                                                                                 |
| Sticky Note2               | Sticky Note                      | -                                      | -                            | -                             |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When chat message received" Node**  
   - Type: LangChain Chat Trigger  
   - Configure webhook to receive chat messages with prospecting queries.

2. **Create "OpenAI Chat Model" Node**  
   - Type: LangChain OpenAI Chat Model  
   - Configure with OpenAI API credentials.  
   - Use default or customized model parameters for conversational AI.

3. **Create "Window Buffer Memory" Node**  
   - Type: LangChain Memory Buffer Window  
   - Use default sliding window size to maintain conversation context.

4. **Create "Call n8n Workflow Tool" Node**  
   - Type: LangChain Tool Workflow  
   - Configure to call any required sub-workflows (optional, depending on your setup).

5. **Create "AI Agent" Node**  
   - Type: LangChain Agent  
   - Connect inputs:  
     - From "When chat message received" (main input)  
     - From "OpenAI Chat Model" (ai_languageModel)  
     - From "Window Buffer Memory" (ai_memory)  
     - From "Call n8n Workflow Tool" (ai_tool)  
   - No additional parameters needed unless customizing AI behavior.

6. **Create "Execute Workflow Trigger" Node**  
   - Type: Execute Workflow Trigger  
   - Configure with example or dynamic query parameters (e.g., jobTitle, companyIndustry, location).  
   - This node triggers the search process.

7. **Create "OpenAI" Node**  
   - Type: LangChain OpenAI Node  
   - Configure with OpenAI API credentials.  
   - Use to refine or generate search queries based on input parameters.

8. **Create "Set Search Page Numbers" Node**  
   - Type: Set Node  
   - Configure to set an array of page numbers for pagination (e.g., [1,2,3]).  
   - Connect input from "OpenAI" node.

9. **Create "Split Search Pages" Node**  
   - Type: SplitOut Node  
   - Connect input from "Set Search Page Numbers" node.  
   - Splits the pagination array into individual page numbers.

10. **Create "Google Custom Search API Request" Node**  
    - Type: HTTP Request  
    - Configure with Google Custom Search API credentials.  
    - Set method to GET.  
    - Construct query parameters using extracted jobTitle, companyIndustry, location, and page number.  
    - Connect input from "Split Search Pages" node.

11. **Create "Filter LinkedIn Profiles" Node**  
    - Type: Code Node (JavaScript)  
    - Write code to parse API response JSON, filter results to include only LinkedIn profile URLs (e.g., URLs containing "linkedin.com/in").  
    - Extract profile titles/headlines as well.  
    - Connect input from "Google Custom Search API Request" node.

12. **Create "Save to Google Sheets" Node**  
    - Type: Google Sheets Node  
    - Configure with Google Sheets API credentials.  
    - Set target spreadsheet and worksheet.  
    - Map columns to profile URL and title fields.  
    - Connect input from "Filter LinkedIn Profiles" node.

13. **Connect "When chat message received" node output to "AI Agent" node input.**

14. **Connect "AI Agent" node output to "Execute Workflow Trigger" node input or to "OpenAI" node as per workflow logic.**

15. **Ensure all API credentials (OpenAI, Google Custom Search, Google Sheets) are properly set up in n8n credentials manager.**

16. **Test the workflow with sample queries such as "Find marketing managers in Paris" or "Find software developers in London working in fintech".**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow automates LinkedIn prospecting using AI and Google Custom Search API.                  | Workflow description and use case.                                                                     |
| Adaptable to other platforms by changing search domain, URL filters, and AI prompts.                 | Allows customization for Twitter, GitHub, company websites, blogs, etc.                                |
| Requires OpenAI API key, Google Custom Search API credentials, and Google Sheets access.             | Prerequisites for operation.                                                                            |
| Setup instructions include API configuration, workflow import, credential linking, and testing.     | Setup guide summary.                                                                                    |
| Example input: "Find software developers in London working in fintech"                               | Demonstrates expected input and output.                                                                |
| #AIAgent #WebScraping #Automation #n8n #Workflow #LinkedInProspecting                               | Relevant hashtags for context and sharing.                                                             |
| For detailed AI prompt customization and code snippets, refer to the original workflow JSON or n8n community resources. | Additional resources for advanced users.                                                               |

---

This document provides a complete, structured reference to understand, reproduce, and modify the "AI Agent: Find the Right LinkedIn Profiles in Seconds" workflow in n8n. It covers all nodes, their roles, configurations, and integration points, enabling robust automation and easy troubleshooting.
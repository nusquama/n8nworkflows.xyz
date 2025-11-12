Personalized Outreach for Lawyers with LinkedIn Scraping, GPT-4o, Google Sheets

https://n8nworkflows.xyz/workflows/personalized-outreach-for-lawyers-with-linkedin-scraping--gpt-4o--google-sheets-4879


# Personalized Outreach for Lawyers with LinkedIn Scraping, GPT-4o, Google Sheets

---

### 1. Workflow Overview

This workflow automates personalized outreach for solo lawyers and small law firm founders by scraping LinkedIn profiles via Google Custom Search, researching each lead with AI, and generating tailored outreach messages. The gathered data and generated messages are stored and updated in Google Sheets for easy management.

**Target Use Cases:**  
- Legal marketing teams aiming to automate lead generation and personalized outreach.  
- Solo practitioners or small law firms wanting to efficiently connect with peers or potential clients.  
- Users integrating LinkedIn lead scraping with AI-driven content generation and CRM updates.

**Logical Blocks:**  
- **1.1 Scheduled Trigger & Initialization:** Starts the workflow on a schedule and initializes pagination parameters.  
- **1.2 LinkedIn Lead Scraping via Google Search:** Performs paginated Google Custom Search queries to scrape LinkedIn profiles of lawyers.  
- **1.3 Results Processing & Pagination Control:** Parses search results and controls loop continuation based on pagination availability.  
- **1.4 Data Storage to Google Sheets:** Appends or updates lead data into a Google Sheet to maintain a centralized database.  
- **1.5 AI-Powered Research Agent:** Uses an AI model to research each lead’s professional background and public information for outreach personalization.  
- **1.6 AI-Powered Outreach Message Generation:** Uses GPT-4o to generate tailored, concise outreach messages based on researched data.  
- **1.7 Final Outreach Message Storage:** Updates the Google Sheet with generated outreach messages.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Initialization

- **Overview:**  
  Triggers the workflow at defined intervals and sets the starting pagination index for Google search.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Set Fields

- **Node Details:**  

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Initiates workflow automatically on a time-based schedule (interval unspecified, defaults to once per minute).  
    - *Configuration:* Default interval with no specific parameters set.  
    - *Connections:* Outputs to "Set Fields" node.  
    - *Edge cases:* Schedule misconfiguration may cause missed or repeated runs.

  - **Set Fields**  
    - *Type:* Set Node  
    - *Role:* Initializes the pagination index (`currentStartIndex = 1`) to start from the first page of Google search results.  
    - *Configuration:* Single assignment setting `currentStartIndex` to 1.  
    - *Connections:* Outputs to "Google Search" node.  
    - *Edge cases:* Incorrect initial index could skip results or cause infinite loops.

#### 2.2 LinkedIn Lead Scraping via Google Search

- **Overview:**  
  Queries Google Custom Search API with a refined query targeting LinkedIn profiles of solo or founder lawyers, respecting pagination.

- **Nodes Involved:**  
  - Google Search

- **Node Details:**  

  - **Google Search**  
    - *Type:* HTTP Request  
    - *Role:* Sends HTTP GET requests to Google Custom Search API with query parameters.  
    - *Configuration:*  
      - URL: `https://www.googleapis.com/customsearch/v1`  
      - Query params:  
        - `key` and `cx` for API authentication (placeholders `YOUR_API_KEY_OR_CX_ID` need real credentials).  
        - Query: `("solo practitioner" OR "founder" OR "managing partner") lawyer "law firm" site:linkedin.com/in -government -gov -biglaw` — focused on LinkedIn profiles excluding government and big law firms.  
        - `start`: pagination start index, dynamically set from input JSON `startIndex` or defaults to 1.  
    - *Connections:* Inputs from "Set Fields" or "Pages Check" nodes; outputs to "Results" node.  
    - *Edge cases:*  
      - API key or cx invalid/expired causing auth failure.  
      - API rate limits or quota exceeded.  
      - Empty or malformed responses.  
      - Pagination index exceeding Google's maximum (typically max 100 results).  

#### 2.3 Results Processing & Pagination Control

- **Overview:**  
  Parses the Google Search response, extracts relevant lead data fields, checks for pagination continuation, and controls workflow looping.

- **Nodes Involved:**  
  - Results  
  - Pages  
  - Pages Check

- **Node Details:**  

  - **Results**  
    - *Type:* Code (JavaScript)  
    - *Role:* Processes Google API response, extracts lead name, position, LinkedIn URL, snippet, and image thumbnail; determines next pagination index and if more results exist.  
    - *Key logic:*  
      - Splits title to separate name and position.  
      - Includes pagination data (`startIndex`, `hasMoreResults`) on each item.  
      - Returns at least one item with pagination info if no results found.  
    - *Connections:* Inputs from "Google Search"; outputs to "Add to Google" and "Pages" nodes.  
    - *Edge cases:*  
      - Empty or missing fields in the response handled gracefully.  
      - Pagination beyond 100 results capped.  

  - **Pages**  
    - *Type:* Code (JavaScript)  
    - *Role:* Extracts pagination control flags (`continueLoop` and `startIndex`) from input items to inform the loop condition.  
    - *Connections:* Inputs from "Results" node; outputs to "Pages Check".  
    - *Edge cases:*  
      - Missing or undefined pagination fields; defaults set to continue with `startIndex=1`.  

  - **Pages Check**  
    - *Type:* If Node  
    - *Role:* Checks if the workflow should continue looping based on `continueLoop` boolean from "Pages" node.  
    - *Configuration:* Condition checks if `continueLoop` is true.  
    - *Connections:* If true, loops back to "Google Search" for next page; else ends the paginated search.  
    - *Edge cases:* Misinterpretation of the flag may cause infinite loops or premature termination.

#### 2.4 Data Storage to Google Sheets

- **Overview:**  
  Stores the scraped lead data into a Google Sheet, appending new entries for each lead.

- **Nodes Involved:**  
  - Add to Google  
  - Google Sheets (final)

- **Node Details:**  

  - **Add to Google**  
    - *Type:* Google Sheets  
    - *Role:* Appends lead name, position (mapped as "title"), and profile URL into the Google Sheet named "linkedin" (document ID provided).  
    - *Configuration:*  
      - Operation: Append  
      - Columns mapped: Name, title, profile_url  
      - Sheet Name: `gid=0` (Sheet1)  
      - Document ID: `1jajDVL6_sMlrK6cQNkKPqPpUVOPUst_A7hyMg0cuyrM`  
    - *Connections:* Inputs from "Results" node; outputs to "Research Agent" node.  
    - *Edge cases:*  
      - Google Sheets API limits or permission errors.  
      - Duplicate entries if not controlled.  

  - **Google Sheets (final)**  
    - *Type:* Google Sheets  
    - *Role:* Appends or updates rows with lead name and the personalized outreach message generated by AI, matching by `name`.  
    - *Configuration:*  
      - Operation: Append or Update (match on "name")  
      - Document ID: `1smDBk1Eh50sRyTEzEhzWkDnaWaexZHiHTxYlp2M6_J0` (different sheet)  
      - Columns: name, outreach message 1  
      - Sheet Name: `gid=0`  
    - *Connections:* Inputs from "Outreach Agent" node.  
    - *Edge cases:*  
      - Matching logic may fail if names differ slightly.  
      - API quota or permission errors.

#### 2.5 AI-Powered Research Agent

- **Overview:**  
  Uses an AI agent with an OpenRouter Chat model to research each lead, gathering publicly available relevant details to enrich outreach personalization.

- **Nodes Involved:**  
  - OpenRouter Chat Model  
  - Research Agent

- **Node Details:**  

  - **OpenRouter Chat Model**  
    - *Type:* LangChain AI Language Model (OpenRouter)  
    - *Role:* Provides the language model backend to the Research Agent.  
    - *Configuration:* Model set to "perplexity/sonar-pro" (custom or third-party AI model).  
    - *Connections:* Connected as `ai_languageModel` to "Research Agent".  
    - *Edge cases:* API key or service failure, model unavailability.  

  - **Research Agent**  
    - *Type:* LangChain Agent  
    - *Role:* Constructs a prompt with lead details (Name, Title, About) and instructs the AI to research the lawyer or founder focusing on relevant practice areas and public info; excludes irrelevant sources.  
    - *Configuration:*  
      - System message: Instructions on scope and tone for research output.  
      - Prompt includes dynamic insertion of lead data from previous nodes.  
    - *Connections:* Inputs from "Add to Google"; outputs to "Outreach Agent".  
    - *Edge cases:*  
      - AI generation failures or irrelevant outputs.  
      - Rate limits or service downtime.

#### 2.6 AI-Powered Outreach Message Generation

- **Overview:**  
  Generates a concise, personalized outreach icebreaker message for each lead based on AI research results.

- **Nodes Involved:**  
  - Outreach Agent

- **Node Details:**  

  - **Outreach Agent**  
    - *Type:* LangChain OpenAI Node  
    - *Role:* Uses GPT-4o to create a brief, professional, personalized outreach message starting with "Hey [name] ..." referring to practice area, community involvement, or personal brand.  
    - *Configuration:*  
      - Model: `gpt-4o`  
      - Messages:  
        - System role: Message instructions for tone and style.  
        - User content: Dynamically constructed with lead info from "Results" node.  
    - *Connections:* Input from "Research Agent"; outputs to "Google Sheets (final)".  
    - *Edge cases:*  
      - Prompt injection risks or irrelevant message generation.  
      - API quota or latency issues.

---

### 3. Summary Table

| Node Name           | Node Type                                                 | Functional Role                                 | Input Node(s)           | Output Node(s)                | Sticky Note                                         |
|---------------------|-----------------------------------------------------------|------------------------------------------------|-------------------------|------------------------------|----------------------------------------------------|
| Schedule Trigger     | Schedule Trigger                                          | Initiates workflow on schedule                   |                         | Set Fields                   |                                                    |
| Set Fields          | Set                                                      | Initializes pagination index for Google Search  | Schedule Trigger         | Google Search                |                                                    |
| Google Search        | HTTP Request                                            | Queries Google Custom Search API for LinkedIn profiles | Set Fields, Pages Check  | Results                     | Requires valid Google API Key and CX ID            |
| Results              | Code                                                     | Parses search results, extracts leads, handles pagination | Google Search            | Add to Google, Pages          | Handles empty results and pagination limits        |
| Pages                | Code                                                     | Extracts pagination control flags                 | Results                  | Pages Check                  |                                                    |
| Pages Check          | If                                                       | Controls loop continuation based on pagination   | Pages                    | Google Search                |                                                    |
| Add to Google        | Google Sheets                                           | Appends scraped leads to Google Sheet             | Results                   | Research Agent               | Sheet: linkedin; watch for API limits               |
| OpenRouter Chat Model| LangChain AI Language Model                             | Provides AI model for research agent              |                         | Research Agent (ai_languageModel) |                                                    |
| Research Agent       | LangChain Agent                                         | Researches lead details for outreach personalization | Add to Google             | Outreach Agent               | Excludes irrelevant sources, focuses on solo/small law firms |
| Outreach Agent       | LangChain OpenAI Node                                   | Generates personalized outreach message           | Research Agent            | Google Sheets (final)         | Uses GPT-4o, keeps message brief and professional  |
| Google Sheets (final)| Google Sheets                                           | Updates Google Sheet with outreach messages       | Outreach Agent            |                              | Sheet: linkedin; matches by name                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure interval as desired (e.g., every hour).  
   - Connect output to "Set Fields" node.

2. **Create Set Fields Node ("Set Fields")**  
   - Type: Set  
   - Add a field `currentStartIndex` (Number) with value `1`.  
   - Connect output to "Google Search" node.

3. **Create HTTP Request Node ("Google Search")**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://www.googleapis.com/customsearch/v1`  
   - Query Parameters:  
     - `key`: Your Google API Key  
     - `cx`: Your Custom Search Engine ID  
     - `q`: `("solo practitioner" OR "founder" OR "managing partner") lawyer "law firm" site:linkedin.com/in -government -gov -biglaw`  
     - `start`: Set to expression `={{ $json.currentStartIndex || 1 }}`  
   - Connect output to "Results" node.

4. **Create Code Node ("Results")**  
   - Type: Code (JavaScript)  
   - Paste the provided code to parse items, extract name, position, link, snippet, thumbnail, and pagination.  
   - Connect outputs to "Add to Google" and "Pages" nodes.

5. **Create Code Node ("Pages")**  
   - Type: Code (JavaScript)  
   - Paste code to extract `continueLoop` and `startIndex` from input.  
   - Connect output to "Pages Check" node.

6. **Create If Node ("Pages Check")**  
   - Type: If  
   - Condition: Expression checks if `{{$json.continueLoop}}` is true.  
   - True branch connects back to "Google Search" (to continue pagination).  
   - False branch ends the loop.

7. **Create Google Sheets Node ("Add to Google")**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Your Google Sheets file ID (e.g., `1jajDVL6_sMlrK6cQNkKPqPpUVOPUst_A7hyMg0cuyrM`)  
   - Sheet Name: `Sheet1` or `gid=0`  
   - Columns mapped:  
     - Name: `={{ $json.name }}`  
     - title: `={{ $json.position }}`  
     - profile_url: `={{ $json.link }}`  
   - Connect output to "Research Agent" node.

8. **Create LangChain OpenRouter Model Node ("OpenRouter Chat Model")**  
   - Type: LangChain AI Language Model (OpenRouter)  
   - Model: `perplexity/sonar-pro` (or substitute with valid model)  
   - Connect output to "Research Agent" as `ai_languageModel`.

9. **Create LangChain Agent Node ("Research Agent")**  
   - Type: LangChain Agent  
   - System Message: Instruction to research lawyer info focusing on solo/small firms, avoiding irrelevant sources.  
   - Prompt: Include dynamic values for Name, Title, About from previous node.  
   - Connect output to "Outreach Agent".

10. **Create LangChain OpenAI Node ("Outreach Agent")**  
    - Type: LangChain OpenAI  
    - Model: `gpt-4o`  
    - Messages:  
      - System: Instructions for generating short, genuine outreach icebreaker starting with "Hey [name] ...".  
      - User: Dynamic insertion of lead data (Name, Position, About).  
    - Connect output to final Google Sheets node.

11. **Create Google Sheets Node ("Google Sheets (final)")**  
    - Type: Google Sheets  
    - Operation: Append or Update  
    - Document ID: Your Google Sheets file ID (e.g., `1smDBk1Eh50sRyTEzEhzWkDnaWaexZHiHTxYlp2M6_J0`)  
    - Sheet Name: `Sheet1` or `gid=0`  
    - Matching Column: `name`  
    - Columns mapped:  
      - name: `={{ $('Results').all()[ $itemIndex ].json.name }}`  
      - outreach message 1: `={{ $json.message.content }}`  
    - No output connection (end node).

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The Google Custom Search API requires valid API Key and CX ID credentials; replace placeholders accordingly. | Google Custom Search API documentation: https://developers.google.com/custom-search/v1/overview  |
| GPT-4o is used for personalized message generation; ensure you have appropriate OpenAI API access.           | OpenAI API docs: https://platform.openai.com/docs/models/gpt-4o                                |
| The workflow targets solo practitioners and small firms; large firm or government attorneys are excluded by design. | This focus is embedded in search query and AI prompt instructions.                             |
| The Google Sheets nodes append and update two different sheets; ensure correct permissions and sheet IDs.     | Google Sheets API documentation: https://developers.google.com/sheets/api                      |
| The AI research agent uses OpenRouter's "perplexity/sonar-pro" model; verify service availability and credentials. | OpenRouter LangChain integration docs: https://docs.openrouter.ai/                            |
| Pagination in Google Search is capped at 100 results; ensure this limit fits your use case.                    | Google Search API limits: https://developers.google.com/custom-search/v1/using_rest#pagination  |

---

*Disclaimer:*  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.
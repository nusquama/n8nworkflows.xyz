AI-Powered Product Hunt Data Collection & Translation with GPT-4 and Google Sheets

https://n8nworkflows.xyz/workflows/ai-powered-product-hunt-data-collection---translation-with-gpt-4-and-google-sheets-10791


# AI-Powered Product Hunt Data Collection & Translation with GPT-4 and Google Sheets

### 1. Workflow Overview

This workflow automates the daily collection, enrichment, and storage of Product Hunt tool data. It targets users who want an up-to-date, bilingual (English/French) catalog of Product Hunt launches enriched with AI-generated translations, category mappings, technology extraction, and documentation. The process includes pagination handling to retrieve all new posts incrementally.

The workflow is logically divided into the following blocks:

- **1.1 Initialization**: Starts the workflow on a daily schedule and initializes pagination.
- **1.2 Data Fetching**: Queries the Product Hunt GraphQL API for the latest tools using cursor-based pagination and restructures API responses.
- **1.3 Duplicate Check**: Checks if a tool already exists in the Google Sheets database by title to avoid processing duplicates.
- **1.4 AI Enrichment**: Sends new tools to an AI agent that translates text, updates category translations using a Google Sheets dictionary, extracts technology keywords, and generates documentation in English and French.
- **1.5 Persistence & Control**: Saves enriched tool data back to Google Sheets, updates pagination cursors, and controls looping with rate limits and safety checks to iterate over all pages.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization

- **Overview:**  
  This block triggers the workflow daily at 9 AM and sets the initial cursor for pagination to `null` to begin fetching from the first page.

- **Nodes Involved:**  
  - Daily Trigger at 9 AM  
  - Initialize Starting Cursor  
  - Workflow Overview (Sticky Note)

- **Node Details:**  
  - *Daily Trigger at 9 AM*  
    - Type: Schedule Trigger  
    - Role: Triggers workflow execution daily at 9 AM.  
    - Configuration: Interval set to trigger at hour 9 daily.  
    - Inputs: None (start node).  
    - Outputs: Initializes the cursor setting node.  
    - Failures: None typical; ensure n8n server is running.  

  - *Initialize Starting Cursor*  
    - Type: Set Node  
    - Role: Sets the initial pagination cursor to string `"null"`.  
    - Configuration: Assigns variable `cursor` with value `"null"`.  
    - Input: Triggered by schedule node.  
    - Output: Passes cursor to API fetch node.  
    - Failures: None expected unless variable assignment fails.

  - *Workflow Overview*  
    - Type: Sticky Note  
    - Role: Documentation describing workflow purpose and setup steps.  
    - Static content for user reference only.

#### 1.2 Data Fetching

- **Overview:**  
  Retrieves Product Hunt posts via GraphQL API using cursor pagination, then restructures the raw data into a clean JSON format with relevant fields.

- **Nodes Involved:**  
  - Fetch Products from Product Hunt API  
  - Restructure API Response to JSON  
  - Section: Data Fetching (Sticky Note)

- **Node Details:**  
  - *Fetch Products from Product Hunt API*  
    - Type: HTTP Request (POST)  
    - Role: Calls Product Hunt’s GraphQL endpoint with OAuth2 authentication.  
    - Configuration: Sends query requesting 5 posts ordered by ranking, with cursor-based pagination using variable `cursor`.  
    - Inputs: Receives `cursor` from initialization or previous pagination step.  
    - Outputs: Returns raw JSON containing posts, pagination info.  
    - Failures: OAuth errors, network timeouts, API limits.  

  - *Restructure API Response to JSON*  
    - Type: Code (JavaScript)  
    - Role: Extracts relevant fields from API response edges, maps to defined schema fields for each post.  
    - Configuration: Maps title, description, topics, media URLs, and initializes empty translation and documentation fields.  
    - Inputs: API JSON response.  
    - Outputs: Array of structured post JSON objects.  
    - Failures: Unexpected API response format, missing fields.  

#### 1.3 Duplicate Check

- **Overview:**  
  For each fetched tool, checks Google Sheets "Tools" sheet by title to determine if the tool is already known. Only new tools proceed to enrichment.

- **Nodes Involved:**  
  - Lookup Existing Tool by Title (Google Sheets Read)  
  - Is Tool New? (If Node)  
  - Section: Duplicate Check (Sticky Note)

- **Node Details:**  
  - *Lookup Existing Tool by Title*  
    - Type: Google Sheets (Read)  
    - Role: Searches "Tools" sheet for existing entries matching current tool title.  
    - Configuration: Matches on column `Title` with exact equality.  
    - Inputs: Structured tool JSON from previous block.  
    - Outputs: Matching rows if any, or empty.  
    - Failures: Google Sheets API errors, auth issues.  

  - *Is Tool New?*  
    - Type: If Node  
    - Role: Checks if lookup result is empty (indicating new tool).  
    - Configuration: Condition `isEmpty()` true means new tool.  
    - Inputs: Lookup results.  
    - Outputs:  
      - True path: New tool → send to AI enrichment.  
      - False path: Existing tool → skip enrichment, proceed to pagination extraction.  
    - Failures: Expression evaluation errors if input malformed.

#### 1.4 AI Enrichment

- **Overview:**  
  Uses an AI agent powered by OpenAI GPT-4.1-mini to translate tool titles and descriptions into French, map categories via a Google Sheets dictionary (adding new translations if missing), extract explicitly mentioned technologies, generate function summaries, and write English/French documentation.

- **Nodes Involved:**  
  - AI Agent: Translate & Enrich Product Data  
  - OpenAI GPT-4.1 Mini (Language Model)  
  - Lookup Category Dictionary (Google Sheets Read)  
  - Update Category Dictionary (Google Sheets Append/Update)  
  - Clean & Parse AI JSON Output (Code)  
  - Section: AI Enrichment (Sticky Note)

- **Node Details:**  
  - *AI Agent: Translate & Enrich Product Data*  
    - Type: LangChain Agent Node  
    - Role: Orchestrates AI prompt interactions combining translation, category lookup/update, tech extraction, and documentation generation.  
    - Configuration:  
      - Inputs full JSON tool data as string.  
      - System message guides AI through stepwise tasks and output formatting (pure JSON, no markdown).  
      - Connects to Google Sheets category dictionary nodes as AI tools for lookup and update.  
      - Limits iterations to max 8 to avoid infinite loops.  
    - Inputs: New tool JSON from duplicate check.  
    - Outputs: Raw AI JSON string with enriched fields.  
    - Failures: API rate limits, malformed AI output, translation inaccuracies.  

  - *OpenAI GPT-4.1 Mini*  
    - Type: Language Model (Chat)  
    - Role: Provides AI completions for the agent.  
    - Configuration: Model set to `gpt-4.1-mini`.  
    - Inputs: Prompts from agent node.  
    - Outputs: AI-generated text outputs.  
    - Failures: API key issues, timeout, quota exceeded.  

  - *Lookup Category Dictionary (Read)*  
    - Type: Google Sheets (Read)  
    - Role: Reads category dictionary sheet to find French translations for English categories.  
    - Configuration: Reads entire sheet with columns `Category Name English` and `Category Name French`.  
    - Inputs: Invoked by AI agent for category lookup.  
    - Outputs: Category dictionary data.  
    - Failures: Google Sheets access errors.  

  - *Update Category Dictionary (Write)*  
    - Type: Google Sheets AppendOrUpdate  
    - Role: Adds new category translations if missing, preventing duplicates.  
    - Configuration: Matches on `Category Name English` to avoid duplicates, appends new rows as needed.  
    - Inputs: Invoked by AI agent for updating dictionary.  
    - Outputs: Updated sheet.  
    - Failures: Concurrent writes, API errors.  

  - *Clean & Parse AI JSON Output*  
    - Type: Code (JavaScript)  
    - Role: Cleans AI raw output by removing markdown formatting and parses JSON string into n8n JSON objects.  
    - Configuration: Handles empty output and invalid JSON with error reporting in output JSON.  
    - Inputs: Raw AI JSON string.  
    - Outputs: Parsed JSON object or error object for debugging.  
    - Failures: Malformed AI output or unexpected formatting.

#### 1.5 Persistence & Control

- **Overview:**  
  Saves the enriched tool data into the Google Sheets "Tools" sheet, updates pagination cursor info, and manages the loop to fetch subsequent pages with safety limits and rate limiting.

- **Nodes Involved:**  
  - Save Enriched Tool to Sheet (Google Sheets AppendOrUpdate)  
  - Extract Pagination Info (Set)  
  - Has Next Page? (If Node)  
  - Continue Loop (NoOp)  
  - Rate Limit: Wait 15 Seconds (Wait)  
  - Safety Limit Check (>50 iterations) (If Node)  
  - Section: Persistence & Control (Sticky Note)

- **Node Details:**  
  - *Save Enriched Tool to Sheet*  
    - Type: Google Sheets AppendOrUpdate  
    - Role: Updates or appends enriched tool data to the "Tools" sheet keyed on title.  
    - Configuration: Auto-maps all enriched fields including translations and documentation.  
    - Inputs: Parsed AI JSON enriched data.  
    - Outputs: Triggers pagination extraction.  
    - Failures: API quota, concurrent writes.  

  - *Extract Pagination Info*  
    - Type: Set Node  
    - Role: Extracts `endCursor` and `hasNextPage` from last API response to control pagination.  
    - Inputs: Latest API response from Product Hunt.  
    - Outputs: Sets pagination variables for decision nodes.  
    - Failures: Missing or malformed pagination info causes loop issues.  

  - *Has Next Page?*  
    - Type: If Node  
    - Role: Checks if more pages exist to continue fetching.  
    - Outputs:  
      - True: Continue loop.  
      - False: End workflow.  
    - Failures: Logic errors may cause infinite looping or early stop.  

  - *Continue Loop*  
    - Type: NoOp Node  
    - Role: Placeholder to logically separate loop continuation steps.  
    - Outputs: Triggers rate limit wait node.  

  - *Rate Limit: Wait 15 Seconds*  
    - Type: Wait Node  
    - Role: Pauses workflow 15 seconds between API calls to respect rate limits.  
    - Outputs: Triggers safety limit check.  
    - Failures: None expected.  

  - *Safety Limit Check (>50 iterations)*  
    - Type: If Node  
    - Role: Prevents infinite loops by limiting pagination cycles to 50 iterations per run.  
    - Outputs:  
      - True: Stops pagination.  
      - False: Triggers next API fetch.  
    - Failures: Incorrect count may lead to premature stop or infinite loop.  

---

### 3. Summary Table

| Node Name                          | Node Type                    | Functional Role                                 | Input Node(s)                      | Output Node(s)                         | Sticky Note                                                                                      |
|-----------------------------------|------------------------------|------------------------------------------------|----------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow Overview                 | Sticky Note                  | Describes workflow purpose and setup           | None                             | None                                  | ## 📋 Product Hunt AI Enrichment Workflow... (full overview and setup instructions)            |
| Daily Trigger at 9 AM             | Schedule Trigger             | Triggers workflow daily at 9 AM                 | None                             | Initialize Starting Cursor             | ## Initialization Starts the workflow daily and sets the initial cursor for pagination.        |
| Initialize Starting Cursor        | Set                         | Sets initial pagination cursor ("null")        | Daily Trigger at 9 AM            | Fetch Products from Product Hunt API  | ## Initialization Starts the workflow daily and sets the initial cursor for pagination.        |
| Fetch Products from Product Hunt API | HTTP Request               | Calls Product Hunt API with OAuth2 and cursor  | Initialize Starting Cursor       | Restructure API Response to JSON      | ## Data Fetching Calls the Product Hunt API and restructures each post into clean JSON fields.  |
| Restructure API Response to JSON | Code                        | Extracts and cleans API response data           | Fetch Products from Product Hunt API | Lookup Existing Tool by Title       | ## Data Fetching Calls the Product Hunt API and restructures each post into clean JSON fields.  |
| Lookup Existing Tool by Title     | Google Sheets (Read)         | Checks for existing tools by title in sheet     | Restructure API Response to JSON | Is Tool New?                         | ## Duplicate Check Looks up each tool by title in Google Sheets and only processes new entries.|
| Is Tool New?                     | If                          | Filters out existing tools, passes new ones     | Lookup Existing Tool by Title    | AI Agent: Translate & Enrich Product Data (true), Extract Pagination Info (false) | ## Duplicate Check Looks up each tool by title in Google Sheets and only processes new entries.|
| AI Agent: Translate & Enrich Product Data | LangChain Agent Node    | AI translation, category mapping, tech extraction, doc generation | Is Tool New?                    | Clean & Parse AI JSON Output          | ## AI Enrichment Uses an AI agent to translate text, map categories, extract tech, and generate documentation. |
| OpenAI GPT-4.1 Mini              | AI Language Model            | Provides GPT-4.1-mini completions for agent     | AI Agent                       | AI Agent                            | ## AI Enrichment Uses an AI agent to translate text, map categories, extract tech, and generate documentation. |
| Lookup Category Dictionary (Read)| Google Sheets (Read)         | Reads category dictionary for translations      | AI Agent (ai_tool)              | AI Agent (ai_tool)                   | ## AI Enrichment Uses an AI agent to translate text, map categories, extract tech, and generate documentation. |
| Update Category Dictionary (Write)| Google Sheets AppendOrUpdate | Adds new category translations without duplicates | AI Agent (ai_tool)              | AI Agent (ai_tool)                   | ## AI Enrichment Uses an AI agent to translate text, map categories, extract tech, and generate documentation. |
| Clean & Parse AI JSON Output      | Code                        | Cleans and parses AI raw JSON output             | AI Agent                       | Save Enriched Tool to Sheet           | ## AI Enrichment Uses an AI agent to translate text, map categories, extract tech, and generate documentation. |
| Save Enriched Tool to Sheet       | Google Sheets AppendOrUpdate | Saves enriched tool data to Tools sheet          | Clean & Parse AI JSON Output    | Extract Pagination Info               | ## Save & Loop Control Writes enriched tools to Google Sheets and manages pagination until all pages are processed. |
| Extract Pagination Info           | Set                         | Extracts pagination cursor and next page flag   | Save Enriched Tool to Sheet     | Has Next Page?                       | ## Save & Loop Control Writes enriched tools to Google Sheets and manages pagination until all pages are processed. |
| Has Next Page?                   | If                          | Checks if more pages exist to continue loop     | Extract Pagination Info         | Continue Loop (true), End (false)   | ## Save & Loop Control Writes enriched tools to Google Sheets and manages pagination until all pages are processed. |
| Continue Loop                    | NoOp                        | Placeholder to continue loop                      | Has Next Page?                 | Rate Limit: Wait 15 Seconds           | ## Save & Loop Control Writes enriched tools to Google Sheets and manages pagination until all pages are processed. |
| Rate Limit: Wait 15 Seconds      | Wait                        | Waits 15 seconds between API calls                | Continue Loop                  | Safety Limit Check (>50 iterations)   | ## Save & Loop Control Writes enriched tools to Google Sheets and manages pagination until all pages are processed. |
| Safety Limit Check (>50 iterations)| If                         | Stops loop after 50 iterations to avoid infinite loops | Rate Limit: Wait 15 Seconds   | Fetch Products from Product Hunt API (false), End (true) | ## Save & Loop Control Writes enriched tools to Google Sheets and manages pagination until all pages are processed. |
| Section: Initialization          | Sticky Note                 | Notes on initialization block                      | None                          | None                                 | ## Initialization Starts the workflow daily and sets the initial cursor for pagination.        |
| Section: Data Fetching           | Sticky Note                 | Notes on data fetching block                        | None                          | None                                 | ## Data Fetching Calls the Product Hunt API and restructures each post into clean JSON fields.  |
| Section: Duplicate Check         | Sticky Note                 | Notes on duplicate check block                      | None                          | None                                 | ## Duplicate Check Looks up each tool by title in Google Sheets and only processes new entries.|
| Section: AI Enrichment           | Sticky Note                 | Notes on AI enrichment block                        | None                          | None                                 | ## AI Enrichment Uses an AI agent to translate text, map categories, extract tech, and generate documentation. |
| Section: Persistence & Control   | Sticky Note                 | Notes on saving and loop control                    | None                          | None                                 | ## Save & Loop Control Writes enriched tools to Google Sheets and manages pagination until all pages are processed. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger - "Daily Trigger at 9 AM"**  
   - Node Type: Schedule Trigger  
   - Parameters: Set to trigger once daily at 9 AM.  
   - No credentials needed.

2. **Create Set Node - "Initialize Starting Cursor"**  
   - Node Type: Set  
   - Parameters: Assign variable `cursor` with value `"null"` (string).  
   - Connect input from "Daily Trigger at 9 AM".

3. **Create HTTP Request Node - "Fetch Products from Product Hunt API"**  
   - Node Type: HTTP Request (POST)  
   - URL: `https://api.producthunt.com/v2/api/graphql`  
   - Method: POST  
   - Authentication: OAuth2 (Product Hunt API OAuth2 credentials required)  
   - Headers: Content-Type and Accept set to `application/json`  
   - Body (JSON): GraphQL query requesting 5 posts ordered by ranking, using variable `cursor` for pagination as:  
     ```json
     {
       "query": "{ posts(order: RANKING, first:5 , after: \"{{ $json.cursor }}\") { edges { cursor node { name description tagline website thumbnail { url } topics { edges { node { name } } } media { videoUrl type url } productLinks { type url } } } pageInfo { endCursor hasNextPage } } }"
     }
     ```
   - Connect input from "Initialize Starting Cursor".

4. **Create Code Node - "Restructure API Response to JSON"**  
   - Node Type: Code (JavaScript)  
   - Code: Extract fields from API response nodes and map to desired output schema including empty translation fields.  
   - Connect input from "Fetch Products from Product Hunt API".

5. **Create Google Sheets Node - "Lookup Existing Tool by Title"**  
   - Node Type: Google Sheets (Read)  
   - Credentials: Google Sheets OAuth2 connected to your Google account.  
   - Sheet: Select the "Tools" sheet in your target spreadsheet.  
   - Filter: Lookup rows where `Title` equals current item's `Title`.  
   - Connect input from "Restructure API Response to JSON".  

6. **Create If Node - "Is Tool New?"**  
   - Node Type: If  
   - Condition: Check if lookup results are empty (`isEmpty()` true).  
   - Connect input from "Lookup Existing Tool by Title".  
   - True path → New tools; False path → existing tools.

7. **Create LangChain Agent Node - "AI Agent: Translate & Enrich Product Data"**  
   - Node Type: LangChain Agent  
   - Credentials: OpenAI API key for GPT-4.1-mini  
   - Parameters:  
     - Text input: JSON stringified current tool data.  
     - System message: Detailed prompt with tasks to translate, map categories with dictionary, extract tech, generate documentation, and preserve fields.  
     - Max iterations: 8  
     - Connect AI language model input to "OpenAI GPT-4.1 Mini" node.  
     - Connect AI tools input to "Lookup Category Dictionary (Read)" and "Update Category Dictionary (Write)".  
   - Connect input from "Is Tool New?" (True output).

8. **Create OpenAI GPT-4.1 Mini Node**  
   - Node Type: AI Language Model (Chat)  
   - Credentials: OpenAI API key  
   - Model: `gpt-4.1-mini`  
   - Connect input from "AI Agent: Translate & Enrich Product Data".

9. **Create Google Sheets Node - "Lookup Category Dictionary (Read)"**  
   - Node Type: Google Sheets (Read)  
   - Credentials: Same Google Sheets OAuth2 account  
   - Sheet: Category Dictionary sheet with columns `Category Name English` and `Category Name French`.  
   - Connect as AI tool input to "AI Agent: Translate & Enrich Product Data".

10. **Create Google Sheets Node - "Update Category Dictionary (Write)"**  
    - Node Type: Google Sheets AppendOrUpdate  
    - Credentials: Same as above  
    - Sheet: Same Category Dictionary sheet  
    - Mapping: Match on `Category Name English` to avoid duplicates, append new translations if missing.  
    - Connect as AI tool input to "AI Agent: Translate & Enrich Product Data".

11. **Create Code Node - "Clean & Parse AI JSON Output"**  
    - Node Type: Code (JavaScript)  
    - Code: Strip markdown ticks from AI output, parse JSON, handle errors gracefully.  
    - Connect input from "AI Agent: Translate & Enrich Product Data".

12. **Create Google Sheets Node - "Save Enriched Tool to Sheet"**  
    - Node Type: Google Sheets AppendOrUpdate  
    - Credentials: Same Google Sheets OAuth2 account  
    - Sheet: "Tools" sheet  
    - Mapping: Auto-map all enriched fields including translations and documentation.  
    - Matching column: `Title` (to update existing or append new).  
    - Connect input from "Clean & Parse AI JSON Output".

13. **Create Set Node - "Extract Pagination Info"**  
    - Node Type: Set  
    - Parameters: Extract variables:  
      - `cursor` = `$('Fetch Products from Product Hunt API').first().json.data.posts.pageInfo.endCursor`  
      - `hasnextpage` = `$('Fetch Products from Product Hunt API').first().json.data.posts.pageInfo.hasNextPage`  
    - Connect input from "Save Enriched Tool to Sheet".

14. **Create If Node - "Has Next Page?"**  
    - Node Type: If  
    - Condition: `hasnextpage` is true  
    - Connect input from "Extract Pagination Info".  
    - True path → Continue Loop; False path → end workflow.

15. **Create NoOp Node - "Continue Loop"**  
    - Node Type: NoOp  
    - Connect input from "Has Next Page?" (true output).

16. **Create Wait Node - "Rate Limit: Wait 15 Seconds"**  
    - Node Type: Wait  
    - Parameters: Wait 15 seconds  
    - Connect input from "Continue Loop".

17. **Create If Node - "Safety Limit Check (>50 iterations)"**  
    - Node Type: If  
    - Condition: `$runIndex > 50`  
    - Connect input from "Rate Limit: Wait 15 Seconds".  
    - True path → End workflow; False path → back to "Fetch Products from Product Hunt API" to continue pagination.

18. **Connect False branch of "Is Tool New?" directly to "Extract Pagination Info"**  
    - This allows skipping AI enrichment for existing tools and continuing pagination.

19. **Adjust all credentials and sheet/document IDs to your environment**  
    - Product Hunt OAuth2 credentials  
    - Google Sheets OAuth2 account with access to both "Tools" and "Category Dictionary" sheets  
    - OpenAI API key with GPT-4.1-mini access

20. **Validate and test workflow**  
    - Run manually first to verify data flow and correct mapping.  
    - Enable daily trigger once confirmed working.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                           |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
| The workflow includes a safety limit of 50 pagination iterations per run to prevent infinite loops. | Important to avoid excessive API calls or stuck loops during daily runs.                                                  |
| Category Dictionary is a Google Sheet with two columns: "Category Name English" and "Category Name French". | Used to avoid translating categories repeatedly and prevent duplicates.                                                  |
| AI agent prompt enforces strict JSON output without markdown formatting to ensure parsing correctness. | Critical for automation tools like n8n to parse AI responses without errors.                                              |
| Product Hunt API uses OAuth2 authentication; ensure token refresh and permissions are configured.   | OAuth2 credentials setup required for the HTTP Request node to successfully retrieve data.                                |
| OpenAI GPT-4.1-mini model is used as a cost-effective alternative to full GPT-4 with sufficient capability. | This balances cost and performance for translation and content generation tasks.                                           |
| Rate limiting (15 seconds wait) is applied between API calls to respect Product Hunt API usage policies. | Prevents throttling or temporary bans due to excessive request volume.                                                    |
| The workflow is designed to build a bilingual (English/French) Product Hunt tool catalog with AI-enriched metadata. | Useful for building multilingual databases, marketing, or research repositories.                                          |
| Workflow overview sticky notes provide user guidance and setup instructions directly inside n8n.    | Helpful for maintenance and onboarding new users without external documentation.                                          |

---

*Disclaimer:* The provided text is exclusively from an automated workflow built with n8n, an integration and automation tool. This process strictly complies with content policies and contains no illegal or protected elements. All manipulated data is legal and publicly accessible.
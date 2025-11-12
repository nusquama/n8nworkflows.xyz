OpenAI GPT-3: Company Enrichment from website content

https://n8nworkflows.xyz/workflows/openai-gpt-3--company-enrichment-from-website-content-1862


# OpenAI GPT-3: Company Enrichment from website content

### 1. Workflow Overview

This workflow is designed to enrich a list of companies with detailed insights extracted from their website content, using OpenAI’s GPT-3 model. It targets users who maintain company lists (e.g., sales or marketing teams) and want to add valuable contextual information—such as market type, industry, target audience, and value proposition—to enhance personalization and strategic outreach.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Trigger the workflow and read company data (website URLs) from a Google Sheets document.
- **1.2 Batch Processing:** Split the list into manageable batches to process each company sequentially.
- **1.3 Website Content Extraction:** For each company URL, retrieve and clean the website’s HTML content.
- **1.4 AI Analysis:** Submit cleaned website content to OpenAI GPT-3 for semantic enrichment.
- **1.5 Data Parsing and Merging:** Parse the AI response, merge it with existing data, and update the original Google Sheet.
- **1.6 Throttling / Loop Control:** Introduce a wait step before processing the next batch to comply with rate limits or operational pacing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Initializes the workflow execution manually and reads company domains from Google Sheets.
- **Nodes Involved:** 
  - When clicking "Execute Workflow"
  - Read Google Sheets

- **Node Details:**

  - **When clicking "Execute Workflow"**
    - Type: Manual trigger
    - Role: Starts execution on user command
    - Config: No parameters
    - Inputs: None
    - Outputs: Connects to "Read Google Sheets"
    - Edge cases: None, manual start only

  - **Read Google Sheets**
    - Type: Google Sheets read node
    - Role: Fetches the list of companies (domains) from a Google Sheet
    - Config:
      - Document URL: Google Sheet with company list
      - Sheet: gid=0 (Sheet1)
      - Credentials: OAuth2 Google Sheets account
    - Inputs: From manual trigger
    - Outputs: Array of company data objects with at least a "Domain" field
    - Edge cases:
      - Google API errors (auth, quota)
      - Empty or malformed sheet data
      - Network timeouts

#### 1.2 Batch Processing

- **Overview:** Splits the entire list of company domains into smaller batches for sequential processing.
- **Nodes Involved:** 
  - Split In Batches
  - Merge (first connection)

- **Node Details:**

  - **Split In Batches**
    - Type: Split In Batches node
    - Role: Processes the list in batches (default batch size not specified, likely default 1)
    - Config: Default options, no batch size override
    - Inputs: From "Read Google Sheets"
    - Outputs: Single item per batch sent to "HTTP Request"
    - Outputs also connect to "Merge" node for combining results later
    - Edge cases:
      - Large lists causing long execution time
      - Batch size too large for API limits
      - No items to process

  - **Merge (initial merge by position)**
    - Type: Merge node
    - Role: Combines original batch input with enriched data later
    - Config: Combine mode, merge by position
    - Inputs: One input gets raw batch items, the other enriched results downstream
    - Outputs: Connects to "Update Google Sheets"
    - Edge cases:
      - Misaligned batch indices causing incorrect merges

#### 1.3 Website Content Extraction

- **Overview:** Retrieves the HTML content of company websites and extracts the core HTML for processing.
- **Nodes Involved:** 
  - HTTP Request
  - HTML Extract
  - Clean Content (Code node)

- **Node Details:**

  - **HTTP Request**
    - Type: HTTP Request node
    - Role: Fetch company website HTML content
    - Config:
      - URL dynamically built as `https://www.{Domain}`
      - Option enabled to follow redirects
      - Continue on failure (to avoid halting workflow on bad URLs)
    - Inputs: From "Split In Batches"
    - Outputs: Raw HTML content passed to "HTML Extract"
    - Edge cases:
      - Invalid domain or URL errors
      - Timeouts or unreachable websites
      - Redirect loops

  - **HTML Extract**
    - Type: HTML Extract node
    - Role: Extract the entire HTML content using CSS selector `html`
    - Config:
      - Extraction key: "body"
      - CSS selector: `html`
    - Inputs: From "HTTP Request"
    - Outputs: Extracted HTML content to "Clean Content"
    - Edge cases:
      - Empty or malformed HTML
      - Extraction failure if selector not found

  - **Clean Content (Code node)**
    - Type: Code node (JavaScript)
    - Role: Clean and trim extracted HTML content; slice to max length
    - Config:
      - Runs once per item
      - Removes leading/trailing whitespace and line breaks
      - Replaces multiple spaces with single spaces
      - Creates `content` and `contentShort` fields (max 10,000 characters)
    - Inputs: From "HTML Extract"
    - Outputs: Cleaned content to "OpenAI" node
    - Edge cases:
      - Missing `body` field
      - String manipulation errors

#### 1.4 AI Analysis

- **Overview:** Feeds the cleaned website content to OpenAI GPT-3 to generate structured business insights.
- **Nodes Involved:** 
  - OpenAI
  - Parse JSON (Code node)

- **Node Details:**

  - **OpenAI**
    - Type: OpenAI node
    - Role: Submit prompt to GPT-3 and receive JSON response with company enrichment data
    - Config:
      - Prompt: Customized with company domain and content snippet
      - Instructions to output JSON with fields: value_proposition (less than 25 words, casual tone), industry (from fixed list), target_audience (from fixed list), market (B2B or B2C)
      - Options: temperature 0 (deterministic), max tokens 120, top_p 1
      - Credentials: OpenAI API key configured
      - Continue on failure enabled
    - Inputs: Cleaned content from Code node
    - Outputs: Raw text JSON response to "Parse JSON"
    - Edge cases:
      - API key or quota errors
      - Unexpected or malformed responses from GPT-3
      - Timeout or network errors

  - **Parse JSON (Code node)**
    - Type: Code node (JavaScript)
    - Role: Parse the JSON string returned in OpenAI response into separate JSON fields
    - Config:
      - Runs once per item
      - Parses `text` field into individual JSON keys: value_proposition, industry, market, target_audience
    - Inputs: From "OpenAI"
    - Outputs: Parsed JSON to "Merge" node (second input)
    - Edge cases:
      - Invalid JSON parse errors if GPT-3 output malformed
      - Missing fields in output

#### 1.5 Data Parsing and Merging

- **Overview:** Merges original company data with GPT-3 enrichment results and updates the Google Sheet accordingly.
- **Nodes Involved:** 
  - Merge (second connection)
  - Update Google Sheets

- **Node Details:**

  - **Merge (continued)**
    - Combines original data and parsed GPT-3 data by position to create enriched items
    - Inputs: 
      - Input 1: Original batch items (domains)
      - Input 2: Enriched JSON data from Parse JSON node
    - Outputs: Enriched full data to "Update Google Sheets"
    - Edge cases:
      - Data misalignment causing incorrect merges

  - **Update Google Sheets**
    - Type: Google Sheets update node
    - Role: Updates the existing rows in the sheet with new enrichment columns
    - Config:
      - Document and sheet same as reading step
      - Columns updated: Market, Industry, Value Proposition, Target Audience
      - Row matching done on "Domain" column to ensure correct row update
      - Credentials: OAuth2 Google Sheets account
    - Inputs: Enriched data from Merge
    - Outputs: Connects to "Wait" node
    - Edge cases:
      - Sheet write permission errors
      - Race conditions if multiple updates on same rows
      - Domain mismatch causing no updates

#### 1.6 Throttling / Loop Control

- **Overview:** Introduces a delay before continuing to the next batch to avoid rate-limiting or overloading APIs.
- **Nodes Involved:** 
  - Wait

- **Node Details:**

  - **Wait**
    - Type: Wait node
    - Role: Pauses workflow for a defined duration before looping back to batch processing
    - Config:
      - Duration unit: seconds (not specified exact value)
      - Webhook ID provided (for potential external trigger)
    - Inputs: From "Update Google Sheets"
    - Outputs: Loops back to "Split In Batches" to continue processing next batch
    - Edge cases:
      - Excessive delay causing long workflow runtimes
      - No delay may cause API throttling

---

### 3. Summary Table

| Node Name                     | Node Type                | Functional Role                          | Input Node(s)                   | Output Node(s)               | Sticky Note                          |
|-------------------------------|--------------------------|----------------------------------------|--------------------------------|-----------------------------|------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger           | Start workflow execution manually      | None                           | Read Google Sheets           |                                    |
| Read Google Sheets             | Google Sheets            | Read company domains                    | When clicking "Execute Workflow" | Split In Batches             |                                    |
| Split In Batches               | Split In Batches         | Batch the list for sequential processing | Read Google Sheets             | HTTP Request, Merge          |                                    |
| HTTP Request                  | HTTP Request             | Fetch website HTML                      | Split In Batches               | HTML Extract                |                                    |
| HTML Extract                  | HTML Extract             | Extract HTML content                    | HTTP Request                  | Clean Content               |                                    |
| Clean Content                 | Code (JavaScript)        | Clean and truncate extracted HTML      | HTML Extract                  | OpenAI                      |                                    |
| OpenAI                       | OpenAI                   | Analyze website content with GPT-3     | Clean Content                 | Parse JSON                  |                                    |
| Parse JSON                   | Code (JavaScript)        | Parse GPT-3 JSON response fields        | OpenAI                       | Merge                       |                                    |
| Merge                        | Merge                    | Combine original and enriched data      | Split In Batches, Parse JSON  | Update Google Sheets         |                                    |
| Update Google Sheets          | Google Sheets            | Update sheet with enriched data         | Merge                        | Wait                        |                                    |
| Wait                         | Wait                     | Delay before processing next batch      | Update Google Sheets          | Split In Batches             |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**
   - Name: When clicking "Execute Workflow"
   - Purpose: To start the workflow manually.
   - No parameters needed.

2. **Add a Google Sheets Node (Read operation):**
   - Name: Read Google Sheets
   - Operation: Read rows from your company list sheet.
   - Document ID: Paste your Google Sheets URL containing company domains.
   - Sheet Name: Select or input the sheet (e.g., "Sheet1" or gid=0).
   - Credentials: Set up Google Sheets OAuth2 credentials.
   - Connect "When clicking Execute Workflow" output to this node.

3. **Add a Split In Batches Node:**
   - Name: Split In Batches
   - Purpose: To process companies one by one or in small sets.
   - Use default batch size or specify as needed.
   - Connect output of "Read Google Sheets" to this node.

4. **Add an HTTP Request Node:**
   - Name: HTTP Request
   - Purpose: Fetch the HTML content of company websites.
   - URL: Use expression `https://www.{{$json["Domain"]}}`
   - Options: Enable "Follow Redirects"
   - Set "Continue On Fail" to true to prevent workflow halt on errors.
   - Connect output of "Split In Batches" to this node.

5. **Add an HTML Extract Node:**
   - Name: HTML Extract
   - Purpose: Extract the entire HTML content for processing.
   - Extraction: Key `body`, CSS selector `html`
   - Connect output of "HTTP Request" to this node.
   - Set "Continue On Fail" to true.

6. **Add a Code Node to Clean Content:**
   - Name: Clean Content
   - Purpose: Remove whitespace and line breaks, truncate content to 10,000 characters.
   - Mode: Run once per item.
   - JavaScript:
     ```js
     if ($input.item.json.body){
       $input.item.json.content = $input.item.json.body.replaceAll('/^\\s+|\\s+$/g', '').replace('/(\\r\\n|\\n|\\r)/gm', "").replace(/\\s+/g, ' ');
       $input.item.json.contentShort = $input.item.json.content.slice(0, 10000);
     }
     return $input.item;
     ```
   - Connect output of "HTML Extract" to this node.

7. **Add an OpenAI Node:**
   - Name: OpenAI
   - Purpose: Send cleaned content to GPT-3 for analysis.
   - Prompt: Use the template below, replacing placeholders with expressions:
     ```
     This is the content of the website {{ $node["Split In Batches"].json["Domain"] }}:"{{ $json["contentShort"] }}"

     In a JSON format:

     - Give me the value proposition of the company. In less than 25 words. In English. Casual Tone. Format is: "[Company Name] helps [target audience] [achieve desired outcome] and [additional benefit]"

     - Give me the industry of the company. (Classify using this industry list: [Agriculture, Arts, Construction, Consumer Goods, Education, Entertainment, Finance, Other, Health Care, Legal, Manufacturing, Media & Communications, Public Administration, Advertisements, Real Estate, Recreation & Travel, Retail, Software, Transportation & Logistics, Wellness & Fitness] if it's ambiguous between Software and Consumer Goods, prefer Consumer Goods)

     - Guess the target audience of each company.(Classify and choose 1 from this list: [sales teams, marketing teams, HR teams, customer Service teams, consumers, C-levels] Write it in lowercase)

     - Tell me if they are B2B or B2C

     format should be:
     {"value_proposition": value_proposition,
     "industry": industry,
     "target_audience": target_audience, 
     "market": market }
     JSON:
     ```
   - OpenAI parameters:
     - Temperature: 0
     - Max Tokens: 120
     - Top P: 1
   - Credentials: Configure your OpenAI API credentials.
   - Connect output of "Clean Content" to this node.
   - Set "Continue On Fail" to true.

8. **Add a Code Node to Parse JSON:**
   - Name: Parse JSON
   - Purpose: Parse the JSON string returned by OpenAI into separate JSON fields.
   - Mode: Run once per item.
   - JavaScript:
     ```js
     const parsed = JSON.parse($input.item.json.text);
     $input.item.json.value_proposition = parsed.value_proposition;
     $input.item.json.industry = parsed.industry;
     $input.item.json.market = parsed.market;
     $input.item.json.target_audience = parsed.target_audience;
     return $input.item;
     ```
   - Connect output of "OpenAI" to this node.

9. **Add a Merge Node:**
   - Name: Merge
   - Purpose: Combine original batch data (domains) and enriched GPT-3 data into one object.
   - Mode: Combine
   - Combination Mode: Merge by Position
   - Connect:
     - Input 1: Output of "Split In Batches" (original data)
     - Input 2: Output of "Parse JSON" (enriched data)

10. **Add a Google Sheets Node (Update operation):**
    - Name: Update Google Sheets
    - Purpose: Update original sheet rows with enriched data.
    - Document ID & Sheet: Same as initial Google Sheets node
    - Operation: Update
    - Fields to update: Market, Industry, Value Proposition, Target Audience
    - Match rows by "Domain" column (value to match: `={{ $json["Domain"] }}`)
    - Credentials: Google Sheets OAuth2
    - Connect output of "Merge" to this node.

11. **Add a Wait Node:**
    - Name: Wait
    - Purpose: Add delay to throttle workflow and avoid API rate limits.
    - Duration: Set appropriate delay in seconds (not specified in original)
    - Connect output of "Update Google Sheets" to this node.

12. **Loop Back:**
    - Connect output of "Wait" node back to "Split In Batches" node to process next batch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                                                                                                                              |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow uses OpenAI GPT-3 for enriching company data extracted from websites, enhancing personalization and targeting for outreach.                          | Workflow description                                                                                                                                        |
| Industry and target audience classification lists are predefined and part of the OpenAI prompt to standardize output.                                           | Prompt inside OpenAI node                                                                                                                                   |
| Google Sheets integration requires OAuth2 credentials with edit permissions on the target spreadsheet.                                                         | Google Sheets nodes                                                                                                                                         |
| The workflow employs batch processing with a wait period to avoid API throttling and ensure stable execution over large datasets.                             | Split In Batches and Wait node logic                                                                                                                       |
| Continue on Fail is enabled on nodes interacting with external services to prevent the entire workflow from stopping due to single failures.                  | HTTP Request, HTML Extract, OpenAI nodes                                                                                                                   |
| Ensure the OpenAI API key has sufficient quota and permissions to perform the required number of completions.                                                  | OpenAI node credentials                                                                                                                                     |
| For best results, company domains should be valid and active websites to allow meaningful content extraction.                                                  | HTTP Request node                                                                                                                                           |
| The "Parse JSON" node expects the OpenAI response to be strictly valid JSON; improper JSON formatting may cause parse errors and should be monitored.           | Parse JSON (Code node)                                                                                                                                      |

---

This document provides all necessary information to understand, reproduce, and maintain the workflow, as well as anticipate common issues or extend its capabilities.
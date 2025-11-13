Extract YouTube Creator Emails with Apify Scraper and GPT-4o Mini

https://n8nworkflows.xyz/workflows/extract-youtube-creator-emails-with-apify-scraper-and-gpt-4o-mini-7597


# Extract YouTube Creator Emails with Apify Scraper and GPT-4o Mini

---

### 1. Workflow Overview

This workflow automates the extraction of YouTube creator contact emails by leveraging Apify scrapers and OpenAI’s GPT-4o Mini model. It is designed for outreach and partnership use cases, providing a streamlined pipeline that searches YouTube channels based on a keyword, scrapes channel data, and uses AI to extract and format email addresses cleanly from channel descriptions.

The workflow is logically divided into the following blocks:

- **1.1 Input Initialization:** Manual trigger and setting search parameters such as the keyword and number of channels to search.
- **1.2 YouTube Channel Search:** Querying the Apify YouTube Scraper to find relevant channels based on the search term.
- **1.3 Channel Scraping:** Iterating over found channels to scrape detailed information using a secondary Apify scraper.
- **1.4 Email Extraction via AI:** Feeding channel descriptions into an OpenAI-powered agent to extract and format email addresses.
- **1.5 Output Parsing and Loop Control:** Parsing AI output for structured email lists and managing batch processing as needed.
- **1.6 Configuration and Setup Notes:** Sticky notes providing setup instructions and credentials management for OpenAI and Apify integrations.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

- **Overview:** Initiates the workflow manually and sets user-defined search parameters like search keywords and the number of channels to process.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Set Search Term and # of searches  
  - Sticky Note17 (Setup Instructions)  
  - Sticky Note18 (Workflow Purpose Description)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connects to "Set Search Term and # of searches"  
    - Edge cases: None (manual trigger)  

  - **Set Search Term and # of searches**  
    - Type: Set  
    - Role: Defines workflow input variables: search keyword (`Search`) and number of channels (`channels`) to retrieve.  
    - Configuration:  
      - `Search` = "n8n workflows" (default search term)  
      - `channels` = "2" (number of channels to process)  
    - Inputs: From manual trigger  
    - Outputs: To "Search Youtube" HTTP Request node  
    - Edge cases: Input values must be valid strings and integers; invalid inputs may cause API request errors.

  - **Sticky Note17 & Sticky Note18**  
    - Purpose: Provide detailed setup instructions for API credentials and a high-level description of the workflow purpose.  
    - Content includes:  
      - How to set up OpenAI API keys, billing, and credentials in n8n.  
      - How to set up Apify API tokens and create HTTP Query Auth credentials in n8n.  
      - Links to Apify scrapers used.  
      - Contact info for support/customization.

#### 1.2 YouTube Channel Search

- **Overview:** Sends a request to the Apify “YouTube Scraper by streamers” to retrieve channels matching the search term within specified criteria.
- **Nodes Involved:**  
  - Search Youtube  
  - Sticky Note19 (OpenAI Setup Reminder)  
  - Sticky Note20 (Apify Setup Reminder)  
  - Loop Over Items (initial invocation)

- **Node Details:**

  - **Search Youtube**  
    - Type: HTTP Request  
    - Role: Queries Apify’s YouTube scraper synchronously for channel metadata matching the search parameters.  
    - Configuration:  
      - Method: POST  
      - URL: Apify act endpoint for “streamers~youtube-scraper” with synchronous dataset retrieval.  
      - JSON Body Parameters:  
        - Filters like date range (last month), video length (<4 min), sorting by relevance, and max results based on input `channels`.  
        - Search queries set dynamically from the `Search` variable.  
      - Authentication: HTTP Query Auth using Apify API key (credential named “Query Auth account”).  
    - Inputs: From “Set Search Term and # of searches” node (search term and max results).  
    - Outputs: Channel list passed to "Loop Over Items".  
    - Edge cases:  
      - API failures due to invalid token, quota exceeded, or malformed requests.  
      - Empty or zero results if no matching channels found.

  - **Loop Over Items (1st instance)**  
    - Type: SplitInBatches  
    - Role: Processes each channel item returned from the search individually for further scraping.  
    - Configuration: Default batch options (no batch size specified, processes all items sequentially).  
    - Inputs: From “Search Youtube” node.  
    - Outputs: First output branch is empty (no further action), second output branch connects to “Scrape Channels”.  
    - Edge cases: Empty input array leads to no processing; large input size may require batch size tuning.

  - **Sticky Note19 and Sticky Note20**  
    - Provide reminders on setting up OpenAI and Apify connections respectively.  
    - Includes links and step-by-step instructions for credential creation.

#### 1.3 Channel Scraping

- **Overview:** For each channel URL obtained, sends a request to the Apify “YouTube Scraper by apidojo” to scrape detailed channel data.
- **Nodes Involved:**  
  - Scrape Channels  
  - Loop Over Items (2nd instance, for potential looping after email extraction)

- **Node Details:**

  - **Scrape Channels**  
    - Type: HTTP Request  
    - Role: Fetches detailed channel information (including descriptions) required for email extraction.  
    - Configuration:  
      - Method: POST  
      - URL: Apify act endpoint for “apidojo~youtube-scraper” with synchronous dataset retrieval.  
      - JSON Body Parameters:  
        - Includes `startUrls` dynamically set from the `channelUrl` property of each item from the previous node.  
        - Fetches all durations, features, disables trending and shorts, limits max items to 1 per channel.  
      - Authentication: HTTP Query Auth using same Apify API key credential.  
    - Inputs: From “Loop Over Items” second output branch.  
    - Outputs: To “Extract Email Address”.  
    - Edge cases:  
      - Missing or malformed `channelUrl` causes failure.  
      - API limits or authorization errors.  
      - Empty channel data if scraper fails or channel URL invalid.

#### 1.4 Email Extraction via AI

- **Overview:** Processes each channel’s description text through an OpenAI agent to extract and format email addresses in a structured JSON format.
- **Nodes Involved:**  
  - Extract Email Address  
  - OpenAI Chat Model3  
  - Structured Output Parser4

- **Node Details:**

  - **Extract Email Address**  
    - Type: LangChain Agent (OpenAI-powered)  
    - Role: Uses AI to parse channel descriptions and extract email addresses, formatting them consistently.  
    - Configuration:  
      - Text input: Bound to `description` property of the current JSON item.  
      - System prompt instructs to extract emails, correct formatting if needed, and output a JSON object with an `emails` array.  
      - Output parser enabled for structured response.  
      - Uses connected OpenAI Chat Model node for inference.  
    - Inputs: From “Scrape Channels”.  
    - Outputs: Loops back to “Loop Over Items” for further processing or completion.  
    - Edge cases:  
      - AI may fail to extract emails if descriptions are empty or poorly formatted.  
      - API timeouts, quota issues with OpenAI credentials.  
      - Unexpected AI output formats, mitigated by the structured output parser.

  - **OpenAI Chat Model3**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4o Mini language model inference for the AI agent.  
    - Configuration:  
      - Model: “gpt-4o-mini” (optimized lightweight GPT-4 variant).  
      - Credentials: Uses a configured OpenAI API key credential.  
    - Inputs: From “Extract Email Address” agent.  
    - Outputs: To “Extract Email Address” for output parsing.  
    - Edge cases: API rate limits, invalid credentials.

  - **Structured Output Parser4**  
    - Type: LangChain structured output parser  
    - Role: Parses AI text output into JSON structure as defined by schema example.  
    - Configuration: JSON schema example expects an object with `"emails": [ ... ]`.  
    - Inputs: From “OpenAI Chat Model3”.  
    - Outputs: Back to “Extract Email Address”.  
    - Edge cases: Parsing failures if AI output deviates from schema.

#### 1.5 Output Parsing and Loop Control

- **Overview:** Manages the iteration over batches and loops the workflow for each item, ultimately preparing the final structured email list.
- **Nodes Involved:**  
  - Loop Over Items (second instance connected after email extraction)

- **Node Details:**

  - **Loop Over Items** (after Extract Email Address)  
    - Type: SplitInBatches  
    - Role: Controls looping through the batch of channels and ensures each channel’s emails are processed.  
    - Configuration: Default options, no batch size specified.  
    - Inputs: From “Extract Email Address”.  
    - Outputs: No further nodes connected (end of chain).  
    - Edge cases: Infinite loops if not connected properly; empty inputs lead to no outputs.

---

### 3. Summary Table

| Node Name                  | Node Type                                   | Functional Role                                 | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                       |
|----------------------------|---------------------------------------------|------------------------------------------------|---------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                             | Starts workflow manually                        | None                            | Set Search Term and # of searches |                                                                                                                  |
| Set Search Term and # of searches | Set                                        | Defines search keyword and number of channels | When clicking ‘Execute workflow’ | Search Youtube                  |                                                                                                                  |
| Sticky Note17              | Sticky Note                                | Setup instructions for OpenAI and Apify        | None                            | None                            | ⚙️ Setup Instructions including OpenAI and Apify credential setup and contact info                               |
| Sticky Note18              | Sticky Note                                | Workflow purpose description                     | None                            | None                            | 📨 YouTube Creator Email Finder workflow overview                                                                |
| Search Youtube             | HTTP Request                              | Calls Apify YouTube Scraper for channel search | Set Search Term and # of searches | Loop Over Items                 |                                                                                                                  |
| Loop Over Items (1st)       | SplitInBatches                            | Processes each search result channel individually | Search Youtube                 | Scrape Channels                |                                                                                                                  |
| Sticky Note19              | Sticky Note                                | OpenAI connection setup reminder                 | None                            | None                            | 1️⃣ Set Up OpenAI Connection steps                                                                              |
| Sticky Note20              | Sticky Note                                | Apify connection setup reminder                   | None                            | None                            | 2️⃣ Set Up Apify Connection steps                                                                                |
| Scrape Channels            | HTTP Request                              | Scrapes detailed data for each channel           | Loop Over Items (1st)            | Extract Email Address          |                                                                                                                  |
| Extract Email Address      | LangChain Agent                            | Extracts and formats emails from descriptions    | Scrape Channels                | Loop Over Items (2nd)          |                                                                                                                  |
| OpenAI Chat Model3          | LangChain OpenAI Chat Model               | Provides GPT-4o Mini inference                   | Extract Email Address (AI model) | Extract Email Address (Output parser) |                                                                                                                  |
| Structured Output Parser4   | LangChain Structured Output Parser        | Parses AI output into JSON                        | OpenAI Chat Model3              | Extract Email Address           |                                                                                                                  |
| Loop Over Items (2nd)       | SplitInBatches                            | Controls iteration after email extraction        | Extract Email Address            | None                           |                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: “When clicking ‘Execute workflow’”  
   - Purpose: To manually start the workflow.

2. **Add a Set Node**  
   - Name: “Set Search Term and # of searches”  
   - Configuration:  
     - Add fields:  
       - `Search` (string) with default value `"n8n workflows"`  
       - `channels` (string or number) with default value `"2"`  
   - Connect Manual Trigger output to this Set node.

3. **Create an HTTP Request Node for YouTube Search**  
   - Name: “Search Youtube”  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/streamers~youtube-scraper/run-sync-get-dataset-items`  
   - Authentication: Use **HTTP Query Auth** credential configured with:  
     - Query Key: `token`  
     - Value: Your Apify API key  
   - JSON Body:  
     ```json
     {
       "dateFilter": "month",
       "downloadSubtitles": false,
       "hasCC": false,
       "hasLocation": false,
       "hasSubtitles": false,
       "is360": false,
       "is3D": false,
       "is4K": false,
       "isBought": false,
       "isHD": false,
       "isHDR": false,
       "isLive": false,
       "isVR180": false,
       "lengthFilter": "under4",
       "maxResultStreams": 0,
       "maxResults": {{ $json.channels }},
       "maxResultsShorts": 0,
       "oldestPostDate": "10 days",
       "preferAutoGeneratedSubtitles": false,
       "saveSubsToKVS": false,
       "searchQueries": [
           "{{ $json.Search }}"
       ],
       "sortingOrder": "relevance"
     }
     ```
   - Connect Set node output to this HTTP Request node.

4. **Add a SplitInBatches Node**  
   - Name: “Loop Over Items”  
   - Purpose: Process each channel item individually.  
   - Connect output of “Search Youtube” to SplitInBatches input.

5. **Create a Second HTTP Request Node for Channel Scraping**  
   - Name: “Scrape Channels”  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/apidojo~youtube-scraper/run-sync-get-dataset-items`  
   - Authentication: Use same HTTP Query Auth credential.  
   - JSON Body:  
     ```json
     {
       "customMapFunction": "(object) => { return {...object} }",
       "duration": "all",
       "features": "all",
       "getTrending": false,
       "includeShorts": false,
       "maxItems": 1,
       "sort": "r",
       "startUrls": [
         "{{ $json.channelUrl }}"
       ],
       "uploadDate": "all"
     }
     ```
   - Connect second output of SplitInBatches node (“Loop Over Items”) to this HTTP Request node.

6. **Add a LangChain Agent Node for Email Extraction**  
   - Name: “Extract Email Address”  
   - Text Property: `={{ $json.description }}` (binds to the channel description)  
   - System Message (Prompt):  
     ```
     extract email addresses from this. if they are in a bad format, format them.

     output like this:

     {
       "emails": ["email1", "email2"]
     }
     ```
   - Enable Output Parser.  
   - Connect “Scrape Channels” output to this node.

7. **Create LangChain OpenAI Chat Model Node**  
   - Name: “OpenAI Chat Model3”  
   - Model: `gpt-4o-mini`  
   - Credentials: Configure with your OpenAI API Key credentials.  
   - Connect this node as the AI model for “Extract Email Address”.

8. **Add LangChain Structured Output Parser Node**  
   - Name: “Structured Output Parser4”  
   - JSON Schema Example:  
     ```json
     {
       "emails": ["email1", "email2"]
     }
     ```  
   - Connect output of “OpenAI Chat Model3” to this parser node.  
   - Connect parser output back to “Extract Email Address” for structured data handling.

9. **Add a Second SplitInBatches Node**  
   - Name: “Loop Over Items” (second instance)  
   - Purpose: Manage looping after email extraction.  
   - Connect “Extract Email Address” output to this node.

10. **Credential Setup:**  
    - Create HTTP Query Auth credential in n8n for Apify API token with query key `token`.  
    - Create OpenAI API credential with your OpenAI API key.  
    - Assign these credentials to the respective HTTP Request and LangChain OpenAI nodes.

11. **Optional:** Add Sticky Note nodes with setup instructions and workflow description for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                               | Context or Link                                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| ⚙️ Setup Instructions for OpenAI and Apify credentials, including links to API key creation and billing pages.                                                                                                                                                            | Sticky Note17 content with links to [OpenAI Platform](https://platform.openai.com/api-keys) and [Apify Console](https://console.apify.com/) |
| 📨 Workflow overview describing the purpose of automating YouTube creator email extraction using Apify scrapers and OpenAI.                                                                                                                                               | Sticky Note18 content                                                                                               |
| 1️⃣ Detailed steps to set up OpenAI connection, including billing and API key insertion in n8n credentials.                                                                                                                                                               | Sticky Note19 content                                                                                               |
| 2️⃣ Detailed steps for Apify connection, getting API tokens, setting up scrapers, and using HTTP Query Auth in n8n.                                                                                                                                                      | Sticky Note20 content                                                                                               |
| Contact for customization or support: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/, Website: https://ynteractive.com                                                                                                            | Included in Sticky Note17                                                                                           |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---
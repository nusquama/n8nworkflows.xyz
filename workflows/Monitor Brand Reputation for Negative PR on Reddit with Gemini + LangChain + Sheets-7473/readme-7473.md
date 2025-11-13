Monitor Brand Reputation for Negative PR on Reddit with Gemini + LangChain + Sheets

https://n8nworkflows.xyz/workflows/monitor-brand-reputation-for-negative-pr-on-reddit-with-gemini---langchain---sheets-7473


# Monitor Brand Reputation for Negative PR on Reddit with Gemini + LangChain + Sheets

### 1. Workflow Overview

This workflow automates the monitoring of brand reputation by scanning Reddit for potentially negative public relations (PR) posts related to a brand. It leverages Google Vertex AI's Gemini language model via LangChain integration to analyze Reddit posts, extract structured insights, and logs relevant data into Google Sheets for tracking and further review. The key logical blocks are:

- **1.1 Scheduled Trigger:** Initiates the workflow at defined intervals to perform fresh checks.
- **1.2 Reddit Data Retrieval:** Fetches the latest posts from Reddit relevant to the brand.
- **1.3 Data Preparation:** Extracts metadata and splits Reddit posts array into individual items for processing.
- **1.4 AI Processing:** Uses LangChain with Google Vertex Chat Model and a structured output parser to analyze posts for negative PR signals.
- **1.5 Data Storage:** Processes AI output and appends results to a Google Sheet for record-keeping and monitoring.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow on a defined schedule, ensuring automated and periodic monitoring without manual intervention.

- **Nodes Involved:**  
  - Schedule Trigger1

- **Node Details:**  
  - **Schedule Trigger1**  
    - *Type:* Schedule Trigger  
    - *Technical role:* Initiates the workflow based on a set schedule.  
    - *Configuration:* Default parameters with no additional filters or time constraints specified in the JSON; presumably runs at fixed intervals (e.g., hourly/daily).  
    - *Inputs:* None  
    - *Outputs:* Triggers "Look for Latest posts" node.  
    - *Edge Cases:* If the trigger time is misconfigured, workflow may not run as expected. No manual trigger fallback shown.  
    - *Version-specific:* Uses typeVersion 1.2, compatible with standard n8n versions supporting schedule triggers.

#### 2.2 Reddit Data Retrieval

- **Overview:**  
  Retrieves the latest Reddit posts that might contain mentions or discussions about the brand for evaluation.

- **Nodes Involved:**  
  - Look for Latest posts

- **Node Details:**  
  - **Look for Latest posts**  
    - *Type:* HTTP Request  
    - *Technical role:* Makes API calls to Reddit or a relevant endpoint to fetch recent posts.  
    - *Configuration:* Parameters are empty in JSON; in practice, should be configured with Reddit API endpoint, authentication (if needed), query parameters (e.g., brand keywords, subreddit filters), and pagination logic.  
    - *Inputs:* Triggered by Schedule Trigger1  
    - *Outputs:* Passes response data to "Extract Post Metadata1".  
    - *Edge Cases:* Possible API rate limits, authentication failures, empty or malformed responses, network timeouts.  
    - *Version-specific:* Uses typeVersion 4.2, supports modern HTTP request features.

#### 2.3 Data Preparation

- **Overview:**  
  Prepares the raw Reddit data for processing by extracting relevant post metadata and splitting the array of posts into individual objects for granular analysis.

- **Nodes Involved:**  
  - Extract Post Metadata1  
  - Seperate array into individual posts1

- **Node Details:**  
  - **Extract Post Metadata1**  
    - *Type:* Set  
    - *Technical role:* Extracts and possibly transforms or formats metadata fields from the raw Reddit data (e.g., post ID, title, author, timestamp).  
    - *Configuration:* No explicit parameters visible; likely configured to map fields from the incoming data to a simplified output structure.  
    - *Inputs:* Data from "Look for Latest posts"  
    - *Outputs:* Passes structured data to "Seperate array into individual posts1".  
    - *Edge Cases:* If expected fields are missing or differently named, extraction may fail or produce incomplete data.  
    - *Version-specific:* typeVersion 3.4

  - **Seperate array into individual posts1**  
    - *Type:* Split Out  
    - *Technical role:* Splits array of posts into individual post items for individual processing by AI.  
    - *Configuration:* Default split behavior on input array.  
    - *Inputs:* From "Extract Post Metadata1"  
    - *Outputs:* Each post item sent separately to "AI Agent".  
    - *Edge Cases:* Empty arrays produce no output; malformed arrays cause failure.  
    - *Version-specific:* typeVersion 1

#### 2.4 AI Processing

- **Overview:**  
  Analyzes each post individually using a LangChain AI agent powered by Google Vertex AI’s Gemini model, parsing the AI output into a structured format for downstream use.

- **Nodes Involved:**  
  - AI Agent  
  - Google Vertex Chat Model  
  - Structured Output Parser  
  - Code

- **Node Details:**  
  - **Google Vertex Chat Model**  
    - *Type:* LangChain Language Model (Google Vertex)  
    - *Technical role:* Hosts the Gemini language model to process natural language inputs.  
    - *Configuration:* Connected as the AI language model used by "AI Agent". Credentials for Google Vertex AI must be configured externally.  
    - *Inputs:* Receives prompts from "AI Agent"  
    - *Outputs:* Responses sent back to "AI Agent".  
    - *Edge Cases:* Authentication errors, quota limits, latency issues, malformed prompts.  
    - *Version-specific:* typeVersion 1

  - **Structured Output Parser**  
    - *Type:* LangChain Output Parser  
    - *Technical role:* Converts raw AI chat responses into structured data formats (e.g., JSON), enforcing schema to facilitate further processing.  
    - *Configuration:* Used by "AI Agent" to parse responses.  
    - *Inputs:* AI responses from "AI Agent"  
    - *Outputs:* Structured output forwarded to "AI Agent".  
    - *Edge Cases:* Parsing failures if AI output deviates from expected format, schema mismatches.  
    - *Version-specific:* typeVersion 1.3

  - **AI Agent**  
    - *Type:* LangChain Agent Node  
    - *Technical role:* Orchestrates AI interaction, sending each Reddit post prompt to the language model and parsing the output.  
    - *Configuration:* Configured to use Google Vertex Chat Model and Structured Output Parser nodes. Receives individual post data as input.  
    - *Inputs:* Individual posts from "Seperate array into individual posts1"  
    - *Outputs:* Processed AI insights passed to "Code" node.  
    - *Edge Cases:* Expression errors, AI response errors, timeout, data format issues.  
    - *Version-specific:* typeVersion 2.1

  - **Code**  
    - *Type:* Code (JavaScript/TypeScript)  
    - *Technical role:* Post-processes AI outputs, likely formatting or filtering results before storage.  
    - *Configuration:* Custom user code (not provided).  
    - *Inputs:* AI Agent outputs  
    - *Outputs:* Sends processed data to "Append to Google Sheets".  
    - *Edge Cases:* Runtime errors in code, invalid data structures.  
    - *Version-specific:* typeVersion 2

#### 2.5 Data Storage

- **Overview:**  
  Stores the final AI-analyzed data into a Google Sheet to maintain a record of monitored posts and their sentiment or risk assessment.

- **Nodes Involved:**  
  - Append to Google Sheets

- **Node Details:**  
  - **Append to Google Sheets**  
    - *Type:* Google Sheets Node  
    - *Technical role:* Appends rows containing processed post data into a specified Google Sheet for tracking.  
    - *Configuration:* Requires Google Sheets credentials; configured with target spreadsheet ID and worksheet name.  
    - *Inputs:* Data from "Code" node  
    - *Outputs:* None (end of flow)  
    - *Edge Cases:* Authentication errors, quota limits, sheet access permission issues, data format mismatches.  
    - *Version-specific:* typeVersion 4.6

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                         | Input Node(s)               | Output Node(s)                | Sticky Note |
|-----------------------------|----------------------------------|---------------------------------------|-----------------------------|------------------------------|-------------|
| Schedule Trigger1            | Schedule Trigger                  | Initiate workflow periodically        | None                        | Look for Latest posts         |             |
| Look for Latest posts        | HTTP Request                     | Retrieve latest Reddit posts           | Schedule Trigger1            | Extract Post Metadata1        |             |
| Extract Post Metadata1       | Set                              | Extract metadata from Reddit data     | Look for Latest posts        | Seperate array into individual posts1 |             |
| Seperate array into individual posts1 | Split Out                      | Split posts array into individual posts | Extract Post Metadata1       | AI Agent                     |             |
| AI Agent                    | LangChain Agent                  | Analyze individual posts with AI      | Seperate array into individual posts1 | Code                        |             |
| Google Vertex Chat Model     | LangChain LM Chat Model          | Provide Gemini AI language model      | AI Agent (ai_languageModel) | AI Agent                     |             |
| Structured Output Parser     | LangChain Output Parser          | Parse AI responses into structured data | AI Agent (ai_outputParser)   | AI Agent                     |             |
| Code                        | Code                             | Post-process AI output for storage    | AI Agent                    | Append to Google Sheets       |             |
| Append to Google Sheets      | Google Sheets                    | Store analyzed data in spreadsheet    | Code                        | None                         |             |
| Sticky Note                 | Sticky Note                      | (Empty content)                       | None                        | None                         |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configuration: Set desired interval (e.g., every hour or day)  
   - No credentials required  
   - Connect output to "Look for Latest posts" node input.

2. **Create HTTP Request node ("Look for Latest posts")**  
   - Type: HTTP Request  
   - Configure with Reddit API endpoint to fetch latest posts mentioning the brand.  
   - Set HTTP method (GET), authentication if necessary (OAuth2 or API key).  
   - Add query parameters (e.g., search terms, subreddit filters, limit).  
   - Connect input from Schedule Trigger node.  
   - Connect output to "Extract Post Metadata1" node.

3. **Create Set node ("Extract Post Metadata1")**  
   - Type: Set  
   - Configure to map and extract relevant fields from Reddit response JSON, e.g.:  
     - post ID  
     - title  
     - author  
     - timestamp  
     - URL  
   - Connect input from HTTP Request node.  
   - Connect output to "Seperate array into individual posts1" node.

4. **Create Split Out node ("Seperate array into individual posts1")**  
   - Type: Split Out  
   - Configure to split the array of posts into individual post items.  
   - Connect input from Set node.  
   - Connect output to "AI Agent" node.

5. **Create LangChain Google Vertex Chat Model node ("Google Vertex Chat Model")**  
   - Type: LangChain LM Chat Model (Google Vertex)  
   - Configure credentials for Google Vertex AI (service account key, project ID).  
   - Set model name to Gemini or equivalent.  
   - Leave default parameters or tune temperature, max tokens as needed.  
   - No direct input connection; it is linked inside the AI Agent node configuration.

6. **Create LangChain Structured Output Parser node ("Structured Output Parser")**  
   - Type: LangChain Output Parser  
   - Define expected output schema for AI responses (e.g., JSON with sentiment labels, risk scores).  
   - No direct input connection; linked inside AI Agent node.

7. **Create LangChain Agent node ("AI Agent")**  
   - Type: LangChain Agent  
   - Configure to use:  
     - Language Model node: "Google Vertex Chat Model"  
     - Output Parser node: "Structured Output Parser"  
   - Accept input from "Seperate array into individual posts1" node.  
   - Connect output to "Code" node.

8. **Create Code node ("Code")**  
   - Type: Code  
   - Insert JavaScript code to:  
     - Process AI Agent output (e.g., extract relevant fields, format for Google Sheets).  
     - Handle errors and data normalization.  
   - Connect input from "AI Agent" node.  
   - Connect output to "Append to Google Sheets" node.

9. **Create Google Sheets node ("Append to Google Sheets")**  
   - Type: Google Sheets  
   - Configure credentials for Google account with access to target spreadsheet.  
   - Set spreadsheet ID and worksheet name where data will be appended.  
   - Map data fields from Code node output to sheet columns.  
   - Connect input from "Code" node.  
   - No output node (end of workflow).

10. **Connect nodes as per above connection order**  
    - Schedule Trigger1 → Look for Latest posts → Extract Post Metadata1 → Seperate array into individual posts1 → AI Agent → Code → Append to Google Sheets  
    - Google Vertex Chat Model and Structured Output Parser linked within AI Agent node configuration.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                       |
|------------------------------------------------------------------------------|------------------------------------------------------|
| This workflow integrates Google Vertex AI Gemini model through LangChain for advanced NLP analysis. | AI processing best practices                          |
| Reddit API access may require OAuth2 and adherence to Reddit API rate limits. | https://www.reddit.com/dev/api/                      |
| Ensure Google Sheets API is enabled and credentials have write permissions.  | https://developers.google.com/sheets/api             |
| The LangChain nodes require n8n versions supporting LangChain integration (v0.202+). | n8n documentation LangChain integration               |
| For structured output parser schema design, refer to LangChain output parser documentation. | https://js.langchain.com/docs/modules/agents/output_parser |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created using n8n integration and automation software. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected content. All data handled is lawful and publicly accessible.
Generate Daily Pipedrive Deal Summaries with GPT-4o-mini

https://n8nworkflows.xyz/workflows/generate-daily-pipedrive-deal-summaries-with-gpt-4o-mini-7672


# Generate Daily Pipedrive Deal Summaries with GPT-4o-mini

### 1. Workflow Overview

This workflow automates the generation of daily summaries for deals in a Pipedrive sales funnel using GPT-4o-mini, an OpenAI language model. It is designed to fetch all active deals and their associated notes from Pipedrive, transform and aggregate this data, then leverage AI to produce a concise, human-readable summary of the sales pipeline status. The workflow is intended for sales teams or managers who want an automated digest of their funnel without manual data compilation.

The workflow consists of the following logical blocks:

- **1.1 Trigger and Data Retrieval:** Initiates the workflow manually and fetches deal and note data from Pipedrive.
- **1.2 Data Transformation and Aggregation:** Processes raw data to convert stage IDs to names, merges notes per deal, and aggregates data for AI consumption.
- **1.3 AI Summarization:** Sends the aggregated data to OpenAI’s GPT-4o-mini model to generate a natural language summary.
- **1.4 Setup and Documentation:** Contains sticky notes with configuration instructions and contact information for support.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Data Retrieval

**Overview:**  
This block is responsible for starting the workflow manually and retrieving all deals and their related notes from Pipedrive using API calls.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get many deals (Pipedrive)  
- Code (JavaScript)  
- Get many notes (Pipedrive)  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user command. No parameters set.  
  - Inputs: None  
  - Outputs: Triggers “Get many deals” node.  
  - Edge Cases: None; manual trigger requires user intervention.

- **Get many deals**  
  - Type: Pipedrive API (getAll operation on deals)  
  - Role: Fetches all deals without filters (`returnAll`=true).  
  - Configuration: No filters applied; uses saved Pipedrive API credentials.  
  - Inputs: Trigger from manual start  
  - Outputs: Passes deals data to “Code” node.  
  - Edge Cases: API authentication failure, network errors, empty deal list.

- **Code**  
  - Type: Code (JavaScript)  
  - Role: Converts numeric `stage_id` of deals into human-readable stage names.  
  - Configuration: Uses a hardcoded mapping of stage IDs to names (`Prospecting`, `Qualified`, etc.) and adds a `stage_name` property to each deal item.  
  - Inputs: Deals from “Get many deals”  
  - Outputs: Modified deals with stage names to “Get many notes” node.  
  - Edge Cases: Unknown or missing `stage_id` results in “Unknown (ID)” label.

- **Get many notes**  
  - Type: Pipedrive API (getAll operation on notes)  
  - Role: Retrieves all notes linked to each deal by passing the current deal’s `id` as a filter (`deal_id`).  
  - Configuration: `returnAll`=true, filtered by deal ID dynamically via expression.  
  - Inputs: Deals with stage names from “Code” node  
  - Outputs: Sends notes data to “Combine Notes” node.  
  - Edge Cases: API errors, no notes for deal, rate limits.

---

#### 1.2 Data Transformation and Aggregation

**Overview:**  
This block combines notes per deal, sets consistent field names, aggregates data into a single structure to prepare for AI summarization.

**Nodes Involved:**  
- Combine Notes (Summarize)  
- Set Field Names (Set)  
- Aggregate for Agent (Aggregate)  
- Turn Objects to Text (Set)  

**Node Details:**

- **Combine Notes**  
  - Type: Summarize (built-in n8n node)  
  - Role: Groups notes by `deal_id`, concatenates note contents and extracts max deal title.  
  - Configuration:  
    - `fieldsToSplitBy`: “deal_id”  
    - `fieldsToSummarize`:  
      - `content`: concatenate values separated by commas  
      - `deal.title`: max (largest or last value)  
  - Inputs: Notes from “Get many notes”  
  - Outputs: Summarized notes to “Set Field Names” node.  
  - Edge Cases: Deals without notes result in empty content.

- **Set Field Names**  
  - Type: Set  
  - Role: Prepares data fields for AI by assigning `stage_name` from “Code” node, `deal.title` from combined notes, and concatenated `content` of notes.  
  - Configuration:  
    - Sets `stage_name` using expression referencing “Code” node’s `stage_name` field.  
    - Sets `deal.title` from max deal title in combined notes.  
    - Sets `content` from concatenated note content.  
  - Inputs: Output from “Combine Notes”  
  - Outputs: Data to “Aggregate for Agent”  
  - Edge Cases: Missing fields cause empty strings.

- **Aggregate for Agent**  
  - Type: Aggregate  
  - Role: Merges all items into one aggregated object for next processing step.  
  - Configuration: Aggregates all item data together without filters.  
  - Inputs: Set data from “Set Field Names”  
  - Outputs: Aggregated data to “Turn Objects to Text”  
  - Edge Cases: No items to aggregate results in empty data.

- **Turn Objects to Text**  
  - Type: Set  
  - Role: Converts aggregated data object into a JSON string under the field `data` for input to AI nodes.  
  - Configuration: Assigns field `data` as stringified JSON of the aggregated data.  
  - Inputs: Aggregated data from prior node  
  - Outputs: Text data to “Summarize Pipedrive” node  
  - Edge Cases: JSON serialization errors if data malformed.

---

#### 1.3 AI Summarization

**Overview:**  
This block leverages OpenAI GPT-4o-mini via LangChain nodes to generate a daily summary text from the aggregated deal data.

**Nodes Involved:**  
- Summarize Pipedrive (LangChain Agent)  
- OpenAI Chat Model3 (OpenAI GPT-4o-mini)  

**Node Details:**

- **Summarize Pipedrive**  
  - Type: LangChain Agent node  
  - Role: Acts as an agent that sends text input to an AI language model with a system prompt guiding summarization.  
  - Configuration:  
    - Input text template: “Deals: {{ $json.data }}” (injects JSON string from previous node)  
    - System message: “You are a helpful assistant. Do a daily summary of the deals in our pipedrive funnel.”  
    - Prompt type: define (custom prompt)  
  - Inputs: Aggregated text data from “Turn Objects to Text”  
  - Outputs: AI response to “OpenAI Chat Model3” node for processing  
  - Edge Cases: API errors, prompt formatting issues, empty input data.

- **OpenAI Chat Model3**  
  - Type: LangChain OpenAI Chat Model node  
  - Role: Executes the GPT-4o-mini model call to OpenAI with the prompt received from the agent node.  
  - Configuration:  
    - Model selected: “gpt-4o-mini”  
    - No extra options set  
    - Uses stored OpenAI API credentials  
  - Inputs: Prompt from “Summarize Pipedrive”  
  - Outputs: Final AI-generated summary text (end of workflow)  
  - Edge Cases: API key invalid, rate limits, timeouts.

---

#### 1.4 Setup and Documentation

**Overview:**  
Sticky notes provide detailed instructions on configuring OpenAI and Pipedrive API credentials, billing, and contact information for workflow support.

**Nodes Involved:**  
- Sticky Note8  
- Sticky Note56  
- Sticky Note9  
- Sticky Note11  

**Node Details:**

- **Sticky Note8**  
  - Provides step-by-step OpenAI and Pipedrive credential setup instructions, including links to billing and API token pages, plus contact info for support.  
  - Positioned left for easy reference.

- **Sticky Note56**  
  - Describes the workflow’s purpose as a daily Pipedrive deals summary using n8n and OpenAI.  

- **Sticky Note9**  
  - Focuses on detailed Pipedrive API connection instructions, including URL shortcuts and credential input.  

- **Sticky Note11**  
  - Focuses on OpenAI connection setup and billing steps with direct links.  

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                                  | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                           |
|---------------------------|---------------------------------|-------------------------------------------------|-----------------------------|----------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Starts workflow manually                         | None                        | Get many deals             |                                                                                                     |
| Get many deals            | Pipedrive API                   | Retrieves all deals                              | When clicking ‘Execute workflow’ | Code                       | Sticky Note9 (Pipedrive setup instructions)                                                         |
| Code                      | Code (JavaScript)               | Maps stage IDs to stage names                    | Get many deals              | Get many notes             |                                                                                                     |
| Get many notes            | Pipedrive API                   | Retrieves all notes linked to deals              | Code                        | Combine Notes              | Sticky Note9 (Pipedrive setup instructions)                                                         |
| Combine Notes             | Summarize                      | Groups notes by deal_id, concatenates content   | Get many notes              | Set Field Names            |                                                                                                     |
| Set Field Names           | Set                           | Sets normalized field names for AI input        | Combine Notes               | Aggregate for Agent        |                                                                                                     |
| Aggregate for Agent       | Aggregate                     | Aggregates all items into single object          | Set Field Names             | Turn Objects to Text       |                                                                                                     |
| Turn Objects to Text      | Set                           | Converts aggregated object to JSON string        | Aggregate for Agent         | Summarize Pipedrive        |                                                                                                     |
| Summarize Pipedrive       | LangChain Agent               | Sends prompt and data to AI for summarization    | Turn Objects to Text        | OpenAI Chat Model3         |                                                                                                     |
| OpenAI Chat Model3        | LangChain OpenAI Chat Model   | Executes GPT-4o-mini call to generate summary    | Summarize Pipedrive        | None                      | Sticky Note8, Sticky Note11 (OpenAI setup and billing instructions)                                 |
| Sticky Note8              | Sticky Note                   | OpenAI & Pipedrive setup instructions; contact  | None                       | None                      | See cell content                                                                                     |
| Sticky Note56             | Sticky Note                   | Workflow description and summary                 | None                       | None                      | See cell content                                                                                     |
| Sticky Note9              | Sticky Note                   | Pipedrive API connection instructions            | None                       | None                      | See cell content                                                                                     |
| Sticky Note11             | Sticky Note                   | OpenAI connection and billing instructions       | None                       | None                      | See cell content                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: “When clicking ‘Execute workflow’”  
   - No special parameters.

2. **Add a Pipedrive Node to Retrieve Deals**  
   - Type: Pipedrive  
   - Operation: “getAll” deals  
   - Return All: true  
   - Credentials: Set up and select your Pipedrive API credentials (company domain and API token).  
   - Connect output of Manual Trigger to this node.

3. **Add a Code Node to Map Stage IDs to Names**  
   - Type: Code (JavaScript)  
   - Name: “Code”  
   - Code snippet:  
     ```javascript
     const stageMap = {
       1: "Prospecting",
       2: "Qualified",
       3: "Proposal Sent",
       4: "Negotiation",
       5: "Closed Won"
     };

     return items.map(item => {
       const stageId = item.json.stage_id;
       item.json.stage_name = stageMap[stageId] || `Unknown (${stageId})`;
       return item;
     });
     ```  
   - Connect output of “Get many deals” node to this Code node.

4. **Add a Pipedrive Node to Retrieve Notes per Deal**  
   - Type: Pipedrive  
   - Resource: Note  
   - Operation: “getAll” notes  
   - Return All: true  
   - Additional Fields: Set filter “deal_id” with expression: `={{ $json.id }}` to fetch notes for each deal dynamically.  
   - Use the same Pipedrive credentials as before.  
   - Connect output of “Code” node to this node.

5. **Add a Summarize Node to Combine Notes by Deal**  
   - Type: Summarize  
   - Fields to Split By: “deal_id”  
   - Fields to Summarize:  
     - Field: “content”, Aggregation: concatenate, Separate by comma `,`  
     - Field: “deal.title”, Aggregation: max  
   - Connect output of “Get many notes” node to this node.

6. **Add a Set Node to Standardize Field Names**  
   - Type: Set  
   - Assignments:  
     - `stage_name` = `={{ $('Code').item.json.stage_name }}`  
     - `deal.title` = `={{ $json.max_deal_title }}`  
     - `content` = `={{ $json.concatenated_content }}`  
   - Connect output of “Combine Notes” node to this node.

7. **Add an Aggregate Node to Combine All Items**  
   - Type: Aggregate  
   - Aggregate: aggregateAllItemData (combine all items into one)  
   - Connect output of “Set Field Names” node to this node.

8. **Add a Set Node to Convert Object to Text**  
   - Type: Set  
   - Assignments:  
     - `data` = `={{ $json.data }}` (stringify aggregated data)  
   - Connect output of “Aggregate for Agent” node to this node.

9. **Add a LangChain Agent Node for Summarization**  
   - Type: LangChain Agent  
   - Parameters:  
     - Text template: `Deals: {{ $json.data }}`  
     - System message: “You are a helpful assistant. Do a daily summary of the deals in our pipedrive funnel.”  
     - Prompt type: define  
   - Connect output of “Turn Objects to Text” node to this node.

10. **Add a LangChain OpenAI Chat Model Node**  
    - Type: LangChain OpenAI Chat Model  
    - Model: “gpt-4o-mini”  
    - Credentials: Configure OpenAI API key in n8n credentials.  
    - Connect AI model input of “Summarize Pipedrive” node to this node.

11. **(Optional) Add Sticky Notes**  
    - Add sticky notes containing instructions for OpenAI API setup, Pipedrive API setup, billing links, and contact info for support.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                    |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| ⚙️ Setup instructions for OpenAI API key acquisition, billing management, and Pipedrive API token setup including direct URL shortcuts.                                                                                                                                                                                                                                  | Sticky Note8, Sticky Note9, Sticky Note11 nodes                                                                   |
| Contact information for extending workflow functionality (e.g., Slack/email integration or task automation). Email: robert@ynteractive.com, LinkedIn: [Robert Breen](https://www.linkedin.com/in/robert-breen-29429625/), Website: [ynteractive.com](https://ynteractive.com)                                                                                               | Sticky Note8                                                                                                      |
| Workflow purpose summary: Fetches deals and notes from Pipedrive, cleans stage IDs, aggregates info, and generates daily summaries using OpenAI.                                                                                                                                                                                                                           | Sticky Note56                                                                                                     |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.
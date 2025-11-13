Automate Pinterest Analysis & AI-Powered Content Suggestions With Pinterest API

https://n8nworkflows.xyz/workflows/automate-pinterest-analysis---ai-powered-content-suggestions-with-pinterest-api-2865


# Automate Pinterest Analysis & AI-Powered Content Suggestions With Pinterest API

### 1. Workflow Overview

This workflow automates the weekly collection, analysis, and reporting of Pinterest Pin data to assist marketing teams in optimizing their content strategy. It integrates Pinterest API data retrieval, data storage in Airtable, AI-driven trend analysis, and email delivery of actionable insights.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow on a weekly schedule.
- **1.2 Pinterest Data Retrieval & Preparation**: Fetches Pinterest Pins data via API, processes it to label data type, and prepares it for storage.
- **1.3 Data Storage in Airtable**: Upserts the processed Pinterest data into an Airtable base for persistent storage and reference.
- **1.4 AI-Powered Trend Analysis & Summarization**: Uses an AI agent to analyze Pinterest data trends and generate content suggestions, followed by a summarization step.
- **1.5 Email Delivery of Insights**: Sends the AI-generated summary report to the Marketing Manager via email.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution every 7 days at 8:00 AM, ensuring weekly automation of Pinterest data analysis.

- **Nodes Involved:**  
  - `8:00am Morning Scheduled Trigger`  
  - `Sticky Note` (commentary)

- **Node Details:**

  - **8:00am Morning Scheduled Trigger**  
    - *Type & Role:* Schedule Trigger node; initiates workflow on a timed schedule.  
    - *Configuration:* Set to trigger every 7 days at 8:00 AM.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Connects to the Pinterest API data retrieval node.  
    - *Edge Cases:* Misconfiguration of schedule could cause missed or multiple triggers; time zone considerations may affect trigger timing.  
    - *Sticky Note:* Explains the trigger timing and flexibility to adjust schedule.

  - **Sticky Note**  
    - *Type & Role:* Visual comment to explain scheduling purpose.  
    - *Content:* "Scheduled trigger at 8:00am to start the workflow. This can be updated to your schedule preference as an email with marketing trends can be sent to best fit one's schedule."

---

#### 2.2 Pinterest Data Retrieval & Preparation

- **Overview:**  
  This block fetches recent Pinterest Pins data from the Pinterest API, processes the raw data to extract relevant fields, and labels each Pin as "Organic" for clarity.

- **Nodes Involved:**  
  - `Pull List of Pinterest Pins From Account`  
  - `Update Data Field To Include Organic`  
  - `Sticky Note1` (commentary)

- **Node Details:**

  - **Pull List of Pinterest Pins From Account**  
    - *Type & Role:* HTTP Request node; retrieves Pinterest Pins data via Pinterest API v5.  
    - *Configuration:*  
      - URL: `https://api.pinterest.com/v5/pins`  
      - Method: GET (default) with empty body parameters.  
      - Headers: Authorization with Bearer token (Pinterest API OAuth2 token).  
      - Redirects enabled.  
    - *Inputs:* Triggered by schedule node.  
    - *Outputs:* Raw JSON response containing an array of Pins under `items`.  
    - *Edge Cases:*  
      - API rate limits or expired tokens causing auth errors.  
      - Network timeouts or Pinterest API downtime.  
      - Empty or malformed responses.  
    - *Sticky Note1:* Explains this node’s role in gathering Pinterest Pin data and mentions that data is labeled as Organic by default.

  - **Update Data Field To Include Organic**  
    - *Type & Role:* Code node; transforms raw API data into structured format for Airtable and adds a `"type": "Organic"` field to each Pin.  
    - *Configuration:*  
      - JavaScript code iterates over all Pins in the input, extracts fields: `id`, `created_at`, `title`, `description`, `link`, and adds `"type": "Organic"`.  
      - Outputs an array of simplified Pin objects.  
    - *Inputs:* Receives raw API data from previous node.  
    - *Outputs:* Structured Pin data ready for Airtable insertion.  
    - *Edge Cases:*  
      - Missing fields in some Pins handled by defaulting to `null`.  
      - Empty `items` array results in empty output.  
      - Code errors if input structure changes unexpectedly.

  - **Sticky Note1**  
    - *Type & Role:* Visual comment explaining the data gathering and labeling process.  
    - *Content:*  
      "Scheduled trigger begin process to gather Pinterest Pin data and store them within Airtable. This data can be referenced or analyzed accordingly.  
      *If you would like to bring in Pinterest Ads data, the data is already labeled as Organic.  
      This is perfect for those who are creating content calendars to understand content scheduling."

---

#### 2.3 Data Storage in Airtable

- **Overview:**  
  This block upserts the processed Pinterest Pin data into a specified Airtable base and table, enabling persistent storage and future reference for analysis.

- **Nodes Involved:**  
  - `Create Record Within Pinterest Data Table`  
  - `Airtable2` (connected as AI tool data source)  

- **Node Details:**

  - **Create Record Within Pinterest Data Table**  
    - *Type & Role:* Airtable node; upserts Pin records into Airtable.  
    - *Configuration:*  
      - Base: `Pinterest_Metrics` (app ID: `appfsNi1QEhw6WvXK`)  
      - Table: `Pinterest_Organic_Data` (table ID: `tbl9Dxdrwx5QZGFnp`)  
      - Operation: Upsert by matching on `id` field to avoid duplicates.  
      - Columns mapped: `pin_id` (from `id`), `created_at`, `title`, `description`, `link`, `type`.  
      - Converts fields as strings, no type conversion attempted.  
    - *Inputs:* Receives structured Pin data from the code node.  
    - *Outputs:* Confirmation of upserted records.  
    - *Edge Cases:*  
      - Airtable API rate limits or token expiration.  
      - Schema mismatch if Airtable table structure changes.  
      - Network errors or partial failures.  

  - **Airtable2**  
    - *Type & Role:* Airtable Tool node; used as a data source input for AI agent.  
    - *Configuration:* Same base and table as above.  
    - *Inputs:* None (used as AI tool input).  
    - *Outputs:* Provides data to AI agent for analysis.  
    - *Edge Cases:* Same as above for Airtable connectivity.

---

#### 2.4 AI-Powered Trend Analysis & Summarization

- **Overview:**  
  This block uses an AI agent to analyze the Pinterest data stored in Airtable, generate pin suggestions and trend insights, then summarizes the AI output into a concise report.

- **Nodes Involved:**  
  - `Pinterest Analysis AI Agent`  
  - `OpenAI Chat Model` (language model for AI agent)  
  - `Pinterest Data Analysis Summary LLM`  
  - `OpenAI Chat Model1` (language model for summarization)  
  - `Sticky Note2` (commentary)

- **Node Details:**

  - **Pinterest Analysis AI Agent**  
    - *Type & Role:* LangChain AI Agent node; performs data analysis and generates pin suggestions.  
    - *Configuration:*  
      - Prompt: Instructs AI to act as a data analysis expert to identify trends and suggest new pins to reach target audiences.  
      - Uses `OpenAI Chat Model` as language model backend (GPT-4o-mini).  
    - *Inputs:* Receives data from Airtable2 node.  
    - *Outputs:* AI-generated text with pin suggestions and trend analysis.  
    - *Edge Cases:*  
      - AI model quota limits or API errors.  
      - Unexpected data formats causing poor AI output.  
      - Latency or timeout issues.

  - **OpenAI Chat Model**  
    - *Type & Role:* Language model node (GPT-4o-mini) used by AI agent for analysis.  
    - *Configuration:* Model set to GPT-4o-mini with default options.  
    - *Inputs:* Receives prompt and data from AI agent node.  
    - *Outputs:* AI-generated text response.  
    - *Edge Cases:* API key expiration, rate limits.

  - **Pinterest Data Analysis Summary LLM**  
    - *Type & Role:* LangChain summarization chain node; condenses AI agent output into a concise summary.  
    - *Configuration:*  
      - Summarization prompt: "Write a concise summary of the following: {{ $json.output }} CONCISE SUMMARY:"  
      - Uses `OpenAI Chat Model1` as language model backend (GPT-4o-mini).  
    - *Inputs:* Receives AI agent output text.  
    - *Outputs:* Summarized text report.  
    - *Edge Cases:* Same as AI agent node.

  - **OpenAI Chat Model1**  
    - *Type & Role:* Language model node (GPT-4o-mini) used for summarization.  
    - *Configuration:* Same as above.  
    - *Inputs:* Receives prompt from summarization node.  
    - *Outputs:* Summarized text.  
    - *Edge Cases:* API limits, latency.

  - **Sticky Note2**  
    - *Type & Role:* Visual comment explaining AI analysis and reporting flow.  
    - *Content:*  
      "AI Agent will go through Pinterest Pins and analyze any data and trends to be able to reach target audience. The data is then summarized within the Summary LLM.  
      The summarized results are then sent to the Marketing Manager within an email to help lead content creation efforts."

---

#### 2.5 Email Delivery of Insights

- **Overview:**  
  This block sends the AI-generated summary report to the Marketing Manager’s email to facilitate informed content planning.

- **Nodes Involved:**  
  - `Send Marketing Trends & Pinterest Analysis To Marketing Manager`

- **Node Details:**

  - **Send Marketing Trends & Pinterest Analysis To Marketing Manager**  
    - *Type & Role:* Gmail node; sends email with AI-generated Pinterest trends and suggestions.  
    - *Configuration:*  
      - Recipient: `john.n.foster1@gmail.com` (Marketing Manager).  
      - Subject: "Pinterest Trends & Suggestions".  
      - Message body: Uses summarized AI output text.  
      - Credential: Gmail OAuth2 account configured.  
      - Executes once per workflow run.  
    - *Inputs:* Receives summarized text from `Pinterest Data Analysis Summary LLM`.  
    - *Outputs:* Email sent confirmation.  
    - *Edge Cases:*  
      - Gmail OAuth token expiration or permission issues.  
      - Email sending limits or spam filtering.  
      - Network errors.

---

### 3. Summary Table

| Node Name                                    | Node Type                                | Functional Role                                  | Input Node(s)                          | Output Node(s)                                   | Sticky Note                                                                                          |
|----------------------------------------------|-----------------------------------------|-------------------------------------------------|--------------------------------------|-------------------------------------------------|----------------------------------------------------------------------------------------------------|
| 8:00am Morning Scheduled Trigger              | Schedule Trigger                        | Initiates workflow weekly at 8:00 AM            | None                                 | Pull List of Pinterest Pins From Account          | Scheduled trigger at 8:00am to start the workflow. This can be updated to your schedule preference. |
| Pull List of Pinterest Pins From Account      | HTTP Request                           | Fetches Pinterest Pins data via API              | 8:00am Morning Scheduled Trigger     | Update Data Field To Include Organic              | Scheduled trigger begin process to gather Pinterest Pin data and store them within Airtable. This data can be referenced or analyzed accordingly. *If you would like to bring in Pinterest Ads data, the data is already labeled as Organic. This is perfect for those who are creating content calendars to understand content scheduling. |
| Update Data Field To Include Organic          | Code                                  | Processes raw data, adds "Organic" type label   | Pull List of Pinterest Pins From Account | Create Record Within Pinterest Data Table, Pinterest Analysis AI Agent |                                                                                                    |
| Create Record Within Pinterest Data Table     | Airtable                              | Upserts Pin data into Airtable                    | Update Data Field To Include Organic | Pinterest Analysis AI Agent                       |                                                                                                    |
| Airtable2                                     | Airtable Tool                         | Provides Pinterest data as input to AI agent     | None                                 | Pinterest Analysis AI Agent                        |                                                                                                    |
| Pinterest Analysis AI Agent                    | LangChain AI Agent                   | Analyzes Pinterest data, generates pin suggestions | Airtable2, OpenAI Chat Model         | Pinterest Data Analysis Summary LLM                | AI Agent will go through Pinterest Pins and analyze any data and trends to be able to reach target audience. The data is then summarized within the Summary LLM. The summarized results are then sent to the Marketing Manager within an email to help lead content creation efforts. |
| OpenAI Chat Model                             | LangChain Language Model (GPT-4o-mini) | Language model backend for AI agent analysis     | Pinterest Analysis AI Agent          | Pinterest Analysis AI Agent                        |                                                                                                    |
| Pinterest Data Analysis Summary LLM           | LangChain Summarization Chain        | Summarizes AI agent output into concise report   | Pinterest Analysis AI Agent          | Send Marketing Trends & Pinterest Analysis To Marketing Manager |                                                                                                    |
| OpenAI Chat Model1                            | LangChain Language Model (GPT-4o-mini) | Language model backend for summarization          | Pinterest Data Analysis Summary LLM | Pinterest Data Analysis Summary LLM                |                                                                                                    |
| Send Marketing Trends & Pinterest Analysis To Marketing Manager | Gmail                                | Sends summarized report email to Marketing Manager | Pinterest Data Analysis Summary LLM | None                                            |                                                                                                    |
| Sticky Note                                   | Sticky Note                          | Explains scheduled trigger timing                 | None                                 | None                                            | Scheduled trigger at 8:00am to start the workflow. This can be updated to your schedule preference. |
| Sticky Note1                                  | Sticky Note                          | Explains Pinterest data gathering and labeling   | None                                 | None                                            | Scheduled trigger begin process to gather Pinterest Pin data and store them within Airtable. This data can be referenced or analyzed accordingly. *If you would like to bring in Pinterest Ads data, the data is already labeled as Organic. This is perfect for those who are creating content calendars to understand content scheduling. |
| Sticky Note2                                  | Sticky Note                          | Explains AI analysis and email reporting          | None                                 | None                                            | AI Agent will go through Pinterest Pins and analyze any data and trends to be able to reach target audience. The data is then summarized within the Summary LLM. The summarized results are then sent to the Marketing Manager within an email to help lead content creation efforts. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Scheduled Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger every 7 days at 8:00 AM (adjust as needed).  
   - No credentials required.

2. **Create HTTP Request Node to Fetch Pinterest Pins**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.pinterest.com/v5/pins`  
   - Headers: Add `Authorization` header with value `Bearer <Your Pinterest API Token>` (replace with your valid token).  
   - Enable redirects.  
   - Connect output of Scheduled Trigger to this node.

3. **Create Code Node to Process Pinterest Data**  
   - Type: Code (JavaScript)  
   - Paste the following code to extract relevant fields and add `"type": "Organic"`:

     ```javascript
     const outputItems = [];

     for (const item of $input.all()) {
       if (item.json.items && Array.isArray(item.json.items)) {
         for (const subItem of item.json.items) {
           outputItems.push({
             id: subItem.id || null,
             created_at: subItem.created_at || null,
             title: subItem.title || null,
             description: subItem.description || null,
             link: subItem.link || null,
             type: "Organic"
           });
         }
       }
     }

     return outputItems;
     ```

   - Connect output of HTTP Request node to this node.

4. **Create Airtable Node to Upsert Pin Data**  
   - Type: Airtable  
   - Credentials: Configure with your Airtable Personal Access Token.  
   - Base: Select or enter your Airtable base ID for Pinterest metrics.  
   - Table: Select or enter your Pinterest data table name/ID.  
   - Operation: Upsert  
   - Matching Column: `id` (to avoid duplicates)  
   - Map columns:  
     - `pin_id` → `id`  
     - `created_at` → `created_at`  
     - `title` → `title`  
     - `description` → `description`  
     - `link` → `link`  
     - `type` → `type`  
   - Connect output of Code node to this Airtable node.

5. **Create Airtable Tool Node for AI Input**  
   - Type: Airtable Tool  
   - Credentials: Same Airtable token as above.  
   - Base and Table: Same as above.  
   - No input connections; this node will serve as data source for AI agent.

6. **Create OpenAI Chat Model Node for AI Agent**  
   - Type: LangChain OpenAI Chat Model  
   - Credentials: Configure with your OpenAI API key.  
   - Model: Select `gpt-4o-mini` or preferred GPT-4 variant.  
   - No input connections; will be linked to AI Agent node.

7. **Create LangChain AI Agent Node for Pinterest Analysis**  
   - Type: LangChain AI Agent  
   - Parameters:  
     - Prompt:  
       ```
       You are a data analysis expert. You will pull data from the table and provide any information in regards to trends in the data.

       Your output should be suggestions of new pins that we can post to reach the target audiences.

       Analyze the data and just summary of the pin suggestions that the team should build.
       ```  
     - Language Model: Connect to OpenAI Chat Model node.  
     - AI Tool: Connect to Airtable Tool node.  
   - Connect output of Airtable node and OpenAI Chat Model node as inputs.

8. **Create OpenAI Chat Model Node for Summarization**  
   - Type: LangChain OpenAI Chat Model  
   - Credentials: Same OpenAI API key.  
   - Model: `gpt-4o-mini`.

9. **Create LangChain Summarization Chain Node**  
   - Type: LangChain Chain Summarization  
   - Parameters:  
     - Summarization prompt:  
       ```
       Write a concise summary of the following:

       "{{ $json.output }}"

       CONCISE SUMMARY:
       ```  
     - Language Model: Connect to the second OpenAI Chat Model node.  
   - Connect output of AI Agent node to this summarization node.

10. **Create Gmail Node to Send Email**  
    - Type: Gmail  
    - Credentials: Configure with Gmail OAuth2 credentials.  
    - To: Marketing Manager’s email (e.g., `john.n.foster1@gmail.com`).  
    - Subject: "Pinterest Trends & Suggestions".  
    - Message: Use expression to insert summarized text from summarization node (`{{$json.response.text}}`).  
    - Connect output of summarization node to this node.

11. **Connect Workflow**  
    - Scheduled Trigger → HTTP Request  
    - HTTP Request → Code Node  
    - Code Node → Airtable Upsert Node  
    - Code Node → AI Agent (via Airtable Tool and OpenAI Chat Model)  
    - AI Agent → Summarization Node → Gmail Node

12. **Test and Validate**  
    - Ensure all credentials are valid and have required permissions.  
    - Test the workflow manually or wait for scheduled trigger.  
    - Monitor for errors such as API limits, authentication failures, or data format issues.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Pinterest API access requires a developer account and OAuth token.                                            | https://developers.pinterest.com/                    |
| Airtable API key and base/table setup are required for data storage.                                          | https://airtable.com/                                |
| OpenAI GPT-4o-mini model is used for AI analysis and summarization.                                           | OpenAI API documentation                             |
| Gmail OAuth2 credentials must be configured for sending emails securely.                                      | https://developers.google.com/identity/protocols/oauth2 |
| This workflow is ideal for marketing teams needing weekly Pinterest trend insights and content suggestions.   | Workflow description section                         |
| Adjust schedule trigger timing to fit marketing calendar and reporting needs.                                  | Sticky Note on schedule trigger                      |

---

This documentation provides a complete, detailed reference to understand, reproduce, and maintain the Pinterest Analysis & AI-Powered Content Suggestions workflow.
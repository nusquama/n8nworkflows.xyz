Course Recommendation System for Surveys with Data Tables and GPT-4.1-Mini

https://n8nworkflows.xyz/workflows/course-recommendation-system-for-surveys-with-data-tables-and-gpt-4-1-mini-9437


# Course Recommendation System for Surveys with Data Tables and GPT-4.1-Mini

### 1. Workflow Overview

This workflow automates personalized course recommendations based on user survey responses. It collects survey data via a web form, stores and retrieves records using n8n Data Tables, and leverages OpenAI’s GPT-4.1-Mini model via the LangChain integration to analyze responses and available courses. The AI then recommends the most suitable course with reasoning and a URL, delivering the result back to the user in a styled HTML format.

The logical flow is grouped into these blocks:

- **1.1 Input Reception:** Collect survey responses through a form trigger node.
- **1.2 Data Storage & Retrieval:** Store survey responses and fetch available courses from two n8n Data Tables.
- **1.3 Data Processing & Formatting:** Aggregate and convert course data into a text format consumable by the AI.
- **1.4 AI Processing:** Use LangChain OpenAI agent to analyze inputs and recommend the best course, with structured JSON output parsing.
- **1.5 Output Delivery:** Format and present the recommendation to the user as an HTML response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block captures user input from a web-based survey form with questions about n8n familiarity and automation needs.

- **Nodes Involved:**  
  - Survey Submission (Form Trigger)

- **Node Details:**

  - **Survey Submission**  
    - *Type:* Form Trigger (Webhook-based form submission)  
    - *Configuration:*  
      - Form titled "Survey" with fields: Name (text), Q1 (text), Q2 (dropdown with Beginner, Intermediate, Advanced), Q3 (text).  
      - Configured to respond with the last node's output.  
      - Webhook ID assigned for external form submissions.  
    - *Inputs:* HTTP request from user form submission.  
    - *Outputs:* JSON object with survey responses for downstream nodes.  
    - *Failure cases:* Webhook errors, malformed or missing form fields, timeout if submission delays occur.  
    - *Version:* 2.2

---

#### 1.2 Data Storage & Retrieval

- **Overview:**  
  Stores incoming survey data into the "Survey Responses" Data Table, then retrieves all courses from the "Courses" Data Table for AI evaluation.

- **Nodes Involved:**  
  - Store Survey Result (Data Table)  
  - Get Available Courses (Data Table)

- **Node Details:**

  - **Store Survey Result**  
    - *Type:* Data Table node (upsert operation)  
    - *Configuration:*  
      - Targets "Survey Responses" Data Table.  
      - Columns mapped from survey fields: Name, Q1, Q2, Q3.  
      - Upsert filter on Name to update existing records or add new ones.  
      - No type conversion, fields stored as strings.  
    - *Inputs:* Survey Submission node output.  
    - *Outputs:* Confirmation of data insertion/upsert.  
    - *Failure cases:* Data Table connectivity issues, schema mismatch, duplicate key conflicts.  
    - *Version:* 1

  - **Get Available Courses**  
    - *Type:* Data Table node (get operation)  
    - *Configuration:*  
      - Retrieves all records from "Courses" Data Table.  
      - No filters, fetches complete dataset.  
    - *Inputs:* Survey Submission node output (triggered after storing survey).  
    - *Outputs:* List of all courses with their descriptions.  
    - *Failure cases:* Data Table unavailability, permission errors.  
    - *Version:* 1

---

#### 1.3 Data Processing & Formatting

- **Overview:**  
  Aggregates all course data into a single JSON array, then converts it into a text string for AI input. Combines survey results and course data to feed into the LangChain prompt.

- **Nodes Involved:**  
  - Aggregate Courses (Aggregate)  
  - Convert to Text (Set)  
  - Combine Results (Merge)

- **Node Details:**

  - **Aggregate Courses**  
    - *Type:* Aggregate node  
    - *Configuration:*  
      - Aggregates all items from the "Get Available Courses" node into a single JSON array.  
    - *Inputs:* Output from Get Available Courses.  
    - *Outputs:* Single aggregated item containing all course records.  
    - *Failure cases:* Empty dataset, aggregation errors.  
    - *Version:* 1

  - **Convert to Text**  
    - *Type:* Set node  
    - *Configuration:*  
      - Sets a single string property "Available Courses" from the aggregated data JSON.  
      - Expression: `={{ $json.data }}` extracts aggregated course data.  
    - *Inputs:* Aggregate Courses output.  
    - *Outputs:* Object with "Available Courses" string for AI prompt.  
    - *Failure cases:* Expression evaluation errors if data is malformed.  
    - *Version:* 3.4

  - **Combine Results**  
    - *Type:* Merge node (combine mode)  
    - *Configuration:*  
      - Combines survey result (via Store Survey Result) and course text (via Convert to Text) into a single item.  
      - Combines all data streams to prepare AI input context.  
    - *Inputs:* Store Survey Result and Convert to Text nodes.  
    - *Outputs:* Combined JSON object with survey and courses.  
    - *Failure cases:* Data mismatch, missing inputs.  
    - *Version:* 3.2

---

#### 1.4 AI Processing

- **Overview:**  
  Uses LangChain’s OpenAI agent to process combined survey and course data, recommending the best course with reasoning and URL. The output is parsed to a structured JSON format for downstream use.

- **Nodes Involved:**  
  - Choose Best Course (LangChain Agent)  
  - Structured Output Parser (LangChain Output Parser)  
  - OpenAI Chat Model (LM Chat OpenAI)

- **Node Details:**

  - **Choose Best Course**  
    - *Type:* LangChain Agent node  
    - *Configuration:*  
      - Prompt template includes survey questions with dynamic insertion of user responses and available courses.  
      - System message instructs to recommend the best course and output JSON with "course", "reasoning", and "url".  
      - Uses a defined prompt type with output parser enabled.  
    - *Inputs:* Combined Results node output.  
    - *Outputs:* Parsed recommendation JSON.  
    - *Failure cases:* OpenAI API errors (rate limits, auth), malformed responses, parsing failures.  
    - *Version:* 2.2  
    - *Sub-workflow:* Utilizes OpenAI Chat Model for completion.

  - **Structured Output Parser**  
    - *Type:* LangChain Output Parser  
    - *Configuration:*  
      - JSON schema example enforcing keys: "course", "reasoning", "url".  
      - Ensures AI output is structured and valid JSON.  
    - *Inputs:* Output from Choose Best Course node’s AI output parser port.  
    - *Outputs:* Clean, structured JSON object.  
    - *Failure cases:* Parsing errors if AI output is invalid JSON.  
    - *Version:* 1.3

  - **OpenAI Chat Model**  
    - *Type:* LangChain LM Chat OpenAI node  
    - *Configuration:*  
      - Model: gpt-4.1-mini via LangChain integration.  
      - No additional options set.  
    - *Credentials:* Requires OpenAI API key configured in n8n credentials.  
    - *Inputs:* AI prompt from Choose Best Course node.  
    - *Outputs:* AI-generated chat completion for recommendation.  
    - *Failure cases:* API key invalid, network errors, billing issues.  
    - *Version:* 1.2

---

#### 1.5 Output Delivery

- **Overview:**  
  Presents the AI-generated course recommendation back to the user in a styled HTML page, embedding course name, URL, and reasoning.

- **Nodes Involved:**  
  - Form (Completion Response)

- **Node Details:**

  - **Form**  
    - *Type:* Form node configured for completion response  
    - *Configuration:*  
      - Renders an HTML page with embedded placeholders for the course recommendation output.  
      - Displays course name as a strong highlight, clickable URL link, and reasoning in italic style.  
      - Uses inline CSS styles for clean UI.  
    - *Inputs:* Output JSON from Choose Best Course node’s main output (structured recommendation).  
    - *Outputs:* HTTP response serving formatted HTML to the user.  
    - *Failure cases:* Missing data causing broken template, HTML rendering issues.  
    - *Version:* 2.3

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                           | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
|-----------------------|----------------------------------|------------------------------------------|-------------------------|--------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Survey Submission      | Form Trigger                     | Collect survey inputs                     | (Webhook)               | Store Survey Result, Get Available Courses | ### Recommend the Best n8n Course from a User Survey (Form Trigger + **Data Tables** + OpenAI Agent) Use the **n8n Data Tables** feature to store, retrieve, and analyze survey results — then let OpenAI automatically recommend the most relevant course for each respondent.                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Store Survey Result    | Data Table                      | Store survey responses                    | Survey Submission        | Combine Results          | #### 🧾 Table 1: `Survey Responses` Columns: - `Name` - `Q1` — Where did you learn about n8n? - `Q2` — What is your experience with n8n? - `Q3` — What kind of automations do you need help with? To create: 1. Add a **Data Table node** to your workflow.  2. From the list, click **“Create New Data Table.”**  3. Name it **Survey Responses** and add the columns above.                                                                                                                                                                                                                                                                                                                            |
| Get Available Courses  | Data Table                      | Retrieve all courses                     | Survey Submission        | Aggregate Courses         | #### 📚 Table 2: `Courses` Columns: - `Course` - `Description` To create: 1. Add another **Data Table node**.  2. Click **“Create New Data Table.”**  3. Name it **Courses** and create the columns above.  4. Copy course data from this Google Sheet:  👉 https://docs.google.com/spreadsheets/d/1Y0Q0CnqN0w47c5nCpbA1O3sn0mQaKXPhql2Bc1UeiFY/edit?usp=sharing This **Courses Data Table** is where you’ll store all available learning paths or programs for the AI to compare against survey inputs.                                                                                                                                                                                                                                         |
| Aggregate Courses      | Aggregate                      | Aggregate courses data into one          | Get Available Courses    | Convert to Text           |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Convert to Text        | Set                            | Convert aggregated courses JSON to text  | Aggregate Courses        | Combine Results           |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Combine Results        | Merge                          | Combine survey and course data            | Store Survey Result, Convert to Text | Choose Best Course        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Choose Best Course     | LangChain Agent                | AI recommends best course based on inputs | Combine Results          | Form                     | ### 2️⃣ Set Up OpenAI Connection 1. Go to [OpenAI Platform](https://platform.openai.com/api-keys)  2. Navigate to [OpenAI Billing](https://platform.openai.com/settings/organization/billing/overview)  3. Add funds to your billing account  4. Copy your API key into the **OpenAI credentials** in n8n                                                                                                                                                                                                                                                                                                                                                                                                        |
| Structured Output Parser | LangChain Output Parser       | Parse AI JSON output into structured data | Choose Best Course (ai_outputParser) | Choose Best Course (main) |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| OpenAI Chat Model      | LangChain LM Chat OpenAI       | Connects LangChain agent to OpenAI model  | Choose Best Course (ai_languageModel) | Choose Best Course (ai_outputParser) |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Form                  | Form                           | Display recommendation as HTML response  | Choose Best Course       | (HTTP response)           |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| Sticky Note55          | Sticky Note                    | Workflow overview and purpose description | —                       | —                        | ### Recommend the Best n8n Course from a User Survey (Form Trigger + **Data Tables** + OpenAI Agent) Use the **n8n Data Tables** feature to store, retrieve, and analyze survey results — then let OpenAI automatically recommend the most relevant course for each respondent.                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Sticky Note9           | Sticky Note                    | Setup instructions, branding, and resources | —                       | —                        | ### Recommend the Best n8n Course from a User Survey (Form Trigger + **Data Tables** + OpenAI Agent) @[youtube](lFbjJAcWII8) ## ⚙️ How to set it up 1️⃣ Create your **n8n Data Tables** This workflow uses **two Data Tables** — both created directly inside n8n. ... For full setup instructions, see sticky note content above. Contact: robert@ynteractive.com, [Robert Breen LinkedIn](https://www.linkedin.com/in/robert-breen-29429625/), [ynteractive.com](https://ynteractive.com) |
| Sticky Note61          | Sticky Note                    | Details for Courses Data Table setup      | —                       | —                        | #### 📚 Table 2: `Courses` Columns: - `Course` - `Description` To create: 1. Add another **Data Table node**.  2. Click **“Create New Data Table.”**  3. Name it **Courses** and create the columns above.  4. Copy course data from this Google Sheet:  👉 https://docs.google.com/spreadsheets/d/1Y0Q0CnqN0w47c5nCpbA1O3sn0mQaKXPhql2Bc1UeiFY/edit?usp=sharing This **Courses Data Table** is where you’ll store all available learning paths or programs for the AI to compare against survey inputs. |
| Sticky Note63          | Sticky Note                    | Details for Survey Responses Data Table setup | —                       | —                        | #### 🧾 Table 1: `Survey Responses` Columns: - `Name` - `Q1` — Where did you learn about n8n? - `Q2` — What is your experience with n8n? - `Q3` — What kind of automations do you need help with? To create: 1. Add a **Data Table node** to your workflow.  2. From the list, click **“Create New Data Table.”**  3. Name it **Survey Responses** and add the columns above.                                                        |
| Sticky Note31          | Sticky Note                    | OpenAI Credential Setup Instructions       | —                       | —                        | ### 2️⃣ Set Up OpenAI Connection 1. Go to [OpenAI Platform](https://platform.openai.com/api-keys)  2. Navigate to [OpenAI Billing](https://platform.openai.com/settings/organization/billing/overview)  3. Add funds to your billing account  4. Copy your API key into the **OpenAI credentials** in n8n                                                                                                                                                                                                                                                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Survey Form Trigger Node:**  
   - Add a **Form Trigger** node named "Survey Submission".  
   - Configure form fields:  
     - Name (text)  
     - Q1: Where did you learn about n8n? (text)  
     - Q2: What is your experience with n8n? (dropdown: Beginner, Intermediate, Advanced)  
     - Q3: What kind of automations do you need help with? (text)  
   - Save and note the webhook URL for external form submission.

2. **Create Survey Responses Data Table:**  
   - Add a **Data Table** node named "Store Survey Result".  
   - Create a new Data Table called "Survey Responses" with columns: Name, Q1, Q2, Q3 (all strings).  
   - Set operation to "upsert" with filter on "Name".  
   - Map survey fields accordingly.

3. **Create Courses Data Table:**  
   - Add another **Data Table** node named "Get Available Courses".  
   - Create a new Data Table called "Courses" with columns: Course, Description (strings).  
   - Populate this table manually or via import using the Google Sheet:  
     https://docs.google.com/spreadsheets/d/1Y0Q0CnqN0w47c5nCpbA1O3sn0mQaKXPhql2Bc1UeiFY/edit?usp=sharing  
   - Set the operation to "get" all records.

4. **Aggregate Courses Data:**  
   - Add an **Aggregate** node named "Aggregate Courses".  
   - Connect from "Get Available Courses".  
   - Aggregate all items into one JSON array.

5. **Convert Aggregated Data to Text:**  
   - Add a **Set** node named "Convert to Text".  
   - Set a string field "Available Courses" with expression: `={{ $json.data }}` from aggregate output.

6. **Combine Survey and Courses Data:**  
   - Add a **Merge** node named "Combine Results".  
   - Set mode to "combine" and "combine all".  
   - Connect outputs from "Store Survey Result" and "Convert to Text".

7. **Add OpenAI Credential:**  
   - In n8n, create new credentials for OpenAI API.  
   - Obtain API key from https://platform.openai.com/api-keys and paste into credential.

8. **Add OpenAI Chat Model Node:**  
   - Add a **LangChain LM Chat OpenAI** node named "OpenAI Chat Model".  
   - Select model "gpt-4.1-mini".  
   - Attach OpenAI credentials.

9. **Configure LangChain Agent Node:**  
   - Add a **LangChain Agent** node named "Choose Best Course".  
   - Use prompt type "define" with a system message instructing recommendation of best course with output JSON keys: course, reasoning, url.  
   - Text prompt includes: survey responses dynamically inserted and the "Available Courses" text.  
   - Enable output parser.  
   - Connect "Combine Results" as input.  
   - Connect "OpenAI Chat Model" as AI language model.  
   - Connect a **Structured Output Parser** node (LangChain Output Parser) configured with JSON schema example to parse AI output.

10. **Create Structured Output Parser Node:**  
    - Add **LangChain Output Parser** node named "Structured Output Parser".  
    - Set JSON schema example with keys: course, reasoning, url.  
    - Connect as AI output parser input from "Choose Best Course".

11. **Create Form Node for HTML Response:**  
    - Add a **Form** node named "Form".  
    - Configure operation as "completion".  
    - Response type: "showText".  
    - Use the provided HTML template embedding output.course, output.url, output.reasoning with styling.  
    - Connect output of "Choose Best Course" (main output) as input.

12. **Connect Nodes:**  
    - Survey Submission → Store Survey Result → Combine Results → Choose Best Course → Form  
    - Survey Submission → Get Available Courses → Aggregate Courses → Convert to Text → Combine Results  
    - Choose Best Course → OpenAI Chat Model (ai_languageModel)  
    - Choose Best Course → Structured Output Parser (ai_outputParser) → back to Choose Best Course main output.

13. **Test the Workflow:**  
    - Submit test survey via webhook URL.  
    - Verify data stored in Data Tables.  
    - Confirm AI returns structured recommendation.  
    - Confirm HTML response renders correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow leverages the new **n8n Data Tables** feature to store and query structured data internally without external databases. It integrates with OpenAI's GPT-4.1-mini via LangChain for AI-driven decision-making.                                                                                                                                                                                                                                                                                                                                                                                       | Core workflow description.                                                                       |
| Setup instructions and credentials management for OpenAI API: go to https://platform.openai.com/api-keys to create and manage your API keys, and ensure billing is funded at https://platform.openai.com/settings/organization/billing/overview.                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note31                                                                                   |
| Courses Data Table can be populated using the provided Google Sheet: https://docs.google.com/spreadsheets/d/1Y0Q0CnqN0w47c5nCpbA1O3sn0mQaKXPhql2Bc1UeiFY/edit?usp=sharing                                                                                                                                                                                                                                                                                                                                                                                                                                          | Sticky Note61                                                                                   |
| Survey Responses Data Table schema: Name, Q1 (Where did you learn about n8n?), Q2 (Experience level), Q3 (Automation needs).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note63                                                                                   |
| For help customizing or expanding this workflow (e.g., multi-survey support, follow-ups), contact Robert Breen via robert@ynteractive.com or LinkedIn https://www.linkedin.com/in/robert-breen-29429625/, or visit https://ynteractive.com.                                                                                                                                                                                                                                                                                                                                                                  | Contact information and support resources.                                                      |
| Video walkthrough available: @[youtube](lFbjJAcWII8)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note9                                                                                   |

---

Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. It adheres strictly to all applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.
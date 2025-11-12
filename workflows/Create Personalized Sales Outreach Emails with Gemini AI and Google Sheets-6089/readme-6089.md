Create Personalized Sales Outreach Emails with Gemini AI and Google Sheets

https://n8nworkflows.xyz/workflows/create-personalized-sales-outreach-emails-with-gemini-ai-and-google-sheets-6089


# Create Personalized Sales Outreach Emails with Gemini AI and Google Sheets

### 1. Workflow Overview

This workflow automates the creation of personalized sales outreach emails using Google Sheets data and Google Gemini AI. It is designed for sales and marketing teams aiming to generate customized email content for prospects listed in a Google Sheet. The workflow reads leads from a Google Sheet, uses AI (Google Gemini) to craft tailored email subjects and bodies based on prospect data and predefined instructions, then updates the original Google Sheet with the generated emails.

Logical blocks included:

- **1.1 Input Reception:** Triggering the workflow on a schedule and reading lead data from Google Sheets.
- **1.2 AI Processing:** Generating personalized email content using Google Gemini AI with a LangChain prompt and parsing the structured output.
- **1.3 Data Preparation & Output:** Preparing the AI output data and updating the Google Sheet with personalized email subjects and bodies.
- **1.4 Conditional Logic:** Checking if AI output is empty to decide whether to generate new content or reuse existing data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block triggers the workflow every 30 minutes and reads prospect data from a specified Google Sheet containing lead information for outreach.

- **Nodes Involved:**
  - Schedule Trigger
  - Read Leads from Sheet
  - If (conditional check)

- **Node Details:**

  - **Schedule Trigger**
    - Type: Trigger node
    - Role: Initiates workflow execution every 30 minutes.
    - Configuration: Interval set to 30 minutes.
    - Input: None
    - Output: Triggers downstream nodes.
    - Edge Cases: Workflow might be delayed or throttled if execution time exceeds 30 minutes; ensure no execution overlap.
  
  - **Read Leads from Sheet**
    - Type: Google Sheets node
    - Role: Reads rows from a specified Google Sheet (spreadsheet ID and sheet name configured).
    - Configuration: Uses OAuth2 credentials for Google Sheets API access; reads all rows from the designated sheet.
    - Key Expressions: Sheet name and document ID are dynamically set from credential/cache.
    - Input: Trigger output
    - Output: JSON array of lead data including fields like Name, Email, Company, Title, etc.
    - Edge Cases: API limits, invalid credentials, empty or malformed sheet data.
  
  - **If (conditional check)**
    - Type: Conditional node
    - Role: Checks if the AI-generated email subject and body are empty to decide if AI generation is needed.
    - Configuration: Conditions check if `email_body` and `subject` fields are empty strings.
    - Input: Output of Read Leads from Sheet
    - Output: Routes to AI generation or skips it.
    - Edge Cases: False negatives if fields are missing or null but not empty string; expression errors if fields undefined.

#### 2.2 AI Processing

- **Overview:** This block uses Google Gemini AI via LangChain to generate personalized email subject lines and bodies based on prospect data and a detailed prompt. The output is parsed into structured JSON format.

- **Nodes Involved:**
  - Google Gemini Chat Model
  - Structured Output Parser
  - Basic LLM Chain

- **Node Details:**

  - **Google Gemini Chat Model**
    - Type: LangChain AI chat model node
    - Role: Sends prompt to Google Gemini AI model ("models/gemini-2.5-flash") to generate email copy.
    - Configuration: Uses Google Palm API credentials; model name specified.
    - Input: Prompt text prepared in Basic LLM Chain node.
    - Output: Raw AI response text.
    - Edge Cases: API rate limits, authentication failure, model downtime.
  
  - **Structured Output Parser**
    - Type: LangChain output parser node
    - Role: Parses AI response into JSON with keys `subject` and `email_body`.
    - Configuration: JSON schema example provided to enforce response format.
    - Input: Raw AI output from Google Gemini Chat Model.
    - Output: Parsed JSON with email subject and body.
    - Edge Cases: Parsing errors if AI response is malformed or not JSON; fallback handling required.
  
  - **Basic LLM Chain**
    - Type: LangChain chain node integrating prompt, AI, and parser
    - Role: Defines the prompt with embedded prospect data; orchestrates AI call and structured parsing.
    - Configuration:
      - Prompt text instructs AI to generate a concise (100-120 words) professional and conversational email body and catchy subject line.
      - Includes context variables from Google Sheets lead data (Name, Company, Title).
      - Specifies tone, structure, formatting requirements, and constraints.
      - Enforces JSON output with keys `subject` and `email_body`.
    - Key Expressions: Uses expressions like `{{ $('Read Leads from Sheet').item.json.Name }}` to inject data from spreadsheet.
    - Input: Conditional node output (when AI generation is needed).
    - Output: Structured AI-generated email content.
    - Edge Cases: Expression errors if referenced fields missing; prompt misinterpretation by AI; output format deviation.
    - Version: Uses LangChain version supporting prompt definition and output parsing.

#### 2.3 Data Preparation & Output

- **Overview:** This block prepares the AI-generated email data and updates the original Google Sheet with new email subjects and bodies matched by email address.

- **Nodes Involved:**
  - Prepare Data for Sheet
  - Update Sheet with Email

- **Node Details:**

  - **Prepare Data for Sheet**
    - Type: Set node
    - Role: Creates a new field `updated_email` from AI generated text (though the source appears to be a generic `text` field, likely a legacy or placeholder).
    - Configuration: Sets `updated_email` to `={{ $input.item.json.text }}`.
    - Input: Output from Basic LLM Chain (structured AI output).
    - Output: Item with `updated_email` field added.
    - Edge Cases: Potential mismatch if `text` field not present; redundant or unused in downstream nodes.
  
  - **Update Sheet with Email**
    - Type: Google Sheets node
    - Role: Updates rows in the Google Sheet with newly generated `subject` and `email_body` fields using the email as a matching key.
    - Configuration:
      - Operation: Update
      - Matching column: Email
      - Columns updated: `subject` and `email_body` populated from AI output.
      - Uses dynamic expressions to fetch values:
        - Email from original lead data row.
        - Subject and email_body from AI output.
      - Credentials: Google Sheets OAuth2.
      - Sheet and document IDs configured to target same spreadsheet and sheet.
    - Input: Output from Prepare Data for Sheet.
    - Output: Confirmation of updated rows.
    - Edge Cases: Update failures if email not found, data type mismatches, API errors.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                             | Input Node(s)          | Output Node(s)                | Sticky Note                                  |
|-------------------------|----------------------------------|---------------------------------------------|------------------------|------------------------------|----------------------------------------------|
| Schedule Trigger        | Trigger                          | Triggers workflow every 30 minutes          | None                   | Read Leads from Sheet         |                                              |
| Read Leads from Sheet   | Google Sheets                    | Reads leads data from Google Sheet          | Schedule Trigger       | If                           |                                              |
| If                      | Conditional                     | Checks if AI-generated email content exists | Read Leads from Sheet  | Basic LLM Chain (if empty)    |                                              |
| Google Gemini Chat Model | LangChain AI Chat Model          | Calls Google Gemini AI for email generation | Basic LLM Chain        | Structured Output Parser      |                                              |
| Structured Output Parser | LangChain Output Parser          | Parses AI response into structured JSON     | Google Gemini Chat Model| Basic LLM Chain               |                                              |
| Basic LLM Chain          | LangChain Chain Node             | Prepares AI prompt and processes output     | If (true branch), Structured Output Parser | Prepare Data for Sheet |                                              |
| Prepare Data for Sheet   | Set Node                        | Prepares AI output for sheet update          | Basic LLM Chain        | Update Sheet with Email       |                                              |
| Update Sheet with Email  | Google Sheets                   | Updates Google Sheet with personalized emails| Prepare Data for Sheet | None                         |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and name it "Personalised Mail".**

2. **Add a Schedule Trigger node:**
   - Set interval to trigger every 30 minutes.
   - Position it as the workflow’s start.

3. **Add a Google Sheets node named "Read Leads from Sheet":**
   - Operation: Read rows.
   - Configure OAuth2 credentials for Google Sheets access.
   - Set the Spreadsheet ID to your target Google Sheet.
   - Set the Sheet Name to your leads sheet’s GID or name.
   - Connect Schedule Trigger node output to this node input.

4. **Add an If node:**
   - Conditions: Check if both `email_body` and `subject` fields are empty strings in the data received.
   - If true, proceed to AI generation.
   - Connect "Read Leads from Sheet" node output to this If node.

5. **Add a LangChain LLM Chain node named "Basic LLM Chain":**
   - Configure prompt text with detailed instructions to generate a professional sales outreach email and subject line.
   - Embed dynamic expressions to inject prospect data fields from "Read Leads from Sheet" (e.g., Name, Company, Title).
   - Define output parsing format as JSON with `subject` and `email_body`.
   - Connect If node’s true branch to this node.

6. **Add a LangChain Google Gemini Chat Model node:**
   - Select model "models/gemini-2.5-flash".
   - Provide Google Gemini API credentials.
   - Connect "Basic LLM Chain" node’s AI model input to this node.

7. **Add a LangChain Structured Output Parser node:**
   - Define JSON schema example with keys `subject` and `email_body`.
   - Connect Google Gemini Chat Model output to this node.
   - Connect this node output back to "Basic LLM Chain" node’s output parser input.

8. **Add a Set node named "Prepare Data for Sheet":**
   - Add a string value field named `updated_email`.
   - Set its value to `={{ $input.item.json.text }}` (this can be adapted or omitted if not used).
   - Connect "Basic LLM Chain" output to this node.

9. **Add another Google Sheets node named "Update Sheet with Email":**
   - Operation: Update.
   - Matching column: Email.
   - Columns to update: `subject` and `email_body` with values from AI output.
   - Use dynamic expressions to map:
     - Email: from original lead row.
     - Subject: `{{$json.output.subject}}`.
     - Email Body: `{{$json.output.email_body}}`.
   - Set Spreadsheet ID and Sheet Name to same as the input sheet.
   - Connect "Prepare Data for Sheet" output to this node.

10. **Connect the false branch of the If node to the Basic LLM Chain node as well if you want to force regeneration or to skip AI generation accordingly.**

11. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Use Google Sheets OAuth2 credentials for API access; ensure scopes include read and write access.| Google Sheets API documentation: https://developers.google.com/sheets/api/guides/authorizing         |
| Google Gemini AI requires Google Palm API credentials with billing enabled.                      | Google Cloud AI & ML services documentation: https://cloud.google.com/vertex-ai                      |
| Prompt includes a Calendly link placeholder; update with your actual CTA scheduling link.        | Replace `[https://calendly.com/YOUR_LINK]` with your real link in the prompt.                         |
| The AI output must strictly follow JSON format for parsing; consider adding error handling for malformed responses.| Best practices for LangChain output parsers: https://js.langchain.com/docs/modules/chains/output-parsers |
| The workflow updates the sheet by matching on the "Email" column; ensure this column is unique and consistent.| Duplicate emails or missing values may cause update errors or overwrites.                            |

---

**Disclaimer:** This document is generated from an n8n workflow automation exported in JSON format. It complies fully with all relevant content policies and does not include any illegal, offensive, or protected information. All data processed is assumed to be legal and public.
Automate Assignment Grading with GPT-4-Turbo and Multi-Format Reports

https://n8nworkflows.xyz/workflows/automate-assignment-grading-with-gpt-4-turbo-and-multi-format-reports-10316


# Automate Assignment Grading with GPT-4-Turbo and Multi-Format Reports

### 1. Workflow Overview

This workflow automates the grading of student engineering assignments using GPT-4-Turbo, producing multi-format reports (HTML and CSV) for educators. The process starts by receiving an uploaded test paper via webhook, extracting its text, and preparing the data alongside a predefined answer script. The AI agent then grades the assignment by comparing student answers to the answer key, awarding marks, and providing detailed feedback. Finally, results are formatted into human-readable reports and returned through the webhook.

The workflow is logically divided into three primary blocks:

- **1.1 Input Reception and Data Preparation:** Handles receiving assignment submissions, extracting textual content, and assembling grading data and answer keys.

- **1.2 AI-Powered Grading:** Utilizes the GPT-4-Turbo model with a structured output parser to grade the assignment, producing detailed marks and feedback.

- **1.3 Results Generation and Response:** Formats the grading results into HTML and CSV formats, prepares a structured response, and sends it back to the requester.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Preparation

**Overview:**  
This block receives the student's test paper via a webhook, extracts the text content from the uploaded file, and prepares the assignment data and answer script for grading.

**Nodes Involved:**  
- Webhook - Upload Test Paper  
- Extract Text from Test Paper  
- Prepare Assignment Data  
- Load Answer Script

**Node Details:**

- **Webhook - Upload Test Paper**  
  - *Type & Role:* Webhook node; entry point receiving HTTP POST submissions containing test papers.  
  - *Configuration:*  
    - Path set to "grade-assignment".  
    - Raw body option enabled to accept file data.  
    - Response mode configured to use the response node downstream.  
  - *Inputs/Outputs:* No inputs; outputs to "Extract Text from Test Paper".  
  - *Potential Failures:* Network issues, invalid file uploads, unsupported file types.  
  - *Version:* Type version 2.  

- **Extract Text from Test Paper**  
  - *Type & Role:* File extraction node; converts uploaded file content into plain text.  
  - *Configuration:* Set to operation "toText" to extract textual content.  
  - *Inputs/Outputs:* Receives file data from webhook; outputs extracted text.  
  - *Edge cases:* Unsupported file formats, corrupted files, extraction errors.  
  - *Version:* Type version 1.  

- **Prepare Assignment Data**  
  - *Type & Role:* Set node; organizes key data fields for grading.  
  - *Configuration:*  
    - Extracts student name and assignment title from webhook JSON body, defaults to 'Unknown Student' and 'Engineering Assignment' if missing.  
    - Sets the extracted test paper text from the previous node.  
  - *Inputs/Outputs:* Input from text extraction node; outputs to "Load Answer Script".  
  - *Expressions:* Uses expressions to safely access JSON data with fallback values.  
  - *Edge cases:* Missing fields in submission JSON, malformed data.  
  - *Version:* Type version 3.4.  

- **Load Answer Script**  
  - *Type & Role:* Set node; loads predefined correct answers and marking scheme as a string.  
  - *Configuration:* Contains a multi-question answer script with question texts, answers, and allocated marks embedded as a string.  
  - *Inputs/Outputs:* Input from "Prepare Assignment Data"; outputs to "AI Agent - Grade Assignment".  
  - *Edge cases:* Large script size, format changes requiring prompt updates.  
  - *Version:* Type version 3.4.  

---

#### 2.2 AI-Powered Grading

**Overview:**  
This block compares the student's extracted answers against the answer script using GPT-4-Turbo, grades each question, and generates detailed feedback in a structured JSON format.

**Nodes Involved:**  
- AI Agent - Grade Assignment  
- OpenAI Chat Model  
- Structured Output Parser

**Node Details:**

- **AI Agent - Grade Assignment**  
  - *Type & Role:* Langchain Agent node; orchestrates prompt composition and output parsing for grading.  
  - *Configuration:*  
    - Prompt instructs the agent to grade based on correctness, completeness, and accuracy.  
    - Requires output in a predefined JSON schema with question-wise marks, total marks, percentage, grade, and overall feedback.  
    - System message enforces valid JSON output.  
  - *Inputs/Outputs:* Inputs answer script and test paper text; outputs raw AI response to "Generate Results Table".  
  - *Expressions:* Embeds variables such as answer script and student submission dynamically into prompt text.  
  - *Version:* Type version 1.7.  
  - *Edge cases:* AI hallucinations, malformed JSON output, API rate limits or timeouts.  

- **OpenAI Chat Model**  
  - *Type & Role:* Language model node; executes the GPT-4-Turbo model call.  
  - *Configuration:* Model set to "gpt-4-turbo".  
  - *Inputs/Outputs:* Receives prompt from AI Agent; outputs completion to the output parser.  
  - *Credentials:* Uses OpenAI API credentials (configured with OAuth or API key).  
  - *Edge cases:* API authentication failure, network issues, model unavailability.  
  - *Version:* Type version 1.  

- **Structured Output Parser**  
  - *Type & Role:* Langchain output parser node; validates and parses AI output into structured JSON.  
  - *Configuration:* Default structured parser expecting the JSON format described in the prompt.  
  - *Inputs/Outputs:* Input from OpenAI Chat Model; output back to AI Agent node for workflow continuation.  
  - *Edge cases:* Parsing errors if AI output deviates from schema, incomplete data.  
  - *Version:* Type version 1.2.  

---

#### 2.3 Results Generation and Response

**Overview:**  
Converts the graded results into HTML and CSV formats, prepares a formatted response JSON including summary and detailed feedback, and sends the final reply to the webhook caller.

**Nodes Involved:**  
- Generate Results Table  
- Convert to HTML File  
- Prepare CSV Data  
- Convert to CSV File  
- Format Response  
- Respond to Webhook

**Node Details:**

- **Generate Results Table**  
  - *Type & Role:* Code node; formats AI grading results into HTML table and CSV string.  
  - *Configuration:*  
    - Builds an HTML table with question-wise marks, feedback, totals, and overall grade summary.  
    - Constructs CSV with the same data for export or download.  
    - Includes student name and assignment title in outputs.  
  - *Inputs/Outputs:* Input from AI Agent grading results; outputs HTML, CSV data, and a summary string.  
  - *Edge cases:* Missing or malformed grading data causing runtime errors.  
  - *Version:* Type version 2.  

- **Convert to HTML File**  
  - *Type & Role:* File conversion node; converts the HTML string into a downloadable file object.  
  - *Configuration:* Operation set to "text".  
  - *Inputs/Outputs:* Inputs HTML from "Generate Results Table"; outputs binary file.  
  - *Edge cases:* Large HTML causing memory issues.  
  - *Version:* Type version 1.1.  

- **Prepare CSV Data**  
  - *Type & Role:* Set node; prepares CSV string for file conversion.  
  - *Configuration:* Assigns CSV string from previous code node.  
  - *Inputs/Outputs:* Input from "Generate Results Table"; outputs to "Convert to CSV File".  
  - *Version:* Type version 3.4.  

- **Convert to CSV File**  
  - *Type & Role:* File conversion node; converts CSV string to downloadable file.  
  - *Configuration:* Operation "text".  
  - *Inputs/Outputs:* Input from "Prepare CSV Data"; outputs binary CSV file.  
  - *Version:* Type version 1.1.  

- **Format Response**  
  - *Type & Role:* Set node; assembles the final JSON response including status, message, detailed results, and HTML report.  
  - *Configuration:*  
    - Status set to "success".  
    - Message includes a summary score string.  
    - Embeds detailed grading results and HTML report string.  
  - *Inputs/Outputs:* Takes data from "Generate Results Table"; outputs to "Respond to Webhook".  
  - *Version:* Type version 3.4.  

- **Respond to Webhook**  
  - *Type & Role:* Webhook response node; sends back the assembled JSON response.  
  - *Configuration:*  
    - Content-Type header set to "application/json".  
    - Responds with all incoming items (including files and JSON).  
  - *Inputs/Outputs:* Input from "Format Response"; no outputs (end of workflow).  
  - *Edge cases:* Client disconnects, response size limits.  
  - *Version:* Type version 1.1.  

---

### 3. Summary Table

| Node Name                   | Node Type                                  | Functional Role                    | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                                             |
|-----------------------------|--------------------------------------------|----------------------------------|------------------------------|--------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Webhook - Upload Test Paper  | n8n-nodes-base.webhook                      | Receive assignment submission    | None                         | Extract Text from Test Paper    | ## Introduction Automates AI-driven assignment grading with HTML and CSV output. Designed for educators evaluating submissions with consistent criteria and exportable results. |
| Extract Text from Test Paper | n8n-nodes-base.extractFromFile              | Extract text from uploaded file  | Webhook - Upload Test Paper   | Prepare Assignment Data         | ## Introduction (continued)                                                                                                             |
| Prepare Assignment Data      | n8n-nodes-base.set                          | Prepare data for grading         | Extract Text from Test Paper  | Load Answer Script              | ## Introduction (continued)                                                                                                             |
| Load Answer Script           | n8n-nodes-base.set                          | Load correct answers and rubric  | Prepare Assignment Data       | AI Agent - Grade Assignment     | ## Introduction (continued)                                                                                                             |
| AI Agent - Grade Assignment  | @n8n/n8n-nodes-langchain.agent              | Grade assignment via AI          | Load Answer Script, OpenAI Chat Model | Generate Results Table    | ## Setup Instructions 1. Trigger & Processing: Configure webhook URL, set text extraction parameters. 2. AI Configuration: Add OpenAI API key, customize grading prompts, define Output Parser JSON schema. |
| OpenAI Chat Model            | @n8n/n8n-nodes-langchain.lmChatOpenAi       | GPT-4-Turbo language model call | AI Agent - Grade Assignment   | Structured Output Parser        |                                                                                                                                         |
| Structured Output Parser     | @n8n/n8n-nodes-langchain.outputParserStructured | Parse AI output to structured JSON | OpenAI Chat Model            | AI Agent - Grade Assignment     |                                                                                                                                         |
| Generate Results Table       | n8n-nodes-base.code                         | Format grading results to HTML & CSV | AI Agent - Grade Assignment | Convert to HTML File, Prepare CSV Data, Format Response |                                                                                                                                         |
| Convert to HTML File         | n8n-nodes-base.convertToFile                 | Convert HTML string to file      | Generate Results Table        | None                           |                                                                                                                                         |
| Prepare CSV Data             | n8n-nodes-base.set                          | Prepare CSV string for conversion | Generate Results Table       | Convert to CSV File             |                                                                                                                                         |
| Convert to CSV File          | n8n-nodes-base.convertToFile                 | Convert CSV string to file       | Prepare CSV Data              | None                           |                                                                                                                                         |
| Format Response             | n8n-nodes-base.set                          | Assemble final JSON response     | Generate Results Table        | Respond to Webhook              |                                                                                                                                         |
| Respond to Webhook           | n8n-nodes-base.respondToWebhook              | Send response to submitter       | Format Response               | None                           |                                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Name: "Webhook - Upload Test Paper"  
   - Path: "grade-assignment"  
   - Enable raw body processing  
   - Set response mode to "responseNode"  
   - No credentials needed  
   - Position appropriately (e.g., leftmost)  

2. **Add Extract Text from File Node**  
   - Type: ExtractFromFile  
   - Name: "Extract Text from Test Paper"  
   - Operation: "toText"  
   - Connect webhook node output to this node input  

3. **Add Set Node for Assignment Data**  
   - Type: Set  
   - Name: "Prepare Assignment Data"  
   - Add fields with expressions:  
     - studentName: `{{$json.body.studentName || 'Unknown Student'}}`  
     - assignmentTitle: `{{$json.body.assignmentTitle || 'Engineering Assignment'}}`  
     - testPaperText: `{{$node["Extract Text from Test Paper"].json.data}}`  
   - Connect previous node output to this node input  

4. **Add Set Node for Answer Script**  
   - Type: Set  
   - Name: "Load Answer Script"  
   - Add field "answerScript" with a multi-line string containing all questions, answers, and marks as provided in the original script  
   - Connect "Prepare Assignment Data" output to this node  

5. **Add Langchain Agent Node for Grading**  
   - Type: @n8n/n8n-nodes-langchain.agent  
   - Name: "AI Agent - Grade Assignment"  
   - Configure prompt:  
     - Embed `answerScript` and `testPaperText` from previous nodes in the prompt text  
     - Specify JSON output format with question numbers, marks, feedback, totals, percentage, grade, overall feedback  
   - Add system message enforcing valid JSON output  
   - Connect "Load Answer Script" output here  
   - Version: 1.7  

6. **Add OpenAI Chat Model Node**  
   - Type: @n8n/n8n-nodes-langchain.lmChatOpenAi  
   - Name: "OpenAI Chat Model"  
   - Model: "gpt-4-turbo"  
   - Connect AI Agent node’s language model input to this node  
   - Provide OpenAI API credentials (API key or OAuth2)  

7. **Add Structured Output Parser Node**  
   - Type: @n8n/n8n-nodes-langchain.outputParserStructured  
   - Name: "Structured Output Parser"  
   - Connect OpenAI Chat Model node output here  
   - Connect parser output back to AI Agent node’s output parser input  

8. **Add Code Node to Generate Results Table**  
   - Type: Code  
   - Name: "Generate Results Table"  
   - Use provided JavaScript code to build HTML table and CSV data from grading result JSON  
   - Connect AI Agent node main output to this node  

9. **Add Convert to HTML File Node**  
   - Type: ConvertToFile  
   - Name: "Convert to HTML File"  
   - Operation: "text"  
   - Connect "Generate Results Table" output here  

10. **Add Set Node to Prepare CSV Data**  
    - Type: Set  
    - Name: "Prepare CSV Data"  
    - Field "data" assigned from `{{ $('Generate Results Table').json.csvData }}`  
    - Connect "Generate Results Table" output here  

11. **Add Convert to CSV File Node**  
    - Type: ConvertToFile  
    - Name: "Convert to CSV File"  
    - Operation: "text"  
    - Connect "Prepare CSV Data" output here  

12. **Add Set Node to Format Response**  
    - Type: Set  
    - Name: "Format Response"  
    - Fields:  
      - status: "success"  
      - message: summary string from "Generate Results Table"  
      - results: gradingResult JSON object  
      - htmlReport: HTML string from "Generate Results Table"  
    - Connect "Generate Results Table" output here  

13. **Add Respond to Webhook Node**  
    - Type: RespondToWebhook  
    - Name: "Respond to Webhook"  
    - Response Headers: Content-Type = application/json  
    - Respond with all incoming items  
    - Connect "Format Response" output here  

14. **Establish Connections** following this order:  
    - Webhook → Extract Text → Prepare Assignment Data → Load Answer Script → AI Agent (with OpenAI Chat Model and Output Parser integration) → Generate Results Table → Convert to HTML File, Prepare CSV Data → Convert to CSV File, Format Response → Respond to Webhook  

15. **Credentials Setup:**  
    - Add OpenAI API credentials with proper API key or OAuth2 token under "OpenAI Chat Model" node.  

16. **Testing:**  
    - Deploy the workflow and test by sending an HTTP POST request to the webhook URL with a test paper file and optional JSON body containing `studentName` and `assignmentTitle`.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                            | Context or Link                                                                                                    |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Automates AI-driven assignment grading with HTML and CSV output. Designed for educators evaluating submissions with consistent criteria and exportable results.                         | Sticky Note near workflow start                                                                                   |
| Setup Instructions: 1. Configure webhook URL and text extraction parameters. 2. Add OpenAI API key, customize grading prompts, define Output Parser JSON schema.                        | Sticky Note near AI Agent node                                                                                    |
| Prerequisites: OpenAI API key, webhook platform, n8n instance. Use Cases: University exam grading, corporate training assessments. Customization: Add PDF output, integrate LMS systems. | Sticky Note near "Prepare Assignment Data" node                                                                   |
| Benefits include consistent AI grading, multi-format exports, and reducing grading time by approximately 90%.                                                                           | Sticky Note near "Prepare Assignment Data" node                                                                   |
| GPT-4-Turbo model used for efficient and cost-effective grading with detailed feedback generation.                                                                                      | Implied from node configuration                                                                                   |
| For best results, ensure uploaded test papers are in supported formats and legible for text extraction.                                                                                  | Operational recommendation inferred from file extraction node                                                    |

---

**Disclaimer:** The provided description and documentation arise exclusively from an automated n8n workflow designed for lawful, ethical use. All processed data is legal and public. This documentation omits raw JSON except where necessary for clarity.
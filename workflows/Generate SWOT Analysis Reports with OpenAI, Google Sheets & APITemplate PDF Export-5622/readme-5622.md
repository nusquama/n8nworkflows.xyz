Generate SWOT Analysis Reports with OpenAI, Google Sheets & APITemplate PDF Export

https://n8nworkflows.xyz/workflows/generate-swot-analysis-reports-with-openai--google-sheets---apitemplate-pdf-export-5622


# Generate SWOT Analysis Reports with OpenAI, Google Sheets & APITemplate PDF Export

### 1. Workflow Overview

This workflow automates the generation of professional SWOT analysis reports for companies, leveraging structured input data from Google Sheets, AI-powered text extraction and narrative generation (using OpenAI and optionally DeepSeek), and automated PDF report creation and emailing. It is designed for business analysts, consultants, or investors who want to rapidly produce detailed, investor-grade SWOT reports.

The workflow is logically divided into these main functional blocks:

- **1.1 Input Reception & Data Loading:** Manual trigger initiates the workflow and loads company data from a Google Sheets document.
- **1.2 AI-Driven SWOT Extraction:** Processes raw company data to extract categorized SWOT elements (strengths, weaknesses, opportunities, threats) using a structured AI agent.
- **1.3 SWOT Narrative Generation:** For each SWOT category, generates detailed strategic narrative sections using AI agents tailored to produce investor-grade report text.
- **1.4 SWOT Sections Formatting:** Converts the AI-generated markdown/plain text SWOT narratives into structured HTML for report formatting.
- **1.5 Report Assembly:** Pulls all formatted sections from Google Sheets, generates introduction, conclusion, table of contents, and title page using AI agents, then combines all content into a single HTML document.
- **1.6 PDF Creation & Distribution:** Converts the assembled HTML report into a multi-page PDF via APITemplate.io, downloads the PDF, and emails it to a designated recipient.
- **1.7 Data Persistence:** Uploads the generated report sections and metadata back into Google Sheets for record-keeping.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Loading

- **Overview:** Starts workflow on user command and loads structured company data from Google Sheets.
- **Nodes Involved:**  
  - *When clicking ‘Test workflow’* (Manual Trigger)  
  - *Google Sheets* (Data Loader)
- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts workflow execution manually.  
    - Config: No parameters.  
    - Inputs: None  
    - Outputs: Connected to Google Sheets node  
    - Potential Failures: None expected.

  - **Google Sheets**  
    - Type: Google Sheets node (version 4.5)  
    - Role: Reads the "Company Info Input" tab from a specific Google Sheet document by ID and sheet name gid=0.  
    - Config: Reads all rows to provide company profile data including fields like Company Name, Industry, Market Region, Financials, etc.  
    - Inputs: Manual Trigger  
    - Outputs: To AI Agent node  
    - Edge Cases: Authentication errors, sheet not found, or no data present.

#### 2.2 AI-Driven SWOT Extraction

- **Overview:** Uses an AI agent to parse raw company data input and extract a structured JSON of SWOT categories without interpretation.
- **Nodes Involved:**  
  - *AI Agent* (LangChain Agent for SWOT extraction)  
  - *Structured Output Parser* (Parse AI JSON output)
- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent (OpenAI-powered)  
    - Role: Receives company data fields and applies a strict extraction prompt to produce a JSON with arrays for strengths, weaknesses, opportunities, threats.  
    - Config: Custom prompt with detailed instructions to extract only clearly classified SWOT phrases.  
    - Inputs: Google Sheets output  
    - Outputs: Structured JSON text for SWOT categories  
    - Edge Cases: API rate limits, incomplete input data, or parsing errors.

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Converts AI JSON text output into parsed structured data for downstream use.  
    - Inputs: AI Agent output  
    - Outputs: Parsed SWOT JSON arrays  
    - Edge Cases: Parsing failures if AI output is malformed.

#### 2.3 SWOT Narrative Generation

- **Overview:** Generates investor-grade detailed narrative sections for each SWOT category using strategic analyst AI agents.
- **Nodes Involved:**  
  - *Strengths Analysis*  
  - *Weaknesses Analysis*  
  - *Opportunities Analysis*  
  - *Threats Analysis*
- **Node Details:**

  Each node is a LangChain Agent configured with a system prompt instructing it to write a multi-paragraph, investor-level analysis section for the assigned SWOT category. They receive:

  - Full company data from Google Sheets  
  - The extracted SWOT category content from the Structured Output Parser

  Outputs are raw markdown/plain text narratives.

  - Inputs: Parsed SWOT JSON + company data  
  - Outputs: SWOT narrative text  
  - Edge Cases: API errors, incomplete inputs, or AI hallucinations (mitigated by prompt design).

#### 2.4 SWOT Sections Formatting

- **Overview:** Converts plaintext or markdown SWOT narratives into well-structured HTML suitable for professional report formatting.
- **Nodes Involved:**  
  - *Strengths Section Formatting*  
  - *Weaknesses Section Formatting*  
  - *Opportunities Section Formatting*  
  - *Threats Section Formatting*
- **Node Details:**

  Each node is a LangChain Agent designed to strictly convert input text into semantic HTML, following a defined structure:

  - Main section headers as `<h2>`  
  - Sub-section headers as `<h3>`  
  - Paragraphs wrapped in `<p>` tags  
  - No rephrasing or content alteration

  - Inputs: SWOT narrative text  
  - Outputs: HTML-formatted SWOT sections  
  - Edge Cases: Handling loosely structured input, missing headings, or multi-paragraph blocks.

#### 2.5 Report Assembly

- **Overview:** Collects all formatted sections, generates introduction, conclusion, table of contents, and title page, then merges all into a single HTML report.
- **Nodes Involved:**  
  - *Pull Info Again*, *Pull Info Again1*, *Pull Info Again2*, *Pull Info Again3* (Google Sheets readers)  
  - *Write The Introduction* (AI Agent)  
  - *Write The Conclusion* (AI Agent)  
  - *Table of Contents* (AI Agent)  
  - *Title Page* (AI Agent)  
  - *GetName* (Set node for company name)  
  - *Combine Content* (Code node to concatenate HTML sections)
- **Node Details:**

  - Google Sheets nodes re-fetch updated sections from the spreadsheet to ensure latest content is used.
  - Introduction and Conclusion nodes generate executive-level multi-paragraph HTML introductions and conclusions.
  - Table of Contents node generates a styled HTML ToC based on the report structure.
  - Title Page node produces a minimal centered HTML cover page.
  - Combine Content node merges all HTML parts with spacing for final report body.

  - Inputs: SWOT HTML sections, company metadata  
  - Outputs: Single combined HTML document  
  - Edge Cases: Data sync issues, partial data, formatting inconsistencies.

#### 2.6 PDF Creation & Distribution

- **Overview:** Converts the assembled HTML report into a multi-page PDF and emails it to a specified recipient.
- **Nodes Involved:**  
  - *Generate PDF* (HTTP Request to APITemplate.io)  
  - *Download PDF* (HTTP Request to download generated PDF)  
  - *Send Report* (Gmail node)
- **Node Details:**

  - Generate PDF sends the combined HTML content to APITemplate.io PDF generation API with layout and styling options.
  - Download PDF retrieves the PDF file URL after generation.
  - Send Report emails the PDF as an attachment to a predefined email address.

  - Inputs: Combined HTML content  
  - Outputs: PDF file emailed  
  - Edge Cases: Network/API errors, file size limits, email delivery failures.

#### 2.7 Data Persistence

- **Overview:** Saves generated SWOT sections, introduction, conclusion, table of contents, and title page HTML back to Google Sheets for archival and further use.
- **Nodes Involved:**  
  - *Upload Strengths*  
  - *Upload Weaknesses*  
  - *Upload Opportunities*  
  - *Upload Threats*  
  - *Upload Introduction*  
  - *Upload Conclusion*  
  - *Send ToC*  
  - *Send ToC1* (for Title Page)
- **Node Details:**

  - Each node updates respective columns in the Google Sheet based on company name matching.
  - Ensures synchronized record of all generated report components.

  - Inputs: HTML sections, narrative text  
  - Outputs: Updated Google Sheet rows  
  - Edge Cases: Sheet write permissions, concurrency conflicts.

---

### 3. Summary Table

| Node Name                   | Node Type                           | Functional Role                              | Input Node(s)                             | Output Node(s)                      | Sticky Note                                                                                                  |
|-----------------------------|-----------------------------------|----------------------------------------------|------------------------------------------|------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger                    | Initiates workflow manually                   | None                                     | Google Sheets                      | Setup guide and prerequisites detailed in Sticky Note1                                                     |
| Google Sheets               | Google Sheets (v4.5)               | Loads company data from Google Sheets         | When clicking ‘Test workflow’             | AI Agent                          | See Setup Guide                                                                                              |
| AI Agent                   | LangChain Agent                   | Extracts structured SWOT JSON from raw data  | Google Sheets                            | Structured Output Parser           | Strict extraction prompt to isolate SWOT elements                                                           |
| Structured Output Parser    | LangChain Output Parser           | Parses AI JSON output into structured data    | AI Agent                                | Strengths Analysis, Weaknesses Analysis, Opportunities Analysis, Threats Analysis |                                                                                                              |
| Strengths Analysis          | LangChain Agent                   | Generates investor-grade “Strengths” narrative| Structured Output Parser                 | Strengths Section Formatting       |                                                                                                              |
| Weaknesses Analysis         | LangChain Agent                   | Generates investor-grade “Weaknesses” narrative| Structured Output Parser                 | Weaknesses Section Formatting      |                                                                                                              |
| Opportunities Analysis      | LangChain Agent                   | Generates investor-grade “Opportunities” narrative| Structured Output Parser                 | Opportunities Section Formatting   |                                                                                                              |
| Threats Analysis            | LangChain Agent                   | Generates investor-grade “Threats” narrative  | Structured Output Parser                 | Threats Section Formatting         |                                                                                                              |
| Strengths Section Formatting| LangChain Agent                   | Converts Strengths narrative into HTML         | Strengths Analysis                      | Upload Strengths                  |                                                                                                              |
| Weaknesses Section Formatting| LangChain Agent                  | Converts Weaknesses narrative into HTML        | Weaknesses Analysis                     | Upload Weaknesses                 |                                                                                                              |
| Opportunities Section Formatting| LangChain Agent              | Converts Opportunities narrative into HTML    | Opportunities Analysis                  | Upload Opportunities              |                                                                                                              |
| Threats Section Formatting  | LangChain Agent                   | Converts Threats narrative into HTML           | Threats Analysis                       | Upload Threats                   |                                                                                                              |
| Upload Strengths            | Google Sheets                    | Saves formatted Strengths HTML to Sheets      | Strengths Section Formatting            | Merge                            |                                                                                                              |
| Upload Weaknesses           | Google Sheets                    | Saves formatted Weaknesses HTML to Sheets     | Weaknesses Section Formatting           | Merge                            |                                                                                                              |
| Upload Opportunities        | Google Sheets                    | Saves formatted Opportunities HTML to Sheets  | Opportunities Section Formatting        | Merge                            |                                                                                                              |
| Upload Threats              | Google Sheets                    | Saves formatted Threats HTML to Sheets         | Threats Section Formatting               | Merge                            |                                                                                                              |
| Merge                      | Merge Node                      | Merges the four SWOT upload outputs            | Upload Strengths, Upload Weaknesses, Upload Opportunities, Upload Threats | Limit                            |                                                                                                              |
| Limit                      | Limit Node                     | Ensures only one item proceeds downstream      | Merge                                  | Pull Info Again                  |                                                                                                              |
| Pull Info Again             | Google Sheets                   | Reads updated report sections from Sheets      | Limit                                   | Write The Introduction           |                                                                                                              |
| Write The Introduction      | LangChain Agent                 | Generates HTML executive introduction          | Pull Info Again                        | Upload Introduction             |                                                                                                              |
| Upload Introduction         | Google Sheets                   | Saves introduction HTML text to Sheets         | Write The Introduction                 | Pull Info Again1                |                                                                                                              |
| Pull Info Again1            | Google Sheets                   | Reads updated introduction and SWOT sections   | Upload Introduction                    | Write The Conclusion            |                                                                                                              |
| Write The Conclusion        | LangChain Agent                 | Generates HTML executive conclusion             | Pull Info Again1                       | Upload Conclusion              |                                                                                                              |
| Upload Conclusion           | Google Sheets                   | Saves conclusion HTML text to Sheets            | Write The Conclusion                  | Pull Info Again2               |                                                                                                              |
| Pull Info Again2            | Google Sheets                   | Reads updated SWOT sections and intro/conclusion| Upload Conclusion                    | Table of Contents, GetName      |                                                                                                              |
| Table of Contents           | LangChain Agent                 | Generates styled HTML Table of Contents          | Pull Info Again2                      | Send ToC                      |                                                                                                              |
| Send ToC                   | Google Sheets                   | Saves Table of Contents HTML to Sheets          | Table of Contents                    | GetName                       |                                                                                                              |
| GetName                    | Set Node                       | Stores company name for title page generation    | Send ToC                            | Title Page                    |                                                                                                              |
| Title Page                 | LangChain Agent                 | Generates minimal HTML title page                 | GetName                            | Send ToC1                    |                                                                                                              |
| Send ToC1                  | Google Sheets                   | Saves title page HTML to Sheets                   | Title Page                         | Pull Info Again3              |                                                                                                              |
| Pull Info Again3            | Google Sheets                   | Reads all report components for final assembly  | Send ToC1                           | Combine Content              |                                                                                                              |
| Combine Content            | Code Node                     | Concatenates all report HTML sections into one   | Pull Info Again3                    | Generate PDF                 |                                                                                                              |
| Generate PDF               | HTTP Request (APITemplate.io)  | Sends HTML to API to generate multi-page PDF     | Combine Content                    | Download PDF                 |                                                                                                              |
| Download PDF               | HTTP Request                  | Downloads the generated PDF file                   | Generate PDF                      | Send Report                 |                                                                                                              |
| Send Report                | Gmail Node                   | Sends the PDF report as email attachment           | Download PDF                      | None                        |                                                                                                              |
| Sticky Note1               | Sticky Note                   | Setup guide and prerequisite documentation         | None                             | None                        | Contains detailed setup instructions and resource links                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: "When clicking ‘Test workflow’"  
   - Purpose: Manual start of the workflow.

2. **Add Google Sheets Node:**  
   - Name: "Google Sheets"  
   - Credentials: Connect your Google Sheets OAuth2 credentials.  
   - Operation: Read rows from the "Company Info Input" tab, specifying document ID and sheet gid=0.  
   - Output: Pass company data to next node.

3. **Add LangChain Agent Node for SWOT Extraction:**  
   - Name: "AI Agent"  
   - Model: OpenAI GPT-4.1-mini or equivalent.  
   - Prompt: Provide strict instructions to extract SWOT components from company data fields as JSON with `strengths`, `weaknesses`, `opportunities`, `threats` arrays.  
   - Input: Company data from Google Sheets node.  
   - Output: Raw JSON text.

4. **Add LangChain Structured Output Parser:**  
   - Name: "Structured Output Parser"  
   - Purpose: Parse AI JSON output to structured JSON data.  
   - Input: AI Agent output.

5. **Create Four Parallel LangChain Agent Nodes for SWOT Narratives:**  
   - Names: "Strengths Analysis", "Weaknesses Analysis", "Opportunities Analysis", "Threats Analysis"  
   - Model: OpenAI GPT-4.1-mini or equivalent.  
   - Prompt: Each node uses a specialized prompt to convert raw SWOT lists and company data into detailed investor-grade narrative text.  
   - Input: Structured Output Parser + company data.

6. **Create Four LangChain Agent Nodes for HTML Formatting:**  
   - Names: "Strengths Section Formatting", "Weaknesses Section Formatting", "Opportunities Section Formatting", "Threats Section Formatting"  
   - Prompt: Convert plaintext markdown SWOT narratives into strict semantic HTML with `<h2>`, `<h3>`, and `<p>` tags.  
   - Input: Corresponding SWOT narrative text.

7. **Add Google Sheets Nodes to Upload Each Formatted Section:**  
   - Names: "Upload Strengths", "Upload Weaknesses", "Upload Opportunities", "Upload Threats"  
   - Operation: Update Google Sheets columns with HTML content for each SWOT category.  
   - Matching: Use "Company Name" column to identify rows.

8. **Add Merge Node:**  
   - Number of inputs: 4  
   - Inputs: All four Upload nodes.  
   - Purpose: Synchronize completion of all SWOT uploads.

9. **Add Limit Node:**  
   - Purpose: Pass only one item downstream to prevent concurrency issues.

10. **Add Google Sheets Node "Pull Info Again":**  
    - Reads updated SWOT sections from the sheet.

11. **Add LangChain Agent "Write The Introduction":**  
    - Prompt: Generate HTML introduction for the report using all SWOT sections and company metadata.  
    - Input: Pull Info Again output.

12. **Add Google Sheets Node "Upload Introduction":**  
    - Upload introduction HTML back to the sheet.

13. **Add Google Sheets Node "Pull Info Again1":**  
    - Reads updated introduction and SWOT sections.

14. **Add LangChain Agent "Write The Conclusion":**  
    - Prompt: Generate HTML conclusion section.  
    - Input: Pull Info Again1 output.

15. **Add Google Sheets Node "Upload Conclusion":**  
    - Upload conclusion HTML back to the sheet.

16. **Add Google Sheets Node "Pull Info Again2":**  
    - Reads all report parts including intro, SWOT sections, conclusion for ToC and title page generation.

17. **Add LangChain Agent "Table of Contents":**  
    - Prompt: Generate styled HTML Table of Contents based on report structure.  
    - Input: Pull Info Again2 output.

18. **Add Google Sheets Node "Send ToC":**  
    - Upload ToC HTML to sheet.

19. **Add Set Node "GetName":**  
    - Store company name for use in title page generation.

20. **Add LangChain Agent "Title Page":**  
    - Prompt: Generate minimal centered HTML title page with company name.  
    - Input: From GetName.

21. **Add Google Sheets Node "Send ToC1":**  
    - Upload title page HTML.

22. **Add Google Sheets Node "Pull Info Again3":**  
    - Reads all HTML report components for final assembly.

23. **Add Code Node "Combine Content":**  
    - Concatenate all report HTML parts with spacing into single HTML document.

24. **Add HTTP Request Node "Generate PDF":**  
    - Send combined HTML to APITemplate.io API for PDF generation.  
    - Configure paper size, margins, header/footer styling.  
    - Use API key credentials.

25. **Add HTTP Request Node "Download PDF":**  
    - Download generated PDF via provided URL.

26. **Add Gmail Node "Send Report":**  
    - Email the PDF as an attachment to a specified recipient.  
    - Configure Gmail OAuth2 credentials.

27. **Add Sticky Note:**  
    - Add detailed setup guide and instructions for operators.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                          |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| Setup guide authored by Sebastian/OptiLever with prerequisites including OpenAI API, Google Sheets, APITemplate.io, Gmail OAuth2 | Setup Guide Sticky Note in workflow; author LinkedIn: https://www.linkedin.com/in/sebastian-9ab9b9242/ |
| Google Sheets template for input data: "SWOT Analysis" with "Company Info Input" tab                                 | https://docs.google.com/spreadsheets/d/19k1nKNIj8J63e4LoR2yVDq2YN5fOFMHBOpMnrakLNOM/edit?usp=sharing         |
| APITemplate.io used for HTML to PDF conversion with professional layout features                                    | https://apitemplate.io/?via=lew                                         |
| DeepSeek API optionally integrated as alternative AI reasoning model                                               | Optional node in workflow                                                |

---

**Disclaimer:** The provided text and nodes come exclusively from an automated workflow created with n8n, respecting all applicable content policies. The data processed is legal and public.
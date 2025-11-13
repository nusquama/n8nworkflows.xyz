Comprehensive Research Report Generator with Gemini AI, Web Search & PDF Delivery

https://n8nworkflows.xyz/workflows/comprehensive-research-report-generator-with-gemini-ai--web-search---pdf-delivery-6455


# Comprehensive Research Report Generator with Gemini AI, Web Search & PDF Delivery

### 1. Workflow Overview

This workflow automates the generation of a comprehensive research report based on a user-submitted search topic. It integrates AI-driven topic planning, deep web research, content synthesis, structured formatting, and delivery of the final report as a PDF via email. The workflow leverages multiple AI models including Google Gemini AI and OpenRouter (Anthropic Claude), integrates with Tavily for advanced web search, uses Google Sheets as a persistent data store, and generates a styled, multi-chapter HTML report converted to PDF.

The logical flow is divided into these main blocks:

- **1.1 Input Reception**: Accepts user input through a web form.
- **1.2 Topic Planning (AI Processing)**: AI generates five detailed research topics from the input.
- **1.3 Introduction and Chapter Titles Generation**: AI composes report title, introduction, and chapter headings.
- **1.4 Web Research and Content Gathering**: For each of the five topics, performs deep web research via Tavily API.
- **1.5 Content Synthesis and Formatting**: AI synthesizes research content into professional HTML reports for each topic.
- **1.6 Content Aggregation and Google Sheets Update**: Extracts, aggregates, and updates Google Sheets with content, sources, sections.
- **1.7 Table of Contents and Sources Section Generation**: AI creates a structured Table of Contents and sources list in HTML.
- **1.8 Final Report Assembly and PDF Generation**: Combines all content, generates a PDF via an external API.
- **1.9 Report Delivery**: Sends the PDF report via Gmail to the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview**: Captures user input (search topic and email) via a web form to trigger the workflow.
- **Nodes Involved**: `On form submission`
- **Node Details**:
  - Type: `Form Trigger`
  - Role: Entry point; listens for HTTP form submissions.
  - Configuration: Form titled "The Deepest Research" with fields "Search Topic" (required) and "Email" (required).
  - Inputs: External user input.
  - Outputs: JSON with user data.
  - Edge cases: Form submission failures, missing required fields.
  - Sub-workflow: None.

#### 1.2 Topic Planning (AI Processing)

- **Overview**: Generates five in-depth subtopics from the main search topic using AI.
- **Nodes Involved**: `Plan Topics`, `5 Topics`, `Google Gemini Chat Model`, `OpenRouter Chat Model`
- **Node Details**:
  - `Plan Topics`:
    - Type: LangChain Agent (AI)
    - Role: Creates five distinct research topics in JSON format.
    - Uses a detailed system prompt to ensure depth, uniqueness, and relevance.
    - Inputs: Search Topic from form.
    - Outputs: JSON object with keys topic_1 to topic_5.
    - Version: 1.8
    - Edge cases: AI prompt failures, malformed JSON output.
  - `5 Topics`:
    - Type: Output Parser Structured
    - Role: Parses and validates AI JSON output into structured fields.
    - Inputs: AI raw output.
    - Outputs: Parsed JSON with topics.
  - `Google Gemini Chat Model` and `OpenRouter Chat Model` nodes:
    - Type: AI language model nodes.
    - Role: Alternate language models configured (Google Gemini and OpenRouter Anthropic Claude) for topic planning.
    - Credentials: Google Palm API and OpenRouter API.
    - Edge cases: API rate limits, authentication failures.

#### 1.3 Introduction and Chapter Titles Generation

- **Overview**: Using AI, generates the report's main title, an introduction section, and five chapter headings based on the planned topics.
- **Nodes Involved**: `Intro`, `Title, Intro, Chapters`, `Google Gemini Chat Model1`, `OpenRouter Chat Model1`
- **Node Details**:
  - `Intro`:
    - Type: LangChain Agent (AI)
    - Role: Creates HTML formatted title, intro, and chapter headers.
    - Uses a system prompt with detailed styling instructions (HTML tags, colors, layout).
    - Inputs: Search Topic and planned topics.
    - Outputs: JSON with title, introduction, and chapters formatted in HTML.
  - `Title, Intro, Chapters`:
    - Type: Output Parser Structured
    - Role: Parses AI output into structured JSON fields.
  - `Google Gemini Chat Model1` and `OpenRouter Chat Model1`:
    - AI nodes for introduction generation.
    - Credentials as above.
  - Edge cases: Improper HTML formatting, AI API errors.

#### 1.4 Web Research and Content Gathering

- **Overview**: For each subtopic, performs advanced web research using Tavily API to gather relevant search results.
- **Nodes Involved**: `Switch`, `Tavily`, `Tavily1`, `Tavily2`, `Tavily3`, `Tavily4`, `Split Out2`, `Split Out3`, `Split Out4`, `Split Out5`, `Split Out6`, `Code`, `Code1`, `Code2`, `Code3`, `Code4`
- **Node Details**:
  - `Switch`:
    - Type: Switch node
    - Role: Routes each topic to a corresponding Tavily HTTP Request node.
    - Inputs: Topics from the planning stage.
    - Outputs: One of five branches, one per topic.
  - `Tavily` through `Tavily4`:
    - Type: HTTP Request
    - Role: POST requests to Tavily API with query = subtopic.
    - Parameters: Search depth "advanced", max results 5, chunks per source 3.
    - Auth: HTTP Header Authentication with Tavily API key.
    - Outputs: JSON search results.
    - Edge cases: API rate limits, network errors, empty results.
  - `Split Out` nodes:
    - Type: Split Out
    - Role: Extracts the "results" array from Tavily responses for further processing.
  - `Code` nodes:
    - Type: Code (JavaScript)
    - Role: Number and format URLs starting with incremental numbering (1-5, then 6-10, etc.).
    - Outputs: Items with numbered URLs for source attribution.
    - Edge cases: Empty arrays, unexpected data formats.

#### 1.5 Content Synthesis and Formatting

- **Overview**: Synthesizes gathered research and formats it into a professional HTML report per topic using AI.
- **Nodes Involved**: `Writer`, `Writer1`, `Writer2`, `Writer3`, `Writer4`, `Aggregate`, `Aggregate1`, `Aggregate2`, `Aggregate3`, `Aggregate4`, `Aggregate5`, `Aggregate6`, `Aggregate7`, `Aggregate8`, `Aggregate9`, `Google Gemini Chat Model2` through `Google Gemini Chat Model6`, `OpenRouter Chat Model2` through `OpenRouter Chat Model6`
- **Node Details**:
  - `Writer` nodes:
    - Type: LangChain Agent (AI)
    - Role: Generate detailed HTML reports combining title, research content, source links, and style guide.
    - Uses system prompts for strict HTML structure and source attribution with clickable links.
    - Inputs: Research content, numbered URLs, style guide from intro.
    - Outputs: Fully formatted HTML report section per topic.
  - `Aggregate` nodes:
    - Type: Aggregate
    - Role: Combines multiple AI outputs or URLs into arrays for merging or next steps.
  - `Google Gemini Chat Model2-6` and `OpenRouter Chat Model2-6`:
    - AI nodes performing the writing task with different AI providers.
    - Credentials as above.
  - Edge cases: AI output inconsistencies, malformed HTML, API errors.

#### 1.6 Content Aggregation and Google Sheets Update

- **Overview**: Extracts sections from HTML reports, aggregates them, and updates Google Sheets with content, sources, and sections per topic.
- **Nodes Involved**: `HTML`, `HTML1`, `HTML2`, `HTML3`, `HTML4`, `Combine`, `Combine1`, `Combine2`, `Combine3`, `Combine4`, `Update row in sheet`, `Update row in sheet1`, `Update row in sheet2`, `Update row in sheet3`, `Update row in sheet4`, `Google Sheets` nodes (`Get Sources`, `Send Sources`, `Send Intro`, `Get All Content`, `Get All Content1`)
- **Node Details**:
  - `HTML` nodes:
    - Type: HTML Extract
    - Role: Extract specific HTML content (titles) from the AI-generated reports.
  - `Combine` nodes:
    - Type: Code
    - Role: Combine extracted HTML sections into single arrays or strings.
  - `Update row in sheet` nodes:
    - Type: Google Sheets Update
    - Role: Update corresponding rows with topic content, sources, and sections.
    - Inputs: Combined HTML, numbered URLs, sections arrays.
    - Credentials: Google Sheets OAuth2.
  - `Get Sources`, `Send Sources`, `Send Intro`, `Get All Content`, `Get All Content1`:
    - Type: Google Sheets Read/Write
    - Role: Retrieve or update data related to topics, sources, intro, ToC.
  - Edge cases: Google Sheets API rate limits, permission errors, mismatched row keys.

#### 1.7 Table of Contents and Sources Section Generation

- **Overview**: AI generates a well-structured HTML Table of Contents and Sources section for the entire research report.
- **Nodes Involved**: `Table of Contents`, `Sources`, `Send ToC`
- **Node Details**:
  - `Table of Contents`:
    - Type: LangChain Agent (AI)
    - Role: Create an HTML ToC with chapters as `<h2>` headers and sections as ordered lists.
    - Inputs: Chapter titles and section lists.
    - Outputs: Formatted HTML ToC.
  - `Sources`:
    - Type: LangChain Agent (AI)
    - Role: Generate an HTML-formatted Sources section with clickable links and preserved numbering.
  - `Send ToC`:
    - Type: Google Sheets Update
    - Role: Save the generated ToC HTML back to Google Sheets.
  - Edge cases: Improper HTML formatting, missing sources.

#### 1.8 Final Report Assembly and PDF Generation

- **Overview**: Combines all content fields into one document, sends it to an external API to generate a PDF, then downloads the PDF.
- **Nodes Involved**: `Combine Content`, `Generate PDF`, `Download PDF`
- **Node Details**:
  - `Combine Content`:
    - Type: Code
    - Role: Concatenate all report sections, chapters, intro, ToC, sources into a single HTML string.
  - `Generate PDF`:
    - Type: HTTP Request
    - Role: POST combined HTML content to API Template.io for PDF generation.
    - Parameters: Paper size A4, margins, header/footer styling.
    - Credentials: API Template.io API key.
    - Outputs: JSON with download URL.
  - `Download PDF`:
    - Type: HTTP Request
    - Role: Downloads the generated PDF file.
    - Inputs: URL from previous node.
  - Edge cases: API failures, content encoding issues, timeout.

#### 1.9 Report Delivery

- **Overview**: Sends the generated PDF report as an email attachment to the user.
- **Nodes Involved**: `Send Report`
- **Node Details**:
  - Type: Gmail node
  - Role: Sends email with the PDF attached.
  - Parameters: Recipient email from form; subject includes search topic.
  - Credential: Gmail OAuth2.
  - Edge cases: Email sending failures, attachment size limits.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                   | Input Node(s)                       | Output Node(s)                     | Sticky Note                                              |
|-------------------------|----------------------------------|-------------------------------------------------|-----------------------------------|----------------------------------|----------------------------------------------------------|
| On form submission      | Form Trigger                     | Entry point; captures user input                 | -                                 | Plan Topics                      |                                                          |
| Plan Topics             | LangChain Agent                  | Generates 5 in-depth search topics               | On form submission                | Intro, Split Out, Set Topics     |                                                          |
| 5 Topics                | Output Parser Structured         | Parses AI output JSON for topics                  | Plan Topics (AI output)           | Plan Topics (parsed JSON)        |                                                          |
| Intro                   | LangChain Agent                  | Generates title, intro, chapter headings          | Plan Topics                      | Send Intro                      |                                                          |
| Title, Intro, Chapters  | Output Parser Structured         | Parses Intro AI output                             | Intro                           | Intro                           |                                                          |
| Switch                  | Switch                          | Routes topics to respective web search nodes     | Merge (Set Topics outputs)       | Tavily, Tavily1, Tavily2, Tavily3, Tavily4 |                                                          |
| Set Topics              | Set                             | Prepares array of topic keys for splitting        | Plan Topics                      | Split Out1                      |                                                          |
| Split Out1              | Split Out                       | Splits topics array into single topic items       | Set Topics                      | Merge1                         |                                                          |
| Merge1                  | Merge                           | Combines split topics for switch routing           | Split Out1                     | Switch                         |                                                          |
| Tavily                  | HTTP Request                    | Queries Tavily API for Topic 1                    | Switch                         | Split Out2                     |                                                          |
| Split Out2              | Split Out                       | Extracts search results for Topic 1                | Tavily                         | Code                          |                                                          |
| Code                    | Code                            | Numbers URLs for Topic 1                            | Split Out2                     | Writer                        |                                                          |
| Writer                  | LangChain Agent                 | Writes HTML report section for Topic 1             | Code                          | Aggregate                     |                                                          |
| Aggregate               | Aggregate                       | Aggregates output for Topic 1                       | Writer                        | Merge3                        |                                                          |
| Merge3                  | Merge                           | Combines content and URLs for Topic 1              | Aggregate, HTML               | Update row in sheet            |                                                          |
| HTML                    | HTML Extract                   | Extracts HTML title from Topic 1 report             | Merge3                        | Combine                       |                                                          |
| Combine                 | Code                            | Combines extracted HTML sections for Topic 1       | HTML                         | Update row in sheet            |                                                          |
| Update row in sheet     | Google Sheets Update            | Updates Google Sheet with Topic 1 content           | Combine                      | Merge2                        |                                                          |
| Tavily1                 | HTTP Request                    | Queries Tavily API for Topic 2                    | Switch                         | Split Out3                    |                                                          |
| Split Out3              | Split Out                       | Extracts search results for Topic 2                | Tavily1                       | Code1                         |                                                          |
| Code1                   | Code                            | Numbers URLs for Topic 2                            | Split Out3                    | Writer1                       |                                                          |
| Writer1                 | LangChain Agent                 | Writes HTML report section for Topic 2             | Code1                        | Aggregate3                    |                                                          |
| Aggregate3              | Aggregate                       | Aggregates output for Topic 2                       | Writer1                      | Merge4                        |                                                          |
| Merge4                  | Merge                           | Combines content and URLs for Topic 2              | Aggregate3, HTML1             | Update row in sheet1          |                                                          |
| HTML1                   | HTML Extract                   | Extracts HTML title from Topic 2 report             | Merge4                       | Combine1                      |                                                          |
| Combine1                | Code                            | Combines extracted HTML sections for Topic 2       | HTML1                        | Update row in sheet1          |                                                          |
| Update row in sheet1    | Google Sheets Update            | Updates Google Sheet with Topic 2 content           | Combine1                     | Merge2                        |                                                          |
| Tavily2                 | HTTP Request                    | Queries Tavily API for Topic 3                    | Switch                         | Split Out4                    |                                                          |
| Split Out4              | Split Out                       | Extracts search results for Topic 3                | Tavily2                       | Code2                         |                                                          |
| Code2                   | Code                            | Numbers URLs for Topic 3                            | Split Out4                    | Writer2                       |                                                          |
| Writer2                 | LangChain Agent                 | Writes HTML report section for Topic 3             | Code2                        | Aggregate5                    |                                                          |
| Aggregate5              | Aggregate                       | Aggregates output for Topic 3                       | Writer2                      | Merge5                        |                                                          |
| Merge5                  | Merge                           | Combines content and URLs for Topic 3              | Aggregate5, HTML2             | Update row in sheet2          |                                                          |
| HTML2                   | HTML Extract                   | Extracts HTML title from Topic 3 report             | Merge5                       | Combine2                      |                                                          |
| Combine2                | Code                            | Combines extracted HTML sections for Topic 3       | HTML2                        | Update row in sheet2          |                                                          |
| Update row in sheet2    | Google Sheets Update            | Updates Google Sheet with Topic 3 content           | Combine2                     | Merge2                        |                                                          |
| Tavily3                 | HTTP Request                    | Queries Tavily API for Topic 4                    | Switch                         | Split Out5                    |                                                          |
| Split Out5              | Split Out                       | Extracts search results for Topic 4                | Tavily3                       | Code3                         |                                                          |
| Code3                   | Code                            | Numbers URLs for Topic 4                            | Split Out5                    | Writer3                       |                                                          |
| Writer3                 | LangChain Agent                 | Writes HTML report section for Topic 4             | Code3                        | Aggregate7                    |                                                          |
| Aggregate7              | Aggregate                       | Aggregates output for Topic 4                       | Writer3                      | Merge6                        |                                                          |
| Merge6                  | Merge                           | Combines content and URLs for Topic 4              | Aggregate7, HTML3             | Update row in sheet3          |                                                          |
| HTML3                   | HTML Extract                   | Extracts HTML title from Topic 4 report             | Merge6                       | Combine3                      |                                                          |
| Combine3                | Code                            | Combines extracted HTML sections for Topic 4       | HTML3                        | Update row in sheet3          |                                                          |
| Update row in sheet3    | Google Sheets Update            | Updates Google Sheet with Topic 4 content           | Combine3                     | Merge2                        |                                                          |
| Tavily4                 | HTTP Request                    | Queries Tavily API for Topic 5                    | Switch                         | Split Out6                    |                                                          |
| Split Out6              | Split Out                       | Extracts search results for Topic 5                | Tavily4                       | Code4                         |                                                          |
| Code4                   | Code                            | Numbers URLs for Topic 5                            | Split Out6                    | Writer4                       |                                                          |
| Writer4                 | LangChain Agent                 | Writes HTML report section for Topic 5             | Code4                        | Aggregate9                    |                                                          |
| Aggregate9              | Aggregate                       | Aggregates output for Topic 5                       | Writer4                      | Merge7                        |                                                          |
| Merge7                  | Merge                           | Combines content and URLs for Topic 5              | Aggregate9, HTML4             | Update row in sheet4          |                                                          |
| HTML4                   | HTML Extract                   | Extracts HTML title from Topic 5 report             | Merge7                       | Combine4                      |                                                          |
| Combine4                | Code                            | Combines extracted HTML sections for Topic 5       | HTML4                        | Update row in sheet4          |                                                          |
| Update row in sheet4    | Google Sheets Update            | Updates Google Sheet with Topic 5 content           | Combine4                     | Merge2                        |                                                          |
| Merge2                  | Merge                           | Combines all topic update nodes                     | Update rows 1-5              | Limit                         |                                                          |
| Limit                   | Limit                           | Controls number of rows processed                    | Merge2                      | Get Sources                   |                                                          |
| Get Sources             | Google Sheets Read              | Retrieves sources from Google Sheets                 | Limit                       | Sources                       |                                                          |
| Sources                 | LangChain Agent                 | Generates HTML sources section                        | Get Sources                 | Send Sources                  |                                                          |
| Send Sources            | Google Sheets Update            | Saves generated sources HTML to Google Sheets        | Sources                     | Get All Content               |                                                          |
| Get All Content         | Google Sheets Read              | Reads all content for final document                  | Send Sources                | Table of Contents             |                                                          |
| Table of Contents       | LangChain Agent                 | Generates HTML Table of Contents                      | Get All Content             | Send ToC                     |                                                          |
| Send ToC                | Google Sheets Update            | Saves Table of Contents to Google Sheets              | Table of Contents           | Get All Content1              |                                                          |
| Get All Content1        | Google Sheets Read              | Reads combined content for PDF generation             | Send ToC                    | Combine Content              |                                                          |
| Combine Content         | Code                            | Concatenates all report sections and sources          | Get All Content1            | Generate PDF                 |                                                          |
| Generate PDF            | HTTP Request                   | Sends HTML to API Template.io to create PDF           | Combine Content             | Download PDF                 |                                                          |
| Download PDF            | HTTP Request                   | Downloads generated PDF file                            | Generate PDF                | Send Report                  |                                                          |
| Send Report             | Gmail Send                    | Sends PDF report via email to user                      | Download PDF                | -                            |                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: `On form submission`  
   - Configure form with title "The Deepest Research"  
   - Fields:  
     - "Search Topic" (required)  
     - "Email" (required)  
   - This node triggers the workflow on submission.

2. **Add AI Agent node for topic planning**  
   - Name: `Plan Topics`  
   - Type: LangChain Agent  
   - Input: Text expression `=Search Topic: {{ $json['Search Topic'] }}`  
   - System prompt: Instruct AI to produce 5 distinct, in-depth subtopics in a JSON format.  
   - Output parser: Add `5 Topics` node (Output Parser Structured) with example JSON schema to parse AI output.

3. **Add AI Agent node for intro and chapter titles**  
   - Name: `Intro`  
   - Input: Expression combining form input and planned topics from `Plan Topics`.  
   - System prompt: Generate HTML title, introduction, and 5 chapter headers with styling instructions.  
   - Output parser: `Title, Intro, Chapters` node to parse structured output.

4. **Split topics array**  
   - Add a `Set` node `Set Topics` to create an array of topic keys: `['topic_1','topic_2','topic_3','topic_4','topic_5']`  
   - Attach `Split Out1` node to split this array.  
   - Attach `Merge1` node to combine items.

5. **Add `Switch` node**  
   - Configure rules to route each topic key (`topic_1`..`topic_5`) to corresponding `Tavily` API HTTP Request nodes.

6. **Add five `HTTP Request` nodes** (`Tavily`, `Tavily1`, … `Tavily4`)  
   - URL: `https://api.tavily.com/search`  
   - Method: POST  
   - Body: JSON with `query` set to the current topic, `search_depth: advanced`, `max_results: 5`, `chunks_per_source: 3`  
   - Authentication: HTTP Header with Tavily API Bearer token.

7. **Add five `Split Out` nodes** to extract `results` array from each Tavily response.

8. **Add five `Code` nodes** to number URLs starting from 1, 6, 11, 16, and 21 respectively for proper source numbering.

9. **Add five `LangChain Agent` writer nodes** (`Writer` through `Writer4`)  
   - Input: Title, research content, numbered URLs, and style guide from `Switch` first output (intro).  
   - System prompt: Generate full HTML report sections for each topic with proper styling, clickable source links, and logical structure.

10. **Add five `Aggregate` nodes** to collect outputs from writer nodes.

11. **Add five `Merge` nodes** to combine aggregated content with extracted HTML titles.

12. **Add five HTML Extract nodes** (`HTML` through `HTML4`)  
    - Operation: Extract HTML content for titles from writer outputs.

13. **Add five `Code` nodes** (`Combine` through `Combine4`) to combine HTML sections.

14. **Add five Google Sheets Update nodes** (`Update row in sheet` through `Update row in sheet4`)  
    - Update content, sources, and sections for each topic in the Google Sheet by matching `Search Topic`.

15. **Add a `Merge2` node** to combine all five Google Sheets update nodes.

16. **Add `Limit` node** to control processing volume before fetching sources.

17. **Add a Google Sheets Read node `Get Sources`**  
    - Filter by `Search Topic`.

18. **Add an AI Agent node `Sources`**  
    - Generates HTML formatted clickable sources list.

19. **Add Google Sheets Update `Send Sources`** to save sources HTML.

20. **Add Google Sheets Read `Get All Content`** to fetch all report content and sections.

21. **Add AI Agent `Table of Contents`**  
    - Generates an HTML Table of Contents structured by chapters and sections.

22. **Add Google Sheets Update `Send ToC`** to save ToC HTML.

23. **Add Google Sheets Read `Get All Content1`** to fetch final combined content.

24. **Add a `Code` node `Combine Content`**  
    - Concatenate all report parts into one large HTML string.

25. **Add HTTP Request `Generate PDF`**  
    - POST HTML content to API Template.io to generate PDF.  
    - Use parameters for A4, margins, header/footer styling.  
    - Credential: API Template.io API key.

26. **Add HTTP Request `Download PDF`**  
    - Download generated PDF using returned URL.

27. **Add Gmail Send node `Send Report`**  
    - Email PDF attachment to email captured from form.  
    - Credential: Gmail OAuth2.

28. **Connect nodes respecting the data flow and trigger conditions** especially managing parallel branches for the five topics.

29. **Configure credentials** for Google Sheets OAuth2, Tavily API, OpenRouter API, Google Palm API, API Template.io, Gmail OAuth2.

30. **Test the entire workflow end-to-end** with sample inputs to ensure smooth operation and error handling.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                                                                |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| **Setup Guide** by Nate Herk detailing required accounts and API keys: Google Sheets template, Tavily API, API Template.io, OpenRouter, and Google Gemini (PaLM) API.                                                                                                                                                                                                                                                                                                                                                                                                       | [YouTube - Nate Herk](https://www.youtube.com/@nateherk) and Google Sheet Template: [Google Sheets Template Link](https://docs.google.com/spreadsheets/d/16WekkajqKqMAwrERVjQo2XdzhKCU7QcprfasZnyK0CA/edit?usp=sharing) |
| Use lightweight AI models (e.g., Gemini 2.5 Flash Lite) to balance cost and performance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Sticky notes in workflow mention "Use Lite model - affordable. Subscribe on Youtube Channel to learn more. https://www.youtube.com/@greatcommissionerofking?sub_confirmation=1" |
| AI-generated HTML reports strictly follow style guides with color #00366D for titles, background #f4f4f4 for intro, and structured use of `<h1>`, `<h2>`, `<p>`, `<hr>`.                                                                                                                                                                                                                                                                                                                                                                                                 | System prompts in `Intro` and `Writer` nodes enforce these styling rules to ensure consistent professional report presentation.              |
| Numbering of source URLs starts uniquely for each topic to avoid overlap in citations (1-5, 6-10, 11-15, 16-20, 21-25).                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Code nodes handle this numbering scheme explicitly.                                                                                           |
| Google Sheets serves as persistent storage for intermediate and final data, enabling manual review or reuse.                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Multiple Google Sheets nodes read and write to a shared sheet with defined schema.                                                           |
| External PDF generation is done through API Template.io, which requires an API key and specific JSON payload.                                                                                                                                                                                                                                                                                                                                                                                                                                                             | See `Generate PDF` node configuration.                                                                                                       |
| Email delivery uses Gmail OAuth2 credentials; ensure proper consent and quota are configured.                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | `Send Report` node sends the report to the user's email captured on form submission.                                                         |

---

This detailed reference document fully describes the workflow’s structure, logic, and configurations, enabling reproduction, modification, and troubleshooting by technical users and AI agents.
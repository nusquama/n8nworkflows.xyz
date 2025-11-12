Extract & Analyze Brand Content with Bright Data and Google Gemini

https://n8nworkflows.xyz/workflows/extract---analyze-brand-content-with-bright-data-and-google-gemini-3846


# Extract & Analyze Brand Content with Bright Data and Google Gemini

### 1. Workflow Overview

This workflow, **“Brand Content Extract, Summarize & Sentiment Analysis with Bright Data”**, automates the extraction, summarization, and sentiment evaluation of brand-related content from web sources using Bright Data’s Web Unlocker API and Google Gemini AI models.  

It is designed for brand managers, marketing analysts, PR teams, data scientists, and growth hackers who need scalable, insightful monitoring of how brands are portrayed online.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Initialization** — Manual trigger and setting input parameters such as target URL and Bright Data zone.
- **1.2 Web Data Extraction** — Extract brand-related content from web pages using Bright Data’s Web Unlocker API.
- **1.3 Textual Data Processing** — Clean and convert raw markdown content into readable textual data using Google Gemini.
- **1.4 Summarization** — Create concise summaries of extracted content using Google Gemini and a summarization chain.
- **1.5 Sentiment Analysis** — Analyze the sentiment of the extracted content with structured responses using Google Gemini.
- **1.6 Output & Persistence** — Send notifications via webhook and save results (text, summary, sentiment) as files on disk.
- **1.7 Utility & Formatting** — Convert JSON results into binary data for file writing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  Starts workflow execution manually and defines the URL and Bright Data zone parameters for data extraction.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set URL and Bright Data Zone (Set Node)  
  - Sticky Note (Instructional note)

- **Node Details:**  

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: No parameters; user triggers workflow on demand.  
    - Connections: Outputs to “Set URL and Bright Data Zone”.  
    - Edge Cases: None typical; manual trigger failure unlikely.

  - **Set URL and Bright Data Zone**  
    - Type: Set Node  
    - Role: Sets two key string parameters: `url` (target brand content URL) and `zone` (Bright Data Web Unlocker zone name).  
    - Configuration:  
      - url = https://www.amazon.com/TP-Link-Dual-Band-Archer-BE230-HomeShield/dp/B0DC99N2T8  
      - zone = web_unlocker1  
    - Connections: Outputs to “Perform Bright Data Web Request”.  
    - Edge Cases: Incorrect URL or zone will cause extraction failures downstream.

  - **Sticky Note**  
    - Type: Sticky Note (UI annotation)  
    - Role: Documentation reminder to set URL and Webhook Notification URL before running.  
    - Content includes instructions for initial setup and usage notes.

---

#### 2.2 Web Data Extraction

- **Overview:**  
  Uses Bright Data’s Web Unlocker API to scrape raw markdown content from the given URL.

- **Nodes Involved:**  
  - Perform Bright Data Web Request (HTTP Request)

- **Node Details:**  

  - **Perform Bright Data Web Request**  
    - Type: HTTP Request  
    - Role: Sends POST request to Bright Data API for web unlocking and scraping.  
    - Configuration:  
      - URL: https://api.brightdata.com/request  
      - Method: POST  
      - Authentication: Header Auth with Bearer token (configured in Credentials)  
      - Body Parameters: zone, url (with query params for unlocker & API), format=raw, data_format=markdown  
    - Connections: Outputs to “Markdown to Textual Data Extractor” and “Summarize Content”.  
    - Edge Cases:  
      - Authentication failure if token invalid.  
      - API rate limiting or timeouts.  
      - Invalid zone or URL causing no data or errors.  
    - Credentials: Requires configured Header Auth credential with Bearer token.

---

#### 2.3 Textual Data Processing

- **Overview:**  
  Converts Bright Data’s raw markdown content to clean textual data suitable for downstream AI processing.

- **Nodes Involved:**  
  - Google Gemini Chat Model for Data Extract (Google Gemini LLM Chat Node)  
  - Markdown to Textual Data Extractor (Langchain LLM Chain Node)  
  - Sticky Note4 (Documentation)

- **Node Details:**  

  - **Google Gemini Chat Model for Data Extract**  
    - Type: Google Gemini LLM Chat Node  
    - Role: Provides AI model interface for content extraction.  
    - Configuration: Uses “models/gemini-2.0-flash-exp” model.  
    - Credentials: Google Palm API key configured.  
    - Connections: Outputs to “Markdown to Textual Data Extractor”.  
    - Edge Cases: API quota limits, network errors.

  - **Markdown to Textual Data Extractor**  
    - Type: Langchain LLM Chain (Define Prompt)  
    - Role: Processes markdown input to extract clean textual data without formatting or links.  
    - Configuration:  
      - Prompt instructs the model to output textual data only, no links, scripts, CSS etc.  
      - Input expression: `{{ $json.data }}` (content from Bright Data response)  
    - Connections: Outputs to:  
      - “AI Sentiment Analyzer with the structured response”  
      - “Initiate a Webhook Notification for Markdown to Textual Data Extraction”  
      - “Create a binary data for textual data”  
    - Edge Cases: Model may misinterpret markdown if malformed; connection failures.

  - **Sticky Note4**  
    - Explains the textual data extraction block and its purpose.

---

#### 2.4 Summarization

- **Overview:**  
  Generates a concise summary of the extracted brand content using Google Gemini and a summarization chain.

- **Nodes Involved:**  
  - Google Gemini Chat Model for Summary (Google Gemini LLM Chat Node)  
  - Summarize Content (Langchain Summarization Chain Node)  
  - Create a binary data for summary (Function Node)  
  - Write the summary file to disk (Read/Write File Node)  
  - Initiate a Webhook Notification for Summarization (HTTP Request)  
  - Sticky Note2 (Documentation)

- **Node Details:**  

  - **Google Gemini Chat Model for Summary**  
    - Type: Google Gemini LLM Chat Node  
    - Role: AI model interface for summarization tasks.  
    - Configuration: Uses “models/gemini-2.0-flash-exp” model.  
    - Credentials: Google Palm API key.  
    - Connections: Outputs to “Summarize Content”.  
    - Edge Cases: Same as other Gemini nodes (quota, connectivity).

  - **Summarize Content**  
    - Type: Langchain Chain Summarization  
    - Role: Processes input text to generate concise summary.  
    - Configuration:  
      - Prompt: “Write a concise summary of the following: {text}”  
      - Chunking mode: advanced (handles large inputs by chunking)  
    - Connections: Outputs to “Initiate a Webhook Notification for Summarization” and “Create a binary data for summary”.  
    - Edge Cases: Large input size may cause partial summarization.

  - **Create a binary data for summary**  
    - Type: Function Node  
    - Role: Converts JSON summary into base64-encoded binary for file writing.  
    - Configuration: Uses Buffer to encode JSON string.  
    - Connections: Outputs to “Write the summary file to disk”.  
    - Edge Cases: Encoding failures unlikely.

  - **Write the summary file to disk**  
    - Type: Read/Write File Node  
    - Role: Saves summary JSON to file `d:\Brand-Content-Summary.json`.  
    - Configuration: Write operation, file path fixed.  
    - Connections: Terminal node.  
    - Edge Cases: File permission errors, disk space.

  - **Initiate a Webhook Notification for Summarization**  
    - Type: HTTP Request  
    - Role: Sends summary text to external webhook URL (default is webhook.site).  
    - Configuration: Sends `summary` parameter with summary text from JSON response.  
    - Connections: Terminal node.  
    - Edge Cases: Webhook endpoint down or rejects request.

  - **Sticky Note2**  
    - Short note indicating this block handles summarization.

---

#### 2.5 Sentiment Analysis

- **Overview:**  
  Performs sentiment analysis on the extracted content and outputs structured sentiment data.

- **Nodes Involved:**  
  - Google Gemini Chat Model for Sentiment Analyzer (Google Gemini LLM Chat Node)  
  - AI Sentiment Analyzer with the structured response (Langchain Information Extractor Node)  
  - Create a binary data for sentiment analysis (Function Node)  
  - Write the AI Sentiment analysis file to disk (Read/Write File Node)  
  - Initiate a Webhook Notification for AI Sentiment Analyzer (HTTP Request)  
  - Sticky Note3 (Documentation)

- **Node Details:**  

  - **Google Gemini Chat Model for Sentiment Analyzer**  
    - Type: Google Gemini LLM Chat Node  
    - Role: AI interface for sentiment analysis.  
    - Configuration: Uses “models/gemini-2.0-flash-exp”.  
    - Credentials: Google Palm API.  
    - Connections: Outputs to “AI Sentiment Analyzer with the structured response”.  
    - Edge Cases: Same as other Gemini nodes.

  - **AI Sentiment Analyzer with the structured response**  
    - Type: Langchain Information Extractor  
    - Role: Performs sentiment analysis and outputs JSON with schema validation.  
    - Configuration:  
      - System prompt: “You are an expert sentiment analyzer.”  
      - Input text expression: content from “Perform Bright Data Web Request” node’s `data` field.  
      - Output schema: Array of objects with sentiment ("Positive", "Neutral", "Negative"), confidence_score (0-1), and sentence explanation.  
    - Connections: Outputs to “Initiate a Webhook Notification for AI Sentiment Analyzer” and “Create a binary data for sentiment analysis”.  
    - Edge Cases: Model may misclassify sentiment; schema validation errors.

  - **Create a binary data for sentiment analysis**  
    - Type: Function Node  
    - Role: Encodes JSON sentiment analysis output into base64 binary for file writing.  
    - Connections: Outputs to “Write the AI Sentiment analysis file to disk”.  
    - Edge Cases: Encoding failures unlikely.

  - **Write the AI Sentiment analysis file to disk**  
    - Type: Read/Write File Node  
    - Role: Writes sentiment JSON to `d:\Brand-Content-Sentiment-Analysis.json`.  
    - Edge Cases: File system errors.

  - **Initiate a Webhook Notification for AI Sentiment Analyzer**  
    - Type: HTTP Request  
    - Role: Sends sentiment analysis results to webhook endpoint.  
    - Edge Cases: Endpoint availability.

  - **Sticky Note3**  
    - Notes this block is dedicated to sentiment analysis.

---

#### 2.6 Output & Persistence

- **Overview:**  
  Handles notifications and persistence of all outputs: textual data, summary, and sentiment.

- **Nodes Involved:**  
  - Initiate a Webhook Notification for Markdown to Textual Data Extraction (HTTP Request)  
  - Create a binary data for textual data (Function Node)  
  - Write the textual file to disk (Read/Write File Node)  

- **Node Details:**  

  - **Initiate a Webhook Notification for Markdown to Textual Data Extraction**  
    - Type: HTTP Request  
    - Role: Sends textual data extracted from markdown to configured webhook.  
    - Configuration: Sends body parameter `summary` with textual data string.  
    - Edge Cases: Network or endpoint issues.

  - **Create a binary data for textual data**  
    - Type: Function Node  
    - Role: Converts textual JSON data into base64 binary for file save.  
    - Edge Cases: None significant.

  - **Write the textual file to disk**  
    - Type: Read/Write File Node  
    - Role: Writes textual JSON to `d:\Brand-Content-Textual.json`.  
    - Edge Cases: Disk write errors.

---

#### 2.7 Utility & Documentation

- **Nodes Involved:**  
  - Sticky Note1 (Describes LLM usage)  
  - Sticky Note (described in 2.1)  
  - Sticky Note2, Sticky Note3, Sticky Note4 (document blocks)  

- These nodes provide essential guidance and architectural notes for users setting up and customizing the workflow.

---

### 3. Summary Table

| Node Name                                          | Node Type                          | Functional Role                        | Input Node(s)                                      | Output Node(s)                                           | Sticky Note                                                                                                                     |
|----------------------------------------------------|----------------------------------|-------------------------------------|---------------------------------------------------|---------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’                      | Manual Trigger                   | Entry point manual start             | —                                                 | Set URL and Bright Data Zone                            | This workflow deals with the brand content extraction by utilizing the Bright Data Web Unlocker Product. ... (see note)         |
| Set URL and Bright Data Zone                       | Set                              | Defines URL and Bright Data zone    | When clicking ‘Test workflow’                      | Perform Bright Data Web Request                          | Same as above                                                                                                                   |
| Perform Bright Data Web Request                     | HTTP Request                    | Web Unlocker API call to extract content | Set URL and Bright Data Zone                       | Markdown to Textual Data Extractor, Summarize Content   | Same as above                                                                                                                   |
| Google Gemini Chat Model for Data Extract          | Langchain LLM Chat (Google Gemini) | AI model interface for data extraction | Perform Bright Data Web Request                   | Markdown to Textual Data Extractor                       | LLM Usages: Google Gemini Flash Exp model is being used. Basic LLM Chain Data Extractor... (see note)                            |
| Markdown to Textual Data Extractor                  | Langchain LLM Chain             | Converts markdown to clean text     | Google Gemini Chat Model for Data Extract          | AI Sentiment Analyzer with the structured response, Initiate a Webhook Notification for Markdown to Textual Data Extraction, Create a binary data for textual data | Sticky Note4: ## Textual Data Extract                                                                                         |
| AI Sentiment Analyzer with the structured response | Langchain Information Extractor  | Performs structured sentiment analysis | Markdown to Textual Data Extractor                  | Initiate a Webhook Notification for AI Sentiment Analyzer, Create a binary data for sentiment analysis | Sticky Note3: ## Sentiment Analysis                                                                                             |
| Google Gemini Chat Model for Sentiment Analyzer    | Langchain LLM Chat (Google Gemini) | AI model interface for sentiment analysis | AI Sentiment Analyzer with the structured response | AI Sentiment Analyzer with the structured response       | Sticky Note3                                                                                                                   |
| Initiate a Webhook Notification for AI Sentiment Analyzer | HTTP Request                    | Sends sentiment analysis results via webhook | AI Sentiment Analyzer with the structured response | —                                                       | Sticky Note3                                                                                                                   |
| Create a binary data for sentiment analysis        | Function                        | Converts sentiment JSON to binary   | AI Sentiment Analyzer with the structured response | Write the AI Sentiment analysis file to disk             | Sticky Note3                                                                                                                   |
| Write the AI Sentiment analysis file to disk       | Read/Write File                 | Saves sentiment JSON to disk        | Create a binary data for sentiment analysis        | —                                                       | Sticky Note3                                                                                                                   |
| Summarize Content                                  | Langchain Summarization Chain   | Creates summary of extracted content | Perform Bright Data Web Request                      | Initiate a Webhook Notification for Summarization, Create a binary data for summary | Sticky Note2: ## Summarization                                                                                                 |
| Google Gemini Chat Model for Summary                | Langchain LLM Chat (Google Gemini) | AI model interface for summarization | Summarize Content                                   | Summarize Content                                        | Sticky Note2                                                                                                                   |
| Initiate a Webhook Notification for Summarization  | HTTP Request                    | Sends summary via webhook           | Summarize Content                                   | —                                                       | Sticky Note2                                                                                                                   |
| Create a binary data for summary                    | Function                        | Converts summary JSON to binary     | Summarize Content                                   | Write the summary file to disk                            | Sticky Note2                                                                                                                   |
| Write the summary file to disk                       | Read/Write File                 | Saves summary JSON to disk          | Create a binary data for summary                     | —                                                       | Sticky Note2                                                                                                                   |
| Initiate a Webhook Notification for Markdown to Textual Data Extraction | HTTP Request                    | Sends textual data via webhook      | Markdown to Textual Data Extractor                   | —                                                       | Sticky Note4                                                                                                                   |
| Create a binary data for textual data               | Function                        | Converts textual JSON to binary     | Markdown to Textual Data Extractor                   | Write the textual file to disk                            | Sticky Note4                                                                                                                   |
| Write the textual file to disk                       | Read/Write File                 | Saves textual JSON to disk          | Create a binary data for textual data                | —                                                       | Sticky Note4                                                                                                                   |
| Sticky Note                                         | Sticky Note                    | Instructional note                  | —                                                 | —                                                       | See note under Input Initialization                                                                                            |
| Sticky Note1                                        | Sticky Note                    | LLM usage explanations             | —                                                 | —                                                       | Explains usage of Google Gemini and Langchain chains                                                                          |
| Sticky Note2                                        | Sticky Note                    | Summarization block label          | —                                                 | —                                                       | ## Summarization                                                                                                               |
| Sticky Note3                                        | Sticky Note                    | Sentiment analysis block label    | —                                                 | —                                                       | ## Sentiment Analysis                                                                                                         |
| Sticky Note4                                        | Sticky Note                    | Textual data extraction block label | —                                                 | —                                                       | ## Textual Data Extract                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: “When clicking ‘Test workflow’”  
   - No configuration needed.  

2. **Create Set Node to Define URL and Zone**  
   - Type: Set  
   - Name: “Set URL and Bright Data Zone”  
   - Add two string fields:  
     - `url` = `https://www.amazon.com/TP-Link-Dual-Band-Archer-BE230-HomeShield/dp/B0DC99N2T8` (replace with your target URL)  
     - `zone` = `web_unlocker1` (replace with your Bright Data zone)  
   - Connect Manual Trigger output to this node.  

3. **Create HTTP Request Node to Call Bright Data API**  
   - Type: HTTP Request  
   - Name: “Perform Bright Data Web Request”  
   - HTTP Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Authentication: Generic Header Auth using Bright Data Bearer token credential  
   - Body Parameters (form-data or JSON):  
     - `zone` = `{{$json.zone}}`  
     - `url` = `{{$json.url}}?product=unlocker&method=api`  
     - `format` = `raw`  
     - `data_format` = `markdown`  
   - Connect “Set URL and Bright Data Zone” output to this node.  

4. **Create Google Gemini LLM Chat Node for Data Extraction**  
   - Type: Langchain LLM Chat Google Gemini  
   - Name: “Google Gemini Chat Model for Data Extract”  
   - Model: `models/gemini-2.0-flash-exp`  
   - Credentials: Google Palm API key configured  
   - Connect “Perform Bright Data Web Request” output to this node.  

5. **Create Langchain Chain LLM Node to Extract Textual Data**  
   - Type: Langchain Chain LLM  
   - Name: “Markdown to Textual Data Extractor”  
   - Prompt Type: Define  
   - Prompt Text:  
     ```
     You need to analyze the below markdown and convert to textual data. Please do not output with your own thoughts. Make sure to output with textual data only with no links, scripts, css etc.

     {{ $json.data }}
     ```  
   - Messages: add system message “You are a markdown expert”  
   - Connect “Google Gemini Chat Model for Data Extract” output to this node.  

6. **Create HTTP Request Node to Send Webhook Notification of Textual Data**  
   - Type: HTTP Request  
   - Name: “Initiate a Webhook Notification for Markdown to Textual Data Extraction”  
   - Method: POST  
   - URL: your webhook endpoint (default example: https://webhook.site/...)  
   - Send body parameters: `summary` = `{{$json.text}}` (text output from previous node)  
   - Connect “Markdown to Textual Data Extractor” output to this node.  

7. **Create Function Node to Convert Textual Data JSON to Binary**  
   - Type: Function  
   - Name: “Create a binary data for textual data”  
   - Code:  
     ```javascript
     items[0].binary = {
       data: {
         data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64'),
       },
     };
     return items;
     ```  
   - Connect “Markdown to Textual Data Extractor” output to this node.  

8. **Create Read/Write File Node to Save Textual Data**  
   - Type: Read/Write File  
   - Name: “Write the textual file to disk”  
   - Operation: Write  
   - File Name: `d:\Brand-Content-Textual.json` (adjust path as needed)  
   - Connect “Create a binary data for textual data” output to this node.  

9. **Create Google Gemini LLM Chat Node for Summarization**  
   - Type: Langchain LLM Chat Google Gemini  
   - Name: “Google Gemini Chat Model for Summary”  
   - Model: `models/gemini-2.0-flash-exp`  
   - Credentials: Google Palm API  
   - Connect “Perform Bright Data Web Request” output to this node.  

10. **Create Langchain Summarization Chain Node**  
    - Type: Langchain Chain Summarization  
    - Name: “Summarize Content”  
    - Prompt: “Write a concise summary of the following: {text}”  
    - Chunking Mode: Advanced  
    - Connect “Google Gemini Chat Model for Summary” output to this node.  

11. **Create HTTP Request Node for Summary Webhook Notification**  
    - Type: HTTP Request  
    - Name: “Initiate a Webhook Notification for Summarization”  
    - Method: POST  
    - URL: your webhook endpoint  
    - Send body parameter: `summary` = `{{$json.response.text}}`  
    - Connect “Summarize Content” output to this node.  

12. **Create Function Node to Convert Summary JSON to Binary**  
    - Type: Function  
    - Name: “Create a binary data for summary”  
    - Code: use same code as textual data binary creation  
    - Connect “Summarize Content” output to this node.  

13. **Create Read/Write File Node to Save Summary**  
    - Type: Read/Write File  
    - Name: “Write the summary file to disk”  
    - File Name: `d:\Brand-Content-Summary.json`  
    - Connect “Create a binary data for summary” output to this node.  

14. **Create Google Gemini LLM Chat Node for Sentiment Analysis**  
    - Type: Langchain LLM Chat Google Gemini  
    - Name: “Google Gemini Chat Model for Sentiment Analyzer”  
    - Model: `models/gemini-2.0-flash-exp`  
    - Credentials: Google Palm API  
    - Connect “Markdown to Textual Data Extractor” output to this node.  

15. **Create Langchain Information Extractor Node for Sentiment Analysis**  
    - Type: Langchain Information Extractor  
    - Name: “AI Sentiment Analyzer with the structured response”  
    - Text Input: `Perform Bright Data Web Request` node's `.data` field (or markdown extractor output as per workflow)  
    - System Prompt: “You are an expert sentiment analyzer.”  
    - Schema: JSON schema defining array of sentiment objects with sentiment enum, confidence_score (0-1), and sentence explanation  
    - Connect “Google Gemini Chat Model for Sentiment Analyzer” output to this node.  

16. **Create HTTP Request Node for Sentiment Webhook Notification**  
    - Type: HTTP Request  
    - Name: “Initiate a Webhook Notification for AI Sentiment Analyzer”  
    - Method: POST  
    - URL: your webhook endpoint  
    - Body parameter: `summary` = `{{$json.output}}` (sentiment analysis output)  
    - Connect “AI Sentiment Analyzer with the structured response” output to this node.  

17. **Create Function Node to Convert Sentiment JSON to Binary**  
    - Type: Function  
    - Name: “Create a binary data for sentiment analysis”  
    - Code: same as other binary conversion nodes  
    - Connect “AI Sentiment Analyzer with the structured response” output to this node.  

18. **Create Read/Write File Node to Save Sentiment Analysis**  
    - Type: Read/Write File  
    - Name: “Write the AI Sentiment analysis file to disk”  
    - File Name: `d:\Brand-Content-Sentiment-Analysis.json`  
    - Connect “Create a binary data for sentiment analysis” output to this node.  

19. **Create Sticky Notes for Documentation**  
    - Add four Sticky Note nodes at relevant positions with the provided content for user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                               | Context or Link                                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This workflow deals with brand content extraction by utilizing Bright Data Web Unlocker Product. Setup requires Bright Data Web Unlocker API and Google Gemini API key. | Setup instructions and credential configuration.                                                                       |
| LLM Usages: Google Gemini Flash Exp model is used for data extraction, summarization, and sentiment analysis with structured responses.                                  | Clarifies AI model usage and Langchain chain types.                                                                     |
| Customize source input to Google Sheets or Airbase to track multiple brands dynamically. AI prompt customization options include summary length and detailed sentiment.   | Extension ideas for workflow scalability and customization.                                                             |
| Output destinations can be adapted to Slack, CRM, or databases instead of webhooks for integration flexibility.                                                           | Integration flexibility note.                                                                                           |
| Official Bright Data Web Unlocker setup guide: https://brightdata.com/                                                                                                    | Official documentation link.                                                                                            |
| Google Gemini (PaLM) API documentation: https://cloud.google.com/vertex-ai/docs/generative-ai/overview                                                                     | Google Gemini API reference.                                                                                            |

---

This documentation provides a complete and precise understanding of the workflow, enabling reproduction, modification, and troubleshooting for users and automation agents alike.
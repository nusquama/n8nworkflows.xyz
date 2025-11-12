Extract & Summarize Bing Copilot Search Results with Gemini AI and Bright Data

https://n8nworkflows.xyz/workflows/extract---summarize-bing-copilot-search-results-with-gemini-ai-and-bright-data-3536


# Extract & Summarize Bing Copilot Search Results with Gemini AI and Bright Data

### 1. Workflow Overview

This workflow automates querying Bing’s Copilot Search via Bright Data’s API, extracting structured data from the search results using AI, summarizing the extracted content, and sending notifications with the results via webhooks. It targets data analysts, developers, and digital marketers who need efficient, automated access to Bing search insights.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Search Trigger**: Manual trigger initiates the workflow and sends a Bing Copilot search request through Bright Data’s API.
- **1.2 Snapshot Management and Polling**: Captures the snapshot ID from the search trigger response, polls Bright Data API until the snapshot is ready, and downloads the snapshot data.
- **1.3 Structured Data Extraction**: Uses Google Gemini AI models and LangChain nodes to parse the raw snapshot data into structured JSON format.
- **1.4 Summarization**: Summarizes the extracted structured data into concise summaries using Google Gemini AI and LangChain summarization chains.
- **1.5 Notification Dispatch**: Sends the structured data and summary results to configured webhook endpoints.
- **1.6 Error Handling and Wait Logic**: Implements conditional checks for errors and waits between polling cycles.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Search Trigger

**Overview:**  
This block starts the workflow manually and sends a Bing Copilot search request to Bright Data’s API, initiating the data retrieval process.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Perform a Bing Copilot Request (HTTP Request)  
- Set Snapshot Id (Set)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual workflow execution  
  - Configuration: Default manual trigger, no parameters  
  - Inputs: None  
  - Outputs: Triggers next node  
  - Edge cases: None, manual trigger only

- **Perform a Bing Copilot Request**  
  - Type: HTTP Request  
  - Role: Sends POST request to Bright Data API to trigger Bing Copilot search  
  - Configuration:  
    - URL: `https://api.brightdata.com/datasets/v3/trigger`  
    - Method: POST  
    - Body: JSON array with search URL and prompt (default: "Top hotels in New York")  
    - Query Parameters: `dataset_id=gd_m7di5jy6s9geokz8w`, `include_errors=true`  
    - Authentication: Header Auth (configured in credentials)  
  - Inputs: Manual Trigger  
  - Outputs: JSON response containing snapshot ID  
  - Edge cases: API auth failure, invalid prompt, network timeout

- **Set Snapshot Id**  
  - Type: Set  
  - Role: Extracts and stores `snapshot_id` from the previous HTTP response for downstream use  
  - Configuration: Assigns `snapshot_id` from `$json.snapshot_id`  
  - Inputs: HTTP Request output  
  - Outputs: Snapshot ID for polling  
  - Edge cases: Missing or malformed snapshot ID in response

---

#### 2.2 Snapshot Management and Polling

**Overview:**  
Polls Bright Data API to check the snapshot status until it is ready, then downloads the snapshot data.

**Nodes Involved:**  
- Check Snapshot Status (HTTP Request)  
- If (Conditional)  
- Check on the errors (If)  
- Wait for 30 seconds (Wait)  
- Download Snapshot (HTTP Request)

**Node Details:**

- **Check Snapshot Status**  
  - Type: HTTP Request  
  - Role: Queries Bright Data API for snapshot progress status  
  - Configuration:  
    - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
    - Authentication: Header Auth  
  - Inputs: Set Snapshot Id output or Wait node output  
  - Outputs: JSON with snapshot status  
  - Edge cases: API errors, auth failure, snapshot ID invalid

- **If**  
  - Type: If (Conditional)  
  - Role: Checks if snapshot status equals `"ready"`  
  - Configuration: Compares `{{$json.status}}` to `"ready"` string  
  - Inputs: Check Snapshot Status output  
  - Outputs:  
    - True branch: proceeds to download snapshot  
    - False branch: triggers wait and re-poll  
  - Edge cases: Unexpected status values, missing status field

- **Wait for 30 seconds**  
  - Type: Wait  
  - Role: Pauses workflow for 30 seconds before re-polling snapshot status  
  - Configuration: Wait time set to 30 seconds  
  - Inputs: If node false branch  
  - Outputs: Triggers Check Snapshot Status again  
  - Edge cases: None significant, but long wait times may delay workflow

- **Download Snapshot**  
  - Type: HTTP Request  
  - Role: Downloads snapshot data in JSON format once ready  
  - Configuration:  
    - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
    - Query Parameter: `format=json`  
    - Authentication: Header Auth  
  - Inputs: If node true branch  
  - Outputs: Raw snapshot data JSON  
  - Edge cases: Download failure, auth errors, invalid snapshot ID

- **Check on the errors**  
  - Type: If (Conditional)  
  - Role: Checks if the snapshot data contains errors (`errors` field equals 0)  
  - Configuration: Compares `$json.errors.toString()` to `"0"`  
  - Inputs: Download Snapshot output  
  - Outputs:  
    - True branch: proceeds to data extraction  
    - False branch: could be extended for error handling (not detailed)  
  - Edge cases: Missing errors field, non-zero errors

---

#### 2.3 Structured Data Extraction

**Overview:**  
Processes the raw snapshot data through AI models to extract structured JSON data representing the search results.

**Nodes Involved:**  
- Structured Output Parser (LangChain Output Parser)  
- Google Gemini Chat Model1 (Google Gemini AI)  
- Structured Data Extractor (LangChain LLM Chain)  
- Default Data Loader (LangChain Document Loader)  
- Recursive Character Text Splitter (LangChain Text Splitter)

**Node Details:**

- **Structured Output Parser**  
  - Type: LangChain Output Parser (Structured)  
  - Role: Defines expected JSON schema for extracted data  
  - Configuration: JSON schema example specifying city and hotels with name, address, description, website, optional area  
  - Inputs: Google Gemini Chat Model1 output  
  - Outputs: Parsed structured JSON for downstream use  
  - Edge cases: Parsing failures if AI output deviates from schema

- **Google Gemini Chat Model1**  
  - Type: LangChain AI Language Model (Google Gemini)  
  - Role: Processes raw snapshot text to generate structured data output  
  - Configuration: Model name `models/gemini-2.0-flash-exp`  
  - Credentials: Google Gemini API key  
  - Inputs: Structured Output Parser input chain  
  - Outputs: AI-generated structured JSON text  
  - Edge cases: API errors, rate limits, unexpected AI output

- **Structured Data Extractor**  
  - Type: LangChain Chain LLM  
  - Role: Extracts structured JSON from snapshot answer text using AI and output parser  
  - Configuration:  
    - Prompt: "Extract the content as a structured JSON. Here's the content - {{ $json.answer_text }}"  
    - Message: "You are an expert data formatter"  
    - Uses output parser (Structured Output Parser)  
  - Inputs: Download Snapshot output (raw answer_text)  
  - Outputs: Structured JSON data  
  - Edge cases: AI parsing errors, missing answer_text field

- **Default Data Loader**  
  - Type: LangChain Document Default Data Loader  
  - Role: Loads structured data documents for summarization  
  - Configuration: Default options  
  - Inputs: Recursive Character Text Splitter output  
  - Outputs: Documents for summarization chain  
  - Edge cases: Document loading errors

- **Recursive Character Text Splitter**  
  - Type: LangChain Recursive Character Text Splitter  
  - Role: Splits large text into chunks with 100 character overlap for processing  
  - Configuration: Chunk overlap set to 100 characters  
  - Inputs: Structured Data Extractor output (structured JSON text)  
  - Outputs: Text chunks for loading  
  - Edge cases: Improper splitting if input text is malformed

---

#### 2.4 Summarization

**Overview:**  
Generates concise summaries of the extracted structured data using Google Gemini AI and LangChain summarization chains.

**Nodes Involved:**  
- Google Gemini Chat Model (Google Gemini AI)  
- Concise Summary Creator (LangChain Summarization Chain)

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: LangChain AI Language Model (Google Gemini)  
  - Role: Supports summarization chain as the language model  
  - Configuration: Model name `models/gemini-2.0-flash-thinking-exp-01-21` (experimental)  
  - Credentials: Google Gemini API key  
  - Inputs: Summarization chain input  
  - Outputs: Summarized text  
  - Edge cases: API errors, rate limits

- **Concise Summary Creator**  
  - Type: LangChain Chain Summarization  
  - Role: Creates concise summaries from loaded documents  
  - Configuration:  
    - Summarization prompt: "Write a concise summary of the following: {{ $('Download Snapshot').item.json.answer_text }}"  
    - Operation mode: documentLoader  
  - Inputs: Default Data Loader documents and Google Gemini Chat Model output  
  - Outputs: Concise summary text  
  - Edge cases: Summarization quality depends on input data and AI model

---

#### 2.5 Notification Dispatch

**Overview:**  
Sends the extracted structured data and the summary to configured webhook endpoints for further processing or notification.

**Nodes Involved:**  
- Structured Data Webhook Notifier (HTTP Request)  
- Summary Webhook Notifier (HTTP Request)

**Node Details:**

- **Structured Data Webhook Notifier**  
  - Type: HTTP Request  
  - Role: Sends structured JSON data to a webhook URL  
  - Configuration:  
    - URL: Configured webhook endpoint (default: https://webhook.site/bc804ce5-4a45-4177-a68a-99c80e5c86e6)  
    - Method: POST  
    - Body: JSON with `response` field set to structured data output (`{{$json.output}}`)  
  - Inputs: Structured Data Extractor output  
  - Outputs: None (end node)  
  - Edge cases: Webhook endpoint unreachable, network errors

- **Summary Webhook Notifier**  
  - Type: HTTP Request  
  - Role: Sends summary text to a webhook URL  
  - Configuration:  
    - URL: Configured webhook endpoint (default: https://webhook.site/bc804ce5-4a45-4177-a68a-99c80e5c86e6)  
    - Method: POST  
    - Body: JSON with `response` field set to summary output (`{{$json.output}}`)  
  - Inputs: Concise Summary Creator output  
  - Outputs: None (end node)  
  - Edge cases: Webhook endpoint unreachable, network errors

---

#### 2.6 Error Handling and Wait Logic

**Overview:**  
Implements conditional checks for errors and manages wait cycles during snapshot polling.

**Nodes Involved:**  
- Check on the errors (If)  
- Wait for 30 seconds (Wait)

**Node Details:**

- **Check on the errors**  
  - Type: If (Conditional)  
  - Role: Verifies if the snapshot data contains errors (errors field equals 0)  
  - Configuration: Compares `$json.errors.toString()` to `"0"`  
  - Inputs: Download Snapshot output  
  - Outputs:  
    - True branch: proceeds to structured data extraction  
    - False branch: could be extended to error handling (not detailed)  
  - Edge cases: Missing errors field, non-zero errors

- **Wait for 30 seconds**  
  - See details in Block 2.2

---

### 3. Summary Table

| Node Name                     | Node Type                          | Functional Role                         | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                           |
|-------------------------------|----------------------------------|---------------------------------------|---------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger                   | Workflow entry point                   | None                            | Perform a Bing Copilot Request     | ## Note\n\nDeals with the Bing Copilot Search using the Bright Data Web Scraper API.\n\nThe Basic LLM Chain and summarization is done to demonstrate the usage of the N8N AI capabilities.\n\n**Please make sure to update the Webhook Notification URL** |
| Perform a Bing Copilot Request | HTTP Request                    | Triggers Bing Copilot search           | When clicking ‘Test workflow’   | Set Snapshot Id                   |                                                                                                     |
| Set Snapshot Id                | Set                             | Stores snapshot ID from search trigger | Perform a Bing Copilot Request  | Check Snapshot Status             |                                                                                                     |
| Check Snapshot Status          | HTTP Request                    | Polls snapshot status                  | Set Snapshot Id, Wait for 30 sec| If                              |                                                                                                     |
| If                            | If                              | Checks if snapshot status is ready     | Check Snapshot Status           | Download Snapshot, Wait for 30 sec|                                                                                                     |
| Wait for 30 seconds            | Wait                            | Waits before re-polling snapshot status| If                            | Check Snapshot Status             |                                                                                                     |
| Download Snapshot             | HTTP Request                    | Downloads snapshot data                 | If                            | Check on the errors               |                                                                                                     |
| Check on the errors            | If                              | Checks for errors in snapshot data     | Download Snapshot              | Structured Data Extractor, Wait for 30 sec|                                                                                                     |
| Structured Output Parser       | LangChain Output Parser         | Defines JSON schema for extraction     | Google Gemini Chat Model1       | Structured Data Extractor         |                                                                                                     |
| Google Gemini Chat Model1      | LangChain AI Language Model     | AI model for structured data extraction| Structured Output Parser        | Structured Data Extractor         | ## LLM Usages\n\nGoogle Gemini Flash Exp model is being used.\n\nBasic LLM Chain makes use of the Output formatter for formatting the response\n\nSummarization Chain is being used for summarization of the content |
| Structured Data Extractor      | LangChain Chain LLM             | Extracts structured JSON from snapshot | Download Snapshot              | Concise Summary Creator, Structured Data Webhook Notifier|                                                                                                     |
| Recursive Character Text Splitter| LangChain Text Splitter       | Splits text for document loading       | Structured Data Extractor       | Default Data Loader               |                                                                                                     |
| Default Data Loader            | LangChain Document Loader       | Loads documents for summarization      | Recursive Character Text Splitter| Concise Summary Creator          |                                                                                                     |
| Google Gemini Chat Model       | LangChain AI Language Model     | AI model for summarization             | Concise Summary Creator         | Concise Summary Creator           | See note on Google Gemini Chat Model1                                                               |
| Concise Summary Creator        | LangChain Chain Summarization   | Creates concise summary                 | Default Data Loader, Google Gemini Chat Model| Summary Webhook Notifier          |                                                                                                     |
| Structured Data Webhook Notifier| HTTP Request                  | Sends structured data to webhook       | Structured Data Extractor       | None                            | **Please make sure to update the Webhook Notification URL**                                         |
| Summary Webhook Notifier       | HTTP Request                    | Sends summary to webhook                | Concise Summary Creator         | None                            | **Please make sure to update the Webhook Notification URL**                                         |
| Sticky Note                   | Sticky Note                    | Notes on workflow usage                 | None                          | None                            | See note content above                                                                              |
| Sticky Note1                  | Sticky Note                    | Notes on LLM usage                      | None                          | None                            | See note content above                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually

2. **Create HTTP Request Node: Perform a Bing Copilot Request**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Body (JSON):  
     ```json
     [
       {
         "url": "https://copilot.microsoft.com/chats",
         "prompt": "Top hotels in New York"
       }
     ]
     ```  
   - Query Parameters:  
     - `dataset_id`: `gd_m7di5jy6s9geokz8w`  
     - `include_errors`: `true`  
   - Authentication: Header Auth (configure with Bright Data API key)  
   - Connect Manual Trigger → Perform Bing Request

3. **Create Set Node: Set Snapshot Id**  
   - Type: Set  
   - Assign variable `snapshot_id` with expression: `{{$json.snapshot_id}}`  
   - Connect Perform Bing Request → Set Snapshot Id

4. **Create HTTP Request Node: Check Snapshot Status**  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{$json.snapshot_id}}`  
   - Authentication: Header Auth (same as above)  
   - Connect Set Snapshot Id → Check Snapshot Status

5. **Create If Node: If snapshot is ready**  
   - Type: If  
   - Condition: `{{$json.status}}` equals `"ready"` (case sensitive)  
   - Connect Check Snapshot Status → If

6. **Create Wait Node: Wait for 30 seconds**  
   - Type: Wait  
   - Duration: 30 seconds  
   - Connect If (false branch) → Wait  
   - Connect Wait → Check Snapshot Status (loop back)

7. **Create HTTP Request Node: Download Snapshot**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{$json.snapshot_id}}`  
   - Query Parameter: `format=json`  
   - Authentication: Header Auth  
   - Connect If (true branch) → Download Snapshot

8. **Create If Node: Check on the errors**  
   - Type: If  
   - Condition: `{{$json.errors.toString()}}` equals `"0"`  
   - Connect Download Snapshot → Check on the errors

9. **Create LangChain Output Parser Node: Structured Output Parser**  
   - Type: LangChain Output Parser (Structured)  
   - JSON Schema Example:  
     ```json
     [{
       "city": "string",
       "hotels": [
         {
           "name": "string",
           "address": "string",
           "description": "string",
           "website": "string",
           "area": "string (optional)"
         }
       ]
     }]
     ```
   - Connect Google Gemini Chat Model1 → Structured Output Parser

10. **Create LangChain AI Language Model Node: Google Gemini Chat Model1**  
    - Type: LangChain AI Language Model (Google Gemini)  
    - Model Name: `models/gemini-2.0-flash-exp`  
    - Credentials: Google Gemini API key  
    - Connect Structured Output Parser → Structured Data Extractor

11. **Create LangChain Chain LLM Node: Structured Data Extractor**  
    - Type: LangChain Chain LLM  
    - Prompt:  
      ```
      Extract the content as a structured JSON.

      Here's the content - {{$json.answer_text}}
      ```  
    - Message: "You are an expert data formatter"  
    - Enable Output Parser, select Structured Output Parser  
    - Connect Download Snapshot → Structured Data Extractor

12. **Create LangChain Text Splitter Node: Recursive Character Text Splitter**  
    - Type: Recursive Character Text Splitter  
    - Chunk Overlap: 100 characters  
    - Connect Structured Data Extractor → Recursive Character Text Splitter

13. **Create LangChain Document Loader Node: Default Data Loader**  
    - Type: Document Default Data Loader  
    - Default options  
    - Connect Recursive Character Text Splitter → Default Data Loader

14. **Create LangChain AI Language Model Node: Google Gemini Chat Model**  
    - Type: LangChain AI Language Model (Google Gemini)  
    - Model Name: `models/gemini-2.0-flash-thinking-exp-01-21`  
    - Credentials: Google Gemini API key  
    - Connect Default Data Loader → Concise Summary Creator (ai_languageModel input)

15. **Create LangChain Chain Summarization Node: Concise Summary Creator**  
    - Type: Chain Summarization  
    - Summarization Prompt:  
      ```
      Write a concise summary of the following:

      {{$json.answer_text}}
      ```  
    - Operation Mode: documentLoader  
    - Connect Default Data Loader → Concise Summary Creator (ai_document input)  
    - Connect Google Gemini Chat Model → Concise Summary Creator (ai_languageModel input)

16. **Create HTTP Request Node: Structured Data Webhook Notifier**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Your webhook endpoint (update accordingly)  
    - Body Parameter: `response` = `{{$json.output}}` (structured data)  
    - Connect Structured Data Extractor → Structured Data Webhook Notifier

17. **Create HTTP Request Node: Summary Webhook Notifier**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Your webhook endpoint (update accordingly)  
    - Body Parameter: `response` = `{{$json.output}}` (summary)  
    - Connect Concise Summary Creator → Summary Webhook Notifier

18. **Connect Check on the errors**  
    - True branch → Structured Data Extractor  
    - False branch → (Optional error handling or end)

19. **Connect Wait for 30 seconds**  
    - Connect If (false branch) → Wait  
    - Connect Wait → Check Snapshot Status (loop)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Deals with the Bing Copilot Search using the Bright Data Web Scraper API. The Basic LLM Chain and summarization demonstrate n8n AI capabilities. **Please make sure to update the Webhook Notification URL**                                          | Sticky Note on workflow start node                                                               |
| Google Gemini Flash Exp model is used. Basic LLM Chain uses Output Formatter for response formatting. Summarization Chain is used for content summarization.                                                                                         | Sticky Note near AI nodes                                                                        |
| Setup requires Bright Data account and Web Unlocker API zone creation. Configure Header Auth credentials in n8n for Bright Data API. Google Gemini API key required for AI nodes. Update prompt and webhook URLs as needed.                            | Workflow description and setup instructions                                                     |
| Bright Data API documentation: https://brightdata.com/docs/api                                                                                                                                                                                      | External resource for API details                                                               |
| Google Gemini API (PaLM) documentation: https://developers.generativeai.google/tutorials                                                                                                                                                            | External resource for AI model integration                                                      |

---

This document provides a detailed and structured reference for understanding, reproducing, and modifying the "Extract & Summarize Bing Copilot Search Results with Gemini AI and Bright Data" workflow. It covers all nodes, their configurations, and logical connections, enabling both human developers and AI agents to work effectively with the workflow.
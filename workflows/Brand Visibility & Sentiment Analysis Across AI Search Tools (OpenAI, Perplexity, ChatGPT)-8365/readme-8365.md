Brand Visibility & Sentiment Analysis Across AI Search Tools (OpenAI, Perplexity, ChatGPT)

https://n8nworkflows.xyz/workflows/brand-visibility---sentiment-analysis-across-ai-search-tools--openai--perplexity--chatgpt--8365


# Brand Visibility & Sentiment Analysis Across AI Search Tools (OpenAI, Perplexity, ChatGPT)

---

## 1. Workflow Overview

This workflow is designed to analyze brand visibility and sentiment across multiple AI search tools, specifically OpenAI's GPT models, Perplexity AI, and ChatGPT via APIfy scraping. Its purpose is to help brands understand how they are perceived and mentioned in AI-generated answers to specific prompts or queries, enabling optimization of brand visibility in emerging AI search channels.

The workflow is organized into these logical blocks:

- **1.1 Input Reception:** Collects prompts either manually or via Google Sheets and prepares them for processing.
- **1.2 Multi-Model AI Querying:** Sends the input prompts concurrently to three AI sources: OpenAI GPT models, Perplexity API, and a ChatGPT scraping actor via APIfy.
- **1.3 Sentiment and Brand Analysis:** Uses Langchain agents to analyze the AI responses for sentiment polarity, emotion category, and brand hierarchy.
- **1.4 Result Normalization and Mapping:** Extracts and normalizes relevant data fields from each AI source’s raw output to a consistent format.
- **1.5 Data Persistence:** Appends processed and structured results into designated Google Sheets for further reporting or analysis.
- **1.6 Control Flow and Batching:** Manages looping over multiple prompts with batch limits and conditional checks to ensure smooth execution.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block gathers input prompts either from a manual trigger or a predefined Google Sheets document, limits the number of prompts processed for testing or operational reasons, and initiates batch processing for scalable execution.

**Nodes Involved:**  
- Manual Trigger  
- Read Prompts1 (Google Sheets read)  
- Limit for testing  
- before-loop-input (NoOp)  
- Loop Over prompts (SplitInBatches)  
- loop-input (NoOp)  
- manual input (Set)

**Node Details:**

- **Manual Trigger**  
  - *Type:* Manual start point node.  
  - *Config:* No parameters; triggers workflow manually.  
  - *Connections:* Outputs to Read Prompts1.  
  - *Failure Modes:* None typical, manual start.  

- **Read Prompts1**  
  - *Type:* Google Sheets Read.  
  - *Config:* Reads range A1:A100 from a specified Google Sheet containing prompt strings.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Input:* Manual Trigger.  
  - *Output:* List of prompts to the Limit for testing node.  
  - *Failure Modes:* Auth failure, sheet access issues, empty data.  

- **Limit for testing**  
  - *Type:* Limit node.  
  - *Config:* Limits data flow to maximum of 2 items for testing.  
  - *Input:* Read Prompts1.  
  - *Output:* before-loop-input.  
  - *Failure Modes:* None expected.  

- **before-loop-input**  
  - *Type:* NoOp (pass-through).  
  - *Role:* Prepares control flow for looping.  
  - *Input:* Limit for testing.  
  - *Output:* Loop Over prompts.  

- **Loop Over prompts**  
  - *Type:* SplitInBatches.  
  - *Config:* Splits prompt list to process sequentially or in batches (default batch size).  
  - *Input:* before-loop-input.  
  - *Output:* loop-input (first batch) or ends flow if no items.  
  - *Failure Modes:* Large data sets might exceed limits if not batched properly.  

- **loop-input**  
  - *Type:* NoOp.  
  - *Role:* Acts as starting point inside loop batch.  
  - *Input:* Loop Over prompts.  
  - *Output:* Sends each prompt to the downstream AI query nodes.  

- **manual input**  
  - *Type:* Set node.  
  - *Config:* Hardcoded prompt "Was sind die besten Laufschuhe?" as example input.  
  - *Input:* None connected in main flow (likely used for testing).  
  - *Output:* before-loop-input.  
  - *Failure Modes:* None typical.  

---

### 2.2 Multi-Model AI Querying

**Overview:**  
This block queries three AI sources in parallel with the prompt: OpenAI GPT models (GPT-4, GPT-5), Perplexity API, and ChatGPT scraping via APIfy actor. It collects raw responses and metadata including citations and sources.

**Nodes Involved:**  
- OpenAI Chat Model (GPT 5)  
- OpenAI Anfrage (chain LLM node)  
- OpenAI (Set node for response tagging)  
- Perplexity Request / Perplexity Request1  
- APIfy Call ChatGPT Scraper (HTTP Request)  
- If successful (conditional check)

**Node Details:**

- **OpenAI Chat Model (GPT 5)**  
  - *Type:* Langchain OpenAI chat model.  
  - *Config:* Model set to "gpt-5".  
  - *Credentials:* OpenAI API key configured.  
  - *Input:* loop-input (prompt).  
  - *Output:* OpenAI Anfrage.  
  - *Failure Modes:* API key limits, network errors, model unavailability.  

- **OpenAI Anfrage**  
  - *Type:* Chain LLM node.  
  - *Config:* Uses prompt text from input JSON.  
  - *Input:* OpenAI Chat Model (GPT 5).  
  - *Output:* OpenAI (Set node).  
  - *Failure Modes:* Expression errors if prompt missing.  

- **OpenAI (Set node)**  
  - *Type:* Set node.  
  - *Config:* Assigns "Response" field from LLM output text and labels LLM source as "OpenAI".  
  - *Input:* OpenAI Anfrage.  
  - *Output:* normalized-tool-response (NoOp).  

- **Perplexity Request / Perplexity Request1**  
  - *Type:* Perplexity API Call.  
  - *Config:* Model "sonar" used; sends prompt message as JSON body.  
  - *Credentials:* Perplexity API key.  
  - *Input:* loop-input (for Request), or LLM-Prompts set node (for Request1).  
  - *Output:* Perplexity Mapper (set node) or Map LLM Output.  
  - *Failure Modes:* Auth failure, API limits, malformed prompt.  

- **APIfy Call ChatGPT Scraper**  
  - *Type:* HTTP Request node.  
  - *Config:* Calls APIfy actor endpoint with prompt in JSON body, uses generic HTTP query auth with token.  
  - *Input:* loop-input.  
  - *Output:* If successful (conditional).  
  - *Failure Modes:* API rate limits, IP blocks, unauthorized usage.  
  - *Sticky Note Warning:* Use at own risk; possible violation of OpenAI terms, restrictions may apply.

- **If successful**  
  - *Type:* If node.  
  - *Config:* Checks if prompt field is not empty to continue processing.  
  - *Input:* APIfy Call ChatGPT Scraper.  
  - *Output:* ChatGPT Mapper or loop-end (NoOp).  

---

### 2.3 Sentiment and Brand Analysis

**Overview:**  
This block analyzes the textual AI responses to classify sentiment polarity, emotional category, and brand hierarchy. It uses Langchain agent nodes with custom prompts and output parsers to structure the information.

**Nodes Involved:**  
- Response Sentiment Analyse1 (Langchain Agent)  
- Response Sentiment Analyse3 (Langchain Agent)  
- Output Parser  
- Structured Output Parser3

**Node Details:**

- **Response Sentiment Analyse1**  
  - *Type:* Langchain agent node.  
  - *Config:* Custom prompt instructs to analyze Perplexity AI's JSON result message, outputting JSON with keys: Basic Polarity, Emotion Category, Brand Hierarchy.  
  - *Input:* Map LLM Output (set node with Perplexity results).  
  - *Output:* final mapping (Set node).  
  - *Failure Modes:* Parsing errors if input JSON malformed; output parsing failure if AI returns unexpected format.  

- **Response Sentiment Analyse3**  
  - *Type:* Langchain agent node.  
  - *Config:* Similar sentiment analysis prompt, but input is generic AI response message.  
  - *Input:* normalized-tool-response node output (from OpenAI or ChatGPT).  
  - *Output:* Prepare Sheet Columns (Set node).  
  - *Failure Modes:* Same as above, plus possible empty or incomplete responses.  

- **Output Parser**  
  - *Type:* Langchain structured output parser.  
  - *Config:* Defines JSON schema example for expected sentiment keys.  
  - *Input:* Chat Model (OpenAI GPT).  
  - *Output:* Response Sentiment Analyse1 (ai_outputParser input).  

- **Structured Output Parser3**  
  - *Type:* Langchain structured output parser.  
  - *Config:* JSON schema example for sentiment keys.  
  - *Input:* OpenAI Chat Model1 (GPT 4.1 mini).  
  - *Output:* Response Sentiment Analyse3 (ai_outputParser input).  

---

### 2.4 Result Normalization and Mapping

**Overview:**  
This block transforms raw AI responses and associated metadata into uniform fields for consistent storage and reporting.

**Nodes Involved:**  
- Map LLM Output  
- Perplexity Mapper  
- ChatGPT Mapper  
- final mapping  
- Prepare Sheet Columns  
- normalized-tool-response (NoOp)

**Node Details:**

- **Map LLM Output**  
  - *Type:* Set node.  
  - *Config:* Extracts citations (up to 5) and main message content from Perplexity API response JSON. Also copies prompt text.  
  - *Input:* Perplexity Request1.  
  - *Output:* Response Sentiment Analyse1.  

- **Perplexity Mapper**  
  - *Type:* Set node.  
  - *Config:* Maps Perplexity API results into normalized fields: Response, LLM name, Source1 - Source6 (citations).  
  - *Input:* Perplexity Request.  
  - *Output:* normalized-tool-response.  

- **ChatGPT Mapper**  
  - *Type:* Set node.  
  - *Config:* Maps APIfy ChatGPT output to normalized fields: Response, LLM name, multiple source URLs, and news listing.  
  - *Input:* If successful (post-APIfy HTTP Request).  
  - *Output:* normalized-tool-response.  

- **final mapping**  
  - *Type:* Set node.  
  - *Config:* Combines sentiment analysis outputs with mapped LLM results, assigns fields like Prompt, Message, Emotion Category, Basic Polarity, Brand Hierarchy, and up to five sources.  
  - *Input:* Response Sentiment Analyse1.  
  - *Output:* Sheet/Excel updaten (Google Sheets append).  

- **Prepare Sheet Columns**  
  - *Type:* Set node.  
  - *Config:* Prepares final data fields for Google Sheets append, including prompt, response, brand mentioned (boolean), brand hierarchy, sentiment info, and sources.  
  - *Input:* Response Sentiment Analyse3.  
  - *Output:* Append row in sheet (Google Sheets append).  

- **normalized-tool-response**  
  - *Type:* NoOp.  
  - *Role:* Passes normalized data from mappers to sentiment analysis and sheet preparation nodes.  

---

### 2.5 Data Persistence

**Overview:**  
This block appends the analyzed and normalized data into Google Sheets documents for storage and reporting.

**Nodes Involved:**  
- Sheet/Excel updaten  
- Append row in sheet

**Node Details:**

- **Sheet/Excel updaten**  
  - *Type:* Google Sheets Append.  
  - *Config:* Appends rows to a specific sheet named "Output" within a Google Sheets document. Maps columns automatically from incoming data.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Input:* final mapping.  
  - *Failure Modes:* Auth failure, quota limits, schema mismatches.  

- **Append row in sheet**  
  - *Type:* Google Sheets Append.  
  - *Config:* Appends rows to a different sheet named "Output many models" in the same Google Sheets document.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Input:* Prepare Sheet Columns.  
  - *Failure Modes:* Same as above.  

---

### 2.6 Control Flow and Batching

**Overview:**  
Manages execution flow between prompts including batch looping, conditional routing, and final workflow termination.

**Nodes Involved:**  
- loop-end (NoOp)  
- If successful (conditional check)

**Node Details:**

- **loop-end**  
  - *Type:* NoOp.  
  - *Role:* Marks end of batch processing; connects back to Loop Over prompts to continue with next batch.  

- **If successful**  
  - *Type:* If node.  
  - *Config:* Checks if prompt field is not empty to proceed with mapping and storage or end loop.  

---

## 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                 | Input Node(s)                        | Output Node(s)                     | Sticky Note                                                                                                          |
|-----------------------------|----------------------------------|--------------------------------|------------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Manual Trigger              | Manual Trigger                   | Start workflow manually         | -                                  | Read Prompts1                    |                                                                                                                      |
| Read Prompts1              | Google Sheets                   | Read input prompts              | Manual Trigger                    | Limit for testing                |                                                                                                                      |
| Limit for testing          | Limit                          | Limit number of prompts         | Read Prompts1                    | before-loop-input               |                                                                                                                      |
| before-loop-input          | NoOp                           | Control flow before loop        | Limit for testing                | Loop Over prompts               |                                                                                                                      |
| Loop Over prompts          | SplitInBatches                 | Batch processing of prompts     | before-loop-input               | loop-input                     |                                                                                                                      |
| loop-input                | NoOp                           | Current batch prompt input      | Loop Over prompts               | OpenAI Chat Model (GPT 5), Perplexity Request, APIfy Call ChatGPT Scraper |                                                                                                                      |
| manual input              | Set                            | Manual prompt input (example)   | -                                | before-loop-input               |                                                                                                                      |
| OpenAI Chat Model (GPT 5) | Langchain OpenAI Chat Model     | Query GPT-5 model               | loop-input                     | OpenAI Anfrage                 |                                                                                                                      |
| OpenAI Anfrage            | Chain LLM                      | Send prompt to OpenAI           | OpenAI Chat Model (GPT 5)       | OpenAI                       |                                                                                                                      |
| OpenAI                   | Set                            | Collect OpenAI response         | OpenAI Anfrage                 | normalized-tool-response       |                                                                                                                      |
| Perplexity Request        | Perplexity API                 | Query Perplexity API            | loop-input                     | Perplexity Mapper             |                                                                                                                      |
| Perplexity Request1       | Perplexity API                 | Secondary Perplexity query      | LLM-Prompts                   | Map LLM Output               |                                                                                                                      |
| APIfy Call ChatGPT Scraper| HTTP Request                  | Scrape ChatGPT via APIfy actor | loop-input                     | If successful                 | Use at own risk - may violate OpenAI terms; see APIfy docs for details.                                              |
| If successful             | If                            | Check valid prompt response     | APIfy Call ChatGPT Scraper      | ChatGPT Mapper, loop-end       |                                                                                                                      |
| Response Sentiment Analyse1 | Langchain Agent               | Sentiment & brand analysis (Perplexity) | Map LLM Output               | final mapping                 |                                                                                                                      |
| Response Sentiment Analyse3 | Langchain Agent               | Sentiment & brand analysis (OpenAI/ChatGPT) | normalized-tool-response    | Prepare Sheet Columns          |                                                                                                                      |
| Output Parser             | Langchain Output Parser        | Parse structured output         | Chat Model                   | Response Sentiment Analyse1    |                                                                                                                      |
| Structured Output Parser3 | Langchain Output Parser        | Parse structured output         | OpenAI Chat Model1            | Response Sentiment Analyse3    |                                                                                                                      |
| Map LLM Output            | Set                            | Extract Perplexity data fields  | Perplexity Request1           | Response Sentiment Analyse1    |                                                                                                                      |
| Perplexity Mapper         | Set                            | Normalize Perplexity response   | Perplexity Request            | normalized-tool-response       |                                                                                                                      |
| ChatGPT Mapper            | Set                            | Normalize APIfy ChatGPT response | If successful               | normalized-tool-response       |                                                                                                                      |
| final mapping             | Set                            | Combine sentiment & sources     | Response Sentiment Analyse1    | Sheet/Excel updaten           |                                                                                                                      |
| Prepare Sheet Columns     | Set                            | Prepare normalized fields       | Response Sentiment Analyse3    | Append row in sheet            |                                                                                                                      |
| normalized-tool-response  | NoOp                           | Pass normalized data            | OpenAI, Perplexity Mapper, ChatGPT Mapper | Response Sentiment Analyse3 |                                                                                                                      |
| Sheet/Excel updaten       | Google Sheets Append           | Store results (single model)    | final mapping                 | loop-end                     |                                                                                                                      |
| Append row in sheet       | Google Sheets Append           | Store results (multi-model)     | Prepare Sheet Columns         | loop-end                     |                                                                                                                      |
| loop-end                  | NoOp                           | End of batch loop               | Sheet/Excel updaten, Append row in sheet, If successful | Loop Over prompts           |                                                                                                                      |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Input Reception Setup:**
   - Add a **Manual Trigger** node to start the workflow manually.
   - Add a **Google Sheets** node ("Read Prompts1") configured to read the prompt column (e.g., range `A1:A100`) from your Google Sheet with input prompts. Use OAuth2 credentials linked to your Google account.
   - Connect Manual Trigger → Read Prompts1.
   - Add a **Limit** node ("Limit for testing"), set max items to 2, connect Read Prompts1 → Limit for testing.
   - Add a **NoOp** node ("before-loop-input"), connect Limit for testing → before-loop-input.
   - Add a **SplitInBatches** node ("Loop Over prompts") to batch process prompts, connect before-loop-input → Loop Over prompts.
   - Add a **NoOp** node ("loop-input") to mark batch start, connect Loop Over prompts → loop-input.
   - Optionally, add a **Set** node ("manual input") with a hardcoded prompt string for testing, connect it to before-loop-input for manual testing.

3. **Multi-Model AI Querying:**
   - Add a **Langchain OpenAI Chat Model** node ("OpenAI Chat Model (GPT 5)"), select model "gpt-5", connect loop-input → OpenAI Chat Model (GPT 5).
   - Add a **Chain LLM** node ("OpenAI Anfrage"), set prompt to use `{{$json.Prompt}}`, connect OpenAI Chat Model → OpenAI Anfrage.
   - Add a **Set** node ("OpenAI") to capture response text and label source as "OpenAI". Map response field to `{{$json.text}}` and add field `LLM = "OpenAI"`. Connect OpenAI Anfrage → OpenAI.
   - Add a **Perplexity** node ("Perplexity Request") with model "sonar". Set message content dynamically from `{{$json.Prompt}}`. Use your Perplexity API credentials. Connect loop-input → Perplexity Request.
   - Add a **Perplexity** node ("Perplexity Request1") similarly for the hardcoded prompt or batch. Connect a **Set** node ("LLM-Prompts") with static prompt for testing → Perplexity Request1.
   - Add an **HTTP Request** node ("APIfy Call ChatGPT Scraper"), method POST, URL: `https://api.apify.com/v2/acts/automation_nerd~chatgpt-prompt-actor/run-sync-get-dataset-items`. Body JSON with `prompts` array containing the prompt, use "generic HTTP query auth" with token header named "token". Connect loop-input → APIfy Call ChatGPT Scraper.
   - Add an **If** node ("If successful") that checks if the prompt field is not empty in the APIfy response. Connect APIfy Call ChatGPT Scraper → If successful.
   - From If successful true output, connect to ChatGPT Mapper (see below).
   - From If successful false output, connect to loop-end (NoOp).

4. **Sentiment and Brand Analysis:**
   - Add a **Set** node ("Map LLM Output") for Perplexity results to extract citations and message. Connect Perplexity Request1 → Map LLM Output.
   - Add a **Langchain Agent** node ("Response Sentiment Analyse1") configured with a custom prompt to analyze sentiment, emotion, and brand hierarchy from the Perplexity API JSON message. Connect Map LLM Output → Response Sentiment Analyse1.
   - Add an **Output Parser** node ("Output Parser") with JSON schema for sentiment keys, connect OpenAI Chat Model → Output Parser → Response Sentiment Analyse1.
   - Add a **Set** node ("Perplexity Mapper") to normalize Perplexity API response fields, connect Perplexity Request → Perplexity Mapper.
   - Add a **Set** node ("ChatGPT Mapper") to normalize APIfy ChatGPT scraped response fields (response text, citations URLs, news listing), connect If successful → ChatGPT Mapper.
   - Add a **NoOp** node ("normalized-tool-response") to funnel normalized data from OpenAI, Perplexity Mapper, and ChatGPT Mapper → Response Sentiment Analyse3.
   - Add a **Langchain Agent** node ("Response Sentiment Analyse3") with sentiment analysis prompt for generic AI responses; input connected from normalized-tool-response.
   - Add a **Structured Output Parser** ("Structured Output Parser3") with JSON schema example connected from OpenAI Chat Model1 to Response Sentiment Analyse3.

5. **Result Normalization and Storage Preparation:**
   - Add a **Set** node ("final mapping") that merges sentiment analysis results with LLM response data and citations from Perplexity; connect Response Sentiment Analyse1 → final mapping.
   - Add a **Set** node ("Prepare Sheet Columns") that prepares fields for Google Sheets append, mapping prompt, response, brand mention flag, sentiment fields, and source URLs. Connect Response Sentiment Analyse3 → Prepare Sheet Columns.

6. **Data Persistence:**
   - Add a **Google Sheets Append** node ("Sheet/Excel updaten") to append final mapping data into the designated "Output" sheet of your Google Sheet. Configure columns to auto-map. Connect final mapping → Sheet/Excel updaten.
   - Add a **Google Sheets Append** node ("Append row in sheet") to append Prepare Sheet Columns data into a separate sheet "Output many models". Connect Prepare Sheet Columns → Append row in sheet.

7. **Control Flow and Looping:**
   - Add a **NoOp** node ("loop-end") as batch processing end marker. Connect Sheet/Excel updaten → loop-end and Append row in sheet → loop-end.
   - Connect loop-end → Loop Over prompts to process next batch.
   - Connect If successful false → loop-end to handle empty or failed APIfy responses.

8. **Credentials Setup:**
   - Configure Google Sheets OAuth2 credentials with access to the target spreadsheet.
   - Configure OpenAI API credentials with access to GPT models.
   - Configure Perplexity API credentials.
   - Configure generic HTTP Query Auth credentials with APIfy token for ChatGPT scraping.

9. **Testing and Validation:**
   - Test manually with Manual Trigger.
   - Validate Google Sheets append operations.
   - Monitor for API limits and error handling on failed calls.

---

## 5. General Notes & Resources

| Note Content                                                                                                                            | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Users increasingly rely on AI tools like ChatGPT and Perplexity for information. Brands must understand their visibility in these tools.| Use Case Explanation sticky note.                                                              |
| Workflow can be extended to more AI tools and custom evaluations; acts as a kickstart for brand visibility research in AI search.      | Use Case Explanation sticky note.                                                              |
| APIfy ChatGPT Scraper node should be used cautiously; may violate OpenAI terms or be restricted.                                        | Sticky Note20 attached to APIfy Call ChatGPT Scraper node.                                     |
| For professional AI automation services, see: https://www.aoe.com/de/services/automation-ai/n8n                                      | Use Case Explanation sticky note with link.                                                    |
| Multi-model flow uses Google Sheets as input and output storage; requires two sheets: one for prompts (column "Prompt") and one for results. | Sticky Note22 describes usage and setup instructions.                                          |
| Simple Perplexity flow uses hardcoded prompt and Google Sheets for storing outputs; suitable for demo and extension.                   | Sticky Note21 explains simple Perplexity flow usage.                                          |

---

**Disclaimer:**  
The provided text and workflow are exclusively automated content created with n8n, an integration and automation tool. All processing complies with applicable content policies and contains no illegal or protected elements. All data handled is legal and public.

---
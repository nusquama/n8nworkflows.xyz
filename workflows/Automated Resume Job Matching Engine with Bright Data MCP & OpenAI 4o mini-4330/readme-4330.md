Automated Resume Job Matching Engine with Bright Data MCP & OpenAI 4o mini

https://n8nworkflows.xyz/workflows/automated-resume-job-matching-engine-with-bright-data-mcp---openai-4o-mini-4330


# Automated Resume Job Matching Engine with Bright Data MCP & OpenAI 4o mini

### 1. Workflow Overview

This workflow automates the process of matching a candidate's resume with job listings extracted from LinkedIn using Bright Data's MCP (Massive Crawling Platform) and analyzes the matches using OpenAI's GPT-4o-mini model. It targets HR professionals and recruitment systems aiming to streamline job matching by extracting job postings, parsing job descriptions, and scoring skills alignment with candidate resumes.

The workflow is logically grouped into the following blocks:

- **1.1 Input Initialization and Triggering**: Starts with manual triggering and setting up candidate/job search parameters.
- **1.2 Job Listings Extraction**: Uses Bright Data MCP to scrape LinkedIn job search results and extract individual job URLs.
- **1.3 Job Description Extraction Loop**: Iterates over the job URLs to scrape detailed job descriptions.
- **1.4 AI Processing for Job Description Analysis**: Applies OpenAI LLM (GPT-4o-mini) to extract structured job description content.
- **1.5 AI Job Matching**: Compares the candidate resume against job descriptions using OpenAI LLM to generate a skill match analysis and scoring.
- **1.6 Output Handling and Notifications**: Parses the AI output, sends webhook notifications, and writes results to disk.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization and Triggering

**Overview:**  
This block sets up the workflow trigger and initializes input data including candidate resume, keywords, location, and job search base URL.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- List all tools for Bright Data  
- Set the Input fields

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type & Role:* Manual Trigger node to start workflow execution on demand.  
  - *Config:* No parameters; purely trigger action.  
  - *Inputs/Outputs:* No input; output to next node (List all tools for Bright Data).  
  - *Failure Modes:* None typical; manual trigger.  

- **List all tools for Bright Data**  
  - *Type & Role:* MCP Client node to list available Bright Data tools, verifying API connectivity.  
  - *Config:* Uses MCP Client API credentials (MCP Client (STDIO) account).  
  - *Inputs/Outputs:* Input from Manual Trigger; output to Set the Input fields.  
  - *Failure Modes:* Authentication failure with Bright Data API, network issues.  

- **Set the Input fields**  
  - *Type & Role:* Set node to define workflow input parameters such as resume text, keywords, location, and job search base URL.  
  - *Config:* Hardcoded example values: resume text ("I am Pechi..."), keywords ("Python"), location ("India"), job search URL (LinkedIn jobs search).  
  - *Inputs/Outputs:* Receives from MCP client node; outputs to Bright Data MCP Client for Jobs Extraction.  
  - *Failure Modes:* Misconfigured input values or missing parameters.  

---

#### 1.2 Job Listings Extraction

**Overview:**  
Scrapes LinkedIn job search page using Bright Data MCP to extract raw HTML content, then uses OpenAI LLM to extract all job posting URLs from the page.

**Nodes Involved:**  
- Bright Data MCP Client For Jobs Extraction  
- OpenAI Chat Model for Paginated Job Extract  
- Paginated Job Data Extractor  
- Split Out

**Node Details:**

- **Bright Data MCP Client For Jobs Extraction**  
  - *Type & Role:* MCP Client node scraping the LinkedIn job search URL constructed from input parameters.  
  - *Config:* Tool: "scrape_as_html"; URL parameter dynamically formed as `{{ $json.job_search_base_url }}?keywords={{ $json.keywords }}&location={{ $json.location }}`.  
  - *Credentials:* MCP Client API.  
  - *Inputs/Outputs:* Input from Set the Input fields; output to OpenAI Chat Model for Paginated Job Extract and Paginated Job Data Extractor.  
  - *Failure Modes:* URL errors, scraping failures, rate limits, API errors from Bright Data.  

- **OpenAI Chat Model for Paginated Job Extract**  
  - *Type & Role:* OpenAI GPT-4o-mini chat model; processes scraped HTML to assist in extracting job links.  
  - *Config:* Model set to GPT-4o-mini, no special options.  
  - *Credentials:* OpenAI API.  
  - *Inputs/Outputs:* Input from Bright Data MCP Client; output to Paginated Job Data Extractor.  
  - *Failure Modes:* API quota limits, malformed prompt, response parsing errors.  

- **Paginated Job Data Extractor**  
  - *Type & Role:* Langchain Information Extractor node; extracts an array of job links from the LLM-processed content.  
  - *Config:* Manual JSON schema specifying array of objects with "link" string property; input text references extracted content from previous node.  
  - *Inputs/Outputs:* Input from OpenAI Chat Model for Paginated Job Extract; outputs to Split Out.  
  - *Failure Modes:* Schema mismatch, missing content, extraction failure.  

- **Split Out**  
  - *Type & Role:* Split Out node; splits the array of job links into individual items for looping.  
  - *Config:* Field to split: "output" (holding extracted job URLs).  
  - *Inputs/Outputs:* Input from Paginated Job Data Extractor; output to Loop Over Items.  
  - *Failure Modes:* Empty arrays, null field.  

---

#### 1.3 Job Description Extraction Loop

**Overview:**  
Iterates over each extracted job URL to scrape detailed job descriptions using Bright Data MCP.

**Nodes Involved:**  
- Loop Over Items  
- Bright Data MCP Client For Jobs Extraction within a Loop  
- Job Desc Information Extractor  
- OpenAI Chat Model for Job Desc Extract

**Node Details:**

- **Loop Over Items**  
  - *Type & Role:* Split In Batches node; processes each job URL individually in batches (default batch size).  
  - *Config:* Default options; processes items one-by-one.  
  - *Inputs/Outputs:* Input from Split Out; outputs to Bright Data MCP Client For Jobs Extraction within a Loop.  
  - *Failure Modes:* Empty input, batch size issues.  

- **Bright Data MCP Client For Jobs Extraction within a Loop**  
  - *Type & Role:* MCP Client node; scrapes individual job posting pages by URL.  
  - *Config:* Tool "scrape_as_html"; URL set dynamically from current item’s "output" field (job URL).  
  - *Credentials:* MCP Client API.  
  - *Inputs/Outputs:* Input from Loop Over Items; outputs scraped HTML content to OpenAI Chat Model for Job Desc Extract.  
  - *Failure Modes:* URL invalid, scraping failure, API quota.  

- **OpenAI Chat Model for Job Desc Extract**  
  - *Type & Role:* OpenAI GPT-4o-mini chat model to assist in extracting clean textual job description from scraped content.  
  - *Config:* Model GPT-4o-mini; no special options.  
  - *Credentials:* OpenAI API.  
  - *Inputs/Outputs:* Input from MCP Client within loop; output to Job Desc Information Extractor.  
  - *Failure Modes:* API errors, prompt failures.  

- **Job Desc Information Extractor**  
  - *Type & Role:* Langchain Information Extractor node; extracts a structured "job_description" attribute from the LLM output.  
  - *Config:* Extracts text from `{{ $json.result.content[0].text }}` with attribute named "job_description".  
  - *Inputs/Outputs:* Input from OpenAI Chat Model for Job Desc Extract; outputs to AI Job Match node.  
  - *Failure Modes:* Missing content, extraction failure, retry on fail enabled.  

---

#### 1.4 AI Processing for Job Matching

**Overview:**  
Compares the candidate’s resume against each job description using OpenAI to produce a structured job match analysis, including skill matches and scoring.

**Nodes Involved:**  
- AI Job Match  
- OpenAI Chat Model for AI Job Match  
- Structured Output Parser

**Node Details:**

- **AI Job Match**  
  - *Type & Role:* Langchain Chain LLM node; generates job match analysis and scoring in JSON format.  
  - *Config:* Prompt defines role as job matcher, inputs resume and job description, requests JSON output with skill matches and scores. Output parser enabled for structured response.  
  - *Inputs/Outputs:* Input from Job Desc Information Extractor; outputs parsed JSON to Structured Output Parser.  
  - *Failure Modes:* LLM API errors, prompt generation issues, parsing errors, retry on fail enabled.  

- **OpenAI Chat Model for AI Job Match**  
  - *Type & Role:* OpenAI GPT-4o-mini chat model backing the AI Job Match node.  
  - *Config:* Model GPT-4o-mini; no special options.  
  - *Credentials:* OpenAI API.  
  - *Inputs/Outputs:* Input from AI Job Match node; output to AI Job Match for processing.  
  - *Failure Modes:* API quota, network errors.  

- **Structured Output Parser**  
  - *Type & Role:* Langchain Structured Output Parser node; parses AI Job Match JSON response into usable structured data.  
  - *Config:* JSON schema example provided with fields like job_match_analysis, skill_match array, overall_match_score, rationale, and recommendations.  
  - *Inputs/Outputs:* Input from AI Job Match; output to Create a binary data for AI Job Match and Webhook Notification nodes.  
  - *Failure Modes:* Schema mismatch, parsing failures.  

---

#### 1.5 Output Handling and Notifications

**Overview:**  
Converts AI job match JSON into binary data, sends notification via webhook, writes output to disk, and loops back to continue processing.

**Nodes Involved:**  
- Create a binary data for AI Job Match  
- Webhook Notification for AI Job Match  
- Write the AI job matched response to disk  
- Loop Over Items (loop back)

**Node Details:**

- **Create a binary data for AI Job Match**  
  - *Type & Role:* Function node; converts JSON job match analysis into base64-encoded binary data for file writing.  
  - *Config:* JavaScript creates Buffer from JSON stringified data.  
  - *Inputs/Outputs:* Input from Structured Output Parser; output to Write file node and Webhook Notification.  
  - *Failure Modes:* Data conversion errors.  

- **Webhook Notification for AI Job Match**  
  - *Type & Role:* HTTP Request node; sends job match JSON to a configured webhook URL.  
  - *Config:* POST request to webhook.site URL with multipart form data, sending job_match_response parameter containing JSON string.  
  - *Inputs/Outputs:* Input from Structured Output Parser; output loops back to Loop Over Items for next iteration.  
  - *Failure Modes:* Network errors, webhook unavailability.  

- **Write the AI job matched response to disk**  
  - *Type & Role:* ReadWriteFile node; saves the job match JSON data to a local file with timestamped filename.  
  - *Config:* Writes to `d:\Job-Match-<timestamp>.json`.  
  - *Inputs/Outputs:* Input from Create a binary data node; no outputs (workflow ends here).  
  - *Failure Modes:* File system permission errors, invalid path.  

- **Loop Over Items (loop back)**  
  - *Role:* Continues processing next job listing by looping over items.  

---

### 3. Summary Table

| Node Name                                | Node Type                          | Functional Role                               | Input Node(s)                          | Output Node(s)                                         | Sticky Note                                                                                       |
|-----------------------------------------|----------------------------------|----------------------------------------------|--------------------------------------|--------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’            | Manual Trigger                   | Workflow start trigger                        | None                                 | List all tools for Bright Data                          | Note: Deals with LinkedIn profile data extraction; ensure input fields and webhook URL setup.    |
| List all tools for Bright Data           | MCP Client                      | Validate Bright Data API connectivity         | When clicking ‘Test workflow’         | Set the Input fields                                    |                                                                                                 |
| Set the Input fields                     | Set                             | Define resume, keywords, location, base URL  | List all tools for Bright Data        | Bright Data MCP Client For Jobs Extraction              |                                                                                                 |
| Bright Data MCP Client For Jobs Extraction | MCP Client                      | Scrape LinkedIn jobs search page HTML         | Set the Input fields                  | OpenAI Chat Model for Paginated Job Extract, Paginated Job Data Extractor | Bright Data MCP Job Extract via Job Listings; scrapes job info for AI matching.                   |
| OpenAI Chat Model for Paginated Job Extract | OpenAI Chat Model (GPT-4o-mini) | Extract job links from scraped HTML           | Bright Data MCP Client For Jobs Extraction | Paginated Job Data Extractor                             |                                                                                                 |
| Paginated Job Data Extractor             | Info Extractor                  | Extract array of job URLs from AI response    | OpenAI Chat Model for Paginated Job Extract | Split Out                                             |                                                                                                 |
| Split Out                               | Split Out                      | Split job links array into individual items   | Paginated Job Data Extractor           | Loop Over Items                                        |                                                                                                 |
| Loop Over Items                         | Split In Batches               | Iterate over each job URL                      | Split Out                           | Bright Data MCP Client For Jobs Extraction within a Loop |                                                                                                 |
| Bright Data MCP Client For Jobs Extraction within a Loop | MCP Client                      | Scrape individual job posting HTML            | Loop Over Items                     | OpenAI Chat Model for Job Desc Extract                 |                                                                                                 |
| OpenAI Chat Model for Job Desc Extract   | OpenAI Chat Model (GPT-4o-mini) | Extract job description text from scraped HTML | Bright Data MCP Client For Jobs Extraction within a Loop | Job Desc Information Extractor                         |                                                                                                 |
| Job Desc Information Extractor           | Langchain Information Extractor | Extract structured job description attribute  | OpenAI Chat Model for Job Desc Extract | AI Job Match                                           |                                                                                                 |
| AI Job Match                           | Langchain Chain LLM             | Generate job match analysis JSON                | Job Desc Information Extractor       | Structured Output Parser                                |                                                                                                 |
| OpenAI Chat Model for AI Job Match       | OpenAI Chat Model (GPT-4o-mini) | LLM backend for job match analysis             | AI Job Match                       | AI Job Match                                           |                                                                                                 |
| Structured Output Parser                 | Langchain Output Parser Structured | Parse AI JSON job match response               | AI Job Match                       | Create a binary data for AI Job Match, Webhook Notification for AI Job Match |                                                                                                 |
| Create a binary data for AI Job Match    | Function                       | Convert JSON response to base64 binary data    | Structured Output Parser            | Write the AI job matched response to disk, Webhook Notification for AI Job Match |                                                                                                 |
| Webhook Notification for AI Job Match   | HTTP Request                   | Send job match results to external webhook     | Structured Output Parser            | Loop Over Items                                        | Note: Update webhook URL of your interest.                                                     |
| Write the AI job matched response to disk | ReadWriteFile                 | Save job match result JSON to local disk       | Create a binary data for AI Job Match | None                                                  |                                                                                                 |
| Sticky Note2                            | Sticky Note                    | Disclaimer about template availability          | None                              | None                                                  | Disclaimer: Template available only on n8n self-hosted due to community MCP Client node.         |
| Sticky Note4                            | Sticky Note                    | Notes on LLM usage                              | None                              | None                                                  | LLM Usages: OpenAI 4o mini LLM used for structured data extraction.                            |
| Sticky Note                             | Sticky Note                    | Notes on Bright Data MCP job extraction         | None                              | None                                                  | Bright Data MCP job extract via job listings feeding AI job matching using GPT 4o mini.          |
| Sticky Note5                            | Sticky Note                    | Logo display                                    | None                              | None                                                  | Logo: Bright Data logo [https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png] |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** called `When clicking ‘Test workflow’` to serve as the workflow start.

2. **Add an MCP Client node** named `List all tools for Bright Data`:  
   - Credentials: Assign your Bright Data MCP API credentials.  
   - Leave parameters default to list tools.

3. **Add a Set node** named `Set the Input fields`:  
   - Assign string fields:  
     - `resume`: e.g. "I am Pechi, Senior Python Developer with 9+ years of experience."  
     - `keywords`: e.g. "Python"  
     - `location`: e.g. "India"  
     - `job_search_base_url`: e.g. "https://www.linkedin.com/jobs/search/"

4. **Add MCP Client node** named `Bright Data MCP Client For Jobs Extraction`:  
   - Credentials: MCP API.  
   - ToolName: "scrape_as_html"  
   - Operation: "executeTool"  
   - Tool Parameters: JSON expression:  
     ```json
     { "url": "{{ $json.job_search_base_url }}?keywords={{ $json.keywords }}&location={{ $json.location }}" }
     ```

5. **Add OpenAI Chat Model node** named `OpenAI Chat Model for Paginated Job Extract`:  
   - Model: gpt-4o-mini  
   - Credentials: OpenAI API

6. **Add Information Extractor node** named `Paginated Job Data Extractor`:  
   - Text: `Extract all the job links from the provided content. Here's the content: {{ $json.result.content[0].text }}`  
   - Input Schema: array of objects with a string field "link"

7. **Add Split Out node** named `Split Out`:  
   - Field to split out: `output`

8. **Add Split In Batches node** named `Loop Over Items`:  
   - Default batch size (process one item at a time)

9. **Add MCP Client node** named `Bright Data MCP Client For Jobs Extraction within a Loop`:  
   - Tool: "scrape_as_html"  
   - Tool Parameters: `{ "url": "{{ $json.output }}" }`  
   - Credentials: MCP Client API

10. **Add OpenAI Chat Model node** named `OpenAI Chat Model for Job Desc Extract`:  
    - Model: gpt-4o-mini  
    - Credentials: OpenAI API

11. **Add Information Extractor node** named `Job Desc Information Extractor`:  
    - Text: `Extract the job description in a textual format. Here's the content: {{ $json.result.content[0].text }}`  
    - Attribute: `job_description`

12. **Add Langchain Chain LLM node** named `AI Job Match`:  
    - Text prompt:  
      ```
      Hi, you are a helpful job matcher, you analyze the given resume and job description and providing a job matching skills and score in a JSON format.

      Here's the Resume: {{ $('Set the Input fields').item.json.resume }}

      Here's the Job Desc:

      {{ $json.output.job_description }}
      ```  
    - Enable output parser.  
    - Retry on fail enabled.

13. **Add OpenAI Chat Model node** named `OpenAI Chat Model for AI Job Match`:  
    - Model: gpt-4o-mini  
    - Credentials: OpenAI API

14. **Add Structured Output Parser node** named `Structured Output Parser`:  
    - Provide a JSON schema example with fields like job_match_analysis, skill_match, overall_match_score, rationale, recommendations.

15. **Add Function node** named `Create a binary data for AI Job Match`:  
    - Function code:  
      ```javascript
      items[0].binary = {
        data: {
          data: Buffer.from(JSON.stringify(items[0].json, null, 2)).toString('base64')
        }
      };
      return items;
      ```

16. **Add HTTP Request node** named `Webhook Notification for AI Job Match`:  
    - Method: POST  
    - URL: Your webhook URL (e.g., https://webhook.site/...)  
    - Content-Type: multipart/form-data  
    - Body Parameters: `job_match_response` set to the JSON string of the job match analysis.

17. **Add ReadWriteFile node** named `Write the AI job matched response to disk`:  
    - Operation: Write  
    - File Name: Use expression like `d:\Job-Match-{{$now.toSeconds()}}.json`

18. **Connect nodes in the following order:**  
    - `When clicking ‘Test workflow’` → `List all tools for Bright Data` → `Set the Input fields` → `Bright Data MCP Client For Jobs Extraction` → `OpenAI Chat Model for Paginated Job Extract` → `Paginated Job Data Extractor` → `Split Out` → `Loop Over Items` → `Bright Data MCP Client For Jobs Extraction within a Loop` → `OpenAI Chat Model for Job Desc Extract` → `Job Desc Information Extractor` → `AI Job Match` → `Structured Output Parser` → `Create a binary data for AI Job Match` → `Write the AI job matched response to disk`  
    - Also connect `Structured Output Parser` → `Webhook Notification for AI Job Match` → loop back to `Loop Over Items` for iterative processing.

19. **Credential Setup:**  
    - Configure MCP Client API credentials for all MCP Client nodes.  
    - Configure OpenAI API credentials for all OpenAI Chat nodes.

20. **Optional:** Add Sticky Notes for context and disclaimers as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This template is only available on n8n self-hosted as it's making use of the community node for MCP Client.             | Sticky Note2 (Disclaimer)                                                                           |
| OpenAI 4o mini LLM is utilized for structured data extraction and AI job matching.                                       | Sticky Note4 (LLM Usages)                                                                           |
| Bright Data MCP used to extract job listings, feeding AI job matching with OpenAI GPT 4o mini LLM.                       | Sticky Note (Bright Data MCP Job Extract via Job Listings)                                         |
| Deals with LinkedIn profile data extraction; ensure input fields node is updated with LinkedIn profile URL, resume, etc. | Sticky Note1                                                                                        |
| Update the Webhook Notification URL to your endpoint of interest to receive job matching results.                        | Sticky Note1, Webhook Notification node parameter                                                  |
| Bright Data Logo: ![logo](https://images.seeklogo.com/logo-png/43/1/brightdata-logo-png_seeklogo-439974.png)             | Sticky Note5                                                                                       |

---

**Disclaimer:**  
The provided text is derived solely from an automated workflow created with n8n, adhering strictly to content policies without containing any illegal, offensive, or protected elements. All handled data is legal and public.
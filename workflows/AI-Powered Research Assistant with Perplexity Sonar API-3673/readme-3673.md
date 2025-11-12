AI-Powered Research Assistant with Perplexity Sonar API

https://n8nworkflows.xyz/workflows/ai-powered-research-assistant-with-perplexity-sonar-api-3673


# AI-Powered Research Assistant with Perplexity Sonar API

### 1. Workflow Overview

This workflow, titled **AI-Powered Research Agent using Perplexity Sonar**, serves as an AI-driven research assistant that integrates with the Perplexity AI Sonar API. It is designed to be triggered by another workflow and processes user queries by sending them to the Perplexity API for up-to-date research results. The workflow then parses and formats the API response for downstream use.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the trigger and input query from an external workflow.
- **1.2 Prompt Preparation:** Sets up system and user prompt variables dynamically.
- **1.3 API Request:** Sends a POST request to the Perplexity Sonar API with authentication.
- **1.4 Response Extraction:** Parses the API response to extract the relevant message content.
- **1.5 Documentation:** Provides an overview note describing the workflow purpose.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for an execution trigger from another workflow and passes the incoming data through without modification.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - Type: `Execute Workflow Trigger`  
    - Role: Entry point triggered externally to start this workflow.  
    - Configuration: Passes input data as-is (`inputSource: passthrough`).  
    - Inputs: Triggered externally.  
    - Outputs: Passes data to "Set Prompt Variables" node.  
    - Edge Cases: If no external workflow triggers this node, the workflow remains idle.  
    - Failure Modes: None specific, but if upstream workflow fails to send data, no execution occurs.

#### 1.2 Prompt Preparation

- **Overview:**  
  This block assigns the system prompt and user query variables that will be sent to the Perplexity API. It dynamically sets the user prompt based on incoming data.

- **Nodes Involved:**  
  - Set Prompt Variables

- **Node Details:**  
  - **Set Prompt Variables**  
    - Type: `Set` node  
    - Role: Defines two string variables:  
      - `System`: A detailed system prompt describing the assistant's capabilities and behavior.  
      - `User`: Dynamically assigned from incoming JSON property `query`.  
    - Configuration:  
      - System prompt is a fixed descriptive string about the assistant’s abilities.  
      - User prompt uses expression `={{ $json.query }}` to extract the query from input JSON.  
    - Inputs: Receives data from "When Executed by Another Workflow".  
    - Outputs: Passes variables to the HTTP Request node.  
    - Edge Cases: If `query` is missing or empty, downstream nodes may receive empty user prompt. Consider validation upstream.  
    - Failure Modes: Expression failure if input JSON structure changes unexpectedly.

#### 1.3 API Request

- **Overview:**  
  Sends a POST request to the Perplexity AI Sonar API endpoint `/chat/completions` with the prepared prompts and authentication, requesting AI-generated research content.

- **Nodes Involved:**  
  - Perplexity Research Content1

- **Node Details:**  
  - **Perplexity Research Content1**  
    - Type: `HTTP Request` node  
    - Role: Calls Perplexity API with model "sonar" and sends system and user messages.  
    - Configuration:  
      - Method: POST  
      - URL: `https://api.perplexity.ai/chat/completions`  
      - Body: JSON with parameters including:  
        - `model`: "sonar"  
        - `messages`: array containing system and user roles with content from previous node variables  
        - Other parameters: max_tokens=4000, temperature=0.2, top_p=0.9, return_citations=true, search_domain_filter=["perplexity.ai"], search_recency_filter="month", frequency_penalty=1, etc.  
      - Authentication: HTTP Header Auth with Bearer token (credential named "Header Auth account 2")  
    - Inputs: Receives system and user prompt variables from "Set Prompt Variables".  
    - Outputs: Sends API response JSON to next node.  
    - Edge Cases:  
      - API key invalid or missing → authentication error.  
      - Network timeout or API downtime → request failure.  
      - Rate limiting by Perplexity API.  
      - Unexpected response format changes.  
    - Failure Modes: HTTP errors, JSON parse errors, auth failures.

#### 1.4 Response Extraction

- **Overview:**  
  Extracts the main content message from the API response JSON for clean output.

- **Nodes Involved:**  
  - Extract API Response

- **Node Details:**  
  - **Extract API Response**  
    - Type: `Set` node  
    - Role: Assigns a new string variable `Respone Message Content` (note typo in name) from the first choice's message content in the API response.  
    - Configuration: Uses expression `={{ $json.choices[0].message.content }}` to extract content.  
    - Inputs: Receives full API response JSON from "Perplexity Research Content1".  
    - Outputs: Provides cleaned message content for downstream use or return.  
    - Edge Cases:  
      - If `choices` array is empty or missing → expression fails.  
      - If API response structure changes → extraction fails.  
    - Failure Modes: Expression evaluation errors, missing data.

#### 1.5 Documentation

- **Overview:**  
  Provides a sticky note summarizing the workflow’s purpose and functionality for users viewing the workflow in n8n.

- **Nodes Involved:**  
  - Workflow Overview

- **Node Details:**  
  - **Workflow Overview**  
    - Type: `Sticky Note`  
    - Role: Documentation within the workflow editor.  
    - Content: Brief explanation that the workflow takes a user query, formats it, sends it to Perplexity AI Sonar, and returns extracted responses.  
    - Inputs/Outputs: None (informational only).  
    - Edge Cases: N/A

---

### 3. Summary Table

| Node Name                   | Node Type                   | Functional Role                              | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                      |
|-----------------------------|-----------------------------|----------------------------------------------|------------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger    | Entry trigger receiving input from another workflow | None                         | Set Prompt Variables         | Find the latest content related to the field/knowledge you are interested in. In-depth materials to prepare for the writing section |
| Set Prompt Variables         | Set                         | Assigns system and user prompt variables     | When Executed by Another Workflow | Perplexity Research Content1 |                                                                                                |
| Perplexity Research Content1 | HTTP Request                | Sends POST request to Perplexity Sonar API   | Set Prompt Variables          | Extract API Response         |                                                                                                |
| Extract API Response         | Set                         | Extracts main message content from API response | Perplexity Research Content1  | None                        |                                                                                                |
| Workflow Overview           | Sticky Note                 | Describes workflow purpose and logic         | None                         | None                        | ## Perplexity Research Workflow Overview This workflow takes a user query, formats it using a system prompt, and sends it to the Perplexity AI Sonar model for search. Responses are extracted and returned as clean output. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
   - Set `inputSource` to `passthrough` to accept incoming data unchanged.

2. **Create Prompt Variables Node**  
   - Add a **Set** node named `Set Prompt Variables`.  
   - Connect `When Executed by Another Workflow` → `Set Prompt Variables`.  
   - Configure two string variables:  
     - `System`: Paste the detailed assistant description text (see section 2.2).  
     - `User`: Set value to expression `={{ $json.query }}` to dynamically extract the query from input JSON.

3. **Create HTTP Request Node**  
   - Add an **HTTP Request** node named `Perplexity Research Content1`.  
   - Connect `Set Prompt Variables` → `Perplexity Research Content1`.  
   - Configure:  
     - HTTP Method: POST  
     - URL: `https://api.perplexity.ai/chat/completions`  
     - Authentication: HTTP Header Auth with Bearer token (create credential with your Perplexity AI API key).  
     - Body Content Type: JSON  
     - Request Body: Use the following JSON with expressions:  
       ```json
       {
         "model": "sonar",
         "messages": [
           {
             "role": "system",
             "content": "{{ $json.System }}"
           },
           {
             "role": "user",
             "content": "{{ $json.User || $json.query || $json.question || $json['Research Query'] || 'No input provided' }}"
           }
         ],
         "max_tokens": 4000,
         "temperature": 0.2,
         "top_p": 0.9,
         "return_citations": true,
         "search_domain_filter": ["perplexity.ai"],
         "return_images": false,
         "return_related_questions": false,
         "search_recency_filter": "month",
         "top_k": 0,
         "stream": false,
         "presence_penalty": 0,
         "frequency_penalty": 1
       }
       ```
     - Enable `Send Body` and specify body as JSON.

4. **Create Response Extraction Node**  
   - Add a **Set** node named `Extract API Response`.  
   - Connect `Perplexity Research Content1` → `Extract API Response`.  
   - Configure one string variable:  
     - `Respone Message Content` (note the typo preserved) with expression `={{ $json.choices[0].message.content }}` to extract the main response text.

5. **Add Documentation Sticky Note**  
   - Add a **Sticky Note** named `Workflow Overview`.  
   - Position it clearly near the start of the workflow.  
   - Paste the overview content:  
     ```
     ## Perplexity Research Workflow Overview
     This workflow takes a user query, formats it using a system prompt, and sends it to the Perplexity AI Sonar model for search.
     Responses are extracted and returned as clean output.
     ```

6. **Save and Test**  
   - Ensure your Perplexity API key is valid and set in the HTTP Header Auth credential.  
   - Trigger the workflow from an external workflow passing a JSON with a `query` field.  
   - Verify that the response content is extracted correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                              |
|----------------------------------------------------------------------------------------------------|----------------------------------------------|
| Requires a valid Perplexity AI API Key with Bearer token authentication configured in n8n.         | Credential setup in n8n HTTP Header Auth node |
| Ensure your n8n environment allows outbound HTTP requests to `https://api.perplexity.ai`.           | Network / firewall configuration              |
| Customize the system prompt in `Set Prompt Variables` to tailor the assistant’s behavior per domain. | Workflow customization tip                     |
| This workflow is designed to be triggered from other workflows, enabling chaining with blog creation, summaries, or messaging integrations. | Integration use case                           |
| Replace or extend the final output handling to integrate with Google Sheets, Notion, Telegram, etc. | Post-processing customization                  |

---

This documentation fully describes the workflow structure, node configurations, and reproduction steps, enabling advanced users and automation agents to understand, modify, or recreate the AI-powered research assistant workflow.
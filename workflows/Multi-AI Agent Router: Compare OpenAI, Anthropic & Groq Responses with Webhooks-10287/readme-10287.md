Multi-AI Agent Router: Compare OpenAI, Anthropic & Groq Responses with Webhooks

https://n8nworkflows.xyz/workflows/multi-ai-agent-router--compare-openai--anthropic---groq-responses-with-webhooks-10287


# Multi-AI Agent Router: Compare OpenAI, Anthropic & Groq Responses with Webhooks

### 1. Workflow Overview

This workflow, titled **"Multi-AI Agent Router: Compare OpenAI, Anthropic & Groq Responses with Webhooks"**, is designed to process textual input data through multiple AI language models from different providers (OpenAI, Anthropic, and Groq). It dynamically routes requests based on configurable criteria such as cost, performance, or a balance of both. The system executes AI agents in parallel, merges their structured outputs, computes detailed performance metrics (including processing time, quality scores, and cost efficiency), and returns a comprehensive response via webhook.

**Target Use Cases:**  
- Comparative analysis of AI model responses for the same input to evaluate cost, speed, and quality.  
- Dynamic selection of AI providers and models for tasks based on input complexity and user priority.  
- Real-time enrichment and analysis of data with actionable recommendations.  
- Automated performance monitoring and decision support for AI usage optimization.

**Logical Blocks:**  
- **1.1 Input Reception & Parameter Extraction:** Handles incoming webhook requests and extracts relevant parameters.  
- **1.2 Dynamic Routing Logic:** Evaluates input complexity and priority to determine the best AI provider and model.  
- **1.3 AI Agent Execution:** Runs AI agents for OpenAI, Anthropic, and Groq in parallel with structured output parsing.  
- **1.4 Response Merging:** Combines AI responses into a unified dataset.  
- **1.5 Performance Metrics Calculation:** Computes metrics like processing time, cost efficiency, and quality scores.  
- **1.6 Webhook Response:** Sends the aggregated results and metrics back as an HTTP response.  
- **1.7 Documentation & Setup Notes:** Provides workflow usage instructions and setup guidelines via sticky notes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Parameter Extraction

- **Overview:**  
Receives HTTP POST requests via webhook, extracts the core input data and user preferences such as task type and priority.

- **Nodes Involved:**  
  - Webhook  
  - Extract Input Parameters

- **Node Details:**  
  - **Webhook**  
    - Type: `n8n-nodes-base.webhook`  
    - Role: Entry point for external HTTP POST requests on path `/ai-pipeline`.  
    - Config: POST method, response mode set to respond via a downstream node.  
    - Input: External HTTP requests.  
    - Output: JSON body with raw input data and optional parameters.  
    - Possible Failures: Invalid HTTP method, malformed JSON body, webhook timeout.

  - **Extract Input Parameters**  
    - Type: `n8n-nodes-base.set`  
    - Role: Extracts and assigns `input_data`, `task_type`, and `priority` from the webhook payload.  
    - Config: Reads `data` from `body.data`, defaults `task_type` to `general` and `priority` to `balanced` if missing.  
    - Expressions: `={{ $json.body.data }}`, `={{ $json.body.task_type || 'general' }}`, `={{ $json.body.priority || 'balanced' }}`  
    - Input: JSON from Webhook node.  
    - Output: Structured parameters used for routing.  
    - Edge Cases: Missing `data` field leads to empty input; invalid priority values are not validated here.

---

#### 1.2 Dynamic Routing Logic

- **Overview:**  
Determines which AI provider and model to use based on input length complexity and user priority (cost, performance, balanced). Outputs routing decision metadata.

- **Nodes Involved:**  
  - Dynamic LLM Router  
  - Route to Provider

- **Node Details:**  
  - **Dynamic LLM Router**  
    - Type: `n8n-nodes-base.code` (JavaScript)  
    - Role: Implements routing logic by calculating complexity score from input length and selecting provider/model accordingly.  
    - Logic:  
      - Complexity score: 1 (<500 chars), 2 (500-1999), 3 (2000+).  
      - Priority options: `cost`, `performance`, `balanced`.  
      - Maps to provider/model, estimated cost, expected quality.  
    - Outputs: Original input, task type, priority, routing decision object, timestamp.  
    - Input: Extracted parameters.  
    - Output: Routing metadata for downstream routing.  
    - Edge Cases: Unexpected priority values default to balanced logic. Empty input leads to complexity 0 (handled as 1 here).  
    - Potential Failures: Expression errors, unexpected input structure.

  - **Route to Provider**  
    - Type: `n8n-nodes-base.switch`  
    - Role: Routes workflow execution flow to the appropriate AI agent based on the provider from routing decision.  
    - Config: Switch on `$json.routing_decision.provider` with strict string equality matching for `"openai"`, `"anthropic"`, `"groq"`.  
    - Input: Output from Dynamic LLM Router.  
    - Output: One of three branches to respective AI agents.  
    - Edge Cases: Unknown provider value results in no branch execution (workflow halts silently).

---

#### 1.3 AI Agent Execution

- **Overview:**  
Runs AI models from OpenAI, Anthropic, and Groq in parallel, each configured to output structured JSON results via LangChain agents with parsing.

- **Nodes Involved:**  
  - OpenAI Model  
  - OpenAI Agent  
  - Output Parser1  
  - Anthropic Model  
  - Anthropic Agent  
  - Output Parser  
  - Groq Model  
  - Groq Agent  
  - Output Parser2

- **Node Details:**  
  - **OpenAI Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Connects to OpenAI API with OAuth credentials.  
    - Credentials: OpenAI API key configured (`OpenAi account 2`).  
    - Input: Text prompt from OpenAI Agent node.  
    - Output: AI-generated chat completion.  
    - Edge Cases: API quota exceeded, network issues.

  - **OpenAI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Defines prompt template, system message, and request structure for OpenAI.  
    - Prompt: Includes task type, input data, expects structured JSON with analysis, insights, recommendations, quality score (1-10).  
    - System Message: Shows provider, model, and expected quality from routing decision.  
    - Output Parser: Enabled to parse structured output.  
    - Input: Routed from Switch node (only when provider is openai).  
    - Output: Parsed structured response.  
    - Edge Cases: Malformed AI response, parsing errors.

  - **Output Parser1**  
    - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
    - Role: Parses OpenAI Agent raw AI response into structured JSON.  
    - Input: OpenAI Agent output.  
    - Output: Parsed data fed back to Merge node.

  - **Anthropic Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
    - Role: Connects to Anthropic API with API key credentials.  
    - Credentials: Anthropic API key configured (`Anthropic account`).  
    - Model: `claude-3-5-sonnet-20241022`.  
    - Input/Output: Similar to OpenAI model.  
    - Edge Cases: API key expiry, latency issues.

  - **Anthropic Agent**  
    - Same role and configuration pattern as OpenAI Agent but for Anthropic.

  - **Output Parser**  
    - Parses Anthropic Agent output similarly.

  - **Groq Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatGroq`  
    - Role: Connects to Groq API.  
    - Model: `llama-3.1-70b-versatile`.  
    - Input/Output: Similar pattern.  
    - Edge Cases: Model availability, network issues.

  - **Groq Agent**  
    - Same role and configuration pattern for Groq.

  - **Output Parser2**  
    - Parses Groq Agent output.

---

#### 1.4 Response Merging

- **Overview:**  
Combines the AI responses from the executed agent(s) into a single dataset for unified processing.

- **Nodes Involved:**  
  - Merge Results

- **Node Details:**  
  - **Merge Results**  
    - Type: `n8n-nodes-base.merge`  
    - Role: Combines inputs from all three AI agents into one output array.  
    - Mode: Combine (merges all inputs, preserving all data).  
    - Inputs: Outputs of OpenAI Agent, Anthropic Agent, Groq Agent.  
    - Output: Array of AI responses for metric calculation.  
    - Edge Cases: Missing input if one AI agent is not triggered, resulting in incomplete merge.

---

#### 1.5 Performance Metrics Calculation

- **Overview:**  
Calculates performance metrics such as processing time, quality scores, cost efficiency, and composite performance scores to assess AI response quality and efficiency.

- **Nodes Involved:**  
  - Calculate Performance Metrics

- **Node Details:**  
  - **Calculate Performance Metrics**  
    - Type: `n8n-nodes-base.code` (JavaScript)  
    - Role:  
      - Calculates processing time from initial timestamp to current time.  
      - Extracts actual quality score from AI response or falls back to expected quality.  
      - Computes cost efficiency (quality divided by estimated cost).  
      - Calculates a weighted performance score combining quality and latency.  
      - Outputs enriched data and performance summary.  
    - Input: Merged AI responses and original routing decision.  
    - Output: Comprehensive performance and enriched data object.  
    - Edge Cases: Missing quality score in AI response, timestamp parsing errors.

---

#### 1.6 Webhook Response

- **Overview:**  
Sends the final combined AI enrichment and performance metrics back as the HTTP response to the original webhook caller.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**  
  - **Respond to Webhook**  
    - Type: `n8n-nodes-base.respondToWebhook`  
    - Role: Returns HTTP 200 response with JSON content including all workflow outputs.  
    - Headers: Sets `Content-Type: application/json`.  
    - Input: Output from performance metrics calculation node.  
    - Edge Cases: Network issues preventing response delivery.

---

#### 1.7 Documentation & Setup Notes

- **Overview:**  
Provides embedded documentation and setup instructions as sticky notes within the workflow for user guidance.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**  
  - **Sticky Note**  
    - Content: Overview of workflow purpose, how it processes AI agents in parallel, workflow steps.  
    - Role: Informational only.

  - **Sticky Note1**  
    - Content: Setup instructions including webhook activation, API keys, model configuration, routing logic tuning, prerequisites, and benefits.  
    - Role: Informational only.

---

### 3. Summary Table

| Node Name               | Node Type                                  | Functional Role                             | Input Node(s)              | Output Node(s)              | Sticky Note                                                                                      |
|-------------------------|--------------------------------------------|---------------------------------------------|----------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Webhook                 | n8n-nodes-base.webhook                      | Receives incoming HTTP POST requests        | —                          | Extract Input Parameters     | Setup instructions note covers this node                                                      |
| Extract Input Parameters | n8n-nodes-base.set                          | Extracts input data & parameters             | Webhook                    | Dynamic LLM Router           | Setup instructions note covers this node                                                      |
| Dynamic LLM Router       | n8n-nodes-base.code                         | Determines AI provider/model routing         | Extract Input Parameters   | Route to Provider, Merge Results | Setup instructions note covers this node                                                  |
| Route to Provider        | n8n-nodes-base.switch                       | Routes flow to selected AI provider agent    | Dynamic LLM Router         | OpenAI Agent, Anthropic Agent, Groq Agent | Setup instructions note covers this node                                             |
| OpenAI Model             | @n8n/n8n-nodes-langchain.lmChatOpenAi      | Connects to OpenAI API                        | OpenAI Agent               | OpenAI Agent                | Setup instructions note covers this node                                                      |
| OpenAI Agent             | @n8n/n8n-nodes-langchain.agent              | Runs OpenAI prompt with structured output    | Route to Provider          | Merge Results               | Setup instructions note covers this node                                                      |
| Output Parser1           | @n8n/n8n-nodes-langchain.outputParserStructured | Parses OpenAI agent output to JSON           | OpenAI Agent               | —                          | Setup instructions note covers this node                                                      |
| Anthropic Model          | @n8n/n8n-nodes-langchain.lmChatAnthropic   | Connects to Anthropic API                     | Anthropic Agent            | Anthropic Agent             | Setup instructions note covers this node                                                      |
| Anthropic Agent          | @n8n/n8n-nodes-langchain.agent              | Runs Anthropic prompt with structured output | Route to Provider          | Merge Results               | Setup instructions note covers this node                                                      |
| Output Parser            | @n8n/n8n-nodes-langchain.outputParserStructured | Parses Anthropic agent output to JSON        | Anthropic Agent            | —                          | Setup instructions note covers this node                                                      |
| Groq Model               | @n8n/n8n-nodes-langchain.lmChatGroq         | Connects to Groq API                          | Groq Agent                 | Groq Agent                  | Setup instructions note covers this node                                                      |
| Groq Agent               | @n8n/n8n-nodes-langchain.agent              | Runs Groq prompt with structured output      | Route to Provider          | Merge Results               | Setup instructions note covers this node                                                      |
| Output Parser2           | @n8n/n8n-nodes-langchain.outputParserStructured | Parses Groq agent output to JSON              | Groq Agent                 | —                          | Setup instructions note covers this node                                                      |
| Merge Results            | n8n-nodes-base.merge                        | Combines AI agents’ outputs                   | OpenAI Agent, Anthropic Agent, Groq Agent | Calculate Performance Metrics | Setup instructions note covers this node                                               |
| Calculate Performance Metrics | n8n-nodes-base.code                      | Computes time, quality, cost, and performance | Merge Results              | Respond to Webhook          | Setup instructions note covers this node                                                      |
| Respond to Webhook       | n8n-nodes-base.respondToWebhook             | Sends HTTP response with combined results    | Calculate Performance Metrics | —                          | Setup instructions note covers this node                                                      |
| Sticky Note              | n8n-nodes-base.stickyNote                   | Provides workflow introduction and overview  | —                          | —                           | Contains comprehensive workflow purpose and steps description                                |
| Sticky Note1             | n8n-nodes-base.stickyNote                   | Provides setup instructions and prerequisites | —                          | —                           | Contains setup, configuration, and benefits information                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: `Webhook`  
   - Configure path as `ai-pipeline`  
   - HTTP method: POST  
   - Response mode: `Response Node` (enables responding downstream)  
   - Enable authentication as needed.

2. **Create Set Node: "Extract Input Parameters"**  
   - Type: `Set`  
   - Assign variables:  
     - `input_data` = `={{ $json.body.data }}`  
     - `task_type` = `={{ $json.body.task_type || 'general' }}`  
     - `priority` = `={{ $json.body.priority || 'balanced' }}`  
   - Connect Webhook output to this node.

3. **Create Code Node: "Dynamic LLM Router"**  
   - Type: `Code` (JavaScript)  
   - Paste routing logic:  
     - Calculate complexity score based on `input_data.length`.  
     - Use `priority` to select provider and model.  
     - Output includes routing decision with provider, model, estimated cost, expected quality, timestamp.  
   - Connect "Extract Input Parameters" → "Dynamic LLM Router".

4. **Create Switch Node: "Route to Provider"**  
   - Type: `Switch`  
   - Condition: Check `{{$json.routing_decision.provider}}` equals `"openai"`, `"anthropic"`, or `"groq"`.  
   - Rename outputs accordingly for clarity.  
   - Connect "Dynamic LLM Router" → "Route to Provider".

5. **Create AI Model Nodes:**  
   - **OpenAI Model**  
     - Type: `LM Chat OpenAI`  
     - Credentials: Provide OpenAI API key.  
     - Connect output to "OpenAI Agent" node's AI language model input.

   - **Anthropic Model**  
     - Type: `LM Chat Anthropic`  
     - Credentials: Provide Anthropic API key.  
     - Model: `claude-3-5-sonnet-20241022`.  
     - Connect output to "Anthropic Agent" node's AI language model input.

   - **Groq Model**  
     - Type: `LM Chat Groq`  
     - Model: `llama-3.1-70b-versatile`.  
     - Connect output to "Groq Agent" node's AI language model input.

6. **Create AI Agent Nodes:**  
   For each provider (OpenAI, Anthropic, Groq):  
   - Type: `LangChain Agent`  
   - Prompt template:  
     ```
     You are a data enrichment AI assistant. Analyze and enrich the following data with insights, structure it properly, and provide actionable recommendations.

     Task Type: {{$json.task_type}}

     Input Data:
     {{$json.input_data}}

     Provide:
     1. Structured analysis
     2. Key insights
     3. Data enrichment
     4. Actionable recommendations
     5. Quality score (1-10)

     Format as JSON.
     ```  
   - System message:  
     ```
     You are processing data with {{$json.routing_decision.provider}} ({{$json.routing_decision.model}}). Quality level: {{$json.routing_decision.expected_quality}}/10.
     ```  
   - Enable output parser.  
   - Connect corresponding "Route to Provider" output to each agent node.

7. **Create Output Parser Nodes:**  
   For each AI Agent, create an `Output Parser Structured` node. Connect the AI Agent output to its parser. This ensures the AI JSON output is parsed into n8n data.

8. **Create Merge Node: "Merge Results"**  
   - Type: `Merge`  
   - Mode: `Combine`  
   - Connect outputs from all three AI Agents to this merge node.

9. **Create Code Node: "Calculate Performance Metrics"**  
   - Type: `Code` (JavaScript)  
   - Logic:  
     - Calculate processing time using timestamps.  
     - Extract actual quality score or fallback to expected.  
     - Compute cost efficiency and performance score.  
     - Aggregate all data including input, routing decision, enriched data, metrics, timestamp.  
   - Connect "Merge Results" → "Calculate Performance Metrics".

10. **Create Respond to Webhook Node**  
    - Type: `Respond to Webhook`  
    - Set response code: 200  
    - Response header: `Content-Type: application/json`  
    - Respond with all incoming items.  
    - Connect "Calculate Performance Metrics" → "Respond to Webhook".

11. **Add Sticky Notes for Documentation (Optional)**  
    - Add notes describing workflow overview and setup instructions for future reference.

12. **Credential Setup**  
    - Add API keys/credentials for:  
      - OpenAI (OAuth2 or API key)  
      - Anthropic (API key)  
      - Groq (API key or equivalent)  
    - Ensure these credentials are linked correctly in model nodes.

13. **Testing**  
    - Use an HTTP client to POST JSON data to webhook URL `/ai-pipeline`.  
    - Example POST body:  
      ```json
      {
        "data": "Sample text data to analyze",
        "task_type": "general",
        "priority": "balanced"
      }
      ```  
    - Verify responses contain enriched structured data and performance metrics.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                          | Context or Link                                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This workflow connects to OpenAI, Anthropic, and Groq AI models, running requests in parallel and comparing output quality, cost, and latency.        | Workflow Introduction (Sticky Note)                                                                                      |
| Setup requires API keys for all providers and enabling the webhook with appropriate authentication.                                                   | Setup Instructions (Sticky Note)                                                                                        |
| Routing logic can be customized further to include additional providers like Gemini or Azure OpenAI.                                                  | Setup Instructions (Sticky Note)                                                                                        |
| The workflow is ideal for testing and benchmarking multiple AI models in real time for operational optimization.                                      | Workflow Introduction (Sticky Note)                                                                                      |
| For performance scoring, the formula balances quality and processing speed with cost considerations.                                                  | Node: Calculate Performance Metrics                                                                                      |
| Official n8n docs for LangChain nodes and webhook configuration are recommended for troubleshooting and extension.                                   | https://docs.n8n.io/nodes/                                                                                               |

---

*Disclaimer: The provided text is exclusively derived from an automated n8n workflow. All data processed is legal and publicly available.*
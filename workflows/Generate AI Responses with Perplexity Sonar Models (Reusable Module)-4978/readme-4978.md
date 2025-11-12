Generate AI Responses with Perplexity Sonar Models (Reusable Module)

https://n8nworkflows.xyz/workflows/generate-ai-responses-with-perplexity-sonar-models--reusable-module--4978


# Generate AI Responses with Perplexity Sonar Models (Reusable Module)

### 1. Workflow Overview

This workflow acts as a reusable module for generating AI responses using the Perplexity API's chat completion endpoint with the `sonar` or `sonar-pro` models. It is designed to be invoked by other workflows, accepting dynamic system and user prompts as inputs. The main logical blocks include:

- **1.1 Input Reception:** Accepts external inputs (`SystemPrompt` and `UserPrompt`) when triggered by another workflow.
- **1.2 Parameter Setup:** Defines the model to be used for the API call, defaulting to `sonar`.
- **1.3 API Request:** Sends a POST request to Perplexity’s chat completions API with the structured messages and authentication.
- **1.4 Documentation Notes:** Several sticky notes clarify usage, supported models, input expectations, and API references.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives external inputs when triggered via an “Execute Workflow” node from a parent workflow. It expects two parameters: `SystemPrompt` and `UserPrompt`.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point; receives inputs from external workflow executions.  
    - Configuration: Configured to accept two named inputs: `SystemPrompt` and `UserPrompt`.  
    - Key Expressions: Inputs are accessed via `item.json.SystemPrompt` and `item.json.UserPrompt`.  
    - Input Connections: None (trigger node).  
    - Output Connections: Passes data to the “Parameters” node.  
    - Edge Cases: Missing or malformed inputs may cause the workflow downstream to fail or behave unexpectedly. No explicit validation is present.  
    - Version: 1.1  

---

#### 2.2 Parameter Setup

- **Overview:**  
  This block sets up the model parameter, defaulting to the `sonar` model, which can be changed to `sonar-pro` if desired.

- **Nodes Involved:**  
  - Parameters  
  - Sticky Note1 (documentation on models)

- **Node Details:**

  - **Parameters**  
    - Type: Set  
    - Role: Defines static or dynamic parameters for the workflow, here specifically the `model` string.  
    - Configuration: Sets `"model"` key to `"sonar"` (default).  
    - Key Expressions: Static value `"sonar"`.  
    - Input Connections: Receives data from “When Executed by Another Workflow”.  
    - Output Connections: Outputs to the “Perplexity API Request” node.  
    - Edge Cases: If the model value is changed to an unsupported string, the API request will likely fail or return errors.  
    - Version: 3.4  

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Provides reference on supported models and links to related documentation.  
    - Content Highlights: Supports `sonar` and `sonar-pro`; users can change model in “Parameters” node.  
    - No connections (informational only).  

---

#### 2.3 API Request

- **Overview:**  
  This block constructs and sends the HTTP POST request to the Perplexity chat completions API, passing the model and messages array with system and user prompts. It uses header-based authentication.

- **Nodes Involved:**  
  - Perplexity API Request  
  - Sticky Note (Perplexity API Integration)  
  - Sticky Note2 (Input Parameters explanation)

- **Node Details:**

  - **Perplexity API Request**  
    - Type: HTTP Request  
    - Role: Executes the call to Perplexity’s API endpoint.  
    - Configuration:  
      - URL: `https://api.perplexity.ai/chat/completions`  
      - Method: POST  
      - Body: JSON, dynamically constructed with expressions referencing previous nodes:  
        ```json
        {
          "model": "{{$('Parameters').item.json.model}}",
          "messages": [
            {
              "role": "system",
              "content": "{{ $('When Executed by Another Workflow').item.json.SystemPrompt }}"
            },
            {
              "role": "user",
              "content": "{{ $('When Executed by Another Workflow').item.json.UserPrompt }}"
            }
          ]
        }
        ```  
      - Authentication: Generic HTTP Header Auth (expects credentials to be configured in n8n credentials manager).  
      - Send Body: True, specifying JSON.  
    - Input Connections: From “Parameters” node.  
    - Output Connections: None (terminal node).  
    - Edge Cases:  
      - Authentication failure if credentials are missing or invalid.  
      - API errors if inputs are missing or invalid, or model is unsupported.  
      - Timeout or network issues can cause request failure.  
      - Expression errors if referenced nodes or parameters are missing.  
    - Version: 4.2  

  - **Sticky Note** (initial node)  
    - Type: Sticky Note  
    - Role: Describes the overall API integration, trigger method, inputs, authentication, and links to API docs.  
    - Content includes link: https://docs.perplexity.ai/api-reference/async-chat-completions-post  

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Explains expected input parameters, their roles, and usage notes about passing these inputs from parent workflows.  
    - Content emphasizes two inputs: `SystemPrompt` (behavior/tone), `UserPrompt` (main instruction).  

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                        | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                          |
|-----------------------------|----------------------------------|--------------------------------------|-----------------------------|---------------------------|----------------------------------------------------------------------------------------------------|
| Sticky Note                 | Sticky Note                      | Describes Perplexity API integration | None                        | None                      | ## 🔍 Perplexity API Integration...[API Docs](https://docs.perplexity.ai/api-reference/async-chat-completions-post) |
| When Executed by Another Workflow | Execute Workflow Trigger          | Entry point; accepts external inputs  | None                        | Parameters                |                                                                                                    |
| Parameters                  | Set                              | Sets model parameter (default: sonar) | When Executed by Another Workflow | Perplexity API Request    | ## 🧠 Supported Models...[Model cards](https://docs.perplexity.ai/models/model-cards)               |
| Perplexity API Request      | HTTP Request                    | Sends chat completion request to API | Parameters                  | None                      |                                                                                                    |
| Sticky Note1                | Sticky Note                      | Lists supported models and usage     | None                        | None                      | ## 🧠 Supported Models...[Model cards](https://docs.perplexity.ai/models/model-cards)               |
| Sticky Note2                | Sticky Note                      | Explains input parameters and usage  | None                        | None                      | ## 🧩 Input Parameters...                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
   - Configure it to accept two inputs: `SystemPrompt` (string) and `UserPrompt` (string).

2. **Create Parameters Node:**  
   - Add a **Set** node named `Parameters`.  
   - Connect `When Executed by Another Workflow` → `Parameters`.  
   - In `Parameters`, create a string field `model` with value `"sonar"` (default).  
   - This node can be modified to change the model to `"sonar-pro"` if needed.

3. **Create HTTP Request Node:**  
   - Add an **HTTP Request** node named `Perplexity API Request`.  
   - Connect `Parameters` → `Perplexity API Request`.  
   - Set:  
     - HTTP Method: POST  
     - URL: `https://api.perplexity.ai/chat/completions`  
     - Authentication: Generic Credential Type → HTTP Header Auth (configure credentials separately in n8n Credentials with API key or token).  
     - Send Body: True, specify body type as JSON.  
     - Body Content (JSON):  
       ```json
       {
         "model": "={{$json[\"model\"]}}",
         "messages": [
           {
             "role": "system",
             "content": "={{$parentItem(\"When Executed by Another Workflow\").json.SystemPrompt}}"
           },
           {
             "role": "user",
             "content": "={{$parentItem(\"When Executed by Another Workflow\").json.UserPrompt}}"
           }
         ]
       }
       ```  
       *(In n8n, expressions may differ slightly; ensure references to previous nodes are correct.)*

4. **Add Documentation Sticky Notes:**  
   - Add sticky notes to explain:  
     - Overall API integration and usage.  
     - Supported models and how to switch them.  
     - Expected input parameters and how to provide them when calling this workflow.

5. **Configure Credentials:**  
   - In n8n, create a new credential of type “HTTP Header Auth” with the necessary API key or token for Perplexity API.  
   - Attach these credentials to the `Perplexity API Request` node.

6. **Save and Test:**  
   - Save the workflow.  
   - Test by triggering via an “Execute Workflow” node from a parent workflow, passing valid `SystemPrompt` and `UserPrompt` strings.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This workflow is designed as a reusable module to be triggered from other workflows by passing dynamic system and user prompts. | Usage instruction                                                                                                       |
| Supported models are `sonar` and `sonar-pro`. Change model in Parameters node to switch.                                  | https://docs.perplexity.ai/models/model-cards                                                                           |
| API documentation for Perplexity chat completions endpoint.                                                               | https://docs.perplexity.ai/api-reference/async-chat-completions-post                                                    |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.
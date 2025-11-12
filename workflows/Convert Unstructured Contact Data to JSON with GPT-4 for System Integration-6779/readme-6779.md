Convert Unstructured Contact Data to JSON with GPT-4 for System Integration

https://n8nworkflows.xyz/workflows/convert-unstructured-contact-data-to-json-with-gpt-4-for-system-integration-6779


# Convert Unstructured Contact Data to JSON with GPT-4 for System Integration

### 1. Workflow Overview

This workflow is designed to convert unstructured contact data (e.g., details from emails or messages) into structured JSON format suitable for system integration such as CRM or ERP platforms. It leverages AI (GPT-4 via n8n’s LangChain integration) to extract specific contact fields strictly from the input text without assumptions or inference.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Handles incoming HTTP POST requests with unstructured contact text payload.
- **1.2 AI Processing:** Uses an AI agent configured with GPT-4 to extract structured contact information according to strict extraction rules.
- **1.3 JSON Validation & Parsing:** Cleans and validates the AI output to ensure it is well-formed JSON, throwing errors otherwise.
- **1.4 Response Delivery:** Sends the structured JSON data back as the HTTP response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
This block receives unstructured contact data from external sources via an HTTP POST webhook, serving as the entry point.

- **Nodes Involved:**  
  - Webhook  
  - Sticky Note1 (documentation)

- **Node Details:**

  - **Webhook**  
    - *Type & Role:* HTTP Webhook node; entry point listening for POST requests.  
    - *Configuration:*  
      - Path: `/contact_data_converter`  
      - Method: POST  
      - Response Mode: responseNode (response sent from downstream node)  
    - *Expressions/Variables:* Accesses input text under `{{ $json.body.prompt }}`  
    - *Inputs:* External HTTP POST requests  
    - *Outputs:* Passes received data to AI Agent node  
    - *Edge Cases:*  
      - Missing or malformed POST body  
      - Incorrect content type  
      - Unauthorized or unexpected clients (no auth configured)  
    - *Sticky Note1 content:*  
      > This webhook acts as the entry point for the workflow.  
      > It accepts an HTTP POST request containing unstructured contact information (e.g., the body of an email) in the request body under the key **"prompt"**

#### 1.2 AI Processing

- **Overview:**  
This block uses an AI agent powered by GPT-4 to strictly extract specific contact fields from the unstructured text, returning a JSON string.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Sticky Note2 (documentation)

- **Node Details:**

  - **OpenAI Chat Model**  
    - *Type & Role:* Language model node using OpenAI GPT-4.  
    - *Configuration:*  
      - Model: `gpt-4.1-nano`  
      - No additional options configured  
    - *Credentials:* Requires valid OpenAI API credentials to function  
    - *Inputs:* Receives instructions and prompt from AI Agent  
    - *Outputs:* Provides AI-generated text to AI Agent  
    - *Edge Cases:*  
      - API authentication errors  
      - Rate limits or quota exceeded  
      - Timeout or connectivity problems

  - **AI Agent**  
    - *Type & Role:* LangChain AI Agent node orchestrating prompt and AI call.  
    - *Configuration:*  
      - Text prompt includes strict extraction rules and a JSON output template with exact keys: `company_name`, `tel`, `fax`, `email`, `first_name`, `last_name`, `address`, `zipcode`, `city`, `country`, `website`.  
      - Input text injected as `{{ $json.body.prompt }}` from webhook payload.  
      - Output expected as a JSON string embedded in the AI response.  
    - *Inputs:* Receives webhook data and OpenAI Chat Model output  
    - *Outputs:* Passes AI-generated JSON string to Code node  
    - *Edge Cases:*  
      - AI output not adhering to the JSON format  
      - Missing expected fields (handled by empty strings)  
      - Unexpected prompt format or input data errors  
    - *Sticky Note2 content:*  
      > This node uses an AI agent to extract structured contact information from the unstructured input text.  
      > The input text is passed from the previous Webhook node using: {{ $json.body.prompt }}  
      > The AI prompt can be customized to fit your specific use case or formatting needs.  
      > You can adjust the instructions in the prompt to tailor the output fields, structure, or style of the resulting JSON.

#### 1.3 JSON Validation & Parsing

- **Overview:**  
This block cleans the AI response by stripping Markdown code block formatting and parses it into a valid JSON object. It ensures the output is ready for downstream consumption or system integration.

- **Nodes Involved:**  
  - Code  
  - Sticky Note3 (documentation)

- **Node Details:**

  - **Code**  
    - *Type & Role:* JavaScript code node for data transformation and validation.  
    - *Configuration:*  
      - Strips leading and trailing Markdown code block markers (```json ... ```) from AI output.  
      - Attempts to parse the cleaned string as JSON.  
      - Throws an error if JSON parsing fails, aiding debugging.  
      - Returns JSON object under `output` key for clarity and further use.  
    - *Inputs:* Receives AI Agent node’s output  
    - *Outputs:* Valid JSON object to Respond to Webhook node  
    - *Edge Cases:*  
      - AI output missing code fences or malformed strings  
      - JSON parse exceptions  
      - Unexpected output structure from AI Agent  
    - *Sticky Note3 content:*  
      > This node processes the raw output from the AI Agent and ensures it’s a valid JSON object.  
      > It removes any Markdown-style code block formatting (e.g., json ... ) from the AI response.  
      > It then parses the cleaned string into a proper JSON object.  
      > If the AI output is not valid JSON, it will throw an error to help with debugging.  
      > The parsed and structured data is returned under output, ready for use in downstream nodes or ERP system integration.

#### 1.4 Response Delivery

- **Overview:**  
Sends the final validated JSON contact data back as the HTTP response to the original webhook caller.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**

  - **Respond to Webhook**  
    - *Type & Role:* Sends HTTP response back to client.  
    - *Configuration:* No special options; outputs data received from Code node.  
    - *Inputs:* Receives validated JSON from Code node  
    - *Outputs:* HTTP response with JSON body to the webhook caller  
    - *Edge Cases:*  
      - Response failure if preceding node errors out  
      - Timeout if workflow execution delays beyond client expectations

---

### 3. Summary Table

| Node Name          | Node Type                     | Functional Role                                    | Input Node(s)       | Output Node(s)       | Sticky Note                                                   |
|--------------------|-------------------------------|---------------------------------------------------|---------------------|----------------------|---------------------------------------------------------------|
| Webhook            | n8n-nodes-base.webhook         | Entry point receiving unstructured contact data  | -                   | AI Agent             | This webhook acts as the entry point for the workflow. It accepts an HTTP POST request containing unstructured contact information (e.g., the body of an email) in the request body under the key **"prompt"** |
| AI Agent           | @n8n/n8n-nodes-langchain.agent| Extracts structured contact info using GPT-4     | Webhook, OpenAI Chat Model | Code              | This node uses an AI agent to extract structured contact information from the unstructured input text. The input text is passed from the previous Webhook node using: {{ $json.body.prompt }} The AI prompt can be customized to fit your specific use case or formatting needs. You can adjust the instructions in the prompt to tailor the output fields, structure, or style of the resulting JSON. |
| OpenAI Chat Model  | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4 AI language model service          | AI Agent            | AI Agent             |                                                               |
| Code               | n8n-nodes-base.code            | Cleans and parses AI output JSON                   | AI Agent            | Respond to Webhook   | This node processes the raw output from the AI Agent and ensures it’s a valid JSON object. It removes any Markdown-style code block formatting (e.g., json ... ) from the AI response. It then parses the cleaned string into a proper JSON object. If the AI output is not valid JSON, it will throw an error to help with debugging. The parsed and structured data is returned under output, ready for use in downstream nodes or ERP system integration. |
| Respond to Webhook | n8n-nodes-base.respondToWebhook | Sends structured JSON back as HTTP response       | Code                | -                    |                                                               |
| Sticky Note        | n8n-nodes-base.stickyNote      | Workflow overview documentation                    | -                   | -                    | ## Contact Data Converter – Unstructured Text to JSON for System Integration This n8n workflow converts unstructured contact information—such as customer details from emails or messages—into structured JSON data using an AI agent. It extracts key fields like Company Name, Address, Phone, Email, First Name, Last Name, etc. The structured output is ideal for seamless integration into CRM / ERP systems like Dolibarr, or any other system that accepts JSON input. Whether you're automating CRM data entry or syncing contacts into your ERP, this template saves time and reduces manual errors. |
| Sticky Note1       | n8n-nodes-base.stickyNote      | Documentation for Webhook node                     | -                   | -                    | This webhook acts as the entry point for the workflow. It accepts an HTTP POST request containing unstructured contact information (e.g., the body of an email) in the request body under the key **"prompt"** |
| Sticky Note2       | n8n-nodes-base.stickyNote      | Documentation for AI Agent node                    | -                   | -                    | This node uses an AI agent to extract structured contact information from the unstructured input text. The input text is passed from the previous Webhook node using: {{ $json.body.prompt }} The AI prompt can be customized to fit your specific use case or formatting needs. You can adjust the instructions in the prompt to tailor the output fields, structure, or style of the resulting JSON. |
| Sticky Note3       | n8n-nodes-base.stickyNote      | Documentation for Code node                        | -                   | -                    | This node processes the raw output from the AI Agent and ensures it’s a valid JSON object. It removes any Markdown-style code block formatting (e.g., json ... ) from the AI response. It then parses the cleaned string into a proper JSON object. If the AI output is not valid JSON, it will throw an error to help with debugging. The parsed and structured data is returned under output, ready for use in downstream nodes or ERP system integration. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: `Webhook`  
   - Set HTTP Method: `POST`  
   - Set Path: `contact_data_converter`  
   - Set Response Mode: `responseNode` to allow downstream node to send response  
   - Leave other options default  
   - This node accepts unstructured contact text under `prompt` in the POST body.

2. **Create OpenAI Chat Model Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Set Model to `gpt-4.1-nano` (ensure your OpenAI API key has access)  
   - No special options needed  
   - Add OpenAI API Credentials (create or select existing with valid API key)  
   - This node provides GPT-4 AI completions used by AI Agent.

3. **Create AI Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Set `text` parameter to the extraction prompt below (replace `{{ $json.body.prompt }}` with incoming webhook data):  
     ```
     Extract the following fields from the input text. Follow these rules strictly:

     **EXTRACTION RULES:**
     1. Only extract information that is explicitly stated in the text
     2. Do not infer, guess, or make assumptions
     3. If a field is missing, ambiguous, or uncertain, leave it as an empty string ""
     4. Extract information exactly as written (preserve original formatting for addresses, phone numbers, etc.)
     5. For names: only extract if clearly identified as a person's name (not just any name in the text)
     6. For company names: look for business names, organization names, or company identifiers

     **FIELD DEFINITIONS:**
     - **company_name**: Business or organization name
     - **tel**: Phone/telephone number (any format)
     - **fax**: Fax number (any format)
     - **email**: Email address
     - **first_name**: Person's first/given name
     - **last_name**: Person's last/family name
     - **address**: Street address or mailing address
     - **zipcode**: Postal/ZIP code
     - **city**: City name
     - **country**: Country name
     - **website**: Website URL or domain

     **OUTPUT FORMAT:**
     Return only valid JSON with exactly these keys (no additional text or explanation):

     {
       "company_name": "",
       "tel": "",
       "fax": "",
       "email": "",
       "first_name": "",
       "last_name": "",
       "address": "",
       "zipcode": "",
       "city": "",
       "country": "",
       "website": ""
     }

     **INPUT TEXT:**
     {{ $json.body.prompt }}
     ```
   - Set `promptType` to `define`  
   - Link AI Agent node’s `ai_languageModel` input to the OpenAI Chat Model node  
   - Connect Webhook node’s output to this AI Agent node.

4. **Create Code Node**  
   - Type: `Code` (JavaScript)  
   - Paste the following snippet in the code editor:
     ```javascript
     return items.map(item => {
       let jsonStr = item.json.output;
       jsonStr = jsonStr.replace(/^```json\n?/, '').replace(/\n?```$/, '');
       
       try {
         const parsed = JSON.parse(jsonStr);
         return {
           json: {
             output: parsed
           }
         };
       } catch (error) {
         throw new Error('Invalid JSON string: ' + error.message);
       }
     });
     ```
   - Connect AI Agent node output to this Code node.

5. **Create Respond to Webhook Node**  
   - Type: `Respond to Webhook`  
   - No special parameters needed  
   - Connect Code node output to this node  
   - This node sends back the final structured JSON as HTTP response.

6. **Link Nodes in Order:**  
   Webhook → AI Agent → Code → Respond to Webhook  
   OpenAI Chat Model → AI Agent (via ai_languageModel input)

7. **Set Up Credentials:**  
   - Add OpenAI API credentials with valid API key for GPT-4 access.  
   - No authentication configured on webhook (can be added as enhancement).

8. **Testing:**  
   - Send POST requests to your n8n webhook URL `/contact_data_converter` with JSON body:  
     ```json
     {
       "prompt": "Unstructured contact info text here..."
     }
     ```  
   - Confirm that the response is a JSON object with the extracted contact fields.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                               | Context or Link                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| This workflow converts unstructured contact information—such as customer details from emails or messages—into structured JSON data using an AI agent. It extracts key fields like Company Name, Address, Phone, Email, First Name, Last Name, etc. | Workflow overview sticky note                             |
| The structured output is ideal for seamless integration into CRM / ERP systems like Dolibarr, or any other system that accepts JSON input.                                                                                                 | Workflow overview sticky note                             |
| Whether you're automating CRM data entry or syncing contacts into your ERP, this template saves time and reduces manual errors.                                                                                                            | Workflow overview sticky note                             |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.
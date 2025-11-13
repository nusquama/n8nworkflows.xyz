Financial Data Q&A Chatbot with Google Finance, SerpAPI, and OpenAI

https://n8nworkflows.xyz/workflows/financial-data-q-a-chatbot-with-google-finance--serpapi--and-openai-7743


# Financial Data Q&A Chatbot with Google Finance, SerpAPI, and OpenAI

### 1. Workflow Overview

This workflow implements a **Financial Data Q&A Chatbot** that leverages **live financial market data** from Google Finance via SerpApi and answers user questions using OpenAI’s GPT-5 model. The chatbot processes user queries, fetches real-time market data, converts data into a text format appropriate for AI consumption, and generates context-aware answers. It also maintains session memory for conversational context.

**Target Use Cases:**
- Interactive chatbot for financial data inquiry (e.g., stock market indices, financial instruments).
- Real-time integration with financial market data APIs.
- Contextual question answering with memory of prior conversation.

**Logical Blocks:**

- **1.1 Setup and Configuration**: Contains sticky notes with instructions for API key setup and contact info.
- **1.2 Data Retrieval (SerpApi Finance Search)**: Fetches live financial data from Google Finance via SerpApi HTTP request.
- **1.3 Data Preparation (Turn Objects to Text)**: Converts raw JSON market data into string format for AI processing.
- **1.4 Conversational AI Processing**: Uses Langchain nodes with OpenAI GPT-5 and memory to generate answers based on user input and fetched data.
  - 1.4.1 Memory Management
  - 1.4.2 Chat Agent Processing
  - 1.4.3 Language Model Execution

---

### 2. Block-by-Block Analysis

#### 1.1 Setup and Configuration

**Overview:**  
This block provides detailed setup instructions and contact information for users to configure the necessary API keys and billing accounts for SerpApi and OpenAI within the workflow.

**Nodes Involved:**  
- Sticky Note60  
- Sticky Note17  
- Sticky Note31  
- Sticky Note32

**Node Details:**

- **Sticky Note60**  
  - *Type:* Sticky Note  
  - *Role:* High-level workflow description highlighting financial data chat capabilities.  
  - *Key Content:* Explains connection to live financial data and OpenAI for answering questions.  
  - *Connections:* None  
  - *Failure Modes:* None  

- **Sticky Note17**  
  - *Type:* Sticky Note  
  - *Role:* Step-by-step setup instructions for SerpApi and OpenAI API keys, billing info, and contact details for customization support.  
  - *Content Highlights:*  
    - How to create SerpApi account and update API key in HTTP Request node.  
    - How to set up OpenAI API key and billing.  
    - Contact email and LinkedIn link for support.  
  - *Connections:* None  
  - *Failure Modes:* None  

- **Sticky Note31**  
  - *Type:* Sticky Note  
  - *Role:* Concise OpenAI API key setup instructions.  
  - *Connections:* None  
  - *Failure Modes:* None  

- **Sticky Note32**  
  - *Type:* Sticky Note  
  - *Role:* Concise SerpApi API key setup instructions.  
  - *Connections:* None  
  - *Failure Modes:* None  

---

#### 1.2 Data Retrieval (SerpApi Finance Search)

**Overview:**  
This block performs an HTTP request to SerpApi’s Google Finance engine to fetch current market data for a specified ticker (default: S&P 500 index `^GSPC`). The API key must be configured here.

**Nodes Involved:**  
- SerpAPI Finance Search1

**Node Details:**

- **SerpAPI Finance Search1**  
  - *Type:* HTTP Request  
  - *Role:* Fetch live financial market data JSON from SerpApi Google Finance endpoint.  
  - *Configuration:*  
    - URL template: `https://serpapi.com/search.json?engine=google_finance&q=^GSPC&api_key=yourapikey`  
    - Method: GET (default)  
    - API Key placeholder `yourapikey` must be replaced by user’s SerpApi key.  
  - *Input:* None (triggered externally or on-demand)  
  - *Output:* JSON market data passed downstream  
  - *Version:* 4.2  
  - *Potential Failures:*  
    - HTTP errors (4xx or 5xx) due to invalid API key or network issues.  
    - Timeout if SerpApi is slow to respond.  
    - Rate limiting by SerpApi.  
    - Malformed or unexpected JSON responses if API changes.  

---

#### 1.3 Data Preparation (Turn Objects to Text)

**Overview:**  
Converts the raw JSON market data retrieved from SerpApi into a string format suitable for inclusion in AI prompt context.

**Nodes Involved:**  
- Turn Objects to Text1

**Node Details:**

- **Turn Objects to Text1**  
  - *Type:* Set Node  
  - *Role:* Assigns a new string field `data` containing the stringified form of the JSON field `markets` from the input data.  
  - *Configuration:* Expression used: `={{ $json.markets }}` converted to string.  
  - *Input:* JSON data from `SerpAPI Finance Search1`  
  - *Output:* JSON object with field `data` containing text representation of market data  
  - *Version:* 3.4  
  - *Potential Failures:*  
    - Expression errors if input JSON does not have `markets` property or if the field is undefined.  
    - Data truncation if the string is too large for downstream nodes.  

---

#### 1.4 Conversational AI Processing

##### 1.4.1 Memory Management

**Overview:**  
Manages conversational context memory using Langchain’s buffer window memory keyed by session ID, enabling the chatbot to maintain context across user interactions.

**Nodes Involved:**  
- Simple Memory1

**Node Details:**

- **Simple Memory1**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Stores and retrieves chat session history keyed by a custom session key extracted from input.  
  - *Configuration:*  
    - Session key expression: `={{ $('Sample Chatbot').item.json.sessionId }}` (uses external session ID from input data)  
    - Session ID type: custom key  
  - *Input:* Receives data from `Chat with Google Finance` node’s memory output  
  - *Output:* Provides memory context to the chat agent node  
  - *Version:* 1.3  
  - *Potential Failures:*  
    - Missing or invalid session ID causing memory lookup failure.  
    - Memory state persistence issues (if n8n instance restarts or memory node misconfigured).  

##### 1.4.2 Chat Agent Processing

**Overview:**  
Processes user question input along with market data text and memory context to generate an AI-driven answer using a Langchain agent configured with a system prompt.

**Nodes Involved:**  
- Chat with Google Finance

**Node Details:**

- **Chat with Google Finance**  
  - *Type:* Langchain Agent Node  
  - *Role:* Constructs the AI prompt by combining user question and market data, applies system instructions, and manages conversation turns.  
  - *Configuration:*  
    - Text parameter uses expression:  
      `=question: {{ $('Sample Chatbot').item.json.chatInput }} market data: {{ $json.data }}`  
      This concatenates the user’s chat input and the market data string.  
    - System message:  
      `"You are a helpful assistant. Take questions from the user and answer based on the market data."`  
    - Prompt type: define (custom prompt)  
  - *Input:* Receives prepared market data text and session info  
  - *Output:* Produces AI response text and memory updates  
  - *Version:* 2.2  
  - *Potential Failures:*  
    - Missing user input in `chatInput` field causes empty queries.  
    - Incorrect or incomplete market data in `data` field affects answer quality.  
    - Expression parsing errors if referenced nodes or fields are missing.  

##### 1.4.3 Language Model Execution

**Overview:**  
Executes the OpenAI GPT-5 language model to generate the chatbot’s textual answer based on the prompt created by the chat agent.

**Nodes Involved:**  
- OpenAI Chat Model5

**Node Details:**

- **OpenAI Chat Model5**  
  - *Type:* Langchain OpenAI Chat Model Node  
  - *Role:* Calls OpenAI GPT-5 model to generate responses for the chat agent.  
  - *Configuration:*  
    - Model selected: `gpt-5-nano` (a GPT-5 variant)  
    - No additional options configured  
    - Credentials: OpenAI API key via `openAiApi` credential named "OpenAi account 4"  
  - *Input:* Receives prompt from `Chat with Google Finance`  
  - *Output:* Returns AI-generated chat response  
  - *Version:* 1.2  
  - *Potential Failures:*  
    - API authentication errors if credentials invalid or expired.  
    - OpenAI quota exceeded or billing issues.  
    - Timeout or rate limits from OpenAI API.  
    - Model unavailability or version mismatch.  

---

### 3. Summary Table

| Node Name              | Node Type                           | Functional Role                     | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                                                                                                                                    |
|------------------------|-----------------------------------|-----------------------------------|------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note60          | Sticky Note                       | Workflow high-level description   | —                      | —                       | # 📊 Chat with Google Finance  This workflow connects a chatbot to **live financial data** from SerpApi and uses OpenAI to answer questions.                                                                                                     |
| Sticky Note17          | Sticky Note                       | Setup instructions and contact    | —                      | —                       | ## ⚙️ Setup Instructions  1️⃣ Add Your SerpApi Key and 2️⃣ Set Up OpenAI Connection with detailed steps; includes contact info for customization requests (robert@ynteractive.com).                                                                 |
| Sticky Note31          | Sticky Note                       | OpenAI setup instructions         | —                      | —                       | 2️⃣ Set Up OpenAI Connection: API key creation and billing instructions with links.                                                                                                                                                            |
| Sticky Note32          | Sticky Note                       | SerpApi setup instructions        | —                      | —                       | 1️⃣ Add Your SerpApi Key: Account creation and API key usage instructions with link.                                                                                                                                                           |
| SerpAPI Finance Search1| HTTP Request                     | Fetch live financial market data  | —                      | Turn Objects to Text1    | Replace `yourapikey` in URL with actual SerpApi API key.                                                                                                                                                                                       |
| Turn Objects to Text1  | Set                              | Convert JSON market data to text  | SerpAPI Finance Search1 | Chat with Google Finance |                                                                                                                                                                                                                                                |
| Chat with Google Finance| Langchain Agent                  | Combine user input & data, manage conversation | Turn Objects to Text1, Simple Memory1, OpenAI Chat Model5 | Simple Memory1, OpenAI Chat Model5  |                                                                                                                                                                                                                                                |
| Simple Memory1         | Langchain Memory Buffer Window   | Maintain session conversational memory | Chat with Google Finance | Chat with Google Finance | Session key expression depends on external `Sample Chatbot` input's `sessionId`.                                                                                                                                                                |
| OpenAI Chat Model5     | Langchain OpenAI Chat Model      | Execute GPT-5 model for responses | Chat with Google Finance | Chat with Google Finance | Credentials required: OpenAI API key named "OpenAi account 4". Model set to `gpt-5-nano`.                                                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Setup and Description**  
   - Add Sticky Note with content describing the workflow: live financial data chatbot using SerpApi and OpenAI.  
   - Add Sticky Notes with detailed setup instructions for:  
     - SerpApi API key creation and placement in HTTP request URL.  
     - OpenAI API key creation, billing, and credential setup.  
     - Contact details for customization support.

2. **Add HTTP Request Node for SerpApi Finance Search**  
   - Node Type: HTTP Request  
   - Name: `SerpAPI Finance Search1`  
   - Configure URL:  
     `https://serpapi.com/search.json?engine=google_finance&q=^GSPC&api_key=YOUR_API_KEY`  
     Replace `YOUR_API_KEY` with your SerpApi API key.  
   - Method: GET (default)  
   - No authentication required beyond the API key in URL.  

3. **Add Set Node to Convert JSON Data to Text**  
   - Node Type: Set  
   - Name: `Turn Objects to Text1`  
   - Add assignment:  
     - Variable: `data`  
     - Type: String  
     - Value: Expression `={{ $json.markets }}` (stringify the `markets` field from SerpApi response)  

4. **Add Langchain Agent Node for Chat Processing**  
   - Node Type: `@n8n/n8n-nodes-langchain.agent`  
   - Name: `Chat with Google Finance`  
   - Parameters:  
     - Text input: Expression  
       `=question: {{ $('Sample Chatbot').item.json.chatInput }} market data: {{ $json.data }}`  
       (This merges user question and prepared market data text)  
     - System message:  
       `You are a helpful assistant. Take questions from the user and answer based on the market data.`  
     - Prompt type: `define` (custom prompt)  

5. **Add Langchain Memory Buffer Window Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.memoryBufferWindow`  
   - Name: `Simple Memory1`  
   - Parameters:  
     - Session key: Expression `={{ $('Sample Chatbot').item.json.sessionId }}`  
     - Session ID type: `customKey`  
   - Purpose: store and retrieve conversation history per session  

6. **Add Langchain OpenAI Chat Model Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Name: `OpenAI Chat Model5`  
   - Parameters:  
     - Model: select `gpt-5-nano`  
     - Options: Default (none)  
   - Credentials: Create or select OpenAI API key credential named e.g. "OpenAi account 4" with your valid API key  

7. **Connect Nodes in Sequence:**  
   - `SerpAPI Finance Search1` → `Turn Objects to Text1`  
   - `Turn Objects to Text1` → `Chat with Google Finance` (main input)  
   - `Chat with Google Finance` → `Simple Memory1` (ai_memory output)  
   - `Simple Memory1` → `Chat with Google Finance` (ai_memory input)  
   - `Chat with Google Finance` → `OpenAI Chat Model5` (ai_languageModel output)  
   - `OpenAI Chat Model5` → `Chat with Google Finance` (ai_languageModel input)  

8. **Set up credentials:**  
   - Create SerpApi API key externally and replace in HTTP Request node URL.  
   - Create OpenAI API key in OpenAI platform, add funds, and configure credential in n8n.  

9. **Input Source:**  
   - The workflow expects a node named `Sample Chatbot` upstream providing:  
     - `sessionId` for memory session key  
     - `chatInput` as the user’s question string  

10. **Test and validate:**  
    - Trigger the workflow with a sample user input and valid API keys.  
    - Verify that live financial data is fetched and the chatbot responds appropriately.  
    - Check for errors in API calls and memory handling.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                         | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow connects a chatbot to live financial data from SerpApi and uses OpenAI to answer questions interactively.                                                             | Workflow description Sticky Note60                                                                       |
| Setup requires adding your SerpApi API key in the HTTP request URL and configuring OpenAI API credentials with billing enabled.                                                     | Sticky Note17, Sticky Note31, Sticky Note32                                                             |
| For customization requests (e.g., multiple tickers, daily summaries, exporting to Google Sheets), contact Robert Breen via email: robert@ynteractive.com or LinkedIn.               | Sticky Note17                                                                                            |
| SerpApi documentation: https://serpapi.com/                                                                                                                                        | Referenced for API key and endpoint configuration                                                        |
| OpenAI platform and billing: https://platform.openai.com/api-keys and https://platform.openai.com/settings/organization/billing/overview                                           | Required for API key creation and billing setup                                                          |

---

**Disclaimer:**  
The provided text originates exclusively from an automated n8n workflow created with the n8n integration tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and publicly available.
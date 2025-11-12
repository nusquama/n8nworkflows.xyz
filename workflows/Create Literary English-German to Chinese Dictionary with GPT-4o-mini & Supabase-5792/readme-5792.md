Create Literary English/German to Chinese Dictionary with GPT-4o-mini & Supabase

https://n8nworkflows.xyz/workflows/create-literary-english-german-to-chinese-dictionary-with-gpt-4o-mini---supabase-5792


# Create Literary English/German to Chinese Dictionary with GPT-4o-mini & Supabase

### 1. Workflow Overview

This workflow is designed to create a bilingual literary dictionary that translates English and German words into Chinese with rich literary context. The main use case is to assist language learners, literary analysts, translators, and creative writers by providing not just definitions but also literary-style example sentences that deepen vocabulary understanding.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives user input words via HTTP POST webhooks.
- **1.2 AI Processing:** Uses a specialized AI Agent configured with a literary dictionary prompt to detect the input language, define the word, and generate Chinese meanings along with three literary-style example sentences.
- **1.3 Error Handling:** Checks the AI response for clarity and validity, handling ambiguous or invalid inputs gracefully.
- **1.4 Response Formatting:** Parses AI textual output into structured JSON including word, Chinese meanings, and example sentences with translations.
- **1.5 Data Storage:** Saves validated dictionary entries into a Supabase database for persistent vocabulary collection.
- **1.6 Response Delivery:** Sends the final structured data or error message back to the user through webhook response nodes.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures word input from the user via a webhook HTTP POST request to initiate the dictionary lookup process.

- **Nodes Involved:**  
  - Webhook to fetch user word input (Webhook node)

- **Node Details:**  
  - **Webhook to fetch user word input**  
    - Type: Webhook (HTTP POST)  
    - Configuration: Listens on path `e50639d5-af5b-4523-864a-cb123250887f` for POST requests  
    - Role: Receives JSON input containing the word to define (e.g., `{ "word": "serenity" }`)  
    - Inputs: External HTTP POST request  
    - Outputs: Passes the input JSON to the AI Agent node  
    - Edge cases: Missing or malformed JSON payload, HTTP errors, webhook path conflicts  
    - Notes: This node serves as the external API entry point for dictionary queries.

---

#### 1.2 AI Processing

- **Overview:**  
  Processes the input word through a specialized AI Agent configured to detect language automatically and generate detailed literary-style definitions and examples in Chinese.

- **Nodes Involved:**  
  - AI Agent (Langchain agent node)  
  - Openai translate & give examples (OpenAI GPT-4o-mini node)

- **Node Details:**  
  - **AI Agent**  
    - Type: Langchain Agent node (AI language model orchestrator)  
    - Configuration:  
      - Input taken directly from webhook's `body.word` field  
      - System message instructs the agent to detect English or German words, provide literary-focused Chinese meanings and three example sentences with translations  
    - Key expressions: `={{ $json.body.word }}` for input text  
    - Inputs: From webhook node  
    - Outputs: AI-generated textual response including language, part of speech, Chinese meaning, and examples  
    - Potential failures: API authentication issues, ambiguous input leading to unclear AI responses, timeout on AI response, malformed AI output  
    - Version: Langchain agent node v1.9  
  - **Openai translate & give examples**  
    - Type: OpenAI LM Chat node  
    - Configuration: Uses GPT-4o-mini model for text generation  
    - Credentials: OpenAI API key required  
    - Role: Generates AI completions based on prompt (used internally by AI Agent)  
    - Note: This node is linked internally to the AI Agent and not directly connected in this workflow's main chain.

---

#### 1.3 Error Handling

- **Overview:**  
  Validates AI output to detect unclear or ambiguous responses suggesting the user input was invalid or insufficient.

- **Nodes Involved:**  
  - If (Conditional node)  
  - Format error message (Set node)  
  - Respond to user (Respond to webhook node)

- **Node Details:**  
  - **If**  
    - Type: Conditional node v2.2  
    - Configuration: Checks if the AI Agent's output string contains phrases like "It seems ", "unclear", or "please provide" indicating failed or ambiguous AI responses  
    - Inputs: AI Agent output  
    - Outputs:  
      - True branch: Indicates invalid or unclear input  
      - False branch: Valid AI response  
    - Edge cases: Partial matches, false positives if AI uses those phrases in legitimate answers  
  - **Format error message**  
    - Type: Set node v3.4  
    - Configuration: Sets JSON error response with keys: `error: true`, `message` with user guidance, and echoes the original word input  
    - Inputs: From If node's true branch  
  - **Respond to user**  
    - Type: Respond to webhook node v1.2  
    - Configuration: Sends the formatted error JSON back to the user  
    - Inputs: From Format error message node

---

#### 1.4 Response Formatting

- **Overview:**  
  Parses the AI's textual output into a structured JSON object containing the word, its Chinese meaning, and example sentences paired with their Chinese translations.

- **Nodes Involved:**  
  - Format response (Code node)  
  - Respond to user1 (Respond to webhook node)

- **Node Details:**  
  - **Format response**  
    - Type: Code node (JavaScript) v2  
    - Configuration:  
      - Parses AI output line by line  
      - Extracts the word (removes markdown bold), Chinese meaning, and examples section  
      - Separates example English sentences and their Chinese translations into structured array objects  
      - Returns JSON with keys: `word`, `chineseMeaning`, and `examples` (array of objects with `sentence` and `translation`)  
    - Inputs: AI Agent output (string)  
    - Outputs: Structured JSON object  
    - Edge cases: AI output format deviations, missing or malformed data, empty examples  
  - **Respond to user1**  
    - Type: Respond to webhook node v1.2  
    - Configuration: Sends the formatted JSON response back to the user  
    - Inputs: From Format response node

---

#### 1.5 Data Storage

- **Overview:**  
  Saves validated dictionary entries into a Supabase table named "Dict" for persistent storage and building a personal vocabulary database.

- **Nodes Involved:**  
  - Webhook to save words in Supabase (Webhook node)  
  - Supabase (Supabase node)  
  - Respond message (Respond to webhook node)

- **Node Details:**  
  - **Webhook to save words in Supabase**  
    - Type: Webhook node (POST)  
    - Configuration: Same webhook path as input reception, but logically separated for saving words  
    - Role: Listens for save requests containing word, Chinese meaning, and examples  
  - **Supabase**  
    - Type: Supabase node v1  
    - Configuration:  
      - Table: "Dict"  
      - Fields mapped:  
        - `Words` ← input `word`  
        - `chineseMeaning` ← input `chineseMeaning`  
        - `Examples` ← input `examples` (likely serialized JSON or string)  
    - Credentials: Supabase API key connected  
    - Outputs: Success response when insert completes  
    - Edge cases: Database connectivity errors, invalid data types, duplicate entries  
  - **Respond message**  
    - Type: Respond to webhook node v1.2  
    - Configuration: Sends JSON success confirmation `{ "success": true, "message": "Word saved successfully" }`  
    - Inputs: From Supabase node

---

#### 1.6 Response Delivery

- **Overview:**  
  Completes the workflow by sending either the structured dictionary data or an error message back to the user.

- **Nodes Involved:**  
  - Respond to user (Respond to webhook node) [error branch]  
  - Respond to user1 (Respond to webhook node) [success branch]  
  - Respond message (Respond to webhook node) [save confirmation]

- **Node Details:**  
  - Each node responds with the appropriate JSON or text message to the HTTP request that triggered the workflow.  
  - Edge cases: Network errors delivering response, webhook timeouts, malformed response bodies.

---

### 3. Summary Table

| Node Name                        | Node Type                   | Functional Role                        | Input Node(s)                  | Output Node(s)                 | Sticky Note                                             |
|---------------------------------|-----------------------------|-------------------------------------|-------------------------------|-------------------------------|---------------------------------------------------------|
| Webhook to fetch user word input| Webhook                     | Receive user input word via HTTP POST| None                          | AI Agent                      | ## 1. Webhook Receiver Receives word requests via HTTP POST. It captures the input word to be processed and triggers the entire workflow chain. **I use this Webhook to receive word input by user from a HTML web app** |
| AI Agent                        | Langchain Agent             | Generate literary dictionary definition and examples| Webhook to fetch user word input | If                           | ## 2. AI Agent Uses specialized literary dictionary prompt. Auto-detects language and generates definitions and examples. |
| If                             | If Node                    | Check for unclear or invalid AI responses| AI Agent                      | Format error message (true), Format response (false) | ## 3. Error Handler Checks for unclear or invalid responses and returns appropriate error messages. |
| Format error message            | Set                        | Prepare structured error JSON response| If (true branch)              | Respond to user               |                                                         |
| Respond to user                | Respond to Webhook          | Return error message to user          | Format error message           | None                         | ## 5. Webhook Response Node Completes workflow with error message if needed. |
| Format response                | Code                       | Parse AI output into structured JSON | If (false branch)              | Respond to user1              | ## 4. Data Structure Parser Extracts structured data from AI output and organizes it. |
| Respond to user1               | Respond to Webhook          | Return structured dictionary data to user | Format response               | None                         |                                                         |
| Webhook to save words in Supabase | Webhook                  | Receive requests to save words        | None                          | Supabase                     | ## 1. Save Words Trigger When user clicks “Save to vocabulary list” button, triggers saving workflow. |
| Supabase                      | Supabase                   | Save validated dictionary entries to database | Webhook to save words in Supabase | Respond message             | ## 2. Supabase Integration Saves all valid lookups to database and builds personal dictionary data. |
| Respond message                | Respond to Webhook          | Confirm word saved successfully       | Supabase                      | None                         | ## 3. Webhook Response Node Completes saving workflow by sending success message. |
| Openai translate & give examples | OpenAI LM Chat           | Underlying AI model call for generation | (Internal to AI Agent)         | AI Agent                     |                                                         |
| Sticky Note, Sticky Note1,...  | Sticky Note                | Documentation and explanation notes   | N/A                           | N/A                          | See notes content section below                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook to Receive Word Input**  
   - Node Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique identifier (e.g., `e50639d5-af5b-4523-864a-cb123250887f`)  
   - Purpose: Receive JSON input with key `word`  
   - No authentication required unless desired.

2. **Add AI Agent Node**  
   - Node Type: Langchain Agent  
   - Input: Set text input to `={{ $json.body.word }}` from webhook  
   - System Message: Use the detailed literary dictionary prompt specifying bilingual English/German to Chinese translation with literary context, automatic language detection, expected response format including part of speech, Chinese meanings, and three literary-style examples with translations.  
   - Model: GPT-4o-mini (via OpenAI)  
   - Connect webhook output to this node input.

3. **Add Conditional “If” Node for Error Detection**  
   - Node Type: If (v2.2)  
   - Condition: Check if AI Agent output contains any of the substrings: `"It seems "`, `"unclear"`, or `"please provide"`  
   - Connect AI Agent output to If node.

4. **Add “Format Error Message” Node (Set)**  
   - Node Type: Set (v3.4)  
   - Output JSON:  
     ```json
     {
       "error": true,
       "message": "Please provide a specific word in English or German. The input was not recognized as a valid word.",
       "word": "={{ $('Webhook to fetch user word input').first().json.body.word }}"
     }
     ```  
   - Connect the If node’s true branch to this node.

5. **Add “Respond to User” Node for Errors**  
   - Node Type: Respond to Webhook  
   - Input: JSON from Format Error Message node  
   - Connect Format Error Message node output to this node.

6. **Add “Format Response” Node (Code)**  
   - Node Type: Code (JavaScript)  
   - Paste the provided JS code that parses AI output string into structured JSON `{ word, chineseMeaning, examples[] }`  
   - Connect the If node’s false branch to this node.

7. **Add “Respond to User1” Node for Valid Responses**  
   - Node Type: Respond to Webhook  
   - Response Body: `={{ JSON.stringify($('Format response').first().json) }}`  
   - Connect Format Response node output to this node.

8. **Create Webhook to Save Words to Database**  
   - Node Type: Webhook  
   - HTTP Method: POST  
   - Path: Same or different unique path (can be the same but logically separated)  
   - Purpose: Receive word data to be saved.

9. **Add Supabase Node to Insert Data**  
   - Node Type: Supabase  
   - Table: `Dict`  
   - Map fields:  
     - `Words` ← `={{ $json.body.word }}`  
     - `chineseMeaning` ← `={{ $json.body.chineseMeaning }}`  
     - `Examples` ← `={{ $json.body.examples }}` (serialize to string if necessary)  
   - Configure Supabase credentials with API key and project URL.

10. **Add Respond to User Node for Save Confirmation**  
    - Node Type: Respond to Webhook  
    - Response Body: `{"success": true, "message": "Word saved successfully"}`  
    - Connect Supabase node output to this node.

11. **Connect Nodes According to Workflow Logic**  
    - Webhook (input) → AI Agent → If →  
      - True → Format error message → Respond to user (error)  
      - False → Format response → Respond to user1 (success)  
    - Webhook (save) → Supabase → Respond message (save confirmation)

12. **Credentials Setup**  
    - Configure OpenAI API credentials for AI Agent and OpenAI node  
    - Configure Supabase API credentials for database access

13. **Optional: Add Sticky Notes**  
    - Add descriptive sticky notes near node groups to document purposes and usage for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| This workflow automatically detects input language (English/German) and handles error cases gracefully.                             | Workflow behavior detail                                                        |
| The system prompt prioritizes literary, archaic, or metaphorical meanings and provides elegant, non-academic responses.             | AI Agent prompt design                                                         |
| Saves all valid lookups to Supabase, enabling building a personal bilingual dictionary with Chinese translations and literary examples. | Data persistence and integration                                               |
| Requires OpenAI account with GPT-4o-mini access and a Supabase account for database storage.                                         | External service requirements                                                  |
| The webhook path is a UUID-like string, ensure to use unique paths to avoid conflicts.                                               | Webhook configuration best practice                                            |
| This workflow is ideal for integration with language learning apps, creative writing tools, literary analysis platforms, or reading apps. | Suggested use cases                                                             |
| Sticky notes in the workflow provide detailed explanations and usage instructions directly in the n8n editor for maintainers.      | Documentation practice within n8n                                              |
| For customization, modify the AI system prompt to support additional languages or different literary styles (academic, casual, business). | Workflow extensibility                                                          |

---

**Disclaimer:** The text provided is derived exclusively from an automated n8n workflow. All data processed is legal and public. The workflow complies fully with content policies and contains no illegal or offensive material.
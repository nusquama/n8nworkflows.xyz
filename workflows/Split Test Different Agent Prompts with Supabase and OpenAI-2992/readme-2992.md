Split Test Different Agent Prompts with Supabase and OpenAI

https://n8nworkflows.xyz/workflows/split-test-different-agent-prompts-with-supabase-and-openai-2992


# Split Test Different Agent Prompts with Supabase and OpenAI

### 1. Workflow Overview

This workflow implements an A/B split testing mechanism for chat prompts using Supabase and OpenAI within n8n. Its primary purpose is to randomly assign incoming chat sessions to one of two predefined prompts (baseline or alternative) and maintain consistent prompt usage throughout the session. This allows users to evaluate and compare the effectiveness of different large language model (LLM) prompts in production.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception:** Captures incoming chat messages and extracts session identifiers.
- **1.2 Session Prompt Assignment:** Checks if the chat session already has an assigned prompt in Supabase; if not, assigns one randomly and stores it.
- **1.3 AI Response Generation:** Uses the assigned prompt to generate a response via OpenAI’s chat model, incorporating session memory stored in PostgreSQL.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow upon receiving a chat message, extracting the session ID and chat input for further processing.

**Nodes Involved:**  
- When chat message received

**Node Details:**  

- **When chat message received**  
  - *Type:* Langchain Chat Trigger  
  - *Role:* Entry point that listens for incoming chat messages via webhook.  
  - *Configuration:* Default settings; no additional options configured.  
  - *Key Expressions:* Extracts `sessionId` and `chatInput` from incoming JSON payload.  
  - *Input:* External webhook call (chat message).  
  - *Output:* Passes extracted data downstream.  
  - *Edge Cases:* Missing or malformed session ID or chat input could cause failures downstream.  
  - *Sticky Note:* "## 1. Receive Message"

---

#### 1.2 Session Prompt Assignment

**Overview:**  
This block determines which prompt to use for the current chat session. It queries Supabase to check if the session exists. If not, it randomly assigns the session to either the baseline or alternative prompt and stores this assignment in Supabase.

**Nodes Involved:**  
- Define Path Values  
- Check If Session Exists  
- If Session Does Exist  
- Assign Path To Session  
- Get Correct Prompt

**Node Details:**  

- **Define Path Values**  
  - *Type:* Set  
  - *Role:* Defines the baseline and alternative prompt strings as workflow variables.  
  - *Configuration:*  
    - `baseline_value`: "The dog's name is Ben"  
    - `alternative_value`: "The dog's name is Tom"  
  - *Input:* Receives data from the trigger node.  
  - *Output:* Passes prompt values downstream.  
  - *Sticky Note:* "### Modification\nSet the values of the  baseline and alternative prompts"

- **Check If Session Exists**  
  - *Type:* Supabase  
  - *Role:* Queries the `split_test_sessions` table in Supabase to check if the current session ID exists.  
  - *Configuration:* Filters on `session_id` equal to the incoming session ID.  
  - *Input:* Receives session ID from previous node.  
  - *Output:* Returns session record if found.  
  - *Credentials:* Supabase API credentials required.  
  - *Edge Cases:* Network or authentication errors with Supabase; empty result if session does not exist.

- **If Session Does Exist**  
  - *Type:* If  
  - *Role:* Branches workflow based on whether the session record exists.  
  - *Configuration:* Checks if the returned session record has an `id` field (exists).  
  - *Input:* Output from Supabase query.  
  - *Output:*  
    - *True branch:* Session exists.  
    - *False branch:* Session does not exist.  
  - *Edge Cases:* Expression evaluation errors if data is malformed.

- **Assign Path To Session**  
  - *Type:* Supabase  
  - *Role:* Inserts a new record into `split_test_sessions` with the session ID and a randomly assigned boolean `show_alternative` flag.  
  - *Configuration:*  
    - `session_id`: current session ID  
    - `show_alternative`: random boolean (`Math.random() < 0.5`)  
  - *Input:* Triggered only if session does not exist.  
  - *Output:* Confirmation of insertion.  
  - *Credentials:* Supabase API credentials required.  
  - *Edge Cases:* Insert conflicts if session ID already inserted concurrently; network/auth errors.

- **Get Correct Prompt**  
  - *Type:* Set  
  - *Role:* Selects the prompt string to use based on the `show_alternative` flag from the session record.  
  - *Configuration:* Uses expression:  
    ``` 
    $json.show_alternative ? alternative_value : baseline_value 
    ```  
  - *Input:* Receives session record with `show_alternative` flag.  
  - *Output:* Sets `prompt` field for downstream AI agent.  
  - *Edge Cases:* Missing `show_alternative` field or undefined prompt values.

- *Sticky Notes covering this block:*  
  - "## 2. Determine Prompt for LLM"  
  - "### Modification\nSet the values of the  baseline and alternative prompts"

---

#### 1.3 AI Response Generation

**Overview:**  
This block generates the AI response using the assigned prompt and maintains chat history in PostgreSQL for context.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Postgres Chat Memory

**Node Details:**  

- **AI Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Core node that sends the chat input and system prompt to the language model and receives the response.  
  - *Configuration:*  
    - `text`: chat input from trigger node  
    - `systemMessage`: prompt selected in previous block  
    - `promptType`: "define" (custom prompt)  
  - *Input:* Receives chat input and prompt.  
  - *Output:* AI-generated response.  
  - *Edge Cases:* Timeout or rate limits from OpenAI; invalid prompt format; missing input data.  
  - *Sticky Note:* "## 3. AI Agent"

- **OpenAI Chat Model**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Defines the underlying LLM used by the AI Agent.  
  - *Configuration:*  
    - Model: "gpt-4o-mini" (a GPT-4 variant)  
  - *Input:* Connected as the language model for AI Agent.  
  - *Output:* Provides completions to AI Agent.  
  - *Credentials:* OpenAI API credentials required.  
  - *Sticky Note:* "### Modification\nReplace this sub-node \nto use a different language\n model"

- **Postgres Chat Memory**  
  - *Type:* Langchain Postgres Chat Memory  
  - *Role:* Stores and retrieves chat history from a PostgreSQL table to provide context for the AI Agent.  
  - *Configuration:*  
    - Table: `n8n_split_test_chat_histories`  
    - Session key: current session ID from trigger  
    - Session ID type: custom key  
  - *Input:* Connected as memory source for AI Agent.  
  - *Output:* Provides chat history context.  
  - *Credentials:* PostgreSQL credentials required.  
  - *Edge Cases:* Database connection errors; missing table or schema mismatches.

- *Sticky Notes covering this block:*  
  - "### Modification\nReplace this sub-node \nto use a different language\n model"  
  - "## 3. AI Agent"

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                         | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                          |
|-------------------------|----------------------------------|---------------------------------------|----------------------------|----------------------------|----------------------------------------------------------------------------------------------------|
| When chat message received | Langchain Chat Trigger           | Entry point for incoming chat messages | (Webhook trigger)           | Define Path Values          | ## 1. Receive Message                                                                               |
| Define Path Values       | Set                              | Defines baseline and alternative prompts | When chat message received  | Check If Session Exists     | ### Modification\nSet the values of the  baseline and alternative prompts                          |
| Check If Session Exists  | Supabase                         | Checks if session ID exists in Supabase | Define Path Values          | If Session Does Exist       |                                                                                                    |
| If Session Does Exist    | If                               | Branches based on session existence    | Check If Session Exists     | Get Correct Prompt (true), Assign Path To Session (false) |                                                                                                    |
| Assign Path To Session   | Supabase                         | Inserts new session with random prompt assignment | If Session Does Exist (false) | Get Correct Prompt          |                                                                                                    |
| Get Correct Prompt       | Set                              | Selects prompt string based on session flag | If Session Does Exist (true), Assign Path To Session | AI Agent                   |                                                                                                    |
| AI Agent                | Langchain Agent                  | Generates AI response using prompt and input | Get Correct Prompt          |                            | ## 3. AI Agent                                                                                      |
| OpenAI Chat Model        | Langchain OpenAI Chat Model      | Provides LLM completions to AI Agent  |                            | AI Agent (ai_languageModel) | ### Modification\nReplace this sub-node \nto use a different language\n model                      |
| Postgres Chat Memory     | Langchain Postgres Chat Memory   | Maintains chat history for context    |                            | AI Agent (ai_memory)        |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Supabase Table**  
   - Table name: `split_test_sessions`  
   - Columns:  
     - `session_id` (text, primary key)  
     - `show_alternative` (boolean)  

2. **Create PostgreSQL Table** (for chat memory)  
   - Table name: `n8n_split_test_chat_histories`  
   - Schema as required by Langchain Postgres Chat Memory node (usually includes session ID, messages, timestamps).

3. **Add Credentials in n8n**  
   - Supabase API credentials (for Supabase nodes)  
   - OpenAI API credentials (for OpenAI Chat Model node)  
   - PostgreSQL credentials (for Postgres Chat Memory node)

4. **Create Node: When chat message received**  
   - Type: Langchain Chat Trigger  
   - Default settings  
   - This node listens for incoming chat messages and extracts `sessionId` and `chatInput`.

5. **Create Node: Define Path Values**  
   - Type: Set  
   - Add two string fields:  
     - `baseline_value` = "The dog's name is Ben"  
     - `alternative_value` = "The dog's name is Tom"  
   - Connect output of trigger node to this node.

6. **Create Node: Check If Session Exists**  
   - Type: Supabase  
   - Operation: Get  
   - Table: `split_test_sessions`  
   - Filter: `session_id` equals `={{ $json.sessionId }}` (expression referencing trigger node)  
   - Connect output of Define Path Values node to this node.

7. **Create Node: If Session Does Exist**  
   - Type: If  
   - Condition: Check if `id` field exists in Supabase query result (expression: `{{$json.id}}` exists)  
   - Connect output of Check If Session Exists node to this node.

8. **Create Node: Assign Path To Session**  
   - Type: Supabase  
   - Operation: Insert  
   - Table: `split_test_sessions`  
   - Fields:  
     - `session_id` = `={{ $json.sessionId }}`  
     - `show_alternative` = `={{ Math.random() < 0.5 }}` (random boolean)  
   - Connect false output of If node (session does not exist) to this node.

9. **Create Node: Get Correct Prompt**  
   - Type: Set  
   - Field: `prompt`  
   - Value expression:  
     ```
     {{$json.show_alternative ? $node["Define Path Values"].json["alternative_value"] : $node["Define Path Values"].json["baseline_value"]}}
     ```  
   - Connect true output of If node (session exists) and main output of Assign Path To Session node to this node.

10. **Create Node: OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Model: `gpt-4o-mini` (or your preferred model)  
    - Credentials: OpenAI API  
    - No additional options needed.

11. **Create Node: Postgres Chat Memory**  
    - Type: Langchain Postgres Chat Memory  
    - Table name: `n8n_split_test_chat_histories`  
    - Session key: `={{ $json.sessionId }}`  
    - Session ID type: customKey  
    - Credentials: PostgreSQL

12. **Create Node: AI Agent**  
    - Type: Langchain Agent  
    - Text: `={{ $node["When chat message received"].json["chatInput"] }}`  
    - System Message: `={{ $json.prompt }}` (from Get Correct Prompt node)  
    - Prompt Type: define  
    - Connect:  
      - Language Model input to OpenAI Chat Model node  
      - Memory input to Postgres Chat Memory node  
      - Main input from Get Correct Prompt node

13. **Connect Nodes**  
    - When chat message received → Define Path Values → Check If Session Exists → If Session Does Exist  
    - If Session Does Exist (true) → Get Correct Prompt → AI Agent  
    - If Session Does Exist (false) → Assign Path To Session → Get Correct Prompt → AI Agent  
    - OpenAI Chat Model and Postgres Chat Memory connected as AI Agent’s language model and memory respectively.

14. **Activate Workflow**  
    - Test by sending chat messages with different session IDs to observe prompt assignment and AI responses.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Create a Supabase table named **split_test_sessions** with columns `session_id` (text) and `show_alternative` (bool).              | Setup step for session prompt assignment                                                               |
| Modify the **Define Path Values** node to customize baseline and alternative prompt strings.                                        | Allows testing different prompt texts                                                                  |
| Use n8n’s inbuilt chat interface to send messages and test the workflow.                                                           | Testing and validation                                                                                  |
| Consider extending the workflow to test other LLM parameters such as temperature or to measure prompt efficacy.                    | Suggested next steps                                                                                     |
| Video and blog resources on split testing LLM prompts with n8n and Supabase can be found at [bsde.ai blog](https://bsde.ai/blog). | External resource for deeper understanding and examples                                                |

---

This documentation fully describes the workflow "Split Test Different Agent Prompts with Supabase and OpenAI" and enables reproduction, modification, and error anticipation for advanced users and automation agents.
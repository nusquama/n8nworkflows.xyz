Deep Research Assistant with Perplexity AI and Telegram Citations

https://n8nworkflows.xyz/workflows/deep-research-assistant-with-perplexity-ai-and-telegram-citations-7939


# Deep Research Assistant with Perplexity AI and Telegram Citations

### 1. Workflow Overview

This workflow implements a **Deep Research Assistant** that integrates Telegram messaging with advanced AI research capabilities via Perplexity AI and OpenAI. Its main purpose is to receive user queries through Telegram, process them using a Langchain AI Agent that intelligently selects between Perplexity’s Sonar and Sonar Pro tools for information retrieval and synthesis, and then respond back on Telegram with concise, well-cited answers.

**Target Use Cases:**  
- Real-time research assistant for detailed and accurate information retrieval  
- Business, scientific, and academic research support  
- Providing users with structured answers enriched with citations  
- Restricting usage to authorized Telegram users for security  

**Logical Blocks:**

- **1.1 Input Reception and Security:** Receives incoming Telegram messages and filters them by user ID to prevent unauthorized access.
- **1.2 Contextual Memory:** Maintains conversation context per chat using window buffer memory.
- **1.3 AI Processing Core:** The Langchain AI Agent node that orchestrates model selection and tool usage (OpenAI GPT and Perplexity Sonar/Sonar Pro).
- **1.4 Information Retrieval Tools:** Perplexity Sonar and Sonar Pro nodes that perform web intelligence lookups and deep research.
- **1.5 Output Delivery:** Sends the AI-generated response back to the user on Telegram.
- **1.6 Workflow Setup & Documentation:** Sticky notes providing detailed setup instructions, customization options, and usage guidelines.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Security

- **Overview:**  
  This block captures Telegram messages using a webhook trigger and filters incoming requests to allow only authorized users based on Telegram user ID.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Filter

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger  
    - *Role:* Listens for incoming Telegram messages (update type: message) via webhook.  
    - *Configuration:* Monitors "message" updates. Webhook ID is set for receiving Telegram events.  
    - *Inputs:* External Telegram message events.  
    - *Outputs:* Passes message data downstream.  
    - *Potential Failures:* Webhook misconfiguration, network issues, Telegram API limits.  
    - *Sticky Note:* See "Telegram Setup" for detailed configuration.

  - **Filter**  
    - *Type:* Filter  
    - *Role:* Ensures only messages from a specific Telegram user ID (5675741296) proceed.  
    - *Configuration:* Checks if `message.from.id` equals 5675741296 (must be updated to your user ID).  
    - *Inputs:* Telegram Trigger output.  
    - *Outputs:* Passes authorized messages to the AI Agent; blocks others.  
    - *Edge Cases:* If user ID is incorrect or missing, no messages pass through.  
    - *Sticky Note:* See "Security Filter" for instructions to update user ID.

---

#### 2.2 Contextual Memory

- **Overview:**  
  This block maintains short-term conversational context using a window buffer memory keyed by the Telegram chat ID, allowing the AI Agent to respond with context awareness.

- **Nodes Involved:**  
  - Window Buffer Memory

- **Node Details:**

  - **Window Buffer Memory**  
    - *Type:* Langchain Memory Buffer Window  
    - *Role:* Stores recent conversation history per chat session.  
    - *Configuration:* Uses Telegram chat ID (`message.chat.id`) as session key to group messages.  
    - *Inputs:* Not directly connected; linked by AI Agent as memory source.  
    - *Outputs:* Supplies memory context to AI Agent node.  
    - *Edge Cases:* If chat ID is missing or malformed, context may not persist.  
    - *Sticky Note:* See "Memory Config" for customization options.

---

#### 2.3 AI Processing Core

- **Overview:**  
  This is the central AI Agent node built with n8n’s Langchain integration. It receives user queries, accesses conversation memory, and intelligently routes research requests to appropriate Perplexity tools and OpenAI GPT, generating well-structured, cited responses.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Perplexity Sonar  
  - Perplexity Sonar Pro

- **Node Details:**

  - **AI Agent**  
    - *Type:* Langchain Agent  
    - *Role:* Coordinates input processing, memory usage, language modeling, and tool integrations.  
    - *Configuration:*  
      - Input text is concatenation of chat input and message text.  
      - System message defines research focus, tool usage logic, output guidelines, and style & tone.  
      - Uses two Perplexity tools (Sonar and Sonar Pro) for different research complexity levels.  
      - Uses OpenAI GPT-4o-mini for language modeling.  
      - Memory from Window Buffer Memory node.  
    - *Inputs:*  
      - Main: Filter node (authorized user messages).  
      - ai_memory: Window Buffer Memory.  
      - ai_languageModel: OpenAI Chat Model.  
      - ai_tool: Perplexity Sonar and Sonar Pro.  
    - *Outputs:*  
      - Main: AI-generated responses to send back.  
    - *Edge Cases:*  
      - Expression errors if input JSON fields are missing.  
      - API timeouts or quota exceeded on OpenAI or Perplexity.  
      - Incorrect or missing API credentials.  
      - Ambiguous queries handled internally per system message instructions.  
    - *Sticky Note:* See "AI Agent Config" for customization and behavior tuning.

  - **OpenAI Chat Model**  
    - *Type:* Langchain LM Chat OpenAI  
    - *Role:* Provides GPT-4o-mini language model for natural language understanding and generation.  
    - *Configuration:* Uses model "gpt-4o-mini" with default options.  
    - *Credentials:* OpenAI API key configured in n8n credentials.  
    - *Inputs:* Connected to AI Agent's language model input.  
    - *Outputs:* Feeds generated completions to AI Agent.  
    - *Edge Cases:* API key invalid, rate limits, model unavailability.  
    - *Sticky Note:* See "OpenAI Setup" for API key and model setup.

  - **Perplexity Sonar**  
    - *Type:* Custom Perplexity Tool node (sonar model)  
    - *Role:* Fast factual lookups and summaries for simple queries.  
    - *Configuration:* Model set to "sonar" with default options and empty message template (filled by AI Agent).  
    - *Credentials:* Perplexity API key set in credentials.  
    - *Inputs:* Connected as ai_tool input to AI Agent.  
    - *Outputs:* Provides factual data to AI Agent for response composition.  
    - *Edge Cases:* API key issues, response delays, model updates.  
    - *Sticky Note:* See "Perplexity Pro" note for detailed usage and differentiation.

  - **Perplexity Sonar Pro**  
    - *Type:* Custom Perplexity Tool node (sonar-pro model)  
    - *Role:* Handles complex queries requiring synthesis, deep dives, and structured summarization.  
    - *Configuration:* Model set to "sonar-pro" with default options and empty message template.  
    - *Credentials:* Shares Perplexity API key with Sonar node.  
    - *Inputs:* Connected as ai_tool input to AI Agent.  
    - *Outputs:* Provides in-depth data for AI Agent responses.  
    - *Edge Cases:* Same as Sonar node; also higher computational cost.  
    - *Sticky Note:* See "Perplexity Pro" for detailed tool description.

---

#### 2.4 Output Delivery

- **Overview:**  
  Sends the AI Agent’s generated answer back to the Telegram user’s chat.

- **Nodes Involved:**  
  - Telegram

- **Node Details:**

  - **Telegram**  
    - *Type:* Telegram node (send message)  
    - *Role:* Posts AI-generated responses back to the Telegram chat identified by chat ID.  
    - *Configuration:*  
      - Text set dynamically from AI Agent output (`{{ $json.output }}`).  
      - Chat ID pulled from Telegram Trigger input.  
      - Uses same Telegram credentials as trigger node.  
    - *Inputs:* AI Agent main output.  
    - *Outputs:* External Telegram message delivery.  
    - *Edge Cases:* Telegram API errors, message formatting issues, invalid chat ID.  
    - *Sticky Note:* See "Telegram Response" for optional formatting and threading customization.

---

#### 2.5 Workflow Setup & Documentation

- **Overview:**  
  Multiple sticky note nodes provide comprehensive instructions on setup, security, AI customization, API credentials, and tool usage.

- **Nodes Involved:**  
  - Workflow Overview (sticky note)  
  - Telegram Setup (sticky note)  
  - Security Filter (sticky note)  
  - AI Agent Config (sticky note)  
  - OpenAI Setup (sticky note)  
  - Memory Config (sticky note)  
  - Perplexity Pro (sticky note)  
  - Telegram Response (sticky note)

- **Node Details:**  
  Each sticky note includes detailed guidance for the corresponding block, emphasizing API key management, user security, model selection, and usage tips.

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                                      | Input Node(s)         | Output Node(s)      | Sticky Note                                                                                          |
|---------------------|----------------------------------|-----------------------------------------------------|-----------------------|---------------------|----------------------------------------------------------------------------------------------------|
| Telegram Trigger     | telegramTrigger                  | Receives Telegram messages                           | (external webhook)    | Filter              | See "Telegram Setup" for bot creation, token, webhook configuration                                 |
| Filter              | filter                          | Filters messages by authorized Telegram user ID     | Telegram Trigger      | AI Agent            | See "Security Filter" to update user ID and prevent unauthorized access                            |
| Window Buffer Memory | memoryBufferWindow              | Maintains chat context via session-keyed memory     | (not direct input)    | AI Agent            | See "Memory Config" for session key and memory window customization                                |
| OpenAI Chat Model    | lmChatOpenAi                    | Provides GPT-4o-mini language model                  | None (used by AI Agent) | AI Agent           | See "OpenAI Setup" for API key and model selection                                                |
| Perplexity Sonar     | perplexityTool (sonar)          | Fast factual lookup and summaries                     | None (used by AI Agent) | AI Agent           | See "Perplexity Pro" for usage and distinction vs Sonar Pro                                       |
| Perplexity Sonar Pro | perplexityTool (sonar-pro)      | Deep research and synthesis for complex queries      | None (used by AI Agent) | AI Agent           | See "Perplexity Pro" for usage and distinction vs Sonar                                           |
| AI Agent            | langchain.agent                 | Central AI orchestrator combining memory, LM, tools | Filter, Window Buffer Memory, OpenAI Chat Model, Perplexity Sonar, Perplexity Sonar Pro | Telegram            | See "AI Agent Config" for system message and prompt customization                                 |
| Telegram            | telegram                        | Sends AI-generated response back to Telegram chat   | AI Agent              | External Telegram    | See "Telegram Response" for formatting and reply threading options                               |
| Workflow Overview   | stickyNote                     | Documentation and overview                            | None                  | None                | Provides high-level workflow summary and security notes                                           |
| Telegram Setup      | stickyNote                     | Telegram bot creation and webhook setup instructions | None                  | None                | Detailed Telegram Trigger setup instructions                                                      |
| Security Filter     | stickyNote                     | Instructions for user ID update in Filter node       | None                  | None                | Explains how to find and update Telegram user ID                                                  |
| AI Agent Config     | stickyNote                     | Customization of AI Agent system message and input   | None                  | None                | Details on modifying system message, prompt handling, and behavior                                |
| OpenAI Setup        | stickyNote                     | OpenAI API key and model selection instructions      | None                  | None                | Guidance on adding OpenAI credentials and choosing models                                        |
| Memory Config       | stickyNote                     | Memory node configuration notes                       | None                  | None                | Notes on session key logic and memory window size                                                |
| Perplexity Pro      | stickyNote                     | Perplexity API key setup and tool usage explanation  | None                  | None                | Describes differences between Sonar and Sonar Pro models and API credential setup                |
| Telegram Response   | stickyNote                     | Telegram message sending customization instructions  | None                  | None                | Advice on message formatting and threading options                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Setup Credentials**  
   - Use @BotFather in Telegram to create a bot and obtain the bot token.  
   - In n8n, add Telegram credentials with the bot token.  
   - Configure webhook URL in Telegram bot settings to point to n8n's webhook endpoint.

2. **Add Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Set "Updates" to ["message"].  
   - Webhook ID: Generate or reuse one.  
   - Connect this node as the workflow entry point.

3. **Add Filter Node for User Restriction**  
   - Type: Filter  
   - Condition: `message.from.id` equals your Telegram user ID (find via @userinfobot).  
   - Connect Telegram Trigger's main output to this Filter node.

4. **Add Window Buffer Memory Node**  
   - Type: Langchain Memory Buffer Window  
   - Parameters:  
     - Session Key: Set to `{{ $json.message.chat.id }}` to maintain per-chat context.  
     - Session ID Type: Custom Key.  
   - No direct input connection needed — will be linked to AI Agent.

5. **Add OpenAI Chat Model Node**  
   - Type: Langchain LM Chat OpenAI  
   - Parameters:  
     - Model: Select "gpt-4o-mini" or preferred GPT-4 variant.  
     - Options: Default or customize temperature/parameters as needed.  
   - Credentials: Add OpenAI API key credentials in n8n and select here.

6. **Add Perplexity Sonar Node**  
   - Type: Custom Perplexity Tool node  
   - Parameters:  
     - Model: "sonar"  
     - Messages: Leave default; AI Agent will populate dynamically.  
   - Credentials: Add Perplexity API key credentials in n8n and select here.

7. **Add Perplexity Sonar Pro Node**  
   - Type: Custom Perplexity Tool node  
   - Parameters:  
     - Model: "sonar-pro"  
     - Messages: Leave default; AI Agent will populate dynamically.  
   - Credentials: Use same Perplexity API credentials as Sonar node.

8. **Add AI Agent Node**  
   - Type: Langchain Agent  
   - Parameters:  
     - Text Input: `{{$json.chatInput}} {{$json.message.text}}` (concatenates user chat input).  
     - System Message: Paste the detailed system message defining research focus, tool usage, output style, and tone.  
     - Prompt Type: Define (fixed prompt).  
   - Connect inputs:  
     - Main input from Filter node (authorized messages).  
     - ai_memory input from Window Buffer Memory node.  
     - ai_languageModel input from OpenAI Chat Model node.  
     - ai_tool inputs from both Perplexity Sonar and Sonar Pro nodes.  
   - Outputs: Connect main output to Telegram node.

9. **Add Telegram Node (Send Message)**  
   - Type: Telegram  
   - Parameters:  
     - Text: `{{ $json.output }}` (AI Agent's generated answer).  
     - Chat ID: `{{ $('Telegram Trigger').item.json.message.chat.id }}`.  
   - Credentials: Use same Telegram credentials from the Trigger node.  
   - Connect input from AI Agent node's main output.

10. **Add Sticky Notes (Optional for Documentation)**  
    - Create sticky note nodes to document setup for Telegram, security filter, AI Agent config, OpenAI, Perplexity tools, memory config, and Telegram response formatting.  
    - Position them logically near their related nodes for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Keep all API keys secure and do not expose them publicly. Use environment variables or n8n credentials securely.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Security best practices                                                                                 |
| To find your Telegram user ID, message @userinfobot on Telegram. Update the Filter node condition accordingly to prevent unauthorized access.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Security Filter sticky note                                                                             |
| The AI Agent’s system message guides tool selection between Perplexity Sonar (fast, simple lookups) and Sonar Pro (deep synthesis and research). Modify this message to adjust behavior or disable tools as needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                    | AI Agent Config sticky note                                                                             |
| For cost-effective language modeling, use "gpt-4o-mini" but feel free to switch to other OpenAI models as needed in the OpenAI Chat Model node configuration.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | OpenAI Setup sticky note                                                                                |
| Perplexity API requires adding an HTTP Header "Authorization: Bearer YOUR_API_KEY" in n8n credentials for access. Adjust max tokens and parameters if Perplexity updates their API.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Perplexity Pro sticky note                                                                              |
| Telegram response node can be customized to include Markdown or HTML parsing modes, reply threading, or buttons by expanding the "additionalFields" parameter.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Telegram Response sticky note                                                                           |
| This workflow is designed for a single user ID to maintain security. For multi-user support, consider modifying the Filter node or implementing additional authentication layers.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Security considerations                                                                                 |
| The workflow’s design leverages Langchain’s multi-tool and memory capabilities to create an intelligent routing system and maintain conversational context in research tasks.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Overall architecture insight                                                                            |

---

**Disclaimer:** The provided content is exclusively derived from an automated n8n workflow designed for legal and public data handling. It fully complies with relevant content policies and contains no illegal, offensive, or protected material.
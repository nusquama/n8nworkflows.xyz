Turn Telegram into an AI assistant with OpenAI voice, memory and tools

https://n8nworkflows.xyz/workflows/turn-telegram-into-an-ai-assistant-with-openai-voice--memory-and-tools-13748


# Turn Telegram into an AI assistant with OpenAI voice, memory and tools

This document provides a comprehensive technical analysis of the n8n workflow: **AI Personal Assistant for Telegram with Voice, Memory & Productivity Tools**.

---

### 1. Workflow Overview

This workflow transforms a Telegram bot into a sophisticated AI personal assistant named "Lucy." It is designed to handle multimodal inputs (text and voice), maintain conversation context via short-term memory, and execute complex tasks using a suite of integrated productivity tools.

**Logical Blocks:**
*   **1.1 Input Reception & Security:** Captures Telegram messages and validates the sender's ID to ensure private access.
*   **1.2 Multimodal Processing:** Detects if the input is text or voice. Voice messages are automatically transcribed via OpenAI Whisper.
*   **1.3 AI Reasoning Core (Lucy):** An AI Agent (LangChain) that processes the request using GPT-4, looks up history in memory, and decides which tool to use.
*   **1.4 Response Refinement:** A post-processing stage that formats Markdown lists and cleans the AI output for optimal readability.
*   **1.5 Dynamic Delivery:** A routing system that chooses between a single message delivery or a "multi-bubble" burst delivery for long responses, including artificial delays for a natural feel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Security
*   **Overview:** Acts as the gateway, ensuring only the owner can trigger the AI.
*   **Nodes Involved:** Telegram Trigger2, If1, Edit Fields.
*   **Node Details:**
    *   **Telegram Trigger2:** Listens for "message" updates.
    *   **If1:** Hardcoded security check. It compares the sender's ID (`message.from.id`) against a specific numeric ID (6471510081). 
    *   **Edit Fields:** Standardizes the incoming data schema (chat_id, text, voice file_ids, etc.) for downstream nodes.

#### 2.2 Multimodal Processing
*   **Overview:** Routes the workflow based on the message type (Voice vs. Text).
*   **Nodes Involved:** Switch2, Get a file, OpenAI2 (Audio), audio message, Text Message, Clean Message, final message.
*   **Node Details:**
    *   **Switch2:** Branches the logic. If `mensaje.audio.file_id` exists, it goes to the audio path; otherwise, it checks for text.
    *   **OpenAI2 (Audio):** Uses the `transcribe` operation to convert voice files into text.
    *   **Clean Message (Code Node):** A JavaScript-based node that merges text fragments from various sources (transcriptions, documents, or direct text) into a single string variable `mensaje_final`.

#### 2.3 AI Reasoning Core (Lucy)
*   **Overview:** The "brain" of the operation. It interprets intent and interacts with external services.
*   **Nodes Involved:** Lucy (Agent), OpenAI Chat Model1, Simple Memory, ContactAgent, Email Agent, CalendarAgent, ResearchAgent, Think, Calculator.
*   **Node Details:**
    *   **Lucy (Agent):** A LangChain agent with a detailed system prompt defining its persona (Lucy, assistant to Christian). It has specific instructions on the order of tool usage (e.g., check contacts before emailing).
    *   **Simple Memory:** A Window Buffer Memory that tracks the last 15 interactions using the Telegram User ID as the session key.
    *   **Tools (Sub-workflows):** These nodes invoke external n8n workflows for specialized tasks:
        *   `ContactAgent`: Manages the contact database.
        *   `Email Agent`: Handles Gmail/Outlook drafting and sending.
        *   `CalendarAgent`: Manages scheduling.
        *   `ResearchAgent`: Performs internet searches.

#### 2.4 Response Refinement & Delivery Logic
*   **Overview:** Prepares the AI's response for the Telegram UI.
*   **Nodes Involved:** transform text, swich, path, message breakdown, message.
*   **Node Details:**
    *   **transform text (Code Node):** Normalizes Markdown lists and ensures consistent line breaks between items for better mobile readability.
    *   **swich (Code Node):** Calculates the word count of the response.
    *   **path (If Node):** If the word count is greater than 15, it triggers the "Message Breakdown" logic for multi-bubble delivery. Otherwise, it sends a single message.
    *   **message breakdown (Chain):** Uses GPT-4o-mini and a **Structured Output Parser** to split a long response into up to 4 logical parts (`part_1` to `part_4`).

#### 2.5 Dynamic Multi-Bubble Delivery
*   **Overview:** Delivers long messages in segments to mimic human typing.
*   **Nodes Involved:** 1st bubble, 2nd bubble, 3rd bubble, 4th bubble, 2nd?, 3rd?, 4th?, Wait5, Wait6, Wait7.
*   **Node Details:**
    *   **Wait Nodes:** Configured for 3-second pauses between segments.
    *   **If Nodes (2nd?, 3rd?):** Check if the subsequent parts of the message breakdown actually contain text before attempting to send.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Telegram Trigger2 | Telegram Trigger | Entry Point | None | If1 | Workflow entry & user validation |
| If1 | If | Security Gate | Telegram Trigger2 | Edit Fields / NoOp | Workflow entry & user validation |
| Edit Fields | Set | Data Mapping | If1 | Switch2 | Message type detection |
| Switch2 | Switch | Router | Edit Fields | Get a file / Text Message | Message type detection |
| OpenAI2 | OpenAI (Audio) | Transcription | Get a file | audio message | Message type detection |
| Lucy | AI Agent | Core Logic | final message | transform text | AI personal assistant agent |
| transform text | Code | Formatting | Lucy | swich | Response formatting and delivery |
| swich | Code | Logic | transform text | path | Response formatting and delivery |
| path | If | Router | swich | message breakdown / message | Response formatting and delivery |
| message breakdown | Chain | Content Splitter | path | 1st bubble | Response formatting and delivery |
| 1st bubble | Telegram | Sender | message breakdown | 2nd? | Response formatting and delivery |
| ContactAgent | Tool Workflow | Contact Mgmt | Lucy | Lucy | AI personal assistant agent |
| Email Agent | Tool Workflow | Email Mgmt | Lucy | Lucy | AI personal assistant agent |
| CalendarAgent | Tool Workflow | Scheduling | Lucy | Lucy | AI personal assistant agent |
| ResearchAgent | Tool Workflow | Web Search | Lucy | Lucy | AI personal assistant agent |

---

### 4. Reproducing the Workflow from Scratch

1.  **Environment Setup:**
    *   Create a Telegram Bot via `@BotFather` and obtain the API Token.
    *   Ensure you have an OpenAI API Key with access to GPT-4 and Whisper.
2.  **Triggers & Security:**
    *   Add a **Telegram Trigger** node.
    *   Add an **If Node** immediately after. Set it to check if `{{ $json.message.from.id }}` equals your specific Telegram ID.
3.  **Data Normalization:**
    *   Add a **Set Node** (Edit Fields) to map `chat_id` and nested message properties into a flat structure.
4.  **Multimodal Handling:**
    *   Use a **Switch Node** to check for the existence of `voice.file_id`.
    *   For the voice path: Add a **Telegram Node** (Action: Get File) -> **OpenAI Node** (Action: Transcribe) -> **Set Node** (audio message).
    *   For the text path: Add a **Set Node** (Text Message).
    *   Join both paths into a **Code Node** (Clean Message) to consolidate the input string.
5.  **Agent Assembly:**
    *   Create an **AI Agent Node**.
    *   Connect an **OpenAI Chat Model Node** (GPT-4 recommended).
    *   Connect a **Window Buffer Memory Node** (Window: 15).
    *   Add **Tool Nodes** (Workflow Tool) for Email, Calendar, and Contacts. *Note: You must have separate sub-workflows created for these.*
    *   Add the **Calculator** and **Google Search (Research)** tools.
6.  **Formatting & Smart Routing:**
    *   Add the **Code Node** (transform text) using the JavaScript provided in the JSON to clean Markdown.
    *   Add an **If Node** (path) to branch based on word count (Condition: `{{ $json.output.length > X }}`).
7.  **Delivery Logic:**
    *   **Simple Path:** Direct **Telegram Node** (Send Message).
    *   **Complex Path:** Use a **Chain Node** with an **OpenAI Chat Model** (GPT-4o-mini) and an **Auto-fixing Output Parser** to split text into 4 JSON parts. Use **Wait Nodes** (3 seconds) between the Telegram send nodes for each part.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Language Support** | The system prompt specifically instructs Lucy to respond in the user's language (defaulting to Spanish/Castellano). |
| **Tool Dependency** | This workflow depends on four sub-workflows (EmailAgent, CalendarAgent, ResearchAgent, ContactAgent). |
| **Year Reference** | The AI is explicitly told it is currently the year 2025 to assist with date-based scheduling. |
| **Memory Persistence** | Memory is tied to the Telegram User ID, allowing the agent to remember "Christian's" preferences. |
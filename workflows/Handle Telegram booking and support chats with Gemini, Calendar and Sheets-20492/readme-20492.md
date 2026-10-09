Handle Telegram booking and support chats with Gemini, Calendar and Sheets

https://n8nworkflows.xyz/workflows/handle-telegram-booking-and-support-chats-with-gemini--calendar-and-sheets-20492


# Handle Telegram booking and support chats with Gemini, Calendar and Sheets

### 1. Workflow Overview

This workflow automates customer interaction and booking management via Telegram by leveraging Google Gemini as an intelligent conversational agent. It classifies user intents (such as appointment requests, pricing questions, complaints, or general inquiries), retrieves relevant knowledge from Google Sheets, manages availability and event creation in Google Calendar, and routes administrative alerts to the business owner via Telegram.

The logic is organized into the following functional blocks:
- **1.1 Input Reception & Data Preparation:** Listens for incoming Telegram messages and extracts essential metadata (chat ID, sender name, message body).
- **1.2 AI Processing & Tool Execution:** Processes messages using a Google Gemini language model supported by conversational memory and tool integrations for Google Calendar and Google Sheets.
- **1.3 Response Delivery & Intent Routing:** Sends sanitized replies back to the customer on Telegram and branches execution based on internal tags to notify the business owner of new bookings or required human support.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Preparation
- **Overview:** Captures incoming webhooks from Telegram when a user interacts with the bot and structures the payload for downstream processing.
- **Nodes Involved:** `When Telegram Message Received`, `Prepare Message Data`

##### Node Details:
- **When Telegram Message Received**
  - **Type & Role:** `n8n-nodes-base.telegramTrigger` (Trigger node)
  - **Configuration:** Listens for `message` updates via a configured webhook ID (`telegram-randevu-webhook`).
  - **Input/Output:** No input connections; outputs raw Telegram message objects.
  - **Failure Types:** Webhook registration errors, Telegram API downtime, or invalid bot token credentials.
- **Prepare Message Data**
  - **Type & Role:** `n8n-nodes-base.set` (Data transformation node)
  - **Configuration:** Maps specific fields from the trigger payload into clean variables (`chat_id`, `user_message`, `user_name`).
  - **Key Expressions:** 
    - `chat_id`: `={{ $json.message.chat.id }}`
    - `user_message`: `={{ $json.message.text }}`
    - `user_name`: `={{ $json.message.from.first_name }}`
  - **Input/Output:** Input from `When Telegram Message Received`; outputs to `Message Handling Agent`.
  - **Failure Types:** Expression evaluation errors if the expected JSON structure is altered by Telegram.

---

#### 2.2 AI Processing & Tool Execution
- **Overview:** Acts as the conversational and decision-making core, combining a Gemini model, rolling chat memory, and external tools to query availability, schedule appointments, and look up FAQ data.
- **Nodes Involved:** `Message Handling Agent`, `Gemini Conversation Model`, `Conversation Memory Buffer`, `Check Calendar Availability`, `Create Calendar Appointment`, `Lookup FAQs and Pricing`

##### Node Details:
- **Message Handling Agent**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.agent` (AI Agent orchestration node)
  - **Configuration:** Processes text inputs using a system prompt that defines specific intent categories (`RANDEVU_TALEBI`, `FIYAT_SORUSU`, `SIKAYET`, `GENEL_SORU`) and internal tags (`RANDEVU_OLUSTU`, `İNSAN_DESTEK_GEREKLI`). Sets maximum iterations to 6.
  - **Key Expressions:** Text input: `={{ $json.user_message }}`
  - **Input/Output:** Receives data from `Prepare Message Data`; links to language model, memory, and tools; outputs the final text response to `Send Response to Customer`.
  - **Failure Types:** LLM rate limits, token budget exhaustion, or tool execution timeouts.
- **Gemini Conversation Model**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.lmChatGoogleGemini` (Language model integration)
  - **Configuration:** Uses model `models/gemini-1.5-flash` with a temperature of `0.3`.
  - **Input/Output:** Connects as an AI language model to `Message Handling Agent`.
  - **Failure Types:** API authentication failures or model deprecation.
- **Conversation Memory Buffer**
  - **Type & Role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow` (Memory module)
  - **Configuration:** Maintains a context window length of 10 messages, keyed dynamically by the user's chat ID.
  - **Key Expressions:** Session key: `={{ $('Prepare Message Data').item.json.chat_id }}`
  - **Input/Output:** Connects as AI memory to `Message Handling Agent`.
- **Check Calendar Availability**
  - **Type & Role:** `n8n-nodes-base.googleCalendarTool` (Google Calendar tool)
  - **Configuration:** Targets the `primary` calendar to retrieve event listings (`getAll`).
  - **Input/Output:** Connects as an AI tool to `Message Handling Agent`.
  - **Credentials:** Google Calendar OAuth2 API.
  - **Failure Types:** Calendar permission errors or invalid calendar ID.
- **Create Calendar Appointment**
  - **Type & Role:** `n8n-nodes-base.googleCalendarTool` (Google Calendar tool)
  - **Configuration:** Creates an event on the `primary` calendar using dynamic start/end times and customer metadata provided by the AI.
  - **Key Expressions:** 
    - Start: `={{ $fromAI('start_datetime') }}`
    - End: `={{ $fromAI('end_datetime') }}`
    - Summary: `=Randevu - {{ $fromAI('customer_name') }}`
    - Description: `={{ $fromAI('customer_note') }}`
  - **Input/Output:** Connects as an AI tool to `Message Handling Agent`.
  - **Credentials:** Google Calendar OAuth2 API.
  - **Failure Types:** Invalid datetime formats generated by the LLM or scheduling conflicts.
- **Lookup FAQs and Pricing**
  - **Type & Role:** `n8n-nodes-base.googleSheetsTool` (Google Sheets tool)
  - **Configuration:** Queries a specified spreadsheet document ID (`GOOGLE_SHEETS_FAQ_DOC_ID`).
  - **Input/Output:** Connects as an AI tool to `Message Handling Agent`.
  - **Credentials:** Google Sheets OAuth2 API.
  - **Failure Types:** Incorrect spreadsheet ID, missing permissions, or unreadable sheet structure.

---

#### 2.3 Response Delivery & Intent Routing
- **Overview:** Dispatches the AI response to the user on Telegram, cleans up internal routing tags, and evaluates conditional logic to alert the business owner if a booking was made or human support is requested.
- **Nodes Involved:** `Send Response to Customer`, `If Appointment Created`, `Notify Business of Appointment`, `If Human Support Required`, `Send Support Alert to Admin`

##### Node Details:
- **Send Response to Customer**
  - **Type & Role:** `n8n-nodes-base.telegram` (Action node)
  - **Configuration:** Sends a message via Markdown parse mode, stripping out internal tags (`RANDEVU_OLUSTU`, `İNSAN_DESTEK_GEREKLI`) before the user sees it.
  - **Key Expressions:** 
    - Text: `={{ $json.output.replace('RANDEVU_OLUSTU', '').replace('İNSAN_DESTEK_GEREKLI', '') }}`
    - Chat ID: `={{ $('Prepare Message Data').item.json.chat_id }}`
  - **Input/Output:** Input from `Message Handling Agent`; outputs to both `If Appointment Created` and `If Human Support Required`.
  - **Failure Types:** Telegram broadcast restrictions, invalid chat ID, or Markdown syntax errors in the AI response.
- **If Appointment Created**
  - **Type & Role:** `n8n-nodes-base.if` (Conditional routing node)
  - **Configuration:** Checks if the agent output contains the string `RANDEVU_OLUSTU`.
  - **Key Expressions:** Left value: `={{ $json.output }}` matches `RANDEVU_OLUSTU`.
  - **Input/Output:** Input from `Send Response to Customer`; outputs to `Notify Business of Appointment`.
- **Notify Business of Appointment**
  - **Type & Role:** `n8n-nodes-base.telegram` (Action node)
  - **Configuration:** Sends a notification template to the business owner containing customer details and conversation history.
  - **Key Expressions:** 
    - Text: `=📅 Yeni randevu oluşturuldu!\n\nMüşteri: {{ $('Prepare Message Data').item.json.user_name }}\nMesaj geçmişi: {{ $('Prepare Message Data').item.json.user_message }}\n\nGoogle Calendar'dan detayları kontrol edebilirsiniz.`
    - Chat ID: `BUSINESS_OWNER_TELEGRAM_CHAT_ID`
  - **Input/Output:** Input from `If Appointment Created`.
  - **Failure Types:** Invalid owner chat ID placeholder.
- **If Human Support Required**
  - **Type & Role:** `n8n-nodes-base.if` (Conditional routing node)
  - **Configuration:** Checks if the agent output contains the string `İNSAN_DESTEK_GEREKLI`.
  - **Key Expressions:** Left value: `={{ $json.output }}` matches `İNSAN_DESTEK_GEREKLI`.
  - **Input/Output:** Input from `Send Response to Customer`; outputs to `Send Support Alert to Admin`.
- **Send Support Alert to Admin**
  - **Type & Role:** `n8n-nodes-base.telegram` (Action node)
  - **Configuration:** Sends a complaint/support alert template to the business owner.
  - **Key Expressions:** 
    - Text: `=⚠️ Dikkat: Bir müşteri şikayeti/insan desteği talebi var.\n\nMüşteri: {{ $('Prepare Message Data').item.json.user_name }}\nMesaj: {{ $('Prepare Message Data').item.json.user_message }}`
    - Chat ID: `BUSINESS_OWNER_TELEGRAM_CHAT_ID`
  - **Input/Output:** Input from `If Human Support Required`.
  - **Failure Types:** Invalid owner chat ID placeholder.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation and workflow overview container | None | None | ## Telegram Otomatik Yanıt + Randevu Botu<br><br>### How it works<br><br>This workflow currently contains no nodes... |
| When Telegram Message Received | n8n-nodes-base.telegramTrigger | Triggers workflow on incoming Telegram message | None | Prepare Message Data | |
| Prepare Message Data | n8n-nodes-base.set | Extracts chat ID, message text, and sender name | When Telegram Message Received | Message Handling Agent | chat_id ve gelen mesaj metnini alt akışa temiz şekilde taşır |
| Message Handling Agent | @n8n/n8n-nodes-langchain.agent | Core AI agent managing customer conversation and intent | Prepare Message Data | Send Response to Customer | |
| Gemini Conversation Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Provides language model capabilities (Gemini 1.5 Flash) | None | Message Handling Agent | |
| Conversation Memory Buffer | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains rolling short-term conversation context | None | Message Handling Agent | |
| Check Calendar Availability | n8n-nodes-base.googleCalendarTool | Tool for querying Google Calendar availability | None | Message Handling Agent | |
| Create Calendar Appointment | n8n-nodes-base.googleCalendarTool | Tool for scheduling events in Google Calendar | None | Message Handling Agent | |
| Lookup FAQs and Pricing | n8n-nodes-base.googleSheetsTool | Tool for retrieving pricing and FAQ information | None | Message Handling Agent | |
| If Appointment Created | n8n-nodes-base.if | Evaluates if an appointment was successfully booked | Send Response to Customer | Notify Business of Appointment | |
| Send Response to Customer | n8n-nodes-base.telegram | Sends sanitized AI response back to user on Telegram | Message Handling Agent | If Appointment Created, If Human Support Required | |
| Notify Business of Appointment | n8n-nodes-base.telegram | Alerts business owner about new bookings | If Appointment Created | None | |
| If Human Support Required | n8n-nodes-base.if | Evaluates if human intervention or complaint handling is needed | Send Response to Customer | Send Support Alert to Admin | |
| Send Support Alert to Admin | n8n-nodes-base.telegram | Alerts business owner about support requests or complaints | If Human Support Required | None | |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:** Add a **When Telegram Message Received** node (`n8n-nodes-base.telegramTrigger`). Set updates to `message` and configure your Telegram bot credentials and webhook ID (`telegram-randevu-webhook`).
2. **Add Data Preparation:** Create a **Prepare Message Data** node (`n8n-nodes-base.set`). Assign three string variables: `chat_id` (`={{ $json.message.chat.id }}`), `user_message` (`={{ $json.message.text }}`), and `user_name` (`={{ $json.message.from.first_name }}`). Connect `When Telegram Message Received` to this node.
3. **Configure the AI Agent:** Add a **Message Handling Agent** node (`@n8n/n8n-nodes-langchain.agent`). Set the text input to `={{ $json.user_message }}` and configure the system message with intent rules (`RANDEVU_TALEBI`, `FIYAT_SORUSU`, `SIKAYET`, `GENEL_SORU`) and tags (`RANDEVU_OLUSTU`, `İNSAN_DESTEK_GEREKLI`). Connect `Prepare Message Data` to its main input.
4. **Add the Language Model:** Create a **Gemini Conversation Model** node (`@n8n/n8n-nodes-langchain.lmChatGoogleGemini`). Select `models/gemini-1.5-flash`, set temperature to `0.3`, add Google AI credentials, and connect it to the agent's model input.
5. **Add Conversation Memory:** Create a **Conversation Memory Buffer** node (`@n8n/n8n-nodes-langchain.memoryBufferWindow`). Set context window length to `10` and session key to `={{ $('Prepare Message Data').item.json.chat_id }}` with custom key type. Connect it to the agent's memory input.
6. **Add Google Calendar Tools:**
   - Create **Check Calendar Availability** (`n8n-nodes-base.googleCalendarTool`). Set operation to `getAll` on calendar `primary`. Configure Google Calendar OAuth2 credentials and connect to the agent's tool input.
   - Create **Create Calendar Appointment** (`n8n-nodes-base.googleCalendarTool`). Set start time to `={{ $fromAI('start_datetime') }}`, end time to `={{ $fromAI('end_datetime') }}`, summary to `=Randevu - {{ $fromAI('customer_name') }}`, and description to `={{ $fromAI('customer_note') }}`. Connect to the agent's tool input.
7. **Add Google Sheets Tool:** Create **Lookup FAQs and Pricing** (`n8n-nodes-base.googleSheetsTool`). Set document ID to your spreadsheet identifier (`GOOGLE_SHEETS_FAQ_DOC_ID`), configure Google Sheets OAuth2 credentials, and connect to the agent's tool input.
8. **Add Customer Response Dispatcher:** Create a **Send Response to Customer** node (`n8n-nodes-base.telegram`). Set chat ID to `={{ $('Prepare Message Data').item.json.chat_id }}` and text to `={{ $json.output.replace('RANDEVU_OLUSTU', '').replace('İNSAN_DESTEK_GEREKLI', '') }}` with parse mode set to Markdown. Connect `Message Handling Agent` to this node.
9. **Add Appointment Routing & Notification:**
   - Create an **If Appointment Created** node (`n8n-nodes-base.if`). Add a condition checking if `={{ $json.output }}` contains `RANDEVU_OLUSTU`. Connect `Send Response to Customer` to its main input.
   - Create a **Notify Business of Appointment** node (`n8n-nodes-base.telegram`). Set chat ID to your owner chat ID (`BUSINESS_OWNER_TELEGRAM_CHAT_ID`) and configure the notification text template. Connect `If Appointment Created` to this node.
10. **Add Support Routing & Alerting:**
    - Create an **If Human Support Required** node (`n8n-nodes-base.if`). Add a condition checking if `={{ $json.output }}` contains `İNSAN_DESTEK_GEREKLI`. Connect `Send Response to Customer` to its main input.
    - Create a **Send Support Alert to Admin** node (`n8n-nodes-base.telegram`). Set chat ID to `BUSINESS_OWNER_TELEGRAM_CHAT_ID` and configure the alert text template. Connect `If Human Support Required` to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Telegram Otomatik Yanıt + Randevu Botu | Workflow title and functional scope covering Telegram integration, Google Gemini processing, Calendar management, and Sheets lookups. |
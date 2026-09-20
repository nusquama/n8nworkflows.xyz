Triage and reply to Gmail emails with GPT-4.1 mini and Telegram

https://n8nworkflows.xyz/workflows/triage-and-reply-to-gmail-emails-with-gpt-4-1-mini-and-telegram-17690


# Triage and reply to Gmail emails with GPT-4.1 mini and Telegram

### 1. Workflow Overview

This workflow is designed to automate email triage, categorization, and response routing for a Gmail inbox. It operates continuously by checking for unread messages, analyzing their semantic and contextual content using an advanced language model, and routing them based on predefined business criteria.

The logical execution is divided into four distinct operational blocks:
- **1.1 Input Reception & Retrieval:** Monitors the inbox for incoming unread emails on a schedule and fetches the complete message payload (headers, metadata, and HTML body).
- **1.2 AI Processing & Extraction:** Passes the email content to an OpenAI model configured with a structured system prompt to analyze priority, sentiment, category, and required actions, outputting a strictly formatted JSON structure.
- **1.3 Parsing & Decision Routing:** Sanitizes and validates the LLM response (with fallback mechanisms for malformed outputs) and directs the workflow down one of three branches using a conditional switch node.
- **1.4 Outcome Execution:** Applies the final action according to the routing decision—sending an automated reply, flagging for human review with a Telegram notification, or marking the message as spam.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception & Retrieval
- **Overview:** This block establishes the entry point of the automation by periodically checking for unread email items and retrieving their full raw content for downstream processing.
- **Nodes Involved:** 
  - `When New Email Arrives`
  - `Get Email from Gmail`
- **Node Details:**
  - **When New Email Arrives**
    - *Type and Technical Role:* Gmail Trigger node (`n8n-nodes-base.gmailTrigger`). Acts as the workflow initiator.
    - *Configuration Choices:* Filtered to monitor the `INBOX` label for `unread` status messages, excluding spam and trash. Polling frequency is set to run every minute.
    - *Key Expressions or Variables:* None (triggers execution context).
    - *Input and Output Connections:* Input: None (trigger). Output: Connects to `Get Email from Gmail`.
    - *Version-Specific Requirements:* Version 1.4. Requires valid OAuth2 credentials.
    - *Edge Cases / Potential Failure Types:* OAuth token expiration, API rate limits, or network timeouts during polling.
  - **Get Email from Gmail**
    - *Type and Technical Role:* Gmail node (`n8n-nodes-base.gmail`). Retrieves full message payloads.
    - *Configuration Choices:* Operation set to `get` with `simple` mode disabled to ensure full headers and body parts are accessible.
    - *Key Expressions or Variables:* Message ID parameter uses `{{ $json.id }}` referencing the trigger output.
    - *Input and Output Connections:* Input: `When New Email Arrives`. Output: Connects to `AI Email Analysis Agent`.
    - *Version-Specific Requirements:* Version 2.2.
    - *Edge Cases / Potential Failure Types:* Message deleted before retrieval, resulting in a 404 error from the Gmail API.

#### Block 1.2: AI Processing & Extraction
- **Overview:** Combines the incoming email data with an extensive system and user prompt to classify the communication, determine sentiment and priority, and formulate a response strategy using an OpenAI chat model.
- **Nodes Involved:**
  - `AI Email Analysis Agent`
  - `OpenAI GPT-4 Mini`
- **Node Details:**
  - **AI Email Analysis Agent**
    - *Type and Technical Role:* LangChain LLM Chain node (`@n8n/n8n-nodes-langchain.chainLlm`). Coordinates the prompt template, chat model, and structured output expectations.
    - *Configuration Choices:* Prompt type set to `define`. Text input constructs an analysis request incorporating headers (`from`, `to`, `subject`) and the email body (`html`). System messages establish business rules for categorization (Sales, Support, Billing, Security, Partnership, HR, Spam, General), priority levels, sentiment detection, reply types (Auto Reply, Human Review, No Reply), and explicit JSON output schemas.
    - *Key Expressions or Variables:* 
      - `{{ $json.headers.from }}`
      - `{{ $json.headers.to }}`
      - `{{ $json.headers.subject }}`
      - `{{ $json.html }}`
    - *Input and Output Connections:* Input: `Get Email from Gmail`. Output: Connects to `Parse AI Response`.
    - *Version-Specific Requirements:* Version 1.9.
    - *Edge Cases / Potential Failure Types:* LLM token limits exceeded on extremely large email threads; non-JSON compliant outputs (handled downstream).
  - **OpenAI GPT-4 Mini**
    - *Type and Technical Role:* OpenAI Chat Model sub-node (`@n8n/n8n-nodes-langchain.lmChatOpenAi`). Provides the underlying intelligence for the LLM chain.
    - *Configuration Choices:* Model selected as `gpt-4.1-mini`. Max tokens set to unlimited (`-1`).
    - *Key Expressions or Variables:* None.
    - *Input and Output Connections:* Connected to `AI Email Analysis Agent` via AI language model connection interface.
    - *Version-Specific Requirements:* Version 1.3. Requires valid OpenAI API credentials.
    - *Edge Cases / Potential Failure Types:* API key revocation, insufficient credit balance, or OpenAI service outages.

#### Block 1.3: Parsing & Decision Routing
- **Overview:** Sanitizes the raw string output from the LLM, parses it safely into a structured JavaScript object, enforces strict enum validation, and routes the workflow based on the assigned reply strategy.
- **Nodes Involved:**
  - `Parse AI Response`
  - `Route by AI Decision`
- **Node Details:**
  - **Parse AI Response**
    - *Type and Technical Role:* Code node (`n8n-nodes-base.code`). Executes custom JavaScript to clean markdown formatting wrappers, parse JSON safely, apply fallback structures if parsing fails, and validate enums.
    - *Configuration Choices:* JavaScript execution environment processing `$json.text` or `$json.output`. Includes a `try/catch` block that injects a default fallback JSON object containing a `parse_error` tag and `Human Review` routing if JSON decoding fails. Validates categories, priorities, sentiments, and reply types against strict allowed arrays.
    - *Key Expressions or Variables:* `{{ $json.text }}` / `{{ $json.output }}`.
    - *Input and Output Connections:* Input: `AI Email Analysis Agent`. Output: Connects to `Route by AI Decision`.
    - *Version-Specific Requirements:* Version 2.
    - *Edge Cases / Potential Failure Types:* Syntax errors inside the custom script if input structures deviate radically from expectations.
  - **Route by AI Decision**
    - *Type and Technical Role:* Switch node (`n8n-nodes-base.switch`). Directs execution flow based on evaluated expressions.
    - *Configuration Choices:* Configured with three rules matching the expression `{{$json.output.reply_type}}` against:
      1. `Auto Reply`
      2. `Human Review`
      3. `No Reply`
    - *Key Expressions or Variables:* `{{$json.output.reply_type}}`
    - *Input and Output Connections:* Input: `Parse AI Response`. Output: Three distinct output ports connecting respectively to `Auto Reply via Gmail`, `Label for Human Review`, and `Label as Spam in Gmail`.
    - *Version-Specific Requirements:* Version 3.4.
    - *Edge Cases / Potential Failure Types:* Unmatched strings fall through all outputs if validation fails (mitigated by the upstream code node enforcing strict enums).

#### Block 1.4: Outcome Execution
- **Overview:** Executes the final actions determined by the routing logic, updating Gmail labels or sending automated replies and external notifications.
- **Nodes Involved:**
  - `Auto Reply via Gmail`
  - `Label for Human Review`
  - `Send Telegram Notification`
  - `Label as Spam in Gmail`
- **Node Details:**
  - **Auto Reply via Gmail**
    - *Type and Technical Role:* Gmail node (`n8n-nodes-base.gmail`). Sends an automated response email.
    - *Configuration Choices:* Operation set to `reply`. Email type set to `text`.
    - *Key Expressions or Variables:* 
      - Message content: `{{ $json.output.suggested_reply }}`
      - Message ID: `{{ $('Get Email from Gmail').item.json.id }}`
    - *Input and Output Connections:* Input: Output branch 0 of `Route by AI Decision`. Output: None (terminal node).
    - *Version-Specific Requirements:* Version 2.2.
    - *Edge Cases / Potential Failure Types:* Empty suggested reply strings or API errors during reply threading.
  - **Label for Human Review**
    - *Type and Technical Role:* Gmail node (`n8n-nodes-base.gmail`). Applies organizational labels to emails.
    - *Configuration Choices:* Operation set to `addLabels`.
    - *Key Expressions or Variables:* Message ID: `{{ $('Get Email from Gmail').item.json.id }}`.
    - *Input and Output Connections:* Input: Output branch 1 of `Route by AI Decision`. Output: Connects to `Send Telegram Notification`.
    - *Version-Specific Requirements:* Version 2.2.
    - *Edge Cases / Potential Failure Types:* Non-existent label IDs configured in the Gmail account.
  - **Send Telegram Notification**
    - *Type and Technical Role:* Telegram node (`n8n-nodes-base.telegram`). Sends an alert message to a configured chat.
    - *Configuration Choices:* Text field structured with dynamic markdown summary variables. Attribution append disabled.
    - *Key Expressions or Variables:* 
      - Subject: `{{ $('Get Email from Gmail').item.json.headers.subject }}`
      - From: `{{ $('Get Email from Gmail').item.json.headers.from }}`
      - Company: `{{ $('Parse AI Response').item.json.output.company }}`
      - Category: `{{ $('Parse AI Response').item.json.output.category }}`
      - Priority: `{{ $('Parse AI Response').item.json.output.priority }}`
      - Confidence: `{{ $('Parse AI Response').item.json.output.confidence }}`
      - Summary: `{{ $('Parse AI Response').item.json.output.summary }}`
      - Action Required: `{{ $('Parse AI Response').item.json.output.requires_human }}`
      - Suggested Reply: `{{ $('Parse AI Response').item.json.output.suggested_reply }}`
      - Chat ID: Configured target chat identifier.
    - *Input and Output Connections:* Input: `Label for Human Review`. Output: None (terminal node).
    - *Version-Specific Requirements:* Version 1.2. Requires valid Telegram Bot credentials.
    - *Edge Cases / Potential Failure Types:* Invalid chat ID or bot lacking permissions to post in the target channel/chat.
  - **Label as Spam in Gmail**
    - *Type and Technical Role:* Gmail node (`n8n-nodes-base.gmail`). Flags unwanted incoming emails.
    - *Configuration Choices:* Operation set to `addLabels`.
    - *Key Expressions or Variables:* Message ID: `{{ $('Get Email from Gmail').item.json.id }}`.
    - *Input and Output Connections:* Input: Output branch 2 of `Route by AI Decision`. Output: None (terminal node).
    - *Version-Specific Requirements:* Version 2.2.
    - *Edge Cases / Potential Failure Types:* Target label missing or invalid message references.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When New Email Arrives | `n8n-nodes-base.gmailTrigger` | Triggers workflow on new unread Gmail messages | None | Get Email from Gmail | ## My workflow 3<br><br>### How it works<br><br>This workflow monitors Gmail for new emails, fetches the full message details, and uses an OpenAI-powered LLM chain to analyze the email. It safely parses the AI output as JSON, then routes the email through a switch into different handling paths such as auto-reply, human review notification, or spam handling.<br><br>### Setup steps<br><br>- Connect and authorize the Gmail credentials for the trigger, email fetch, labeling, auto-reply, and spam actions.<br>- Configure the OpenAI credential/model used by the GPT-4.1 Mini chat model attached to the AI Email Analysis chain.<br>- Set the expected AI prompt/output schema so Validate AI JSON can parse the response reliably and the Switch can route on the parsed fields.<br>- Configure the Telegram bot credentials and target chat for human review notifications.<br><br>### Customization<br><br>Adjust the AI classification prompt, Switch rules, Gmail labels, auto-reply message, spam action, and Telegram notification text to match your inbox triage policy. |
| Get Email from Gmail | `n8n-nodes-base.gmail` | Retrieves full email message data | When New Email Arrives | AI Email Analysis Agent | ## Fetch and analyze email<br><br>Watches for new Gmail messages, retrieves the full email details, and sends the content into the AI analysis chain using the attached GPT-4.1 Mini model. |
| AI Email Analysis Agent | `@n8n/n8n-nodes-langchain.chainLlm` | Analyzes email content via LLM and outputs JSON structure | Get Email from Gmail | Parse AI Response | ## Fetch and analyze email<br><br>Watches for new Gmail messages, retrieves the full email details, and sends the content into the AI analysis chain using the attached GPT-4.1 Mini model. |
| OpenAI GPT-4 Mini | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Provides the language model backend | None | AI Email Analysis Agent | ## Fetch and analyze email<br><br>Watches for new Gmail messages, retrieves the full email details, and sends the content into the AI analysis chain using the attached GPT-4.1 Mini model. |
| Parse AI Response | `n8n-nodes-base.code` | Cleans, parses, and validates the LLM response JSON | AI Email Analysis Agent | Route by AI Decision | ## Validate and route<br><br>Parses the AI response safely as JSON, then uses a switch to choose the appropriate handling path based on the analysis result. |
| Route by AI Decision | `n8n-nodes-base.switch` | Routes execution based on `reply_type` | Parse AI Response | Auto Reply via Gmail, Label for Human Review, Label as Spam in Gmail | ## Validate and route<br><br>Parses the AI response safely as JSON, then uses a switch to choose the appropriate handling path based on the analysis result. |
| Auto Reply via Gmail | `n8n-nodes-base.gmail` | Sends automated reply to the email sender | Route by AI Decision | None | ## Apply email outcomes<br><br>Executes the selected branch actions: sending an automatic Gmail reply, labeling messages for human review and notifying Telegram, or marking messages as spam. |
| Label for Human Review | `n8n-nodes-base.gmail` | Applies review label to the email | Route by AI Decision | Send Telegram Notification | ## Apply email outcomes<br><br>Executes the selected branch actions: sending an automatic Gmail reply, labeling messages for human review and notifying Telegram, or marking messages as spam. |
| Send Telegram Notification | `n8n-nodes-base.telegram` | Sends review summary alert to Telegram chat | Label for Human Review | None | ## Apply email outcomes<br><br>Executes the selected branch actions: sending an automatic Gmail reply, labeling messages for human review and notifying Telegram, or marking messages as spam. |
| Label as Spam in Gmail | `n8n-nodes-base.gmail` | Applies spam label to unwanted email | Route by AI Decision | None | ## Apply email outcomes<br><br>Executes the selected branch actions: sending an automatic Gmail reply, labeling messages for human review and notifying Telegram, or marking messages as spam. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Gmail Trigger** node (`When New Email Arrives`).
   - Set filters: `labelIds` = `INBOX`, `readStatus` = `unread`, `includeSpamTrash` = `false`.
   - Set polling interval to run `everyMinute`.
   - Connect a valid **Gmail OAuth2** credential.

2. **Create the Email Retrieval Node:**
   - Add a **Gmail** node (`Get Email from Gmail`).
   - Set operation to `Get` (`get`), uncheck `Simple` mode, and set Message ID to `={{ $json.id }}`.
   - Connect it downstream from the trigger node.

3. **Configure the AI Analysis Chain:**
   - Add an **Advanced AI -> Chain LLM** node (`AI Email Analysis Agent`).
   - Set prompt type to `Define`.
   - Insert the comprehensive system prompt defining business rules, categories (Sales, Support, Billing, Security, Partnership, HR, Spam, General), priorities, sentiment, and the required JSON output schema (`{ "output": { "category": "", "priority": "", ... } }`).
   - Configure the text input to inject headers (`from`, `to`, `subject`) and body (`html`) using expressions referencing the Get Email node output.
   - Add an **OpenAI Chat Model** sub-node (`OpenAI GPT-4 Mini`), configure model as `gpt-4.1-mini`, set max tokens to `-1`, and attach an **OpenAI API** credential.
   - Connect the AI Language Model output of the OpenAI node into the LLM Chain node.
   - Connect the main output of `Get Email from Gmail` into `AI Email Analysis Agent`.

4. **Add and Configure the Parser Code Node:**
   - Add a **Code** node (`Parse AI Response`).
   - Paste JavaScript code that handles JSON string sanitization, markdown removal, fallback JSON generation upon syntax errors, and enum validation against expected category, priority, sentiment, and reply type lists.
   - Connect the output of `AI Email Analysis Agent` into this node.

5. **Configure the Decision Switch Node:**
   - Add a **Switch** node (`Route by AI Decision`).
   - Create 3 output routing rules based on expression `={{$json.output.reply_type}}`:
     - Rule 0: Equals `Auto Reply`
     - Rule 1: Equals `Human Review`
     - Rule 2: Equals `No Reply`
   - Connect `Parse AI Response` output into this switch node.

6. **Build the Auto-Reply Branch:**
   - Add a **Gmail** node (`Auto Reply via Gmail`).
   - Set operation to `Reply` (`reply`), email type to `text`.
   - Set message content to `={{ $json.output.suggested_reply }}` and message ID to `={{ $('Get Email from Gmail').item.json.id }}`.
   - Connect output branch 0 of the Switch node to this node.

7. **Build the Human Review Branch:**
   - Add a **Gmail** node (`Label for Human Review`).
   - Set operation to `Add Labels` (`addLabels`) and provide the target message ID `={{ $('Get Email from Gmail').item.json.id }}`.
   - Connect output branch 1 of the Switch node to this node.
   - Add a **Telegram** node (`Send Telegram Notification`).
   - Configure the message text using markdown formatting referencing email headers and parsed AI output fields (subject, sender, company, category, priority, confidence, summary, required action, suggested reply).
   - Set the `chatId` parameter and connect a valid **Telegram Bot** credential.
   - Connect `Label for Human Review` to `Send Telegram Notification`.

8. **Build the Spam Branch:**
   - Add a **Gmail** node (`Label as Spam in Gmail`).
   - Set operation to `Add Labels` (`addLabels`) with message ID `={{ $('Get Email from Gmail').item.json.id }}`.
   - Connect output branch 2 of the Switch node to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| This workflow is designed as a starting point for AI-powered email automation utilizing structured JSON validation to ensure reliable routing decisions. | Workflow Description and Architecture Overview |
| Credentials required: Gmail OAuth2, OpenAI API, and Telegram Bot. | Integration Setup Requirements |
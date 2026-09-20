Summarize meeting transcripts with DeepSeek and email summaries via Gmail

https://n8nworkflows.xyz/workflows/summarize-meeting-transcripts-with-deepseek-and-email-summaries-via-gmail-17531


# Summarize meeting transcripts with DeepSeek and email summaries via Gmail

### 1. Workflow Overview

This workflow accepts meeting transcripts via an incoming webhook, processes them using the DeepSeek chat-completions API to extract structured insights (such as summaries, decisions, action items, objections, and next steps), and sends the formatted result via plain-text email using Gmail. 

The logic is divided into the following sequential blocks:
- **1.1 Input Reception:** Triggers the workflow upon receiving a POST request and normalizes the payload fields and transcript length.
- **1.2 AI Processing:** Sends the prepared meeting title and transcript to the DeepSeek API with a strict JSON response schema, handling potential extraction failures gracefully.
- **1.3 Email Distribution:** Formats the extracted JSON insights into an organized plain-text report and sends it to a configured recipient through Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
- **Overview:** Receives raw webhook data containing meeting details, handles variable payload field names, assigns a fallback title if needed, and caps the transcript size.
- **Nodes Involved:** 
  - `Receive Transcript Webhook`
  - `Normalize Transcript Data`
- **Node Details:**
  - **Receive Transcript Webhook**
    - *Type and Technical Role:* `n8n-nodes-base.webhook` (Webhook Trigger). Listens for incoming HTTP POST requests.
    - *Configuration Choices:* Configured to listen on path `meeting-notes-lite` for HTTP `POST` requests.
    - *Key Expressions or Variables:* None (triggers execution).
    - *Input/Output Connections:* Output connects to `Normalize Transcript Data`.
    - *Edge Cases / Potential Failures:* Missing POST headers or invalid endpoint paths will result in unhandled HTTP errors or 404 responses.
  - **Normalize Transcript Data**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code execution). Cleans, extracts, and truncates raw input.
    - *Configuration Choices:* Reads payload from `body` or root properties. Checks fields `transcript`, `text`, or `notes`. Defaults title to `'Untitled meeting'` and slices transcript length to a maximum of 60,000 characters.
    - *Key Expressions or Variables:* Uses JavaScript `String()`, `.trim()`, and `.slice(0, 60000)`.
    - *Input/Output Connections:* Input from `Receive Transcript Webhook`; output connects to `Post to DeepSeek API`.
    - *Edge Cases / Potential Failures:* Throws errors if input payload is entirely empty or malformed JSON.

#### 2.2 AI Processing
- **Overview:** Transmits the normalized title and transcript to DeepSeek's chat completions endpoint, enforcing a strict JSON output schema that categorizes decisions separately from discussions and assigns action item owners.
- **Nodes Involved:**
  - `Post to DeepSeek API`
- **Node Details:**
  - **Post to DeepSeek API**
    - *Type and Technical Role:* `n8n-nodes-base.httpRequest` (HTTP Request). Communicates with external LLM APIs.
    - *Configuration Choices:* Sends a POST request to `https://api.deepseek.com/v1/chat/completions` with a 120-second timeout. Uses `HTTP Header Auth` authentication and requests JSON output mode (`response_format: { type: 'json_object' }`). Sets `temperature: 0` and `max_tokens: 4000`. Configured with `continueOnFail: true` to prevent workflow stoppage on API errors.
    - *Key Expressions or Variables:* Uses `={{ JSON.stringify({...}) }}` to construct the payload dynamically using `$json.title` and `$json.transcript`. System prompt enforces strict rules regarding language matching, separation of discussed vs. decided items, and mandatory action item owners.
    - *Input/Output Connections:* Input from `Normalize Transcript Data`; output connects to `Build Email Content`.
    - *Credentials:* Requires a `Header Auth account` credential with header name `Authorization` and value `Bearer <API_KEY>`.
    - *Edge Cases / Potential Failures:* Invalid API keys trigger 401 errors. Insufficient `max_tokens` when using reasoning models can result in empty outputs. Handled gracefully via `continueOnFail`.

#### 2.3 Email Distribution
- **Overview:** Parses the LLM response JSON (or applies a fallback message if extraction failed), formats the content into a readable plain-text report, and delivers it via Gmail.
- **Nodes Involved:**
  - `Build Email Content`
  - `Email Summary through Gmail`
- **Node Details:**
  - **Build Email Content**
    - *Type and Technical Role:* `n8n-nodes-base.code` (JavaScript Code execution). Transforms JSON data into formatted plain text.
    - *Configuration Choices:* Parses the output of `Post to DeepSeek API`. Falls back to `(AI extraction failed)` if parsing fails. Iterates through decisions, action items (formatting owners and due dates), objections, and next steps to generate a structured email body and subject line.
    - *Key Expressions or Variables:* Uses `$('Normalize Transcript Data').first().json.title` and `JSON.parse()`.
    - *Input/Output Connections:* Input from `Post to DeepSeek API`; output connects to `Email Summary through Gmail`.
    - *Edge Cases / Potential Failures:* Malformed JSON strings from the LLM are caught safely via `try/catch`, ensuring the workflow continues execution.
  - **Email Summary through Gmail**
    - *Type and Technical Role:* `n8n-nodes-base.gmail` (Gmail integration). Sends emails via the Gmail API.
    - *Configuration Choices:* Sends plain text emails (`emailType: 'text'`). 
    - *Key Expressions or Variables:* Uses `={{ $json.emailBody }}` for the message body and `={{ $json.subject }}` for the email subject line.
    - *Input/Output Connections:* Input from `Build Email Content`.
    - *Credentials:* Requires a `Gmail OAuth2 account` credential.
    - *Edge Cases / Potential Failures:* Expired OAuth tokens or invalid recipient addresses (`you@example.com`) will cause execution failure on this node.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `Receive Transcript Webhook` | `n8n-nodes-base.webhook` | Webhook Trigger | None | `Normalize Transcript Data` | ## Receive transcript input<br><br>Starts the workflow from an incoming webhook request and normalizes the submitted transcript into a consistent format: it accepts several common field names, falls back to a default title, and caps the transcript length before processing. |
| `Normalize Transcript Data` | `n8n-nodes-base.code` | Data Normalization | `Receive Transcript Webhook` | `Post to DeepSeek API` | ## Receive transcript input<br><br>Starts the workflow from an incoming webhook request and normalizes the submitted transcript into a consistent format: it accepts several common field names, falls back to a default title, and caps the transcript length before processing. |
| `Post to DeepSeek API` | `n8n-nodes-base.httpRequest` | AI Processing | `Normalize Transcript Data` | `Build Email Content` | ## Extract meeting summary<br><br>Sends the normalized transcript to the DeepSeek chat/completions API, which returns JSON containing the summary, decisions, action items with owners, objections, and the next step. |
| `Build Email Content` | `n8n-nodes-base.code` | Report Formatting | `Post to DeepSeek API` | `Email Summary through Gmail` | ## Email formatted summary<br><br>Formats the AI extraction result into an email-friendly structure and sends the final meeting summary through Gmail. If extraction failed, a clearly-marked placeholder is sent instead so no meeting is silently dropped. |
| `Email Summary through Gmail` | `n8n-nodes-base.gmail` | Email Dispatch | `Build Email Content` | None | ## Email formatted summary<br><br>Formats the AI extraction result into an email-friendly structure and sends the final meeting summary through Gmail. If extraction failed, a clearly-marked placeholder is sent instead so no meeting is silently dropped. |

*(Note: The comprehensive descriptive note covering background, setup, customization, and usage instructions is attached as a standalone reference note in the workflow layout.)*

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Webhook Trigger Node:**
   - Add a **Webhook** node.
   - Set **HTTP Method** to `POST`.
   - Set **Path** to `meeting-notes-lite`.
   - Name the node `Receive Transcript Webhook`.

2. **Create the Transcript Normalization Node:**
   - Add a **Code** node and connect it to `Receive Transcript Webhook`.
   - Set **Mode** to `Run Once for All Items`.
   - Add the following JavaScript code:
     ```javascript
     const raw = $input.first().json.body || $input.first().json;
     const transcript = String(raw.transcript || raw.text || raw.notes || '').trim();
     const title = String(raw.title || 'Untitled meeting');
     return [{ json: { title, transcript: transcript.slice(0, 60000) } }];
     ```
   - Name the node `Normalize Transcript Data`.

3. **Create the DeepSeek API Request Node:**
   - Add an **HTTP Request** node and connect it to `Normalize Transcript Data`.
   - Set **Method** to `POST`.
   - Set **URL** to `https://api.deepseek.com/v1/chat/completions`.
   - Configure **Authentication** to `Generic Credential Type` -> `Header Auth`. Set up your credential with Name `Authorization` and Value `Bearer <YOUR_API_KEY>`.
   - Under **Options**, set **Timeout** to `120000` ms and enable **Continue On Fail** (`continueOnFail: true`).
   - Set **Body Content Type** to `JSON`.
   - Set the JSON Body expression to:
     ```json
     ={{ JSON.stringify({ model: 'deepseek-chat', response_format: { type: 'json_object' }, temperature: 0, max_tokens: 4000, messages: [ { role: 'system', content: 'Extract structured data from this meeting transcript. Distinguish DISCUSSED vs DECIDED - only decided items in decisions. Every action item needs an owner, else \'unassigned\'. Same language as transcript. JSON only: {\'summary\': string max 150 words, \'decisions\': [string], \'action_items\': [{\'task\': string, \'owner\': string, \'due\': string}], \'objections\': [string], \'next_step\': string}' }, { role: 'user', content: 'Title: ' + $json.title + '\n\n' + $json.transcript } ] }) }}
     ```
   - Name the node `Post to DeepSeek API`.

4. **Create the Email Content Builder Node:**
   - Add a **Code** node and connect it to `Post to DeepSeek API`.
   - Set **Mode** to `Run Once for All Items`.
   - Add the following JavaScript code:
     ```javascript
     const title = $('Normalize Transcript Data').first().json.title;
     let d = { summary: '(AI extraction failed)', decisions: [], action_items: [], objections: [], next_step: '' };
     try { const j = JSON.parse($input.first().json.choices?.[0]?.message?.content); if (j.summary) d = j; } catch (e) {}
     const lines = [];
     lines.push('MEETING: ' + title, '', 'SUMMARY', d.summary || '', '', 'DECISIONS');
     lines.push(...(d.decisions?.length ? d.decisions.map(x => '- ' + x) : ['(none)']));
     lines.push('', 'ACTION ITEMS');
     lines.push(...(d.action_items?.length ? d.action_items.map(a => '- [' + (a.owner || 'unassigned') + '] ' + a.task + (a.due ? ' (due ' + a.due + ')' : '')) : ['(none)']));
     lines.push('', 'OBJECTIONS / RISKS');
     lines.push(...(d.objections?.length ? d.objections.map(x => '- ' + x) : ['(none)']));
     lines.push('', 'NEXT STEP', d.next_step || '(none)');
     return [{ json: { emailBody: lines.join('\n'), subject: 'Meeting summary: ' + title } }];
     ```
   - Name the node `Build Email Content`.

5. **Create the Gmail Dispatch Node:**
   - Add a **Gmail** node and connect it to `Build Email Content`.
   - Set **Resource** to `Message` and **Operation** to `Send`.
   - Configure **Authentication** using a valid **Gmail OAuth2** credential.
   - Set **Send To** to your recipient address (e.g., `you@example.com`).
   - Set **Subject** expression to `={{ $json.subject }}`.
   - Set **Message** expression to `={{ $json.emailBody }}`.
   - Set **Email Type** to `Text`.
   - Name the node `Email Summary through Gmail`.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Summarize meeting transcripts with DeepSeek and send results via Gmail | Main workflow overview and architectural design |
| Provider flexibility (OpenAI compatibility) | Endpoint can be adapted to `https://api.openai.com/v1/chat/completions` with model `gpt-4o-mini` |
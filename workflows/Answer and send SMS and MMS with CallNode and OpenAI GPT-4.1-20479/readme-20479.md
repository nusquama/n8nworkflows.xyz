Answer and send SMS and MMS with CallNode and OpenAI GPT-4.1

https://n8nworkflows.xyz/workflows/answer-and-send-sms-and-mms-with-callnode-and-openai-gpt-4-1-20479


# Answer and send SMS and MMS with CallNode and OpenAI GPT-4.1

### 1. Workflow Overview

This workflow integrates CallNode and OpenAI to provide automated, context-aware responses to inbound SMS and MMS messages, while also enabling manual outbound messaging via an n8n form. It combines webhook event processing, AI vision analysis, text generation, and HMAC-signed API requests to CallNode.

The operational logic is organized into the following functional blocks:
- **1.1 Input Reception & Configuration:** Receives inbound webhook messages from CallNode, verifies authentication headers, and injects global business configuration variables.
- **1.2 Media Assessment & Conditional Routing:** Inspects incoming messages for media attachments (images), routing them through an OpenAI Vision model if present to extract textual or visual context.
- **1.3 AI Processing & Response Formulation:** Processes the inbound text and image description using an OpenAI Chat Model supplemented by conversation history memory, generating a constrained business response and checking for image-generation instructions.
- **1.4 Webhook Reply & Branching:** Immediately returns the text response to CallNode to maintain SMS thread consistency, then branches execution based on whether an image generation marker was requested.
- **1.5 Picture Generation & API Dispatch:** If an image is requested, generates the visual content via OpenAI, converts it to base64, cryptographically signs an outbound MMS request using HMAC-SHA256, and submits it to CallNode.
- **1.6 Manual Outbound Form Processing:** Alternatively captures outbound messages submitted via an n8n form trigger, normalizes the recipient number, securely signs the payload, and sends an SMS/MMS via CallNode before displaying a completion confirmation.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Configuration
- **Overview:** Captures incoming HTTP POST webhooks from CallNode, enforces header authentication, and assigns default application settings such as the business name.
- **Nodes Involved:** `CallNode Webhook`, `Settings`

- **Node Details:**
  - **CallNode Webhook**
    - *Type and Role:* `n8n-nodes-base.webhook` (Webhook Trigger). Receives incoming text and picture messages from CallNode.
    - *Configuration:* Configured for POST requests using Header Auth (`httpHeaderAuth`).
    - *Key Expressions:* None (Trigger node).
    - *Input/Output:* Inputs: None. Outputs: Connects to `Settings`.
    - *Edge Cases/Failures:* Authentication failure returns 401 if header values mismatch. Unreachable public URL will prevent CallNode from delivering payloads.

  - **Settings**
    - *Type and Role:* `n8n-nodes-base.set` (Data Transformation / Assignment). Sets global runtime parameters.
    - *Configuration:* Assigns a string variable `businessName` to `"Your Business"`.
    - *Key Expressions:* None (Static assignment).
    - *Input/Output:* Input: `CallNode Webhook`. Output: `Has a picture?`.
    - *Edge Cases/Failures:* Low risk; static parameter assignment.

---

#### 1.2 Media Assessment & Conditional Routing
- **Overview:** Evaluates whether the incoming message contains media attachments. If images are present, it invokes an OpenAI vision analysis model to generate a descriptive context summary.
- **Nodes Involved:** `Has a picture?`, `Describe Picture`

- **Node Details:**
  - **Has a picture?**
    - *Type and Role:* `n8n-nodes-base.if` (Conditional Router). Checks if the inbound media array length is greater than zero.
    - *Configuration:* Evaluates array length of webhook media property.
    - *Key Expressions:* `={{ ($('CallNode Webhook').item.json.body.media || []).length }}` (Checks if `> 0`).
    - *Input/Output:* Input: `Settings`. Output 1 (True): `Describe Picture`. Output 2 (False): `AI Assistant`.
    - *Edge Cases/Failures:* Malformed JSON payloads could cause expression evaluations to resolve defensively via fallback empty arrays.

  - **Describe Picture**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.openAi` (OpenAI Advanced Node - Image Analysis). Analyzes attached images using `gpt-4.1-mini`.
    - *Configuration:* Operation set to `analyze` with input type `url`.
    - *Key Expressions:* `={{ ($('CallNode Webhook').item.json.body.media || []).join(',') }}`
    - *Input/Output:* Input: `Has a picture?` (True branch). Output: `AI Assistant`.
    - *Credentials Required:* OpenAI API (`openAiApi`).
    - *Edge Cases/Failures:* API rate limits, invalid media URLs, or unsupported image formats will cause execution faults unless error handling overrides are applied.

---

#### 1.3 AI Processing & Response Formulation
- **Overview:** Synthesizes the inbound message text, extracted image descriptions, business prompt rules, and conversation window memory to generate a short, context-aware customer reply.
- **Nodes Involved:** `AI Assistant`, `OpenAI Chat Model`, `Conversation Memory`

- **Node Details:**
  - **AI Assistant**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.agent` (AI Agent). Orchestrates the conversation flow using the language model and memory buffer.
    - *Configuration:* System message enforces plain-text constraints under 300 characters, business context restrictions, and image request markers (`[image: ...]`).
    - *Key Expressions:* Complex IIFE parsing body captions and image descriptions to form the prompt.
    - *Input/Output:* Inputs: `Has a picture?` (False branch) or `Describe Picture`. Connects language model and memory sub-nodes. Output: `Build CallNode Reply`.
    - *Edge Cases/Failures:* Context window overflow or prompt token limits.

  - **OpenAI Chat Model**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.lmChatOpenAi` (Language Model Sub-Node). Provides the underlying LLM (`gpt-4.1-mini`) for the agent.
    - *Configuration:* Model set to `gpt-4.1-mini`.
    - *Input/Output:* Connected exclusively to `AI Assistant` via AI language model connection.
    - *Credentials Required:* OpenAI API (`openAiApi`).

  - **Conversation Memory**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.memoryBufferWindow` (Memory Sub-Node). Maintains a rolling buffer window of chat history per sender/recipient pair.
    - *Configuration:* Context window length set to 10. Session key uses sender and receiver phone numbers.
    - *Key Expressions:* `={{ $('CallNode Webhook').item.json.body.from + ' ' + $('CallNode Webhook').item.json.body.to }}`
    - *Input/Output:* Connected exclusively to `AI Assistant` via AI memory connection.

---

#### 1.4 Webhook Reply & Branching
- **Overview:** Strips out internal image markers from the AI response, instantly responds to the incoming webhook so CallNode can deliver the text in the thread, and evaluates whether an image generation request was triggered.
- **Nodes Involved:** `Build CallNode Reply`, `Reply to CallNode`, `Wants a picture?`

- **Node Details:**
  - **Build CallNode Reply**
    - *Type and Role:* `n8n-nodes-base.set` (Data Transformation). Parses the AI output to isolate the clean text reply and extract any `[image: ...]` generation prompts.
    - *Configuration:* Assigns `reply` (cleaned text) and `imagePrompt` (extracted prompt).
    - *Key Expressions:* Uses regex matching to clean output and extract brackets (`[image: ...]`).
    - *Input/Output:* Input: `AI Assistant`. Output: `Reply to CallNode`.

  - **Reply to CallNode**
    - *Type and Role:* `n8n-nodes-base.respondToWebhook` (Webhook Response). Returns the HTTP response containing the text reply back to CallNode.
    - *Configuration:* Default response mode.
    - *Input/Output:* Input: `Build CallNode Reply`. Output: `Wants a picture?`.

  - **Wants a picture?**
    - *Type and Role:* `n8n-nodes-base.if` (Conditional Router). Determines if the AI response contained a non-empty image generation prompt.
    - *Configuration:* Checks if `imagePrompt` is not empty.
    - *Key Expressions:* `={{ $json.imagePrompt }}`
    - *Input/Output:* Input: `Reply to CallNode`. Output 1 (True): `API Settings`. Output 2 (False): Terminating path for text-only flows.

---

#### 1.5 Picture Generation & API Dispatch
- **Overview:** Generates requested images via OpenAI, converts the binary image to a base64 string, constructs an HMAC-signed API payload, and posts the MMS message back through the CallNode API.
- **Nodes Involved:** `API Settings`, `Started by a text?`, `Generate Picture`, `Picture to Text`, `Build Picture Request`, `Sign Picture Request`, `Send Picture`

- **Node Details:**
  - **API Settings**
    - *Type and Role:* `n8n-nodes-base.set` (Configuration Node). Houses CallNode API credentials and sending parameters.
    - *Configuration:* Assigns `apiKey`, `apiSecret`, and `fromNumber`.
    - *Input/Output:* Input: `Wants a picture?` (True branch) or `Send a Text` form completion. Output: `Started by a text?`.

  - **Started by a text?**
    - *Type and Role:* `n8n-nodes-base.if` (Conditional Router). Verifies whether the execution originated from an incoming webhook text or the outbound form.
    - *Configuration:* Evaluates execution status of the webhook node.
    - *Key Expressions:* `={{ $('CallNode Webhook').isExecuted }}`
    - *Input/Output:* Input: `API Settings`. Output 1 (True): `Generate Picture`. Output 2 (False): `Build Request`.

  - **Generate Picture**
    - *Type and Role:* `@n8n/n8n-nodes-langchain.openAi` (OpenAI Advanced Node - Image Generation). Generates an image using `gpt-image-1`.
    - *Configuration:* Model set to `gpt-image-1`, resolution `1024x1024`, quality `medium`. Error handling set to continue regular output.
    - *Key Expressions:* `={{ $('Build CallNode Reply').item.json.imagePrompt }}`
    - *Input/Output:* Input: `Started by a text?` (True branch). Output: `Picture to Text`.
    - *Credentials Required:* OpenAI API (`openAiApi`).

  - **Picture to Text**
    - *Type and Role:* `n8n-nodes-base.extractFromFile` (Binary Transformation). Converts generated binary image data into a base64-encoded property.
    - *Configuration:* Operation `binaryToPropery`, destination key `imageBase64`, binary property `data`. Error handling set to continue regular output.
    - *Input/Output:* Input: `Generate Picture`. Output: `Build Picture Request`.

  - **Build Picture Request**
    - *Type and Role:* `n8n-nodes-base.set` (Data Transformation). Formats the JSON payload structure containing recipient phone numbers, message body, and base64 media attachments.
    - *Configuration:* Creates `timestamp`, `idempotencyKey`, and serialized JSON `body`.
    - *Input/Output:* Input: `Picture to Text`. Output: `Sign Picture Request`.

  - **Sign Picture Request**
    - *Type and Role:* `n8n-nodes-base.crypto` (Cryptographic Utility). Generates an HMAC-SHA256 signature for API request verification.
    - *Configuration:* Action `hmac`, algorithm `SHA256`, encoding `hex`.
    - *Key Expressions:* `={{ $json.timestamp + String.fromCharCode(10) + $json.body }}`
    - *Input/Output:* Input: `Build Picture Request`. Output: `Send Picture`.

  - **Send Picture**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (HTTP Client). Dispatches the signed MMS request to the CallNode Messages API.
    - *Configuration:* POST method to `https://api.callnode.io/api/v1/messages`. Sends raw JSON body with custom authentication headers (`X-Api-Key`, `X-Timestamp`, `X-Signature`, `X-Idempotency-Key`).
    - *Input/Output:* Input: `Sign Picture Request`. Outputs: None (Terminal node).
    - *Edge Cases/Failures:* Invalid CallNode API credentials, expired timestamps, or incorrect HMAC secrets will result in 401/403 API rejections.

---

#### 1.6 Manual Outbound Form Processing
- **Overview:** Provides a web-accessible n8n form interface allowing users to manually compose and send outbound SMS or MMS messages through the CallNode API.
- **Nodes Involved:** `Send a Text`, `Build Request`, `Sign Request`, `Send Message`, `Message Sent`

- **Node Details:**
  - **Send a Text**
    - *Type and Role:* `n8n-nodes-base.formTrigger` (Form Trigger). Renders an n8n form capturing phone numbers, messages, and optional image links.
    - *Configuration:* Fields configured for phone number, message textarea, and optional image link.
    - *Input/Output:* Inputs: None (Trigger). Output: `API Settings`.

  - **Build Request**
    - *Type and Role:* `n8n-nodes-base.set` (Data Transformation). Normalizes phone numbers, builds timestamps, and creates the serialized JSON body for standard outbound messages.
    - *Configuration:* Prepares payload properties for standard or media-attached text dispatch.
    - *Input/Output:* Input: `Started by a text?` (False branch). Output: `Sign Request`.

  - **Sign Request**
    - *Type and Role:* `n8n-nodes-base.crypto` (Cryptographic Utility). Signs the outbound request body using HMAC-SHA256.
    - *Configuration:* Action `hmac`, SHA256, hex encoding.
    - *Input/Output:* Input: `Build Request`. Output: `Send Message`.

  - **Send Message**
    - *Type and Role:* `n8n-nodes-base.httpRequest` (HTTP Client). Posts the signed payload to the CallNode Messages API.
    - *Configuration:* POST request with custom HMAC and API key headers.
    - *Input/Output:* Input: `Sign Request`. Output: `Message Sent`.

  - **Message Sent**
    - *Type and Role:* `n8n-nodes-base.form` (Form Completion). Displays a completion confirmation page to the form submitter with delivery status details.
    - *Configuration:* Operation set to `completion`, displaying status codes and message IDs returned by CallNode.
    - *Input/Output:* Input: `Send Message`. Outputs: None (Terminal node).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Overview | n8n-nodes-base.stickyNote | Documentation container | None | None | ## CallNode AI messaging for texts and pictures... (Full guide: [callnode.io/integrations/n8n](https://callnode.io/integrations/n8n)) |
| Section 1 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Receive the message\n\nCallNode posts each incoming text or picture message to the webhook. Settings holds your business name. |
| Section 2 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Write the reply\n\nAn attached picture is described first. The AI assistant then answers, with a memory per conversation. |
| Section 3 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Reply to CallNode\n\nPuts the answer in a reply field and returns it. CallNode delivers it in the same thread. |
| Section 4 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Choose what to send\n\nA picture request or a form submission continues here. API Settings holds your CallNode API key, secret and sending number. |
| Section 5 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Make the picture\n\nOpenAI generates the picture. The file is then turned into text so it can travel inside the request. |
| Section 6 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Send the picture\n\nBuilds the request, signs it with your API secret and posts it to the CallNode messages API. |
| Section 7 | n8n-nodes-base.stickyNote | Documentation container | None | None | ## Send a text from the form\n\nBuilds and signs the request, sends it through the CallNode API and confirms on the form. |
| CallNode Webhook | n8n-nodes-base.webhook | Receives incoming messages | None | Settings | ## Receive the message... |
| Settings | n8n-nodes-base.set | Assigns business configuration | CallNode Webhook | Has a picture? | ## Receive the message... |
| Has a picture? | n8n-nodes-base.if | Checks for media attachments | Settings | Describe Picture, AI Assistant | ## Receive the message... / ## Write the reply... |
| Describe Picture | @n8n/n8n-nodes-langchain.openAi | Analyzes attached images | Has a picture? | AI Assistant | ## Write the reply... |
| AI Assistant | @n8n/n8n-nodes-langchain.agent | Generates context-aware replies | Describe Picture, Has a picture? | Build CallNode Reply | ## Write the reply... |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | LLM provider for agent | None (Sub-node) | AI Assistant | ## Write the reply... |
| Conversation Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Stores chat buffer history | None (Sub-node) | AI Assistant | ## Write the reply... |
| Build CallNode Reply | n8n-nodes-base.set | Formats text and image prompts | AI Assistant | Reply to CallNode | ## Reply to CallNode |
| Reply to CallNode | n8n-nodes-base.respondToWebhook | Returns webhook HTTP response | Build CallNode Reply | Wants a picture? | ## Reply to CallNode |
| Wants a picture? | n8n-nodes-base.if | Checks for image generation flag | Reply to CallNode | API Settings, (End) | ## Choose what to send... |
| Send a Text | n8n-nodes-base.formTrigger | Outbound message form trigger | None | API Settings | ## Send a text from the form... |
| API Settings | n8n-nodes-base.set | Sets CallNode API credentials | Wants a picture?, Send a Text | Started by a text? | ## Choose what to send... |
| Started by a text? | n8n-nodes-base.if | Routes inbound vs form requests | API Settings | Generate Picture, Build Request | ## Choose what to send... |
| Generate Picture | @n8n/n8n-nodes-langchain.openAi | Generates AI images | Started by a text? | Picture to Text | ## Make the picture... |
| Picture to Text | n8n-nodes-base.extractFromFile | Converts image binary to base64 | Generate Picture | Build Picture Request | ## Make the picture... |
| Build Picture Request | n8n-nodes-base.set | Formats MMS API payload | Picture to Text | Sign Picture Request | ## Send the picture... |
| Sign Picture Request | n8n-nodes-base.crypto | Generates HMAC signature | Build Picture Request | Send Picture | ## Send the picture... |
| Send Picture | n8n-nodes-base.httpRequest | Posts MMS via CallNode API | Sign Picture Request | None | ## Send the picture... |
| Build Request | n8n-nodes-base.set | Formats form message payload | Started by a text? | Sign Request | ## Send a text from the form... |
| Sign Request | n8n-nodes-base.crypto | Generates HMAC signature | Build Request | Send Message | ## Send a text from the form... |
| Send Message | n8n-nodes-base.httpRequest | Posts SMS/MMS via CallNode API | Sign Request | Message Sent | ## Send a text from the form... |
| Message Sent | n8n-nodes-base.form | Displays form completion page | Send Message | None | ## Send a text from the form... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Inbound Webhook:**
   - Add a **Webhook** node named `CallNode Webhook`.
   - Set HTTP Method to `POST`, Response Mode to `Response Node`, and Authentication to `Header Auth`.
   - Create an HTTP Header Auth credential with Header Name `X-CallNode-Secret` and a secure random string value.
2. **Add Global Settings:**
   - Add a **Set** node named `Settings`.
   - Add a string assignment: name `businessName`, value `Your Business`.
   - Connect `CallNode Webhook` to `Settings`.
3. **Configure Media Branching:**
   - Add an **If** node named `Has a picture?`.
   - Set condition to check if `{{ ($('CallNode Webhook').item.json.body.media || []).length }}` is greater than `0`.
   - Connect `Settings` to `Has a picture?`.
4. **Add Image Analysis (Vision):**
   - Add an **OpenAI** node named `Describe Picture` (Resource: `Image`, Operation: `analyze`).
   - Configure model `gpt-4.1-mini`, input type `url`, image URLs expression `={{ ($('CallNode Webhook').item.json.body.media || []).join(',') }}`.
   - Connect the `True` output of `Has a picture?` to `Describe Picture`.
   - Configure OpenAI API credentials.
5. **Configure the AI Agent & Sub-Nodes:**
   - Add an **AI Agent** node named `AI Assistant`. Set Prompt Type to `Define`.
   - Connect `Describe Picture` and the `False` output of `Has a picture?` into `AI Assistant`.
   - Add an **OpenAI Chat Model** sub-node (model `gpt-4.1-mini`) and connect it to `AI Assistant` (AI language model connection). Configure OpenAI credentials.
   - Add a **Buffer Window Memory** sub-node, set session key to `={{ $('CallNode Webhook').item.json.body.from + ' ' + $('CallNode Webhook').item.json.body.to }}`, context window length to `10`, and connect it to `AI Assistant` (AI memory connection).
   - Configure the AI Agent's system message with business details and formatting constraints as specified in the workflow description.
6. **Process and Return the Webhook Reply:**
   - Add a **Set** node named `Build CallNode Reply`.
   - Assign `reply` (`={{ ($json.output || '').replace(/[[].*?]/g, '').replace(/ +/g, ' ').trim() }}`) and `imagePrompt` (`={{ ((($json.output || '').match(/[[]image:(.*?)]/i) || ['', ''])[1]).trim() }}`).
   - Connect `AI Assistant` to `Build CallNode Reply`.
   - Add a **Respond to Webhook** node named `Reply to CallNode`. Connect `Build CallNode Reply` to it.
7. **Handle Image Requests & Outbound Triggering:**
   - Add an **If** node named `Wants a picture?`. Check if `imagePrompt` is not empty. Connect `Reply to CallNode` to it.
   - Add a **Form Trigger** node named `Send a Text` with fields: Phone number, Message (textarea), Image link (optional).
   - Add a **Set** node named `API Settings`. Assign `apiKey`, `apiSecret`, and `fromNumber`.
   - Connect the `True` output of `Wants a picture?` and the output of `Send a Text` to `API Settings`.
   - Add an **If** node named `Started by a text?`. Condition: `={{ $('CallNode Webhook').isExecuted }}` is true. Connect `API Settings` to it.
8. **Configure Image Generation & Dispatch:**
   - Add an **OpenAI** node named `Generate Picture` (Resource: `Image`, Operation: `generate`, model `gpt-image-1`, size `1024x1024`). Enable "Continue execution on fail". Connect `Started by a text?` (True branch) to it. Configure OpenAI credentials.
   - Add an **Extract From File** node named `Picture to Text` (operation `binaryToPropery`, destination `imageBase64`, binary property `data`). Enable error continuation. Connect `Generate Picture` to it.
   - Add a **Set** node named `Build Picture Request` to format timestamp, idempotency key, and JSON body containing base64 media. Connect `Picture to Text` to it.
   - Add a **Crypto** node named `Sign Picture Request` (Action: `hmac`, SHA256, hex encoding, value `={{ $json.timestamp + String.fromCharCode(10) + $json.body }}`). Connect `Build Picture Request` to it.
   - Add an **HTTP Request** node named `Send Picture` (POST `https://api.callnode.io/api/v1/messages`, raw JSON, custom headers for API Key, Timestamp, Signature, and Idempotency Key). Connect `Sign Picture Request` to it.
9. **Configure Form Outbound Messaging:**
   - Add a **Set** node named `Build Request` to parse form parameters and format the outbound request body. Connect `Started by a text?` (False branch) to it.
   - Add a **Crypto** node named `Sign Request` (Action: `hmac`, SHA256, hex encoding). Connect `Build Request` to it.
   - Add an **HTTP Request** node named `Send Message` (POST `https://api.callnode.io/api/v1/messages`, raw JSON, custom authentication headers). Connect `Sign Request` to it.
   - Add a **Form** (Completion) node named `Message Sent` with completion title and confirmation message. Connect `Send Message` to it.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Full integration and setup guide | [callnode.io/integrations/n8n](https://callnode.io/integrations/n8n) |
| CallNode platform website | [callnode.io](https://callnode.io) |
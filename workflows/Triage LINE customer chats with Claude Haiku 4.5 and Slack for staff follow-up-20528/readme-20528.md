Triage LINE customer chats with Claude Haiku 4.5 and Slack for staff follow-up

https://n8nworkflows.xyz/workflows/triage-line-customer-chats-with-claude-haiku-4-5-and-slack-for-staff-follow-up-20528


# Triage LINE customer chats with Claude Haiku 4.5 and Slack for staff follow-up

### 1. Workflow Overview

This workflow automates customer support triage and handling between LINE and Slack using Anthropic Claude. Its primary objective is to autonomously answer common customer inquiries (regarding shop information, product stock, and visit availability) using predefined data tables and configuration parameters, while routing complex or unanswerable queries to a dedicated Slack channel for human staff intervention. Staff replies inside Slack are automatically synced back to the customer via LINE.

The workflow logic is divided into four functional blocks:
- **1.1 LINE Ingestion and Verification:** Receives webhook events from LINE, validates cryptographic signatures, drops invalid requests, skips redelivered events, and filters for supported text and image message types.
- **1.2 AI Processing and Information Extraction:** Evaluates text messages or analyzes uploaded furniture images using Claude, classifies the intent into categories (shop info, stock, visit availability, or staff required), extracts relevant parameters, queries internal n8n data tables, and generates structured responses.
- **1.3 Automated Reply & Slack Handoff:** Decides whether the AI can definitively answer the inquiry. If answerable, it replies directly via the LINE Messaging API and logs the success. If not, it sends an acknowledgment message to the customer on LINE, determines if an existing Slack thread exists for that customer, and either updates or creates a thread in Slack.
- **1.4 Staff-to-LINE Synchronization:** Listens for staff replies in Slack threads, filters out internal notes and bot messages, maps the Slack thread back to the LINE user, forwards the message as a LINE push message, and updates the thread status with confirmation reactions or failure reports.

---

### 2. Block-by-Block Analysis

#### 2.1 LINE Ingestion and Verification
- **Overview:** Receives raw webhook payloads from LINE, verifies their HMAC SHA-256 cryptographic signatures to ensure security, filters out duplicates, and extracts core message metadata for incoming text and image messages.
- **Nodes Involved:** `Receive LINE webhook`, `Compute signature`, `Signature valid?`, `Drop invalid request`, `Config`, `Split events`, `Skip redelivered events`, `Extract message fields`, `Keep text and images`, `Text or image?`
- **Node Details:**
  - **Receive LINE webhook** (`n8n-nodes-base.webhook`): 
    - Role: Entry point for HTTP POST requests from LINE.
    - Config: Path `line-slack`, response mode `onReceived`, raw body enabled (`options.rawBody: true`).
    - Connections: Output to `Compute signature`.
    - Edge cases: Webhook URLs must be publicly accessible via HTTPS.
  - **Compute signature** (`n8n-nodes-base.crypto`): 
    - Role: Computes an HMAC SHA-256 signature (Base64 encoding) of the incoming raw request body using the LINE channel secret.
    - Config: Action `hmac`, type `SHA256`, encoding `base64`.
    - Connections: Input from `Receive LINE webhook`, output to `Signature valid?`.
  - **Signature valid?** (`n8n-nodes-base.if`): 
    - Role: Compares the computed cryptographic signature against the `x-line-signature` request header.
    - Config: Loose type validation checking if `computedSignature` equals `headers['x-line-signature']`.
    - Connections: True branch to `Config`, false branch to `Drop invalid request`.
  - **Drop invalid request** (`n8n-nodes-base.noOp`): 
    - Role: Terminates execution flow for invalid signatures.
    - Config: Standard No-Op node.
    - Connections: Input from `Signature valid?` (false).
  - **Config** (`n8n-nodes-base.set`): 
    - Role: Establishes global configuration parameters including timezone (`Asia/Tokyo`), currency (`USD`), customer acknowledgment text, and core shop information details.
    - Config: Manual assignments merging static config parameters with existing input fields.
    - Connections: Input from `Signature valid?` (true), output to `Split events`.
  - **Split events** (`n8n-nodes-base.splitOut`): 
    - Role: Iterates through batch event arrays received in the LINE webhook payload.
    - Config: Field to split out: `body.events`.
    - Connections: Input from `Config`, output to `Skip redelivered events`.
  - **Skip redelivered events** (`n8n-nodes-base.removeDuplicates`): 
    - Role: Prevents duplicate execution of previously processed webhook events.
    - Config: Deduplicates based on `webhookEventId` seen in prior executions.
    - Connections: Input from `Split events`, output to `Extract message fields`.
  - **Extract message fields** (`n8n-nodes-base.set`): 
    - Role: Normalizes incoming event properties into standard variables (`userId`, `replyToken`, `messageType`, `messageId`, `text`).
    - Config: Manual assignments extracting specific nested JSON properties from the LINE payload.
    - Connections: Input from `Skip redelivered events`, output to `Keep text and images`.
  - **Keep text and images** (`n8n-nodes-base.filter`): 
    - Role: Restricts message processing exclusively to text and image message types.
    - Config: Evaluates if `messageType` equals `text` or `image`.
    - Connections: Input from `Extract message fields`, output to `Text or image?`.
  - **Text or image?** (`n8n-nodes-base.switch`): 
    - Role: Routes execution paths based on whether the incoming message contains text or an image.
    - Config: Rule-based routing checking `messageType`.
    - Connections: Input from `Keep text and images`. Text output goes to `Use text as inquiry`; image output goes to `Download image`.

#### 2.2 AI Processing and Information Extraction
- **Overview:** Processes text inquiries directly or downloads image attachments to generate textual descriptions via Claude, classifies the intent, and extracts structured parameters to query internal database tables for answers.
- **Nodes Involved:** `Use text as inquiry`, `Download image`, `Describe photo`, `Use photo as inquiry`, `Inquiry`, `Claude Haiku 4.5`, `Can the AI answer?`, `Answer from shop info`, `Extract product category`, `Look up stock`, `Combine products`, `Answer from stock`, `Extract visit request`, `Look up visit slots`, `Combine slots`, `Answer from visit slots`, `AI answer`
- **Node Details:**
  - **Use text as inquiry** (`n8n-nodes-base.set`): 
    - Role: Normalizes text input into an `inquiry` property.
    - Config: Assigns `inquiry` using `{{ $("Extract message fields").item.json.text }}`.
    - Connections: Input from `Text or image?`, output to `Inquiry`.
  - **Download image** (`n8n-nodes-base.httpRequest`): 
    - Role: Fetches binary image content from LINE Messaging API servers using the message ID.
    - Config: GET request to `https://api-data.line.me/v2/bot/message/{{ $json.messageId }}/content` using HTTP Header Authentication. Output format: file binary data (`data`).
    - Connections: Input from `Text or image?`, output to `Describe photo`.
    - Edge cases: Requires valid LINE channel access token via Header Auth.
  - **Describe photo** (`@n8n/n8n-nodes-langchain.chainLlm`): 
    - Role: Uses Anthropic Claude to analyze the furniture image binary data and output a concise description.
    - Config: Uses linked Anthropic Chat Model (`Claude Haiku 4.5`). Prompt defines strict extraction criteria (type, color, material, rough size).
    - Connections: Input from `Download image`, output to `Use photo as inquiry`.
  - **Use photo as inquiry** (`n8n-nodes-base.set`): 
    - Role: Formulates a stock inquiry string based on the AI's image description.
    - Config: Assigns formatted text to `inquiry`.
    - Connections: Input from `Describe photo`, output to `Inquiry`.
  - **Inquiry** (`n8n-nodes-base.noOp`): 
    - Role: Convergence node merging text and image processing paths prior to classification.
    - Config: Standard No-Op.
    - Connections: Input from `Use text as inquiry` and `Use photo as inquiry`, output to `Can the AI answer?`.
  - **Claude Haiku 4.5** (`@n8n/n8n-nodes-langchain.lmChatAnthropic`): 
    - Role: Provides the foundational Anthropic Claude language model configuration for all downstream AI text classifier and information extractor nodes.
    - Config: Model set to `claude-haiku-4-5-20251001`. Requires valid Anthropic API credentials.
    - Connections: Linked as AI language model to all LangChain nodes in the workflow.
  - **Can the AI answer?** (`@n8n/n8n-nodes-langchain.textClassifier`): 
    - Role: Classifies customer inquiries into predefined categories (`Shop info`, `Stock`, `Visit availability`, or `Needs staff`).
    - Config: Uses system prompt template with strict classification rules and categories.
    - Connections: Input from `Inquiry`. Routes output dynamically: `Shop info` to `Answer from shop info`, `Stock` to `Extract product category`, `Visit availability` to `Extract visit request`, and `Needs staff` (or fallback) to `Send acknowledgment`.
  - **Answer from shop info** (`@n8n/n8n-nodes-langchain.informationExtractor`): 
    - Role: Extracts structured answers regarding general shop policies, hours, and location based strictly on config data.
    - Config: Schema attributes: `answerable` (boolean) and `reply` (string).
    - Connections: Input from `Can the AI answer?`, output to `AI answer`.
  - **Extract product category** (`@n8n/n8n-nodes-langchain.informationExtractor`): 
    - Role: Identifies the specific product category mentioned in the customer's inquiry.
    - Config: Schema attribute: `category` (string: Sofa, Table, Chair, Lighting, Rug, Storage, or Other).
    - Connections: Input from `Can the AI answer?`, output to `Look up stock`.
  - **Look up stock** (`n8n-nodes-base.dataTable`): 
    - Role: Queries the `products` n8n Data Table filtering by the extracted product category.
    - Config: Resource `row`, operation `get`, returning all matching rows where `category` equals the extracted category value. `alwaysOutputData: true`.
    - Connections: Input from `Extract product category`, output to `Combine products`.
  - **Combine products** (`n8n-nodes-base.aggregate`): 
    - Role: Aggregates multiple product rows into a single structured array object (`products`).
    - Config: Aggregate all item data into destination field `products`.
    - Connections: Input from `Look up stock`, output to `Answer from stock`.
  - **Answer from stock** (`@n8n/n8n-nodes-langchain.informationExtractor`): 
    - Role: Evaluates product inventory data against the customer's inquiry to generate an accurate stock reply.
    - Config: Schema attributes: `answerable` (boolean) and `reply` (string).
    - Connections: Input from `Combine products`, output to `AI answer`.
  - **Extract visit request** (`@n8n/n8n-nodes-langchain.informationExtractor`): 
    - Role: Extracts requested store visit dates and time preferences from customer messages.
    - Config: Schema attributes: `desired_date` (YYYY-MM-DD string) and `time_preference` (string).
    - Connections: Input from `Can the AI answer?`, output to `Look up visit slots`.
  - **Look up visit slots** (`n8n-nodes-base.dataTable`): 
    - Role: Queries the `visit_slots` n8n Data Table for availability matching the requested date.
    - Config: Resource `row`, operation `get`, matching `slot_date`. `alwaysOutputData: true`.
    - Connections: Input from `Extract visit request`, output to `Combine slots`.
  - **Combine slots** (`n8n-nodes-base.aggregate`): 
    - Role: Aggregates schedule slot rows into a single structured array object (`slots`).
    - Config: Aggregate all item data into destination field `slots`.
    - Connections: Input from `Look up visit slots`, output to `Answer from visit slots`.
  - **Answer from visit slots** (`@n8n/n8n-nodes-langchain.informationExtractor`): 
    - Role: Generates visit slot availability responses based on data table capacity checks.
    - Config: Schema attributes: `answerable` (boolean) and `reply` (string).
    - Connections: Input from `Combine slots`, output to `AI answer`.
  - **AI answer** (`n8n-nodes-base.noOp`): 
    - Role: Convergence node merging responses from all three AI answer generation paths (`Shop info`, `Stock`, `Visit slots`).
    - Config: Standard No-Op.
    - Connections: Input from extraction answer nodes, output to `Answered by AI?`.

#### 2.3 Automated Reply & Slack Handoff
- **Overview:** Evaluates if the generated AI answer is valid and complete. If valid, replies directly to the customer on LINE and logs success; otherwise, triggers a standard acknowledgment message, checks for an existing customer thread in Slack, and creates or updates the Slack thread accordingly.
- **Nodes Involved:** `Answered by AI?`, `Send AI reply`, `Send acknowledgment`, `Log AI answer`, `Log needs staff`, `Post to Slack`, `Find thread`, `Thread exists?`, `Reply in thread`, `Update thread status`, `Get LINE display name`, `Create new thread`, `Save thread`
- **Node Details:**
  - **Answered by AI?** (`n8n-nodes-base.if`): 
    - Role: Determines whether the AI successfully answered the inquiry (`answerable === true` and reply text is non-empty).
    - Config: Evaluates expression confirming validity of AI output.
    - Connections: True branch to `Send AI reply`; false branch to `Send acknowledgment`.
  - **Send AI reply** (`n8n-nodes-base.httpRequest`): 
    - Role: Sends the AI-generated response back to the customer using the LINE reply API.
    - Config: POST request to `https://api.line.me/v2/bot/message/reply` with JSON body containing `replyToken` and message text. Error handling set to `continueRegularOutput`.
    - Connections: Input from `Answered by AI?` (true), output to `Log AI answer`.
  - **Send acknowledgment** (`n8n-nodes-base.httpRequest`): 
    - Role: Sends the fixed configuration acknowledgment message to the customer if the query requires staff intervention.
    - Config: POST request to `https://api.line.me/v2/bot/message/reply` using the configured acknowledgment text. Error handling set to `continueRegularOutput`.
    - Connections: Input from `Answered by AI?` (false) and `Can the AI answer?`, output to `Log needs staff`.
  - **Log AI answer** (`n8n-nodes-base.set`): 
    - Role: Sets status logs and metadata following an AI response event.
    - Config: Assigns `status: "ai_answered"`, formatted `note`, and `broadcast` boolean flag.
    - Connections: Input from `Send AI reply`, output to `Post to Slack`.
  - **Log needs staff** (`n8n-nodes-base.set`): 
    - Role: Sets status logs and metadata when routing an unanswerable inquiry to staff.
    - Config: Assigns `status: "needs_reply"`, formatted `note`, and `broadcast: true`.
    - Connections: Input from `Send acknowledgment`, output to `Post to Slack`.
  - **Post to Slack** (`n8n-nodes-base.noOp`): 
    - Role: Convergence node preparing log payloads for Slack thread mapping.
    - Config: Standard No-Op.
    - Connections: Input from `Log AI answer` and `Log needs staff`, output to `Find thread`.
  - **Find thread** (`n8n-nodes-base.dataTable`): 
    - Role: Searches the `line_slack_threads` Data Table to locate an existing Slack thread for the incoming LINE user ID.
    - Config: Resource `row`, operation `get`, limit 1, filtering by `line_user_id`. `alwaysOutputData: true`.
    - Connections: Input from `Post to Slack`, output to `Thread exists?`.
  - **Thread exists?** (`n8n-nodes-base.if`): 
    - Role: Checks if a valid Slack thread timestamp (`slack_ts`) was found for the customer.
    - Config: Evaluates if `slack_ts` is not empty.
    - Connections: True branch to `Reply in thread`; false branch to `Get LINE display name`.
  - **Reply in thread** (`n8n-nodes-base.slack`): 
    - Role: Posts a follow-up message into an existing Slack thread.
    - Config: Resource `message`, operation `post`, channel select, passing thread timestamp options.
    - Connections: Input from `Thread exists?` (true), output to `Update thread status`.
  - **Update thread status** (`n8n-nodes-base.dataTable`): 
    - Role: Updates status and last inquiry details for an existing thread in the data table.
    - Config: Resource `row`, operation `update`, mapping `status` and `last_inquiry` filtered by `line_user_id`.
    - Connections: Input from `Reply in thread`.
  - **Get LINE display name** (`n8n-nodes-base.httpRequest`): 
    - Role: Fetches the customer's display profile name from LINE when initiating a brand new Slack thread.
    - Config: GET request to `https://api.line.me/v2/bot/profile/{{ userId }}` using HTTP Header Authentication. Error handling set to `continueRegularOutput`.
    - Connections: Input from `Thread exists?` (false), output to `Create new thread`.
- **Create new thread** (`n8n-nodes-base.slack`): 
    - Role: Creates a brand new top-level message thread in Slack for a new customer inquiry.
    - Config: Resource `message`, operation `post`, channel select, formatting text with customer display name and inquiry details.
    - Connections: Input from `Get LINE display name`, output to `Save thread`.
  - **Save thread** (`n8n-nodes-base.dataTable`): 
    - Role: Saves the newly created Slack thread mapping (`slack_ts`, `line_user_id`, status, and inquiry) into the data table.
    - Config: Resource `row`, operation `insert`.
    - Connections: Input from `Create new thread`.

#### 2.4 Staff-to-LINE Synchronization
- **Overview:** Triggers when staff reply within Slack threads, validates message criteria (ignoring bot posts and internal notes), matches the thread back to the customer, pushes the reply to LINE, and confirms success or failure in Slack.
- **Nodes Involved:** `Slack message`, `Staff thread replies only`, `Find customer`, `Send to LINE`, `Sent?`, `Mark as sent`, `Mark thread replied`, `Report failure`
- **Node Details:**
  - **Slack message** (`n8n-nodes-base.slackTrigger`): 
    - Role: Entry point triggered whenever a message event occurs in the configured Slack channel.
    - Config: Watches workspace events for `message` subtype in selected channel.
    - Connections: Output to `Staff thread replies only`.
  - **Staff thread replies only** (`n8n-nodes-base.filter`): 
    - Role: Filters out unwanted Slack messages, ensuring only valid staff thread replies are processed.
    - Config: Conditions check that `thread_ts` is not empty, message `ts` does not equal `thread_ts` (ensuring it's a threaded reply, not a root post), `bot_id` is empty (ignoring bot messages), text does not start with `//` (ignoring internal staff notes), and message subtype is valid.
    - Connections: Input from `Slack message`, output to `Find customer`.
  - **Find customer** (`n8n-nodes-base.dataTable`): 
    - Role: Queries the `line_slack_threads` Data Table using the Slack thread timestamp (`thread_ts`) to retrieve the corresponding LINE user ID.
    - Config: Resource `row`, operation `get`, limit 1, filtering by `slack_ts`.
    - Connections: Input from `Staff thread replies only`, output to `Send to LINE`.
  - **Send to LINE** (`n8n-nodes-base.httpRequest`): 
    - Role: Forwards the staff's Slack message as a push message to the customer on LINE.
    - Config: POST request to `https://api.line.me/v2/bot/message/push` with JSON body containing recipient `to` (`line_user_id`) and message text. Error handling set to `continueRegularOutput`.
    - Connections: Input from `Find customer`, output to `Sent?`.
  - **Sent?** (`n8n-nodes-base.if`): 
    - Role: Checks whether the push message was successfully sent to the LINE API without errors.
    - Config: Evaluates if `$json.error` is absent (`ok` vs `ng`).
    - Connections: True branch to `Mark as sent`; false branch to `Report failure`.
  - **Mark as sent** (`n8n-nodes-base.slack`): 
    - Role: Adds a white check mark reaction (`white_check_mark`) to the staff message in Slack upon successful delivery.
    - Config: Resource `reaction`, operation `add`, timestamp tied to the Slack message `ts`.
    - Connections: Input from `Sent?` (true), output to `Mark thread replied`.
  - **Mark thread replied** (`n8n-nodes-base.dataTable`): 
    - Role: Updates the thread status in the data table to `replied`.
    - Config: Resource `row`, operation `update`, mapping `status: "replied"` filtered by `slack_ts`.
    - Connections: Input from `Mark as sent`.
  - **Report failure** (`n8n-nodes-base.slack`): 
    - Role: Posts an error warning message into the Slack thread if delivery to LINE fails.
    - Config: Resource `message`, operation `post`, posting error details into the respective thread timestamp.
    - Connections: Input from `Sent?` (false).

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Receive LINE webhook | n8n-nodes-base.webhook | Entry point for LINE webhooks | None | Compute signature | ## Setup (do this first)<br>1. Create credentials and select them in the nodes:<br>   - **Header Auth**: Name `Authorization`, Value `Bearer <LINE channel access token>` (nodes that call LINE)<br>   - **Crypto**: Hmac Secret = LINE channel secret (Compute signature)<br>   - **Anthropic** (Claude Haiku 4.5)<br>   - **Slack API**: Access Token = the bot token (`xoxb-`), Signature Secret = the Slack app's Signing Secret<br>2. Create three Data Tables and select them in the lookup and log nodes:<br>   - `products`: sku, name, category, color, width_cm, price, stock, note<br>   - `visit_slots`: slot_date (YYYY-MM-DD), slot_time, capacity, booked. **Add rows for upcoming dates. A date with no rows is answered as a day the shop does not take visits.**<br>   - `line_slack_threads`: line_user_id, slack_ts, status, last_inquiry (all strings)<br>3. Select the Slack channel in every Slack node. Invite the bot to that channel first.<br>4. Edit the **Config** node: timezone, currency, the acknowledgment text, and the shop information. The AI answers only from what you write there. The AI replies in the language the customer wrote in, but the acknowledgment text is fixed, so write it in your customers' language.<br>5. Publish. Paste the production webhook URL of **Receive LINE webhook** into the LINE Developers console (turn Webhook on, auto-reply messages off). Paste the production URL of **Slack message** into the Slack app's Event Subscriptions Request URL and add the bot event `message.channels`.<br><br>The AI replies only when the answer is decided by your data. Everything else gets a short acknowledgment and goes to Slack. |
| Compute signature | n8n-nodes-base.crypto | Computes HMAC SHA-256 signature | Receive LINE webhook | Signature valid? | ## Setup (do this first)<br>1. Create credentials and select them in the nodes:<br>   - **Header Auth**: Name `Authorization`, Value `Bearer <LINE channel access token>` (nodes that call LINE)<br>   - **Crypto**: Hmac Secret = LINE channel secret (Compute signature)<br>   - **Anthropic** (Claude Haiku 4.5)<br>   - **Slack API**: Access Token = the bot token (`xoxb-`), Signature Secret = the Slack app's Signing Secret<br>2. Create three Data Tables and select them in the lookup and log nodes:<br>   - `products`: sku, name, category, color, width_cm, price, stock, note<br>   - `visit_slots`: slot_date (YYYY-MM-DD), slot_time, capacity, booked. **Add rows for upcoming dates. A date with no rows is answered as a day the shop does not take visits.**<br>   - `line_slack_threads`: line_user_id, slack_ts, status, last_inquiry (all strings)<br>3. Select the Slack channel in every Slack node. Invite the bot to that channel first.<br>4. Edit the **Config** node: timezone, currency, the acknowledgment text, and the shop information. The AI answers only from what you write there. The AI replies in the language the customer wrote in, but the acknowledgment text is fixed, so write it in your customers' language.<br>5. Publish. Paste the production webhook URL of **Receive LINE webhook** into the LINE Developers console (turn Webhook on, auto-reply messages off). Paste the production URL of **Slack message** into the Slack app's Event Subscriptions Request URL and add the bot event `message.channels`.<br><br>The AI replies only when the answer is decided by your data. Everything else gets a short acknowledgment and goes to Slack. |
| Signature valid? | n8n-nodes-base.if | Validates LINE signature header | Compute signature | Config, Drop invalid request | ## Setup (do this first)<br>1. Create credentials and select them in the nodes:<br>   - **Header Auth**: Name `Authorization`, Value `Bearer <LINE channel access token>` (nodes that call LINE)<br>   - **Crypto**: Hmac Secret = LINE channel secret (Compute signature)<br>   - **Anthropic** (Claude Haiku 4.5)<br>   - **Slack API**: Access Token = the bot token (`xoxb-`), Signature Secret = the Slack app's Signing Secret<br>2. Create three Data Tables and select them in the lookup and log nodes:<br>   - `products`: sku, name, category, color, width_cm, price, stock, note<br>   - `visit_slots`: slot_date (YYYY-MM-DD), slot_time, capacity, booked. **Add rows for upcoming dates. A date with no rows is answered as a day the shop does not take visits.**<br>   - `line_slack_threads`: line_user_id, slack_ts, status, last_inquiry (all strings)<br>3. Select the Slack channel in every Slack node. Invite the bot to that channel first.<br>4. Edit the **Config** node: timezone, currency, the acknowledgment text, and the shop information. The AI answers only from what you write there. The AI replies in the language the customer wrote in, but the acknowledgment text is fixed, so write it in your customers' language.<br>5. Publish. Paste the production webhook URL of **Receive LINE webhook** into the LINE Developers console (turn Webhook on, auto-reply messages off). Paste the production URL of **Slack message** into the Slack app's Event Subscriptions Request URL and add the bot event `message.channels`.<br><br>The AI replies only when the answer is decided by your data. Everything else gets a short acknowledgment and goes to Slack. |
| Drop invalid request | n8n-nodes-base.noOp | Terminates invalid requests | Signature valid? | None | ## Setup (do this first)<br>1. Create credentials and select them in the nodes:<br>   - **Header Auth**: Name `Authorization`, Value `Bearer <LINE channel access token>` (nodes that call LINE)<br>   - **Crypto**: Hmac Secret = LINE channel secret (Compute signature)<br>   - **Anthropic** (Claude Haiku 4.5)<br>   - **Slack API**: Access Token = the bot token (`xoxb-`), Signature Secret = the Slack app's Signing Secret<br>2. Create three Data Tables and select them in the lookup and log nodes:<br>   - `products`: sku, name, category, color, width_cm, price, stock, note<br>   - `visit_slots`: slot_date (YYYY-MM-DD), slot_time, capacity, booked. **Add rows for upcoming dates. A date with no rows is answered as a day the shop does not take visits.**<br>   - `line_slack_threads`: line_user_id, slack_ts, status, last_inquiry (all strings)<br>3. Select the Slack channel in every Slack node. Invite the bot to that channel first.<br>4. Edit the **Config** node: timezone, currency, the acknowledgment text, and the shop information. The AI answers only from what you write there. The AI replies in the language the customer wrote in, but the acknowledgment text is fixed, so write it in your customers' language.<br>5. Publish. Paste the production webhook URL of **Receive LINE webhook** into the LINE Developers console (turn Webhook on, auto-reply messages off). Paste the production URL of **Slack message** into the Slack app's Event Subscriptions Request URL and add the bot event `message.channels`.<br><br>The AI replies only when the answer is decided by your data. Everything else gets a short acknowledgment and goes to Slack. |
| Config | n8n-nodes-base.set | Sets global workflow variables | Signature valid? | Split events | ## Setup (do this first)<br>1. Create credentials and select them in the nodes:<br>   - **Header Auth**: Name `Authorization`, Value `Bearer <LINE channel access token>` (nodes that call LINE)<br>   - **Crypto**: Hmac Secret = LINE channel secret (Compute signature)<br>   - **Anthropic** (Claude Haiku 4.5)<br>   - **Slack API**: Access Token = the bot token (`xoxb-`), Signature Secret = the Slack app's Signing Secret<br>2. Create three Data Tables and select them in the lookup and log nodes:<br>   - `products`: sku, name, category, color, width_cm, price, stock, note<br>   - `visit_slots`: slot_date (YYYY-MM-DD), slot_time, capacity, booked. **Add rows for upcoming dates. A date with no rows is answered as a day the shop does not take visits.**<br>   - `line_slack_threads`: line_user_id, slack_ts, status, last_inquiry (all strings)<br>3. Select the Slack channel in every Slack node. Invite the bot to that channel first.<br>4. Edit the **Config** node: timezone, currency, the acknowledgment text, and the shop information. The AI answers only from what you write there. The AI replies in the language the customer wrote in, but the acknowledgment text is fixed, so write it in your customers' language.<br>5. Publish. Paste the production webhook URL of **Receive LINE webhook** into the LINE Developers console (turn Webhook on, auto-reply messages off). Paste the production URL of **Slack message** into the Slack app's Event Subscriptions Request URL and add the bot event `message.channels`.<br><br>The AI replies only when the answer is decided by your data. Everything else gets a short acknowledgment and goes to Slack. |
| Split events | n8n-nodes-base.splitOut | Splits event batches | Config | Skip redelivered events | |
| Skip redelivered events | n8n-nodes-base.removeDuplicates | Removes duplicate event IDs | Split events | Extract message fields | |
| Extract message fields | n8n-nodes-base.set | Normalizes event data | Skip redelivered events | Keep text and images | |
| Keep text and images | n8n-nodes-base.filter | Filters text/image messages | Extract message fields | Text or image? | |
| Text or image? | n8n-nodes-base.switch | Routes by message type | Keep text and images | Use text as inquiry, Download image | |
| Use text as inquiry | n8n-nodes-base.set | Sets text inquiry | Text or image? | Inquiry | |
| Download image | n8n-nodes-base.httpRequest | Downloads image binary from LINE | Text or image? | Describe photo | |
| Describe photo | @n8n/n8n-nodes-langchain.chainLlm | Describes image using Claude | Download image | Use photo as inquiry | |
| Use photo as inquiry | n8n-nodes-base.set | Formulates stock inquiry from photo | Describe photo | Inquiry | |
| Inquiry | n8n-nodes-base.noOp | Merges inquiry paths | Use text as inquiry, Use photo as inquiry | Can the AI answer? | |
| Claude Haiku 4.5 | @n8n/n8n-nodes-langchain.lmChatAnthropic | Anthropic language model provider | None | None (Linked via AI model) | |
| Can the AI answer? | @n8n/n8n-nodes-langchain.textClassifier | Classifies inquiry intent | Inquiry | Answer from shop info, Extract product category, Extract visit request, Send acknowledgment | |
| Answer from shop info | @n8n/n8n-nodes-langchain.informationExtractor | Extracts shop info answers | Can the AI answer? | AI answer | |
| Extract product category | @n8n/n8n-nodes-langchain.informationExtractor | Extracts product category | Can the AI answer? | Look up stock | |
| Look up stock | n8n-nodes-base.dataTable | Queries products Data Table | Extract product category | Combine products | |
| Combine products | n8n-nodes-base.aggregate | Aggregates product rows | Look up stock | Answer from stock | |
| Answer from stock | @n8n/n8n-nodes-langchain.informationExtractor | Generates stock answers | Combine products | AI answer | |
| Extract visit request | @n8n/n8n-nodes-langchain.informationExtractor | Extracts visit date/time preferences | Can the AI answer? | Look up visit slots | |
| Look up visit slots | n8n-nodes-base.dataTable | Queries visit slots Data Table | Extract visit request | Combine slots | |
| Combine slots | n8n-nodes-base.aggregate | Aggregates visit slot rows | Look up visit slots | Answer from visit slots | |
| Answer from visit slots | @n8n/n8n-nodes-langchain.informationExtractor | Generates visit availability answers | Combine slots | AI answer | |
| AI answer | n8n-nodes-base.noOp | Merges AI response paths | Answer from shop info, Answer from stock, Answer from visit slots | Answered by AI? | |
| Answered by AI? | n8n-nodes-base.if | Checks if AI answer is valid | AI answer | Send AI reply, Send acknowledgment | |
| Send AI reply | n8n-nodes-base.httpRequest | Sends AI reply to LINE | Answered by AI? | Log AI answer | |
| Send acknowledgment | n8n-nodes-base.httpRequest | Sends acknowledgment to LINE | Answered by AI?, Can the AI answer? | Log needs staff | |
| Log AI answer | n8n-nodes-base.set | Logs successful AI response | Send AI reply | Post to Slack | |
| Log needs staff | n8n-nodes-base.set | Logs unanswerable inquiry routing | Send acknowledgment | Post to Slack | |
| Post to Slack | n8n-nodes-base.noOp | Prepares log payload for Slack | Log AI answer, Log needs staff | Find thread | |
| Find thread | n8n-nodes-base.dataTable | Queries thread mapping Data Table | Post to Slack | Thread exists? | |
| Thread exists? | n8n-nodes-base.if | Checks if Slack thread exists | Find thread | Reply in thread, Get LINE display name | |
| Reply in thread | n8n-nodes-base.slack | Posts reply in existing Slack thread | Thread exists? | Update thread status | |
| Update thread status | n8n-nodes-base.dataTable | Updates existing thread status | Reply in thread | None | |
| Get LINE display name | n8n-nodes-base.httpRequest | Gets LINE user profile name | Thread exists? | Create new thread | |
| Create new thread | n8n-nodes-base.slack | Creates new Slack thread | Get LINE display name | Save thread | |
| Save thread | n8n-nodes-base.dataTable | Saves new thread mapping | Create new thread | None | |
| Slack message | n8n-nodes-base.slackTrigger | Triggers on Slack message events | None | Staff thread replies only | |
| Staff thread replies only | n8n-nodes-base.filter | Filters valid staff thread replies | Slack message | Find customer | |
| Find customer | n8n-nodes-base.dataTable | Gets LINE user ID from thread timestamp | Staff thread replies only | Send to LINE | |
| Send to LINE | n8n-nodes-base.httpRequest | Sends staff reply to LINE push API | Find customer | Sent? | |
| Sent? | n8n-nodes-base.if | Checks if LINE push succeeded | Send to LINE | Mark as sent, Report failure | |
| Mark as sent | n8n-nodes-base.slack | Adds check mark reaction in Slack | Sent? | Mark thread replied | ## How staff reply in Slack<br>- Each customer gets one thread. **Anything written in the thread is sent to the customer on LINE** (push message, counted in your monthly LINE quota).<br>- A ✅ appears on the message once it is sent. No ✅ means it was not sent.<br>- Messages that start with **//** are internal notes and are NOT sent.<br>- Replies with "also send to channel" are sent too.<br>- Messages outside a thread are ignored. |
| Mark thread replied | n8n-nodes-base.dataTable | Updates thread status to replied | Mark as sent | None | ## How staff reply in Slack<br>- Each customer gets one thread. **Anything written in the thread is sent to the customer on LINE** (push message, counted in your monthly LINE quota).<br>- A ✅ appears on the message once it is sent. No ✅ means it was not sent.<br>- Messages that start with **//** are internal notes and are NOT sent.<br>- Replies with "also send to channel" are sent too.<br>- Messages outside a thread are ignored. |
| Report failure | n8n-nodes-base.slack | Posts failure warning in Slack thread | Sent? | None | ## How staff reply in Slack<br>- Each customer gets one thread. **Anything written in the thread is sent to the customer on LINE** (push message, counted in your monthly LINE quota).<br>- A ✅ appears on the message once it is sent. No ✅ means it was not sent.<br>- Messages that start with **//** are internal notes and are NOT sent.<br>- Replies with "also send to channel" are sent too.<br>- Messages outside a thread are ignored. |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to rebuild the workflow manually in n8n:

1. **Credential Setup:**
   - **HTTP Header Auth:** Name `Authorization`, Value `Bearer <YOUR_LINE_CHANNEL_ACCESS_TOKEN>`.
   - **Crypto (HMAC):** Set Hmac Secret to your LINE channel secret.
   - **Anthropic API:** Configure with your Anthropic API key.
   - **Slack API:** Configure with Bot Token (`xoxb-...`) and Signing Secret.

2. **Data Table Setup:**
   - Create three n8n Data Tables with the following schemas:
     - `products`: columns `sku`, `name`, `category`, `color`, `width_cm`, `price`, `stock`, `note` (all strings/numbers).
     - `visit_slots`: columns `slot_date` (YYYY-MM-DD), `slot_time`, `capacity` (number), `booked` (number).
     - `line_slack_threads`: columns `line_user_id`, `slack_ts`, `status`, `last_inquiry` (all strings).

3. **Node Creation & Configuration (Branch 1 - LINE Ingestion & AI Processing):**
   - **Receive LINE webhook**: Set webhook path to `line-slack`, HTTP method `POST`, response mode `onReceived`, and enable raw body.
   - **Compute signature**: Type `SHA256`, action `hmac`, encoding `base64`, binary data enabled (`computedSignature`, binary property `data`).
   - **Signature valid?**: IF node checking `{{ $json.computedSignature }}` equals `{{ $json.headers['x-line-signature'] }}`.
   - **Drop invalid request**: No-Op node (false branch).
   - **Config**: Set node assigning `timezone` (`Asia/Tokyo`), `currency` (`USD`), `acknowledgment` text, and `shop_info` text.
   - **Split events**: Split out `body.events`.
   - **Skip redelivered events**: Remove duplicates using dedupe value `{{ $json.webhookEventId }}`.
   - **Extract message fields**: Set node extracting `userId`, `replyToken`, `messageType`, `messageId`, and `text`.
   - **Keep text and images**: Filter node ensuring `messageType` equals `text` or `image`.
   - **Text or image?**: Switch node routing by `messageType` (`text` vs `image`).
   - **Use text as inquiry**: Set node mapping `inquiry` to extracted text.
   - **Download image**: HTTP Request GET to `https://api-data.line.me/v2/bot/message/{{ $json.messageId }}/content` with Header Auth; response format file (`data`).
   - **Describe photo**: LangChain LLM Chain node using Anthropic model (`Claude Haiku 4.5`) with prompt describing furniture.
   - **Use photo as inquiry**: Set node formulating inquiry string from image description.
   - **Inquiry**: No-Op convergence node.
   - **Claude Haiku 4.5**: LangChain Anthropic Chat Model (`claude-haiku-4-5-20251001`). Link to all LangChain nodes.
   - **Can the AI answer?**: Text Classifier node with categories `Shop info`, `Stock`, `Visit availability`, and `Needs staff`.
   - **Answer from shop info**: Information Extractor node with schema `answerable` (boolean) and `reply` (string).
   - **Extract product category**: Information Extractor node extracting product category.
   - **Look up stock**: Data Table node querying `products` where `category` matches extracted category.
   - **Combine products**: Aggregate node combining item data into `products`.
   - **Answer from stock**: Information Extractor node evaluating inventory and pricing.
   - **Extract visit request**: Information Extractor node extracting `desired_date` and `time_preference`.
   - **Look up visit slots**: Data Table node querying `visit_slots` where `slot_date` matches `desired_date`.
   - **Combine slots**: Aggregate node combining slot data into `slots`.
   - **Answer from visit slots**: Information Extractor node evaluating slot availability.
   - **AI answer**: No-Op convergence node.

4. **Node Creation & Configuration (Branch 2 - Automated Reply & Slack Handoff):**
   - **Answered by AI?**: IF node checking `{{ $json.output.answerable === true && String($json.output.reply ?? "").trim() !== "" }}`.
   - **Send AI reply**: HTTP Request POST to `https://api.line.me/v2/bot/message/reply` with JSON body (`replyToken`, AI reply text) and Header Auth. Continue on error.
   - **Send acknowledgment**: HTTP Request POST to `https://api.line.me/v2/bot/message/reply` with JSON body (`replyToken`, config acknowledgment text). Continue on error.
   - **Log AI answer**: Set node assigning `status: "ai_answered"`, `note`, and `broadcast`.
   - **Log needs staff**: Set node assigning `status: "needs_reply"`, `note`, and `broadcast: true`.
   - **Post to Slack**: No-Op convergence node.
   - **Find thread**: Data Table node querying `line_slack_threads` where `line_user_id` matches user ID.
   - **Thread exists?**: IF node checking if `slack_ts` is not empty.
   - **Reply in thread**: Slack node posting message to channel with `thread_ts` and `reply_broadcast`.
   - **Update thread status**: Data Table node updating `status` and `last_inquiry` filtered by `line_user_id`.
   - **Get LINE display name**: HTTP Request GET to `https://api.line.me/v2/bot/profile/{{ userId }}` with Header Auth. Continue on error.
   - **Create new thread**: Slack node posting message to channel with formatted customer display name and inquiry.
   - **Save thread**: Data Table node inserting new row (`line_user_id`, `slack_ts`, `status`, `last_inquiry`).

5. **Node Creation & Configuration (Branch 3 - Staff-to-LINE Synchronization):**
   - **Slack message**: Slack Trigger node watching workspace `message` events in target channel.
   - **Staff thread replies only**: Filter node ensuring message has `thread_ts`, `ts !== thread_ts`, empty `bot_id`, text not starting with `//`, and valid subtype.
   - **Find customer**: Data Table node querying `line_slack_threads` where `slack_ts` matches `thread_ts`.
   - **Send to LINE**: HTTP Request POST to `https://api.line.me/v2/bot/message/push` with JSON body (`to: line_user_id`, message text) and Header Auth. Continue on error.
   - **Sent?**: IF node checking `{{ $json.error ? "ng" : "ok" }}` equals `ok`.
   - **Mark as sent**: Slack node adding `white_check_mark` reaction to message `ts`.
   - **Mark thread replied**: Data Table node updating status to `replied` filtered by `slack_ts`.
   - **Report failure**: Slack node posting error warning reply into `thread_ts`.

6. **Finalizing Connections:**
   - Connect nodes step by step exactly as outlined in Section 2 and Section 3.
   - Publish the workflow and configure webhooks in LINE Developers Console and Slack App Event Subscriptions.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| External blog write-up with screenshots and resolved problems (in Japanese) | [Outsider Notes Write-up](https://outsidernotes.com/n8n-template-line-slack-handoff/) |
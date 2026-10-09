Sell clothing on Facebook Messenger with Gemini, Stripe, and an admin app

https://n8nworkflows.xyz/workflows/sell-clothing-on-facebook-messenger-with-gemini--stripe--and-an-admin-app-20421


# Sell clothing on Facebook Messenger with Gemini, Stripe, and an admin app

### 1. Workflow Overview

This workflow implements an automated AI sales agent for an online clothing store integrated with Facebook Messenger, powered by Google Gemini, Stripe Checkout, and PostgreSQL (with `pgvector` for semantic product searching). Additionally, it hosts an embedded administration dashboard serving product management, order processing, customer records, an escalation inbox, a test chat interface, and a setup wizard.

The workflow is categorized into the following logical blocks:
- **1.1 Messenger Intake & Verification:** Receives Meta webhook events, validates the verify token, acknowledges webhooks with an immediate HTTP 200 OK, and filters for valid incoming user messages.
- **1.2 Message Processing & Agent Orchestration:** De-duplicates incoming messages, buffers burst messaging using a wait node, updates user states, loads customer history, and interacts with the Google Gemini AI Agent with PostgreSQL chat memory and specialized data table tools.
- **1.3 Delivery Details & Cart Management:** Collects and validates shipping information, reviews cart contents against live stock levels, and prepares order parameters.
- **1.4 Checkout & Payment Processing:** Manages Stripe checkout sessions, handles expired payment links, creates secure payment webhooks, and processes successful payments via callbacks or webhooks—subsequently decrementing inventory and dispatching Messenger confirmation messages.
- **1.5 Admin Panel & API Backend:** Serves the frontend administration dashboard and provides a unified API handling authentication, product CRUD, Google Drive photo integration, vector database synchronization, and order management.
- **1.6 Nightly Maintenance & Error Handling:** Executes scheduled cleanups for old message IDs, stale carts, and uncompleted draft orders, while logging unhandled execution errors to a dedicated database table.

---

### 2. Block-by-Block Analysis

#### 2.1 Messenger Intake & Verification
- **Overview:** Receives inbound webhooks from Meta, verifies the subscription handshake, and isolates authentic customer messages from echo messages and page event wrappers.
- **Nodes Involved:** `Meta Webhook Trigger`, `Verify Webhook Handshake`, `Confirm Webhook Subscription`, `Load verify token`, `Acknowledge Event (200 OK)`, `Filter Page Events`, `Split the page record`, `split out each message`, `Is Real Customer Message`, `Is Manual Page Reply?`, `Pause Bot For Customer`.
- **Node Details:**
  - `Meta Webhook Trigger`: Webhook node (v2.1) listening at path `chat` supporting multiple methods. Role: Entry point for Meta Webhook events.
  - `Verify Webhook Handshake`: IF node checking `hub.mode === 'subscribe'` and matching `hub.verify_token` against database configurations.
  - `Confirm Webhook Subscription`: Respond to Webhook node returning the `hub.challenge` string.
  - `Load verify token`: Data Table node fetching `metaVerifyToken`.
  - `Acknowledge Event (200 OK)`: Respond to Webhook node returning `EVENT_RECEIVED` with HTTP 200.
  - `Filter Page Events`: IF node checking if `body.object === 'page'`.
  - `Split the page record` & `split out each message`: SplitOut nodes unpacking `body.entry` and `messaging` arrays.
  - `Is Real Customer Message`: IF node validating that the message is not an echo, has a valid `mid` or postback payload, and that sender and recipient IDs do not match.
  - `Is Manual Page Reply?`: IF node identifying manual intervention by page agents via message echoes without the `shop-bot` metadata tag.
  - `Pause Bot For Customer`: Data Table node (upsert) that updates `Sales Customer State` setting `humanPausedUntil` to 12 hours in the future when a human administrator replies manually.

#### 2.2 Message Processing & Agent Orchestration
- **Overview:** De-duplicates customer messages, handles batching of rapid sequential chats, and invokes the Google Gemini agent equipped with chat memory and data table retrieval tools.
- **Nodes Involved:** `Process Each Message Separately`, `When Called By This Workflow`, `Message Context`, `Find Processed Message`, `Is New Message`, `Remember Message ID`, `Sales Agent Config`, `Update Customer Last Seen`, `Get Customer State`, `Bot Allowed For Customer?`, `Wait For More Messages`, `Get Unhandled Messages`, `Combine Messages`, `Is Latest Message?`, `Mark Messages Handled`, `Look Up Existing Customer`, `Check If Customer Is New`, `Fetch Facebook Profile Info`, `Save New Customer Record`, `Get Customer Orders`, `Find Open Cart`, `Has Open Cart?`, `Create New Cart Order`, `Current Order`, `Get Pending Admin Answers`, `Sales Agent (SalesPro)`, `Google Gemini Chat Model`, `Customer Conversation Memory`, `Parse Agent Reply`, `Gemini Output Fixer Model`, `Route Reply`, `Send Text Reply to Customer`, `Mark Admin Answers Delivered`, `Send Product Image to Customer`, `Create Escalation`, `Send Fallback Reply`, `Save Answer To Agent Memory`, `Customer Memory (Admin Answers)`.
- **Node Details:**
  - `Sales Agent (SalesPro)`: Advanced AI Agent node utilizing LangChain bindings. Role: Manages conversational product recommendations, order flows, and tool calling based on system instructions.
  - `Google Gemini Chat Model`: Language model node bound to `models/gemini-2.5-flash`.
  - `Customer Conversation Memory`: Postgres Chat Memory node bound to `Sales Customers` session keys using context window length 16.
  - `Parse Agent Reply`: Structured Output Parser enforcing schema with `message`, `image`, and `escalate_question` strings.
  - `Send Text Reply to Customer`: HTTP Request node (POST to `https://graph.facebook.com/v25.0/me/messages`) transmitting response payloads.

#### 2.3 Delivery Details & Cart Management
- **Overview:** Manages step-by-step collection of recipient delivery information (name, phone, city, address, landmark), updates order states, and verifies cart integrity.
- **Nodes Involved:** `Delivery: Get Order`, `Delivery: Get Customer Orders`, `Delivery: Merge Details`, `Delivery: Save needed?`, `Delivery: Save Order`, `Delivery: Reply`, `save_delivery_details`.
- **Node Details:**
  - `save_delivery_details`: Tool Workflow node that invokes sub-workflow processing to validate delivery fields against missing requirements and retrieve past customer addresses.
  - `Delivery: Merge Details`: Code node performing incremental data validation, normalization of phone numbers, and address completeness checks.

#### 2.4 Checkout & Payment Processing
- **Overview:** Re-prices shopping carts against live database stock, creates Stripe Checkout Sessions, handles session expiration, and processes payment success return webhooks.
- **Nodes Involved:** `Get Checkout Settings`, `Get Order`, `Get Cart Items`, `Read Inventory`, `Get Previous Payment Links`, `Build Checkout`, `Checkout Valid?`, `Return Checkout Problem`, `List Old Payment Links`, `Expire Old Payment Link`, `Mark Old Link Expired`, `Create Stripe Checkout Session`, `Return Payment Error`, `Save Payment Link`, `Update Order For Payment`, `Send Pay Now Button`, `Return Checkout Success`, `Return Link Without Button`, `Is Paid Order Session?`, `Get Paid Order`, `Not Processed Yet?`, `Mark Order Paid`, `Claim Payment`, `Get Paid Line Items`, `Split Paid Items`, `Record Order Items`, `Read Inventory For Stock`, `Calculate New Stock`, `Reduce Inventory Stock`, `Get Buyer Last Seen`, `Send Payment Confirmation`, `Save Payment To Agent Memory`, `Buyer Memory`, `Payment Return Page`, `Look Up Returned Payment`, `Show Payment Result Page`, `Paid Session`, `Get Payment Link Row`.
- **Node Details:**
  - `Build Checkout`: Code node mapping inventory items to Stripe line items, converting currencies to cents, setting metadata, and constructing URL-encoded payloads.
  - `Create Stripe Checkout Session`: HTTP Request node calling `https://api.stripe.com/v1/checkout/sessions` with bearer authentication.
  - `Stripe Webhook` / `Payment Return Page`: Webhook trigger nodes capturing asynchronous payment notifications and user redirection URLs.

#### 2.5 Admin Panel & API Backend
- **Overview:** Serves the HTML administrative single-page application and handles backend administrative operations including product CRUD, image uploads to Google Drive, and vector index syncs.
- **Nodes Involved:** `Admin Page Webhook`, `Render app`, `Respond: app page`, `Admin API Webhook`, `Parse API`, `Load admin`, `Hash submitted password`, `Find session`, `Load settings`, `Decide`, `Route`, `Respond: JSON result`, `New session token`, `Save session`, `Respond: signed in`, `Delete session`, `Respond: signed out`, `Prepare new login`, `Hash new password`, `Update login details`, `Respond: login updated`, `Expand writes`, `Save setting`, `Respond: settings saved`, `Subscribe Page to app`, `Register app webhook`, `Compose Page result`, `Check Stripe key`, `Compose Stripe result`, `FB Callback Webhook`, `Load settings (callback)`, `Check state`, `State valid?`, `Exchange code for token`, `Get long-lived token`, `List Pages`, `Build pending pages`, `Expand callback writes`, `Save callback setting`, `Respond: callback page`, `Build error page`, `Respond: callback error`, `Get products`, `Compose products`, `Compose image result`, `Save product`, `Delete product`, `Look up photo folder`, `Has photo folder?`, `Create photo folder`, `Folder created?`, `Compose folder error`, `Share photo folder`, `Remember photo folder`, `Save folder setting`, `Prepare photo file`, `Upload photo to Drive`, `Share photo publicly`, `Has Drive photo?`, `Delete photo from Drive`, `Find product photo`, `Extend actions`, `Shape vector input`, `Run vector op`, `Compose vector result`, `Get products to sync`, `Find product to archive`, `Prepare archive`, `Archive target found?`, `Archive target missing`, `Connect Stripe`, `Stripe Webhook`, `Read Stripe body`, `Load Stripe secret`, `Verify Stripe signature`, `Signature valid?`, `Save Stripe event`, `Respond: Stripe accepted`, `Respond: Stripe rejected`, `Extend orders`, `Route orders`, `Get all orders`, `Get all order items`, `Get all customers`, `Get all products for orders`, `Compose overview`, `Save order`, `Replace items?`, `Clear old items`, `Any items to save?`, `Expand items`, `Save order item`, `Compose order result`, `Delete order items`, `Delete order record`, `Compose order delete`, `Save customer`, `Compose customer result`, `Find customer orders`, `Customer has orders?`, `Customer in use`, `Delete customer`, `Compose customer delete`, `Orders Module Webhook`, `Render orders module`, `Respond: orders module`, `Extend more`, `Route more`, `Find escalation`, `Plan reply`, `Plan ok?`, `Reply error`, `Send now?`, `Send Messenger reply`, `Finish send`, `Save escalation`, `Compose reply result`, `Get assistant config`, `Compose config`, `Save assistant config`, `Compose config saved`, `Shape preview input`, `Run preview assistant`, `Compose preview`, `Get all escalations`, `More Module Webhook`, `Render more module`, `Respond: more module`, `Find existing product`, `Photo replaced?`, `Delete replaced photo`, `Product to save`, `Guide Module Webhook`, `Render guide module`, `Respond: guide module`, `Find existing product`, `Photo replaced?`, `Delete replaced photo`, `Product to save`.
- **Node Details:**
  - `Admin Page Webhook`: Webhook endpoint serving the embedded SPA dashboard at path `shop-admin`.
  - `Admin API Webhook`: Webhook endpoint routing secure administrator commands at path `shop-api`.
  - `Upload photo to Drive`: Google Drive node storing binary product photos in designated folders.

#### 2.6 Nightly Maintenance & Error Handling
- **Overview:** Executes scheduled database hygiene tasks and captures unhandled execution exceptions.
- **Nodes Involved:** `Every Night At 3`, `Delete Old Message IDs`, `Delete Stale Cart Lines`, `Read Draft Orders`, `Older Than 7 Days`, `Mark Drafts Abandoned`, `When This Workflow Fails`, `Build error details`, `Save error log`, `Log: assistant could not answer`, `Log: reply not delivered`, `Log: payment message not sent`.
- **Node Details:**
  - `Every Night At 3`: Schedule Trigger node firing daily at 03:00 hours.
  - `When This Workflow Fails`: Error Trigger node capturing pipeline failures to record logs in the `Shop Errors` table.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Meta Webhook Trigger | n8n-nodes-base.webhook | Entry point for Meta webhooks | None | Load verify token, Acknowledge Event (200 OK) | 1. Messenger intake |
| Verify Webhook Handshake | n8n-nodes-base.if | Verifies Hub challenge tokens | Load verify token | Confirm Webhook Subscription | 1. Messenger intake |
| Confirm Webhook Subscription | n8n-nodes-base.respondToWebhook | Responds to webhook verification | Verify Webhook Handshake | None | 1. Messenger intake |
| Acknowledge Event (200 OK) | n8n-nodes-base.respondToWebhook | Acknowledges receipt immediately | Meta Webhook Trigger | Filter Page Events | 1. Messenger intake |
| Filter Page Events | n8n-nodes-base.if | Filters for page-level event objects | Acknowledge Event (200 OK) | Split the page record | 1. Messenger intake |
| Split the page record | n8n-nodes-base.splitOut | Unpacks page entry array | Filter Page Events | split out each message | 1. Messenger intake |
| split out each message | n8n-nodes-base.splitOut | Unpacks messaging events | Split the page record | Is Real Customer Message | 1. Messenger intake |
| Is Real Customer Message | n8n-nodes-base.if | Checks if event is valid customer message | split out each message | Process Each Message Separately, Is Manual Page Reply? | 1. Messenger intake |
| Process Each Message Separately | n8n-nodes-base.executeWorkflow | Dispatches message execution | Is Real Customer Message | None | 1. Messenger intake |
| When Called By This Workflow | n8n-nodes-base.executeWorkflowTrigger | Entry for sub-workflow calls | None | Load connections (messages) | 2. One customer message |
| Message Context | n8n-nodes-base.set | Normalizes message metadata | Route By Action | Find Processed Message | 2. One customer message |
| Find Processed Message | n8n-nodes-base.dataTable | Checks if message was processed | Message Context | Is New Message | 2. One customer message |
| Is New Message | n8n-nodes-base.if | Verifies if message is new | Find Processed Message | Remember Message ID | 2. One customer message |
| Remember Message ID | n8n-nodes-base.dataTable | Records processed message identifier | Is New Message | Sales Agent Config | 2. One customer message |
| Sales Agent Config | n8n-nodes-base.dataTable | Loads agent system prompts and configs | Remember Message ID | Update Customer Last Seen | 2. One customer message |
| Show Typing Indicator | n8n-nodes-base.httpRequest | Sends Facebook typing action | Bot Allowed For Customer? | None | 2. One customer message |
| Update Customer Last Seen | n8n-nodes-base.dataTable | Updates customer activity timestamp | Sales Agent Config | Get Customer State | 2. One customer message |
| Look Up Existing Customer | n8n-nodes-base.dataTable | Retrieves existing customer record | Is Latest Message? | Check If Customer Is New | 2. One customer message |
| Check If Customer Is New | n8n-nodes-base.if | Determines if customer record exists | Look Up Existing Customer | Fetch Facebook Profile Info, Get Customer Orders | 2. One customer message |
| Fetch Facebook Profile Info | n8n-nodes-base.httpRequest | Fetches user profile from Meta | Check If Customer Is New | Save New Customer Record | 2. One customer message |
| Save New Customer Record | n8n-nodes-base.dataTable | Creates new customer record | Fetch Facebook Profile Info | Get Customer Orders | 2. One customer message |
| Get Customer Orders | n8n-nodes-base.dataTable | Fetches all orders for customer | Check If Customer Is New, Save New Customer Record | Find Open Cart | 2. One customer message |
| Find Open Cart | n8n-nodes-base.code | Identifies open draft or pending orders | Get Customer Orders | Has Open Cart? | 2. One customer message |
| Has Open Cart? | n8n-nodes-base.if | Checks if open cart exists | Find Open Cart | Current Order, Create New Cart Order | 2. One customer message |
| Create New Cart Order | n8n-nodes-base.dataTable | Creates a new cart order record | Has Open Cart? | Current Order | 2. One customer message |
| Current Order | n8n-nodes-base.set | Formats active order payload | Has Open Cart?, Create New Cart Order | Get Pending Admin Answers | 2. One customer message |
| Get Pending Admin Answers | n8n-nodes-base.dataTable | Loads pending admin responses | Current Order | Sales Agent (SalesPro) | 2. One customer message |
| Sales Agent (SalesPro) | @n8n/n8n-nodes-langchain.agent | Main AI Sales Agent node | Get Pending Admin Answers, Customer Conversation Memory, Customer Memory (Admin Answers) | Route Reply, Send Fallback Reply | 2. One customer message |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM backend for Sales Agent | None | Sales Agent (SalesPro) | 2. One customer message |
| Customer Conversation Memory | @n8n/n8n-nodes-langchain.memoryPostgresChat | Postgres memory backend | None | Sales Agent (SalesPro) | 2. One customer message |
| get_product_details | n8n-nodes-base.dataTableTool | Tool to fetch product details | None | Sales Agent (SalesPro) | 2. One customer message |
| analyze_customer_image | @n8n/n8n-nodes-langchain.googleGeminiTool | Tool to analyze customer images | None | Sales Agent (SalesPro) | 2. One customer message |
| view_cart | n8n-nodes-base.dataTableTool | Tool to view current cart items | None | Sales Agent (SalesPro) | 2. One customer message |
| set_cart_item | n8n-nodes-base.dataTableTool | Tool to add/update cart items | None | Sales Agent (SalesPro) | 2. One customer message |
| remove_cart_item | n8n-nodes-base.dataTableTool | Tool to remove cart items | None | Sales Agent (SalesPro) | 2. One customer message |
| clear_cart | n8n-nodes-base.dataTableTool | Tool to empty cart items | None | Sales Agent (SalesPro) | 2. One customer message |
| checkout_and_send_payment_link | @n8n/n8n-nodes-langchain.toolWorkflow | Tool to initiate checkout workflow | None | Sales Agent (SalesPro) | 2. One customer message |
| get_my_orders | n8n-nodes-base.dataTableTool | Tool to retrieve customer orders | None | Sales Agent (SalesPro) | 2. One customer message |
| Parse Agent Reply | @n8n/n8n-nodes-langchain.outputParserStructured | Parses structured JSON output from agent | None | Sales Agent (SalesPro) | 2. One customer message |
| Gemini Output Fixer Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | Fixes malformed outputs | None | Parse Agent Reply | 2. One customer message |
| Route Reply | n8n-nodes-base.switch | Routes agent output by type | Sales Agent (SalesPro) | Send Text Reply to Customer, Send Product Image to Customer, Create Escalation, Send Fallback Reply | 2. One customer message |
| Send Text Reply to Customer | n8n-nodes-base.httpRequest | Sends text reply via Messenger | Route Reply | Mark Admin Answers Delivered, Log: reply not delivered | 2. One customer message |
| Mark Admin Answers Delivered | n8n-nodes-base.dataTable | Updates escalation delivery status | Send Text Reply to Customer | None | 2. One customer message |
| Send Product Image to Customer | n8n-nodes-base.httpRequest | Sends product image attachment | Route Reply | None | 2. One customer message |
| Create Escalation | n8n-nodes-base.dataTable | Records admin escalation item | Route Reply | None | 2. One customer message |
| Send Fallback Reply | n8n-nodes-base.httpRequest | Sends error recovery message | Route Reply | Log: assistant could not answer | 2. One customer message |
| Save Answer To Agent Memory | @n8n/n8n-nodes-langchain.memoryManager | Injects admin answers into memory | Answer sent? | None | 2. One customer message |
| Customer Memory (Admin Answers) | @n8n/n8n-nodes-langchain.memoryPostgresChat | Postgres connection for admin memory | None | Save Answer To Agent Memory | 2. One customer message |
| Is Manual Page Reply? | n8n-nodes-base.if | Checks if manual page reply occurred | Is Real Customer Message | Pause Bot For Customer | 1. Messenger intake |
| Pause Bot For Customer | n8n-nodes-base.dataTable | Pauses bot when admin replies manually | Is Manual Page Reply? | None | 1. Messenger intake |
| Get Customer State | n8n-nodes-base.dataTable | Loads customer pause state | Update Customer Last Seen | Bot Allowed For Customer? | 2. One customer message |
| Bot Allowed For Customer? | n8n-nodes-base.if | Verifies if bot is not paused | Get Customer State | Show Typing Indicator, Wait For More Messages | 2. One customer message |
| Wait For More Messages | n8n-nodes-base.wait | Batches rapid incoming messages | Bot Allowed For Customer? | Get Unhandled Messages | 2. One customer message |
| Get Unhandled Messages | n8n-nodes-base.dataTable | Fetches unhandled messages buffer | Wait For More Messages | Combine Messages | 2. One customer message |
| Combine Messages | n8n-nodes-base.set | Merges multiple text blocks | Get Unhandled Messages | Is Latest Message? | 2. One customer message |
| Is Latest Message? | n8n-nodes-base.if | Verifies final message in batch | Combine Messages | Mark Messages Handled, Look Up Existing Customer | 2. One customer message |
| Mark Messages Handled | n8n-nodes-base.dataTable | Flags batch messages as handled | Is Latest Message? | None | 2. One customer message |
| Every Night At 3 | n8n-nodes-base.scheduleTrigger | Daily schedule trigger at 03:00 | None | Delete Old Message IDs, Delete Stale Cart Lines, Read Draft Orders | 6. Nightly cleanup |
| Delete Old Message IDs | n8n-nodes-base.dataTable | Purges old processed messages | Every Night At 3 | None | 6. Nightly cleanup |
| Delete Stale Cart Lines | n8n-nodes-base.dataTable | Purges old cart rows | Every Night At 3 | None | 6. Nightly cleanup |
| Read Draft Orders | n8n-nodes-base.dataTable | Reads draft orders for review | Every Night At 3 | Older Than 7 Days | 6. Nightly cleanup |
| Older Than 7 Days | n8n-nodes-base.filter | Filters drafts older than 7 days | Read Draft Orders | Mark Drafts Abandoned | 6. Nightly cleanup |
| Mark Drafts Abandoned | n8n-nodes-base.dataTable | Marks abandoned draft orders | Older Than 7 Days | None | 6. Nightly cleanup |
| Route By Action | n8n-nodes-base.switch | Routes workflow tasks by action parameter | Restore message input | Get Checkout Settings, Message Context, Vector: Ensure store, Preview: Load assistant config, Delivery: Get Order | 2. One customer message |
| list_products | n8n-nodes-base.dataTableTool | Tool to list active products | None | Sales Agent (SalesPro), Preview Assistant | 2. One customer message |
| Get Checkout Settings | n8n-nodes-base.dataTable | Loads sales agent config | Route By Action | Get Order | 4. Checkout |
| Get Order | n8n-nodes-base.dataTable | Loads order details for checkout | Get Checkout Settings | Get Cart Items | 4. Checkout |
| Get Cart Items | n8n-nodes-base.dataTable | Fetches cart items for order | Get Order | Read Inventory | 4. Checkout |
| Read Inventory | n8n-nodes-base.dataTable | Loads product inventory list | Get Cart Items | Get Previous Payment Links | 4. Checkout |
| Get Previous Payment Links | n8n-nodes-base.dataTable | Retrieves existing checkout sessions | Read Inventory | Build Checkout | 4. Checkout |
| Build Checkout | n8n-nodes-base.code | Re-prices cart and builds Stripe payload | Get Previous Payment Links | Checkout Valid? | 4. Checkout |
| Checkout Valid? | n8n-nodes-base.if | Validates checkout parameters | Build Checkout | List Old Payment Links, Create Stripe Checkout Session, Return Checkout Problem | 4. Checkout |
| Return Checkout Problem | n8n-nodes-base.set | Sets checkout validation error | Checkout Valid? | None | 4. Checkout |
| List Old Payment Links | n8n-nodes-base.splitOut | Splits previous checkout sessions | Checkout Valid? | Expire Old Payment Link | 4. Checkout |
| Expire Old Payment Link | n8n-nodes-base.httpRequest | Expires old Stripe checkout sessions | List Old Payment Links | Mark Old Link Expired | 4. Checkout |
| Mark Old Link Expired | n8n-nodes-base.dataTable | Updates session status in DB | Expire Old Payment Link | None | 4. Checkout |
| Create Stripe Checkout Session | n8n-nodes-base.httpRequest | Creates Stripe Checkout session | Checkout Valid? | Return Payment Error, Save Payment Link | 4. Checkout |
| Return Payment Error | n8n-nodes-base.set | Sets Stripe API error response | Create Stripe Checkout Session | None | 4. Checkout |
| Save Payment Link | n8n-nodes-base.dataTable | Saves checkout session record | Create Stripe Checkout Session | Update Order For Payment | 4. Checkout |
| Update Order For Payment | n8n-nodes-base.dataTable | Sets order status to awaiting payment | Save Payment Link | Send Pay Now Button | 4. Checkout |
| Send Pay Now Button | n8n-nodes-base.httpRequest | Sends 'Pay Now' button on Messenger | Update Order For Payment | Return Checkout Success, Return Link Without Button | 4. Checkout |
| Return Checkout Success | n8n-nodes-base.set | Formats successful checkout response | Send Pay Now Button | None | 4. Checkout |
| Return Link Without Button | n8n-nodes-base.set | Formats fallback checkout response | Send Pay Now Button | None | 4. Checkout |
| Is Paid Order Session? | n8n-nodes-base.if | Validates Stripe paid status | Paid Session | Get Payment Link Row | 5. Payment confirmed |
| Get Paid Order | n8n-nodes-base.dataTable | Loads paid order details | Claim Payment | Not Processed Yet? | 5. Payment confirmed |
| Not Processed Yet? | n8n-nodes-base.if | Checks if payment already processed | Get Paid Order | Mark Order Paid | 5. Payment confirmed |
| Mark Order Paid | n8n-nodes-base.dataTable | Updates order status to Paid | Not Processed Yet? | Get Paid Line Items | 5. Payment confirmed |
| Claim Payment | n8n-nodes-base.dataTable | Marks checkout session as paid | Is Paid Order Session? | Get Paid Order | 5. Payment confirmed |
| Get Paid Line Items | n8n-nodes-base.httpRequest | Fetches line items from Stripe session | Mark Order Paid | Split Paid Items | 5. Payment confirmed |
| Split Paid Items | n8n-nodes-base.splitOut | Splits purchased line items | Get Paid Line Items | Record Order Items | 5. Payment confirmed |
| Record Order Items | n8n-nodes-base.dataTable | Upserts order item records | Split Paid Items | Read Inventory For Stock | 5. Payment confirmed |
| Read Inventory For Stock | n8n-nodes-base.dataTable | Loads inventory for stock deduction | Record Order Items | Calculate New Stock | 5. Payment confirmed |
| Calculate New Stock | n8n-nodes-base.code | Calculates remaining stock levels | Read Inventory For Stock | Reduce Inventory Stock | 5. Payment confirmed |
| Reduce Inventory Stock | n8n-nodes-base.dataTable | Updates product stock numbers | Calculate New Stock | None | 5. Payment confirmed |
| Get Buyer Last Seen | n8n-nodes-base.dataTable | Fetches customer last seen timestamp | Get Paid Line Items | Send Payment Confirmation | 5. Payment confirmed |
| Send Payment Confirmation | n8n-nodes-base.httpRequest | Sends payment confirmation on Messenger | Get Buyer Last Seen | Save Payment To Agent Memory | 5. Payment confirmed |
| Save Payment To Agent Memory | @n8n/n8n-nodes-langchain.memoryManager | Injects payment confirmation into memory | Send Payment Confirmation, When This Workflow Fails | Buyer Memory | 5. Payment confirmed |
| Buyer Memory | @n8n/n8n-nodes-langchain.memoryPostgresChat | Postgres memory backend for buyer | None | Save Payment To Agent Memory | 5. Payment confirmed |
| When This Workflow Fails | n8n-nodes-base.errorTrigger | Captures workflow errors | None | Build error details | 7. Error log |
| Payment Return Page | n8n-nodes-base.webhook | Entry point for payment return success page | None | Load connections (return page) | 5. Payment confirmed |
| Look Up Returned Payment | n8n-nodes-base.httpRequest | Queries Stripe session status | Restore return page | Show Payment Result Page | 5. Payment confirmed |
| Show Payment Result Page | n8n-nodes-base.respondToWebhook | Renders HTML payment result page | Look Up Returned Payment | Paid Session | 5. Payment confirmed |
| Paid Session | n8n-nodes-base.set | Normalizes paid session payload | Show Payment Result Page, Restore Stripe event | Is Paid Order Session? | 5. Payment confirmed |
| Get Payment Link Row | n8n-nodes-base.dataTable | Loads checkout session row | Is Paid Order Session? | Claim Payment | 5. Payment confirmed |
| get_order_items | n8n-nodes-base.dataTableTool | Tool to retrieve paid order items | None | Sales Agent (SalesPro) | 2. One customer message |
| search_products | @n8n/n8n-nodes-langchain.toolWorkflow | Semantic product search tool | None | Sales Agent (SalesPro), Preview Assistant | 2. One customer message |
| Vector: Ensure store | n8n-nodes-base.postgres | Ensures pgvector extension & vectors table | Route By Action | Vector: Plan operation | 14. Product search index |
| Vector: Plan operation | n8n-nodes-base.code | Plans vector embedding tasks | Vector: Ensure store | Vector: Needs embedding? | 14. Product search index |
| Vector: Needs embedding? | n8n-nodes-base.if | Checks if text needs embedding | Vector: Plan operation | Vector: Embed texts, Vector: Build SQL | 14. Product search index |
| Vector: Embed texts | n8n-nodes-base.httpRequest | Generates Gemini vector embeddings | Vector: Needs embedding? | Vector: Build SQL | 14. Product search index |
| Vector: Build SQL | n8n-nodes-base.code | Constructs vector SQL queries | Vector: Embed texts, Vector: Needs embedding? | Vector: SQL ready? | 14. Product search index |
| Vector: SQL ready? | n8n-nodes-base.if | Validates generated SQL | Vector: Build SQL | Vector: Run SQL, Vector: Compose result | 14. Product search index |
| Vector: Run SQL | n8n-nodes-base.postgres | Executes vector queries in database | Vector: SQL ready? | Vector: Compose result | 14. Product search index |
| Vector: Compose result | n8n-nodes-base.code | Formats vector operation result | Vector: Run SQL, Vector: SQL ready? | Respond: JSON result | 14. Product search index |
| Preview: Load assistant config | n8n-nodes-base.dataTable | Loads config for test chat | Route By Action | Preview: Build prompt | 17. Test chat |
| Preview: Build prompt | n8n-nodes-base.code | Builds test chat prompt | Preview: Load assistant config | Preview Assistant | 17. Test chat |
| Preview Assistant | @n8n/n8n-nodes-langchain.agent | Test chat AI assistant node | Preview: Build prompt, list_products, search_products | Preview: Compose reply | 17. Test chat |
| Gemini Preview Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | LLM backend for preview | None | Preview Assistant | 17. Test chat |
| Preview: Compose reply | n8n-nodes-base.code | Formats preview response | Preview Assistant | Respond: JSON result | 17. Test chat |
| Run once: create tables and first login | n8n-nodes-base.manualTrigger | Manual setup trigger | None | Create table: Sales Agent Config | 18. Setup (run once) |
| Create table: Sales Agent Config | n8n-nodes-base.dataTable | Creates Agent Config table | Run once: create tables and first login, Setup Webhook | Create table: Sales Cart Items | 18. Setup (run once) |
| Create table: Sales Cart Items | n8n-nodes-base.dataTable | Creates Cart Items table | Create table: Sales Agent Config | Create table: Sales Checkout Sessions | 18. Setup (run once) |
| Create table: Sales Checkout Sessions | n8n-nodes-base.dataTable | Creates Checkout Sessions table | Create table: Sales Cart Items | Create table: Sales Customer State | 18. Setup (run once) |
| Create table: Sales Customer State | n8n-nodes-base.dataTable | Creates Customer State table | Create table: Sales Checkout Sessions | Create table: Sales Customers | 18. Setup (run once) |
| Create table: Sales Customers | n8n-nodes-base.dataTable | Creates Customers table | Create table: Sales Customer State | Create table: Sales Escalations | 18. Setup (run once) |
| Create table: Sales Escalations | n8n-nodes-base.dataTable | Creates Escalations table | Create table: Sales Customers | Create table: Sales Order Items | 18. Setup (run once) |
| Create table: Sales Order Items | n8n-nodes-base.dataTable | Creates Order Items table | Create table: Sales Escalations | Create table: Sales Orders | 18. Setup (run once) |
| Create table: Sales Orders | n8n-nodes-base.dataTable | Creates Orders table | Create table: Sales Order Items | Create table: Sales Processed Messages | 18. Setup (run once) |
| Create table: Sales Processed Messages | n8n-nodes-base.dataTable | Creates Processed Messages table | Create table: Sales Orders | Create table: Sales Products | 18. Setup (run once) |
| Create table: Sales Products | n8n-nodes-base.dataTable | Creates Products table | Create table: Sales Processed Messages | Create table: Shop Admin Login | 18. Setup (run once) |
| Create table: Shop Admin Login | n8n-nodes-base.dataTable | Creates Admin Login table | Create table: Sales Products | Create table: Shop Admin Sessions | 18. Setup (run once) |
| Create table: Shop Admin Sessions | n8n-nodes-base.dataTable | Creates Admin Sessions table | Create table: Shop Admin Login | Create table: Shop Settings | 18. Setup (run once) |
| Create table: Shop Settings | n8n-nodes-base.dataTable | Creates Shop Settings table | Create table: Shop Admin Sessions | Create table: Shop Errors | 18. Setup (run once) |
| Setup: look for an admin login | n8n-nodes-base.dataTable | Checks admin login existence | Setup: create product search table | Setup: no admin login yet? | 18. Setup (run once) |
| Setup: no admin login yet? | n8n-nodes-base.if | Checks if admin login is missing | Setup: look for an admin login | Setup: add the first admin login, Setup: look for assistant instructions | 18. Setup (run once) |
| Setup: add the first admin login | n8n-nodes-base.dataTable | Inserts default admin account | Setup: no admin login yet? | Setup: look for assistant instructions | 18. Setup (run once) |
| Setup: look for assistant instructions | n8n-nodes-base.dataTable | Checks assistant prompt config | Setup: add the first admin login, Setup: no admin login yet? | Setup: no instructions yet? | 18. Setup (run once) |
| Setup: no instructions yet? | n8n-nodes-base.if | Checks if prompt is missing | Setup: look for assistant instructions | Setup: add default instructions, Setup: done | 18. Setup (run once) |
| Setup: add default instructions | n8n-nodes-base.dataTable | Inserts default system prompt | Setup: no instructions yet? | Setup: done | 18. Setup (run once) |
| Setup: done | n8n-nodes-base.set | Formats setup completion notice | Setup: add default instructions, Setup: no instructions yet? | None | 18. Setup (run once) |
| Admin Page Webhook | n8n-nodes-base.webhook | Entry point for admin website SPA | None | Render app | 8. Admin website pages |
| Render app | n8n-nodes-base.code | Renders admin SPA HTML template | Admin Page Webhook | Respond: app page | 8. Admin website pages |
| Respond: app page | n8n-nodes-base.respondToWebhook | Returns admin SPA HTML | Render app | None | 8. Admin website pages |
| Admin API Webhook | n8n-nodes-base.webhook | Entry point for admin REST API | None | Parse API | 9. Admin API: routing, login and settings |
| Parse API | n8n-nodes-base.code | Parses API request headers and body | Admin API Webhook | Load admin | 9. Admin API: routing, login and settings |
| Load admin | n8n-nodes-base.dataTable | Loads admin credentials from DB | Parse API | Hash submitted password | 9. Admin API: routing, login and settings |
| Hash submitted password | n8n-nodes-base.crypto | Hashes submitted admin password | Load admin | Find session | 9. Admin API: routing, login and settings |
| Find session | n8n-nodes-base.dataTable | Validates admin session token | Hash submitted password | Load settings | 9. Admin API: routing, login and settings |
| Load settings | n8n-nodes-base.dataTable | Loads all shop settings | Find session | Decide | 9. Admin API: routing, login and settings |
| Decide | n8n-nodes-base.code | Evaluates API permissions and routing | Load settings | Route | 9. Admin API: routing, login and settings |
| Route | n8n-nodes-base.switch | Routes API actions | Decide | Respond: JSON result, New session token, Delete session, Prepare new login, Connect Stripe, Subscribe Page to app, Check Stripe key, Get products, Look up photo folder, Find existing product, Find product photo, Shape vector input, Get products to sync, Find product to archive, Route orders | 9. Admin API: routing, login and settings |
| Respond: JSON result | n8n-nodes-base.respondToWebhook | Returns JSON API response | Route, Reply error, Compose config, Compose errors, Compose preview, Compose products, Compose Stripe result, Compose vector result, Customer in use, Save setting | None | 9. Admin API: routing, login and settings |
| New session token | n8n-nodes-base.code | Generates new admin session token | Route | Save session | 9. Admin API: routing, login and settings |
| Save session | n8n-nodes-base.dataTable | Saves session token to DB | New session token | Respond: signed in | 9. Admin API: routing, login and settings |
| Respond: signed in | n8n-nodes-base.respondToWebhook | Returns login success response | Save session | None | 9. Admin API: routing, login and settings |
| Delete session | n8n-nodes-base.dataTable | Deletes admin session on logout | Route | Respond: signed out | 9. Admin API: routing, login and settings |
| Respond: signed out | n8n-nodes-base.respondToWebhook | Returns logout confirmation | Delete session | None | 9. Admin API: routing, login and settings |
| Prepare new login | n8n-nodes-base.code | Generates salt for password update | Route | Hash new password | 9. Admin API: routing, login and settings |
| Hash new password | n8n-nodes-base.crypto | Hashes new admin password | Prepare new login | Update login details | 9. Admin API: routing, login and settings |
| Update login details | n8n-nodes-base.dataTable | Updates admin login record | Hash new password | Respond: login updated | 9. Admin API: routing, login and settings |
| Respond: login updated | n8n-nodes-base.respondToWebhook | Returns password update confirmation | Update login details | None | 9. Admin API: routing, login and settings |
| Expand writes | n8n-nodes-base.code | Expands setting write operations | Connect Stripe, Compose Page result | Save setting | 9. Admin API: routing, login and settings |
| Save setting | n8n-nodes-base.dataTable | Upserts shop setting records | Expand writes | Respond: settings saved | 9. Admin API: routing, login and settings |
| Respond: settings saved | n8n-nodes-base.respondToWebhook | Returns settings save confirmation | Save setting | None | 9. Admin API: routing, login and settings |
| Subscribe Page to app | n8n-nodes-base.httpRequest | Subscribes Facebook Page to app | Route | Register app webhook | 10. Facebook connect |
| Register app webhook | n8n-nodes-base.httpRequest | Registers Meta app webhook subscription | Subscribe Page to app | Compose Page result | 10. Facebook connect |
| Compose Page result | n8n-nodes-base.code | Formats Facebook subscription result | Register app webhook | Expand writes | 10. Facebook connect |
| Check Stripe key | n8n-nodes-base.httpRequest | Validates Stripe API key | Route | Compose Stripe result | 11. Stripe connect and webhook |
| Compose Stripe result | n8n-nodes-base.code | Formats Stripe validation result | Check Stripe key | Respond: JSON result | 11. Stripe connect and webhook |
| FB Callback Webhook | n8n-nodes-base.webhook | Entry point for Facebook OAuth callback | None | Load settings (callback) | 10. Facebook connect |
| Load settings (callback) | n8n-nodes-base.dataTable | Loads settings for callback validation | FB Callback Webhook | Check state | 10. Facebook connect |
| Check state | n8n-nodes-base.code | Validates OAuth state parameter | Load settings (callback) | State valid? | 10. Facebook connect |
| State valid? | n8n-nodes-base.if | Checks if OAuth state is valid | Check state | Exchange code for token, Build error page | 10. Facebook connect |
| Exchange code for token | n8n-nodes-base.httpRequest | Exchanges OAuth code for access token | State valid? | Get long-lived token | 10. Facebook connect |
| Get long-lived token | n8n-nodes-base.httpRequest | Requests long-lived Facebook token | Exchange code for token | List Pages | 10. Facebook connect |
| List Pages | n8n-nodes-base.httpRequest | Lists Facebook Pages managed by user | Get long-lived token | Build pending pages | 10. Facebook connect |
| Build pending pages | n8n-nodes-base.code | Formats pending Facebook pages list | List Pages | Expand callback writes | 10. Facebook connect |
| Expand callback writes | n8n-nodes-base.code | Prepares callback setting writes | Build pending pages | Save callback setting | 10. Facebook connect |
| Save callback setting | n8n-nodes-base.dataTable | Saves pending pages to settings | Expand callback writes | Respond: callback page | 10. Facebook connect |
| Respond: callback page | n8n-nodes-base.respondToWebhook | Renders Facebook connection success page | Save callback setting | None | 10. Facebook connect |
| Build error page | n8n-nodes-base.code | Renders OAuth error page | State valid? | Respond: callback error | 10. Facebook connect |
| Respond: callback error | n8n-nodes-base.respondToWebhook | Returns OAuth error HTML | Build error page | None | 10. Facebook connect |
| Get products | n8n-nodes-base.dataTable | Retrieves all products | Route | Compose products | 12. Products |
| Compose products | n8n-nodes-base.code | Formats product list for API | Get products | Respond: JSON result | 12. Products |
| Compose image result | n8n-nodes-base.code | Formats uploaded image URL | Share photo publicly | Respond: JSON result | 13. Product photos |
| Save product | n8n-nodes-base.dataTable | Upserts product record | Product to save, Archive target found? | Shape vector input | 12. Products |
| Delete product | n8n-nodes-base.dataTable | Deletes product record | Find product photo | Has Drive photo? | 12. Products |
| Look up photo folder | n8n-nodes-base.code | Retrieves Google Drive folder ID | Route | Has photo folder? | 13. Product photos |
| Has photo folder? | n8n-nodes-base.if | Checks if Drive folder exists | Look up photo folder | Prepare photo file, Create photo folder | 13. Product photos |
| Create photo folder | n8n-nodes-base.googleDrive | Creates Google Drive folder | Has photo folder? | Folder created? | 13. Product photos |
| Folder created? | n8n-nodes-base.if | Verifies folder creation | Create photo folder | Share photo folder, Compose folder error | 13. Product photos |
| Compose folder error | n8n-nodes-base.code | Formats folder error response | Folder created? | Respond: JSON result | 13. Product photos |
| Share photo folder | n8n-nodes-base.googleDrive | Shares Drive folder publicly | Folder created? | Remember photo folder | 13. Product photos |
| Remember photo folder | n8n-nodes-base.code | Prepares folder setting write | Share photo folder | Save folder setting | 13. Product photos |
| Save folder setting | n8n-nodes-base.dataTable | Saves folder setting | Remember photo folder | Prepare photo file | 13. Product photos |
| Prepare photo file | n8n-nodes-base.code | Prepares binary image data | Has photo folder?, Save folder setting | Upload photo to Drive | 13. Product photos |
| Upload photo to Drive | n8n-nodes-base.googleDrive | Uploads photo to Google Drive | Prepare photo file | Share photo publicly | 13. Product photos |
| Share photo publicly | n8n-nodes-base.googleDrive | Makes photo publicly accessible | Upload photo to Drive | Compose image result | 13. Product photos |
| Has Drive photo? | n8n-nodes-base.if | Checks if image is on Google Drive | Delete product | Delete photo from Drive, Shape vector input | 13. Product photos |
| Delete photo from Drive | n8n-nodes-base.googleDrive | Deletes photo from Google Drive | Has Drive photo? | Shape vector input | 13. Product photos |
| Find product photo | n8n-nodes-base.dataTable | Retrieves product photo record | Route | Delete product | 12. Products |
| Extend actions | n8n-nodes-base.code | Normalizes extended API actions | Decide | Extend orders | 9. Admin API: routing, login and settings |
| Shape vector input | n8n-nodes-base.code | Shapes input for vector workflow | Save product, Delete product, Has Drive photo?, Delete photo from Drive | Run vector op | 14. Product search index |
| Run vector op | n8n-nodes-base.executeWorkflow | Executes vector sub-workflow | Shape vector input | Compose vector result | 14. Product search index |
| Compose vector result | n8n-nodes-base.code | Formats vector operation response | Run vector op | Respond: JSON result | 14. Product search index |
| Get products to sync | n8n-nodes-base.dataTable | Loads products for vector sync | Route | Shape vector input | 14. Product search index |
| Find product to archive | n8n-nodes-base.dataTable | Retrieves product to archive | Route | Prepare archive | 12. Products |
| Prepare archive | n8n-nodes-base.code | Prepares archive payload | Find product to archive | Archive target found? | 12. Products |
| Archive target found? | n8n-nodes-base.if | Verifies archive target exists | Prepare archive | Save product, Archive target missing | 12. Products |
| Archive target missing | n8n-nodes-base.code | Sets archive error response | Archive target found? | Respond: JSON result | 12. Products |
| Connect Stripe | n8n-nodes-base.code | Configures Stripe webhooks and accounts | Route | Expand writes | 11. Stripe connect and webhook |
| Stripe Webhook | n8n-nodes-base.webhook | Entry point for Stripe event webhooks | None | Read Stripe body | 11. Stripe connect and webhook |
| Read Stripe body | n8n-nodes-base.code | Reads raw Stripe webhook body | Stripe Webhook | Load Stripe secret | 11. Stripe connect and webhook |
| Load Stripe secret | n8n-nodes-base.dataTable | Loads Stripe webhook signing secret | Read Stripe body | Verify Stripe signature | 11. Stripe connect and webhook |
| Verify Stripe signature | n8n-nodes-base.code | Verifies Stripe webhook HMAC signature | Load Stripe secret | Signature valid? | 11. Stripe connect and webhook |
| Signature valid? | n8n-nodes-base.if | Checks signature validity | Verify Stripe signature | Save Stripe event, Stripe Payment Event, Respond: Stripe rejected | 11. Stripe connect and webhook |
| Save Stripe event | n8n-nodes-base.dataTable | Records last Stripe event timestamp | Signature valid? | Respond: Stripe accepted | 11. Stripe connect and webhook |
| Respond: Stripe accepted | n8n-nodes-base.respondToWebhook | Returns Stripe success response | Save Stripe event | None | 11. Stripe connect and webhook |
| Respond: Stripe rejected | n8n-nodes-base.respondToWebhook | Returns Stripe error response | Signature valid? | None | 11. Stripe connect and webhook |
| Extend orders | n8n-nodes-base.code | Normalizes order management API actions | Extend actions | Route orders | 15. Orders and customers |
| Route orders | n8n-nodes-base.switch | Routes order/customer API requests | Extend orders | Get all orders, Save order, Delete order items, Save customer, Find customer orders, Route more | 15. Orders and customers |
| Get all orders | n8n-nodes-base.dataTable | Retrieves all shop orders | Route orders | Get all order items | 15. Orders and customers |
| Get all order items | n8n-nodes-base.dataTable | Retrieves all order line items | Get all orders | Get all customers | 15. Orders and customers |
| Get all customers | n8n-nodes-base.dataTable | Retrieves all customer records | Get all order items | Get all products for orders | 15. Orders and customers |
| Get all products for orders | n8n-nodes-base.dataTable | Retrieves all products for orders | Get all customers | Get all escalations | 15. Orders and customers |
| Compose overview | n8n-nodes-base.code | Formats orders overview response | Get all escalations | Respond: JSON result | 15. Orders and customers |
| Save order | n8n-nodes-base.dataTable | Upserts order record | Route orders | Replace items? | 15. Orders and customers |
| Replace items? | n8n-nodes-base.if | Checks if order items should be replaced | Save order | Clear old items, Compose order result | 15. Orders and customers |
| Clear old items | n8n-nodes-base.dataTable | Clears old order line items | Replace items? | Any items to save? | 15. Orders and customers |
| Any items to save? | n8n-nodes-base.if | Checks if new items exist | Clear old items | Expand items, Compose order result | 15. Orders and customers |
| Expand items | n8n-nodes-base.code | Expands order items array | Any items to save? | Save order item | 15. Orders and customers |
| Save order item | n8n-nodes-base.dataTable | Upserts individual order line item | Expand items | Compose order result | 15. Orders and customers |
| Compose order result | n8n-nodes-base.code | Formats order save response | Replace items?, Any items to save?, Save order item | Respond: JSON result | 15. Orders and customers |
| Delete order items | n8n-nodes-base.dataTable | Deletes order line items | Route orders | Delete order record | 15. Orders and customers |
| Delete order record | n8n-nodes-base.dataTable | Deletes order record | Delete order items | Compose order delete | 15. Orders and customers |
| Compose order delete | n8n-nodes-base.code | Formats order delete response | Delete order record | Respond: JSON result | 15. Orders and customers |
| Save customer | n8n-nodes-base.dataTable | Upserts customer record | Route orders | Compose customer result | 15. Orders and customers |
| Compose customer result | n8n-nodes-base.code | Formats customer save response | Save customer | Respond: JSON result | 15. Orders and customers |
| Find customer orders | n8n-nodes-base.dataTable | Retrieves customer orders for deletion check | Route orders | Customer has orders? | 15. Orders and customers |
| Customer has orders? | n8n-nodes-base.if | Checks if customer has existing orders | Find customer orders | Customer in use, Delete customer | 15. Orders and customers |
| Customer in use | n8n-nodes-base.code | Sets error response for active customer | Customer has orders? | Respond: JSON result | 15. Orders and customers |
| Delete customer | n8n-nodes-base.dataTable | Deletes customer record | Customer has orders? | Compose customer delete | 15. Orders and customers |
| Compose customer delete | n8n-nodes-base.code | Formats customer delete response | Delete customer | Respond: JSON result | 15. Orders and customers |
| Orders Module Webhook | n8n-nodes-base.webhook | Entry point for orders SPA module | None | Render orders module | 8. Admin website pages |
| Render orders module | n8n-nodes-base.code | Renders orders module script | Orders Module Webhook | Respond: orders module | 8. Admin website pages |
| Respond: orders module | n8n-nodes-base.respondToWebhook | Returns orders module JS | Render orders module | None | 8. Admin website pages |
| Extend more | n8n-nodes-base.code | Normalizes advanced API actions | Route orders | Route more | 16. Inbox and assistant |
| Route more | n8n-nodes-base.switch | Routes advanced admin actions | Extend more | Find escalation, Get assistant config, Save assistant config, Shape preview input, Get error log, Mark errors seen, Clear seen errors | 16. Inbox and assistant |
| Find escalation | n8n-nodes-base.dataTable | Retrieves escalation record | Route more | Plan reply | 16. Inbox and assistant |
| Plan reply | n8n-nodes-base.code | Plans reply strategy for escalation | Find escalation | Plan ok? | 16. Inbox and assistant |
| Plan ok? | n8n-nodes-base.if | Validates reply plan | Plan reply | Send now?, Reply error | 16. Inbox and assistant |
| Reply error | n8n-nodes-base.code | Formats reply error response | Plan ok? | Respond: JSON result | 16. Inbox and assistant |
| Send now? | n8n-nodes-base.if | Checks if reply should be sent immediately | Plan ok? | Send Messenger reply, Save escalation | 16. Inbox and assistant |
| Send Messenger reply | n8n-nodes-base.httpRequest | Sends admin answer on Messenger | Send now? | Finish send | 16. Inbox and assistant |
| Finish send | n8n-nodes-base.code | Processes Messenger send result | Send Messenger reply | Save escalation | 16. Inbox and assistant |
| Save escalation | n8n-nodes-base.dataTable | Updates escalation status | Send now?, Finish send | Answer sent? | 16. Inbox and assistant |
| Compose reply result | n8n-nodes-base.code | Formats reply result response | Answer sent? | Respond: JSON result | 16. Inbox and assistant |
| Get assistant config | n8n-nodes-base.dataTable | Loads assistant configuration | Route more | Compose config | 16. Inbox and assistant |
| Compose config | n8n-nodes-base.code | Formats assistant config response | Get assistant config | Respond: JSON result | 16. Inbox and assistant |
| Save assistant config | n8n-nodes-base.dataTable | Upserts assistant configuration | Route more | Compose config saved | 16. Inbox and assistant |
| Compose config saved | n8n-nodes-base.code | Formats config save response | Save assistant config | Respond: JSON result | 16. Inbox and assistant |
| Shape preview input | n8n-nodes-base.code | Shapes input for test chat preview | Route more | Run preview assistant | 17. Test chat |
| Run preview assistant | n8n-nodes-base.executeWorkflow | Executes preview sub-workflow | Shape preview input | Compose preview | 17. Test chat |
| Compose preview | n8n-nodes-base.code | Formats preview response | Run preview assistant | Respond: JSON result | 17. Test chat |
| Get all escalations | n8n-nodes-base.dataTable | Retrieves all escalations | Get all products for orders | Compose overview | 15. Orders and customers |
| More Module Webhook | n8n-nodes-base.webhook | Entry point for more admin modules | None | Render more module | 8. Admin website pages |
| Render more module | n8n-nodes-base.code | Renders more module script | More Module Webhook | Respond: more module | 8. Admin website pages |
| Respond: more module | n8n-nodes-base.respondToWebhook | Returns more module JS | Render more module | None | 8. Admin website pages |
| Find existing product | n8n-nodes-base.dataTable | Retrieves existing product for photo replacement check | Route | Photo replaced? | 12. Products |
| Photo replaced? | n8n-nodes-base.if | Checks if product photo changed | Find existing product | Delete replaced photo, Product to save | 12. Products |
| Delete replaced photo | n8n-nodes-base.googleDrive | Deletes old photo from Drive | Photo replaced? | Product to save | 13. Product photos |
| Product to save | n8n-nodes-base.code | Prepares product save payload | Photo replaced?, Delete replaced photo | Save product | 12. Products |
| Guide Module Webhook | n8n-nodes-base.webhook | Entry point for setup guide module | None | Render guide module | 8. Admin website pages |
| Render guide module | n8n-nodes-base.code | Renders setup guide script | Guide Module Webhook | Respond: guide module | 8. Admin website pages |
| Respond: guide module | n8n-nodes-base.respondToWebhook | Returns setup guide JS | Render guide module | None | 8. Admin website pages |
| Setup Webhook | n8n-nodes-base.webhook | Entry point for setup initialization | None | Create table: Sales Agent Config | 18. Setup (run once) |
| Setup: create product search table | n8n-nodes-base.postgres | Creates pgvector table in Supabase | Create table: Shop Errors | Setup: look for an admin login | 18. Setup (run once) |
| Load verify token | n8n-nodes-base.dataTable | Loads verification token | Meta Webhook Trigger | Verify Webhook Handshake | 1. Messenger intake |
| Load connections (messages) | n8n-nodes-base.dataTable | Loads connections for messages | When Called By This Workflow | Restore message input | 2. One customer message |
| Restore message input | n8n-nodes-base.code | Restores message workflow input | Load connections (messages) | Route By Action | 2. One customer message |
| Stripe Payment Event | n8n-nodes-base.code | Parses Stripe payment event | Signature valid? | Load connections (stripe event) | 5. Payment confirmed |
| Load connections (stripe event) | n8n-nodes-base.dataTable | Loads connections for Stripe events | Stripe Payment Event | Restore Stripe event | 5. Payment confirmed |
| Restore Stripe event | n8n-nodes-base.code | Restores Stripe event input | Load connections (stripe event) | Paid Session | 5. Payment confirmed |
| Load connections (return page) | n8n-nodes-base.dataTable | Loads connections for return page | Payment Return Page | Restore return page | 5. Payment confirmed |
| Restore return page | n8n-nodes-base.code | Restores return page input | Load connections (return page) | Look Up Returned Payment | 5. Payment confirmed |
| Create table: Shop Errors | n8n-nodes-base.dataTable | Creates Shop Errors table | Create table: Shop Settings | Setup: create product search table | 18. Setup (run once) |
| Save error log | n8n-nodes-base.dataTable | Records workflow execution errors | Build error details | None | 7. Error log |
| Get error log | n8n-nodes-base.dataTable | Retrieves error logs for admin view | Route more | Compose errors | 16. Inbox and assistant |
| Compose errors | n8n-nodes-base.code | Formats error log response | Get error log | Respond: JSON result | 16. Inbox and assistant |
| Mark errors seen | n8n-nodes-base.dataTable | Marks error logs as seen | Route more | Compose errors seen | 16. Inbox and assistant |
| Compose errors seen | n8n-nodes-base.code | Formats errors seen response | Mark errors seen | Respond: JSON result | 16. Inbox and assistant |
| Clear seen errors | n8n-nodes-base.dataTable | Deletes seen error logs | Route more | Compose errors cleared | 16. Inbox and assistant |
| Compose errors cleared | n8n-nodes-base.code | Formats errors cleared response | Clear seen errors | Respond: JSON result | 16. Inbox and assistant |
| Delivery: Get Order | n8n-nodes-base.dataTable | Loads order for delivery details | Route By Action | Delivery: Get Customer Orders | 3. Delivery details |
| Delivery: Get Customer Orders | n8n-nodes-base.dataTable | Retrieves customer orders for address lookup | Delivery: Get Order | Delivery: Merge Details | 3. Delivery details |
| Delivery: Merge Details | n8n-nodes-base.code | Merges delivery details | Delivery: Get Customer Orders | Delivery: Save needed? | 3. Delivery details |
| Delivery: Save needed? | n8n-nodes-base.if | Checks if delivery details need saving | Delivery: Merge Details | Delivery: Save Order, Delivery: Reply | 3. Delivery details |
| Delivery: Save Order | n8n-nodes-base.dataTable | Updates order delivery details | Delivery: Save needed? | Delivery: Reply | 3. Delivery details |
| Delivery: Reply | n8n-nodes-base.code | Formats delivery tool response | Delivery: Save needed?, Delivery: Save Order | None | 3. Delivery details |
| save_delivery_details | @n8n/n8n-nodes-langchain.toolWorkflow | Tool to save delivery details | None | Sales Agent (SalesPro) | 3. Delivery details |
| Build error details | n8n-nodes-base.set | Formats error log details | When This Workflow Fails | Save error log | 7. Error log |
| Log: assistant could not answer | n8n-nodes-base.dataTable | Logs assistant failure | Send Fallback Reply | None | 7. Error log |
| Log: reply not delivered | n8n-nodes-base.dataTable | Logs Messenger reply failure | Send Text Reply to Customer | None | 7. Error log |
| Payment message failed? | n8n-nodes-base.if | Checks if payment message failed | Save Payment To Agent Memory | Log: payment message not sent | 5. Payment confirmed |
| Log: payment message not sent | n8n-nodes-base.dataTable | Logs payment confirmation failure | Payment message failed? | None | 5. Payment confirmed |
| Answer sent? | n8n-nodes-base.if | Checks if admin answer was sent | Save escalation | Answer sent? (True -> Save Answer To Agent Memory, False -> Compose reply result) | 16. Inbox and assistant |
| Guide Module Webhook | n8n-nodes-base.webhook | Entry point for setup guide module | None | Render guide module | 8. Admin website pages |
| Render guide module | n8n-nodes-base.code | Renders setup guide script | Guide Module Webhook | Respond: guide module | 8. Admin website pages |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Initialize Database Tables and Credentials
1. Create 3 n8n credentials:
   - **Google Gemini (PaLM) API**: Enter your Gemini API key.
   - **Postgres**: Enter Supabase connection details (Session pooler host, port `5432`, database `postgres`, username, password, and set SSL to **Require**).
   - **Google Drive OAuth2 API**: Configure OAuth client credentials with authorized redirect URIs.
2. Create 14 Data Tables in n8n with their exact case-sensitive column schemas:
   - `Sales Agent Config`: `configKey`, `active` (boolean), `systemPrompt`, `storeName`, `paymentSuccessUrl`
   - `Sales Cart Items`: `orderId`, `customerId`, `productName`, `size`, `color`, `quantity` (number), `unitPrice` (number)
   - `Sales Checkout Sessions`: `orderId`, `customerId`, `sessionId`, `checkoutUrl`, `amountCents` (number), `status`
   - `Sales Customer State`: `customerId`, `customerName`, `lastCustomerMessageAt` (date), `humanPausedUntil` (date)
   - `Sales Customers`: `customer_id`, `Name`, `Platform`, `Phone`, `Address`, `Email`, `Interests`, `Birthday`, `Notes`
   - `Sales Escalations`: `escalationId`, `customerId`, `customerName`, `question`, `customerMessage`, `status`, `adminAnswer`, `answeredAt` (date), `deliveredAt` (date)
   - `Sales Order Items`: `LineID`, `OrderID`, `Item_ID`, `Product_Name`, `Size`, `Color`, `Quantity` (number), `Unit_Price` (number), `Image`
   - `Sales Orders`: `OrderID`, `customer_id`, `Full_Name`, `Address`, `City`, `Landmark`, `Address_Confirmed` (boolean), `Phone`, `Status`, `Payment_Method`, `Total` (number), `Created_At`, `Paid_At`, `Tracking`, `Notes`, `Source`
   - `Sales Processed Messages`: `messageId`, `customerId`, `text`, `attachments`, `handled` (boolean)
   - `Sales Products`: `Item_ID`, `Name`, `Price` (number), `Stock_Status` (number), `Product_Image`, `Description`, `Sizes`, `Colors`, `Status`
   - `Shop Admin Login`: `username`, `passwordHash`, `salt`, `mustChange` (boolean)
   - `Shop Admin Sessions`: `token`, `expiresAt` (number)
   - `Shop Settings`: `settingKey`, `settingValue`
   - `Shop Errors`: `occurredAt`, `workflowName`, `failedNode`, `message`, `hint`, `executionId`, `executionUrl`, `status`

#### Step 2: Build the Messenger Intake Pipeline
1. Create a **Webhook** node named `Meta Webhook Trigger` with path `chat`, response mode `Response Node`, and multiple methods enabled.
2. Add a **Data Table** node named `Load verify token` to fetch `metaVerifyToken` from `Shop Settings`.
3. Add an **IF** node named `Verify Webhook Handshake` evaluating `hub.mode === 'subscribe'` and matching `hub.verify_token`. Connect a **Respond to Webhook** node (`Confirm Webhook Subscription`) to return `hub.challenge`.
4. Connect an immediate **Respond to Webhook** node named `Acknowledge Event (200 OK)` returning `EVENT_RECEIVED` with HTTP status `200`.
5. Add an **IF** node named `Filter Page Events` (`body.object === 'page'`), followed by two **SplitOut** nodes (`Split the page record` for `body.entry` and `split out each message` for `messaging`).
6. Add an **IF** node named `Is Real Customer Message` verifying valid message IDs, non-echo status, and distinct sender/recipient IDs.
7. Route valid messages to an **Execute Workflow** node (`Process Each Message Separately`) targeting the current workflow ID with workflow inputs `event`, `action: 'process_message'`, etc., configured in each execution.

#### Step 3: Configure Sub-Workflow Routing & AI Agent Core
1. Create an **Execute Workflow Trigger** node named `When Called By This Workflow` accepting inputs: `action`, `event`, `orderId`, `customerId`, `fullName`, `address`, `phone`, `op`, `product`, `products`, `itemId`, `query`, `limit`, `message`, `history`, `city`, `landmark`, `targetOrderId`, `confirmed`.
2. Add a **Switch** node named `Route By Action` with routes: `checkout`, `process_message`, `vector`, `preview`, `delivery`.
3. For the `process_message` branch, add a **Set** node (`Message Context`) to extract `customerId`, `pageId`, `messageId`, `text`, and `attachmentsSummary`.
4. Check processed messages using `Find Processed Message` and `Is New Message`. If new, record the message ID in `Remember Message ID`.
5. Load agent configurations from `Sales Agent Config`, update customer last seen timestamps in `Sales Customer State`, and verify bot permissions with `Bot Allowed For Customer?`.
6. Add a **Wait** node (`Wait For More Messages`) set to 4 seconds, fetch unhandled messages via `Get Unhandled Messages`, combine them in `Combine Messages`, and verify the latest message flag.
7. Look up or create the customer record using `Look Up Existing Customer`, `Check If Customer Is New`, `Fetch Facebook Profile Info`, and `Save New Customer Record`.
8. Retrieve customer orders (`Get Customer Orders`), find open carts (`Find Open Cart`, `Has Open Cart?`, `Create New Cart Order`), and set the active order context (`Current Order`).
9. Load pending admin answers via `Get Pending Admin Answers`.
10. Create the **AI Agent** node named `Sales Agent (SalesPro)` bound to:
    - **Language Model**: `Google Gemini Chat Model` (`models/gemini-2.5-flash`).
    - **Memory**: `Customer Conversation Memory` (Postgres Chat Memory) and `Customer Memory (Admin Answers)`.
    - **Tools**: `get_product_details`, `search_products`, `list_products`, `view_cart`, `set_cart_item`, `remove_cart_item`, `clear_cart`, `save_delivery_details`, `checkout_and_send_payment_link`, `get_my_orders`, `get_order_items`, `analyze_customer_image`.
    - **Output Parser**: `Parse Agent Reply` (Structured Output Parser with JSON schema enforcing `message`, `image`, and `escalate_question`).

#### Step 4: Implement Delivery Details and Checkout Workflows
1. Configure `save_delivery_details` and `checkout_and_send_payment_link` tool workflows pointing back to the main workflow ID with appropriate `action` parameters (`delivery` and `checkout`).
2. For the `delivery` action route: Fetch the target order, merge incoming address fields via `Delivery: Merge Details`, validate missing properties, and update `Sales Orders` using `Delivery: Save Order`.
3. For the `checkout` action route: Fetch checkout settings, order items, and inventory. Re-price cart items in `Build Checkout`, validate stock, and generate Stripe Checkout Session requests via `Create Stripe Checkout Session`.
4. Send the Stripe Checkout Session URL via HTTP Request to `https://graph.facebook.com/v25.0/me/messages` using button templates (`Send Pay Now Button`).

#### Step 5: Implement Stripe Payment Handling and Webhooks
1. Create a **Webhook** node named `Stripe Payment Event` listening at path `shop-stripe-webhook` with raw body enabled.
2. Read the raw body, load the Stripe webhook signing secret from `Shop Settings`, and verify the HMAC SHA-256 signature in `Verify Stripe signature`.
3. Upon payment success (`checkout.session.completed`), update the order status to `Paid` in `Sales Orders`, fetch line items from Stripe (`Get Paid Line Items`), record order items in `Sales Order Items`, calculate new stock levels, update `Sales Products`, and send payment confirmation messages to the customer via Messenger.
4. Create a **Webhook** node named `Payment Return Page` at path `payment-success` to render HTML success/cancellation pages and redirect users back to Messenger.

#### Step 6: Build the Admin Website and API Backend
1. Create a **Webhook** node named `Admin Page Webhook` at path `shop-admin` returning the single-page application HTML rendered in `Render app`.
2. Create a **Webhook** node named `Admin API Webhook` at path `shop-api` handling POST requests.
3. Parse incoming API requests (`Parse API`), validate admin session tokens against `Shop Admin Sessions`, verify password hashes (`Shop Admin Login`), and route actions via `Decide` and `Route` switches.
4. Implement handlers for authentication (`login`, `me`, `logout`, `change_password`), settings management, Facebook OAuth integration (`fb_start`, `fb_pending`, `fb_select`, `fb_disconnect`), product management (CRUD, Google Drive uploads via `Upload photo to Drive`, and vector embedding sync via `Run vector op`), order/customer management, inbox escalations (`inbox_reply`), and test chat previews (`preview_chat`).

#### Step 7: Finalize Error Handling and Nightly Cleanup
1. In workflow settings, set this workflow as its own **Error Workflow**.
2. Add a **When This Workflow Fails** error trigger node, parse error details in `Build error details`, and log failures to the `Shop Errors` data table.
3. Add a **Schedule Trigger** node (`Every Night At 3`) set to 03:00 hours daily, connecting to data table operations that delete old message IDs (>7 days), purge stale cart lines (>30 days), and mark draft orders older than 7 days as `Abandoned`.
4. Publish the workflow. Open your n8n instance URL at `/webhook/shop-admin`, click **Create database and first login**, sign in with `admin` / `admin`, and update your credentials immediately.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Walkthrough video (2 min walkthrough of Messenger ordering, Stripe payment, and admin website) | https://drive.google.com/file/d/1HchY8S2IfqyadX-hvK1-ZiTB1pnUUkCl/view?usp=sharing |
| Google AI Studio API Key Generation | https://aistudio.google.com/apikey |
| Supabase Database Platform | https://supabase.com |
| Meta for Developers | https://developers.facebook.com/apps |
| Stripe Dashboard | https://dashboard.stripe.com/register |